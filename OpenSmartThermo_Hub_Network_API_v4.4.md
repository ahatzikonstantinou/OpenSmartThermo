# OpenSmartThermo — Main Thermostat & Hub Firmware (Part 2B-1: Network, ESP-NOW Hub, REST API & MQTT)

**Version:** 4.4  
**Target Hardware:** `ESP32-S3-WROOM-1-N16R8`  
**Contents:** `src/network_manager.h`, `src/network_manager.cpp` (ESP-NOW Hub RX/TX & Pairing, `ESPAsyncWebServer` REST API `/api/v1/*`, WebSocket live broadcast `/ws`, OTA Firmware Upload, and `PubSubClient` MQTT + Home Assistant Auto-Discovery).

---

## 1. `src/network_manager.h`

```cpp
#pragma once

#include <Arduino.h>
#include "config.h"

void networkInit();
void networkLoop();
void networkStartPairing(uint16_t duration_sec = 60);
void networkStopPairing();
void networkStartSetupAP(uint16_t duration_sec = 600);
void networkStopSetupAP();
void networkBroadcastWS();
void networkPublishMqttState();
```

---

## 2. `src/network_manager.cpp`

```cpp
#include "network_manager.h"
#include "control_engine.h"
#include <WiFi.h>
#include <DNSServer.h>
#include <esp_now.h>
#include <esp_wifi.h>
#include <ESPmDNS.h>
#include <ESPAsyncWebServer.h>
#include <ArduinoJson.h>
#include <PubSubClient.h>
#include <LittleFS.h>
#include <Update.h>
#include <time.h>

static DNSServer dnsServer;
static AsyncWebServer server(80);
static AsyncWebSocket ws("/ws");
static WiFiClient wifiClient;
static PubSubClient mqtt(wifiClient);

static QueueHandle_t espnowRxQueue = nullptr;
static unsigned long lastMqttReconnectMs = 0;
static unsigned long lastMqttPublishMs = 0;
static unsigned long lastWsBroadcastMs = 0;
static unsigned long lastStaRetryMs = 0;
static bool pendingApShutdown = false;
static unsigned long apShutdownAtMs = 0;
static bool haDiscoveryPublished = false;

typedef struct {
    uint8_t src_mac[6];
    int8_t rssi;
    espnow_sensor_packet_t pkt;
} rx_queue_item_t;

static uint32_t getCurrentEpoch() {
    time_t now = time(nullptr);
    return (now > 100000) ? (uint32_t)now : (millis() / 1000UL);
}

// -------------------------------------------------------------------------
// ESP-NOW ISR Receive Callback (Keep ultra-fast, push to FreeRTOS queue)
// -------------------------------------------------------------------------
static void onEspNowRecv(const esp_now_recv_info_t *info, const uint8_t *data, int len) {
    if (len != sizeof(espnow_sensor_packet_t) || !espnowRxQueue) return;
    rx_queue_item_t item;
    memcpy(item.src_mac, info->src_addr, 6);
    item.rssi = info->rx_ctrl ? info->rx_ctrl->rssi : -70;
    memcpy(&item.pkt, data, sizeof(espnow_sensor_packet_t));
    xQueueSendFromISR(espnowRxQueue, &item, nullptr);
}

// -------------------------------------------------------------------------
// Send ESP-NOW ACK + Piggybacked Config (Sensor Name, Hub Name, Timeout)
// -------------------------------------------------------------------------
static void sendEspNowAck(const uint8_t *peer_mac, uint8_t msg_type, uint8_t assigned_id) {
    esp_now_peer_info_t peerInfo = {};
    memcpy(peerInfo.peer_addr, peer_mac, 6);
    peerInfo.channel = WiFi.channel();
    peerInfo.encrypt = false;

    if (!esp_now_is_peer_exist(peer_mac)) {
        esp_now_add_peer(&peerInfo);
    }

    espnow_hub_ack_t ack = {};
    ack.msg_type          = msg_type;
    ack.assigned_id       = assigned_id;
    ack.wifi_channel      = WiFi.channel();
    ack.report_interval_s = g_config.sensors[assigned_id].report_interval_s;
    ack.cal_offset_c      = g_config.sensors[assigned_id].cal_offset_c;
    ack.display_timeout_s = g_config.sensors[assigned_id].display_timeout_s;
    strlcpy(ack.sensor_name, g_config.sensors[assigned_id].name, sizeof(ack.sensor_name));
    strlcpy(ack.hub_name, g_config.hub_name, sizeof(ack.hub_name));

    esp_now_send(peer_mac, (uint8_t *)&ack, sizeof(ack));
}

// -------------------------------------------------------------------------
// Process Queued ESP-NOW Packets in Core 0 Task
// -------------------------------------------------------------------------
static void processEspNowQueue() {
    if (!espnowRxQueue) return;
    rx_queue_item_t item;
    while (xQueueReceive(espnowRxQueue, &item, 0) == pdTRUE) {
        bool needSave = false;
        bool needBroadcast = false;

        xSemaphoreTake(g_mutex, portMAX_DELAY);

        int found_id = -1;
        for (int i = 1; i <= MAX_SENSORS; i++) {
            if (g_config.sensors[i].paired && memcmp(g_config.sensors[i].mac, item.src_mac, 6) == 0) {
                found_id = i;
                break;
            }
        }

        if (item.pkt.msg_type == 0x02 && g_state.pairing_active) {
            if (found_id == -1) {
                for (int i = 1; i <= MAX_SENSORS; i++) {
                    if (!g_config.sensors[i].paired) {
                        found_id = i;
                        break;
                    }
                }
                if (found_id != -1) {
                    g_config.sensors[found_id].paired = true;
                    memcpy(g_config.sensors[found_id].mac, item.src_mac, 6);
                    snprintf(g_config.sensors[found_id].name, sizeof(g_config.sensors[found_id].name), "Sensor #%02d", found_id);
                    g_config.sensors[found_id].role = ROLE_INDOOR;
                    g_config.sensors[found_id].weight = 1;
                    g_config.sensors[found_id].cal_offset_c = 0.0f;
                    g_config.sensors[found_id].report_interval_s = 300;
                    g_config.sensors[found_id].display_timeout_s = 8;
                    needSave = true;
                }
            }
            if (found_id != -1) {
                g_state.sensors[found_id].temp_c          = item.pkt.temp_c + g_config.sensors[found_id].cal_offset_c;
                g_state.sensors[found_id].humidity_pct    = item.pkt.humidity_pct;
                g_state.sensors[found_id].batt_mv         = item.pkt.batt_mv;
                g_state.sensors[found_id].batt_pct        = item.pkt.batt_pct;
                g_state.sensors[found_id].rssi            = item.rssi;
                g_state.sensors[found_id].online          = true;
                g_state.sensors[found_id].last_seen_epoch = getCurrentEpoch();
                g_state.last_paired_id = found_id;
                sendEspNowAck(item.src_mac, 0x82, found_id);
                needBroadcast = true;
            }
        } else if (item.pkt.msg_type == 0x01 && found_id != -1) {
            g_state.sensors[found_id].temp_c          = item.pkt.temp_c + g_config.sensors[found_id].cal_offset_c;
            g_state.sensors[found_id].humidity_pct    = item.pkt.humidity_pct;
            g_state.sensors[found_id].batt_mv         = item.pkt.batt_mv;
            g_state.sensors[found_id].batt_pct        = item.pkt.batt_pct;
            g_state.sensors[found_id].rssi            = item.rssi;
            g_state.sensors[found_id].online          = true;
            g_state.sensors[found_id].last_seen_epoch = getCurrentEpoch();
            sendEspNowAck(item.src_mac, 0x81, found_id);
            needBroadcast = true;
        }

        xSemaphoreGive(g_mutex);

        if (needSave) saveConfigToNVS();
        if (needBroadcast) {
            networkBroadcastWS();
            networkPublishMqttState();
        }
    }
}

// -------------------------------------------------------------------------
// JSON Serialization Helpers
// -------------------------------------------------------------------------
static String buildStatusJson() {
    JsonDocument doc;
    xSemaphoreTake(g_mutex, portMAX_DELAY);

    const char *sysModes[] = {"AUTO", "MANUAL", "BOOST", "OFF"};
    doc["hub_name"]       = g_config.hub_name;
    doc["mode"]           = sysModes[g_state.mode];
    doc["indoor_temp"]    = roundf(g_state.control_temp_c * 10.0f) / 10.0f;
    doc["indoor_hum"]     = roundf(g_state.sensors[0].humidity_pct);
    doc["target_temp"]    = roundf(g_state.active_setpoint * 10.0f) / 10.0f;
    doc["heating_active"] = g_state.heating_active;
    doc["setup_ap"]       = g_state.setup_ap_active;
    doc["lockout_reason"] = g_state.lockout_warm_weather ? "WARM WEATHER LOCKOUT" : "";

    struct tm timeinfo;
    if (getLocalTime(&timeinfo, 5)) {
        char tBuf[8];
        snprintf(tBuf, sizeof(tBuf), "%02d:%02d", timeinfo.tm_hour, timeinfo.tm_min);
        doc["time_str"] = tBuf;
    }

    JsonArray sArr = doc["sensors"].to<JsonArray>();
    for (int i = 0; i <= MAX_SENSORS; i++) {
        if (i != 0 && !g_config.sensors[i].paired) continue;
        JsonObject s = sArr.add<JsonObject>();
        s["id"]           = i;
        s["name"]         = g_config.sensors[i].name;
        s["role"]         = (uint8_t)g_config.sensors[i].role;
        s["temp"]         = roundf(g_state.sensors[i].temp_c * 10.0f) / 10.0f;
        s["hum"]          = roundf(g_state.sensors[i].humidity_pct);
        s["batt_pct"]     = g_state.sensors[i].batt_pct;
        s["online"]       = g_state.sensors[i].online;
        s["disp_timeout"] = g_config.sensors[i].display_timeout_s;
    }

    xSemaphoreGive(g_mutex);
    String out;
    serializeJson(doc, out);
    return out;
}

static String buildSettingsJson() {
    JsonDocument doc;
    xSemaphoreTake(g_mutex, portMAX_DELAY);

    doc["hub_name"]          = g_config.hub_name;
    doc["algo"]              = (uint8_t)g_config.algorithm;
    doc["hysteresis"]        = g_config.static_hyst_c;
    doc["agg_mode"]          = (uint8_t)g_config.agg_mode;
    doc["learned_overshoot"] = g_config.learned_overshoot_c;
    doc["warm_lockout"]      = g_config.warm_lockout_en ? g_config.warm_lockout_temp : 0.0f;
    doc["frost_temp"]        = g_config.frost_protect_temp;
    doc["min_cycle_s"]       = g_config.min_on_time_s;
    doc["mqtt_host"]         = g_config.mqtt_host;
    doc["mqtt_port"]         = g_config.mqtt_port;
    doc["mqtt_user"]         = g_config.mqtt_user;
    doc["ip"]                = (WiFi.status() == WL_CONNECTED) ? WiFi.localIP().toString() : WiFi.softAPIP().toString();
    doc["wifi_channel"]      = WiFi.channel();
    doc["uptime"]            = String(millis() / 60000UL) + " min";
    doc["free_heap"]         = String(ESP.getFreeHeap() / 1024) + " KB";

    xSemaphoreGive(g_mutex);
    String out;
    serializeJson(doc, out);
    return out;
}

// -------------------------------------------------------------------------
// MQTT & Home Assistant Auto-Discovery
// -------------------------------------------------------------------------
static void publishHaDiscovery() {
    if (!g_config.ha_discovery || !mqtt.connected()) return;
    String base = strlen(g_config.mqtt_base_topic) > 0 ? String(g_config.mqtt_base_topic) : "opensmartthermo";

    JsonDocument doc;
    doc["name"]          = String(g_config.hub_name) + " Thermostat";
    doc["uniq_id"]       = "opensmartthermo_climate";
    doc["mode_cmd_t"]    = base + "/set/mode";
    doc["mode_stat_t"]   = base + "/state";
    doc["mode_stat_tpl"] = "{{ 'off' if value_json.mode == 'OFF' else ('auto' if value_json.mode == 'AUTO' else 'heat') }}";
    doc["temp_cmd_t"]    = base + "/set/target_temp";
    doc["temp_stat_t"]   = base + "/state";
    doc["temp_stat_tpl"] = "{{ value_json.target_temp }}";
    doc["curr_temp_t"]   = base + "/state";
    doc["curr_temp_tpl"] = "{{ value_json.indoor_temp }}";
    doc["act_t"]         = base + "/state";
    doc["act_tpl"]       = "{{ 'heating' if value_json.heating_active else 'idle' }}";
    JsonArray modes = doc["modes"].to<JsonArray>();
    modes.add("off"); modes.add("heat"); modes.add("auto");
    doc["min_temp"] = MIN_SETPOINT_C;
    doc["max_temp"] = MAX_SETPOINT_C;
    doc["temp_step"] = 0.5;

    String payload;
    serializeJson(doc, payload);
    mqtt.publish("homeassistant/climate/opensmartthermo/config", payload.c_str(), true);
    haDiscoveryPublished = true;
}

static void onMqttMessage(char *topic, byte *payload, unsigned int length) {
    String msg;
    for (unsigned int i = 0; i < length; i++) msg += (char)payload[i];
    String t = String(topic);
    String base = strlen(g_config.mqtt_base_topic) > 0 ? String(g_config.mqtt_base_topic) : "opensmartthermo";

    if (t == base + "/set/mode") {
        msg.toUpperCase();
        xSemaphoreTake(g_mutex, portMAX_DELAY);
        if (msg == "AUTO") g_state.mode = MODE_AUTO;
        else if (msg == "MANUAL" || msg == "HEAT") g_state.mode = MODE_MANUAL;
        else if (msg == "BOOST") {
            g_state.mode = MODE_BOOST;
            g_state.boost_until_epoch = getCurrentEpoch() + 3600;
        } else if (msg == "OFF") g_state.mode = MODE_OFF;
        xSemaphoreGive(g_mutex);
        saveConfigToNVS();
        networkBroadcastWS();
        networkPublishMqttState();
    } else if (t == base + "/set/target_temp") {
        float val = constrain(msg.toFloat(), MIN_SETPOINT_C, MAX_SETPOINT_C);
        xSemaphoreTake(g_mutex, portMAX_DELAY);
        g_state.manual_setpoint = val;
        if (g_state.mode == MODE_AUTO) g_state.mode = MODE_MANUAL;
        xSemaphoreGive(g_mutex);
        saveConfigToNVS();
        networkBroadcastWS();
        networkPublishMqttState();
    }
}

void networkPublishMqttState() {
    if (!g_config.mqtt_en || !mqtt.connected()) return;
    String base = strlen(g_config.mqtt_base_topic) > 0 ? String(g_config.mqtt_base_topic) : "opensmartthermo";
    mqtt.publish((base + "/state").c_str(), buildStatusJson().c_str(), true);
}

// -------------------------------------------------------------------------
// Public Control Helpers
// -------------------------------------------------------------------------
void networkStartPairing(uint16_t duration_sec) {
    xSemaphoreTake(g_mutex, portMAX_DELAY);
    g_state.pairing_active = true;
    g_state.pairing_until_epoch = getCurrentEpoch() + duration_sec;
    g_state.last_paired_id = -1;
    xSemaphoreGive(g_mutex);
    networkBroadcastWS();
}

void networkStopPairing() {
    xSemaphoreTake(g_mutex, portMAX_DELAY);
    g_state.pairing_active = false;
    g_state.pairing_until_epoch = 0;
    xSemaphoreGive(g_mutex);
    networkBroadcastWS();
}

void networkStartSetupAP(uint16_t duration_sec) {
    WiFi.mode(WIFI_AP_STA);
    WiFi.softAP("OpenSmartThermo-Setup");
    delay(100);
    dnsServer.start(53, "*", WiFi.softAPIP());

    xSemaphoreTake(g_mutex, portMAX_DELAY);
    g_state.setup_ap_active = true;
    g_state.setup_ap_until_ms = millis() + (duration_sec * 1000UL);
    xSemaphoreGive(g_mutex);
    networkBroadcastWS();
}

void networkStopSetupAP() {
    dnsServer.stop();
    WiFi.softAPdisconnect(true);
    if (WiFi.status() == WL_CONNECTED) {
        WiFi.mode(WIFI_STA);
    }

    xSemaphoreTake(g_mutex, portMAX_DELAY);
    g_state.setup_ap_active = false;
    g_state.setup_ap_until_ms = 0;
    xSemaphoreGive(g_mutex);
    networkBroadcastWS();
}

void networkBroadcastWS() {
    if (ws.count() > 0) {
        ws.textAll(buildStatusJson());
    }
}

// -------------------------------------------------------------------------
// REST API & WebServer Setup
// -------------------------------------------------------------------------
static void setupRestRoutes() {
    ws.onEvent([](AsyncWebSocket *s, AsyncWebSocketClient *c, AwsEventType type, void *arg, uint8_t *data, size_t len) {
        if (type == WS_EVT_CONNECT) c->text(buildStatusJson());
    });
    server.addHandler(&ws);

    // Captive Portal endpoints (Android / iOS / Windows)
    server.on("/generate_204", HTTP_GET, [](AsyncWebServerRequest *req) { req->redirect("http://192.168.4.1"); });
    server.on("/hotspot-detect.html", HTTP_GET, [](AsyncWebServerRequest *req) { req->redirect("http://192.168.4.1"); });
    server.on("/connecttest.txt", HTTP_GET, [](AsyncWebServerRequest *req) { req->redirect("http://192.168.4.1"); });

    server.on("/api/v1/status", HTTP_GET, [](AsyncWebServerRequest *req) {
        req->send(200, "application/json", buildStatusJson());
    });

    server.on("/api/v1/control", HTTP_POST, [](AsyncWebServerRequest *req) {}, nullptr,
        [](AsyncWebServerRequest *req, uint8_t *data, size_t len, size_t index, size_t total) {
            JsonDocument doc;
            if (deserializeJson(doc, data, len)) {
                req->send(400, "application/json", "{\"error\":\"Invalid JSON\"}");
                return;
            }
            xSemaphoreTake(g_mutex, portMAX_DELAY);
            if (doc["mode"].is<const char*>()) {
                String m = doc["mode"].as<String>();
                if (m == "AUTO") g_state.mode = MODE_AUTO;
                else if (m == "MANUAL") g_state.mode = MODE_MANUAL;
                else if (m == "BOOST") {
                    g_state.mode = MODE_BOOST;
                    g_state.boost_until_epoch = getCurrentEpoch() + 3600;
                } else if (m == "OFF") g_state.mode = MODE_OFF;
            }
            if (doc["target_temp"].is<float>()) {
                g_state.manual_setpoint = constrain(doc["target_temp"].as<float>(), MIN_SETPOINT_C, MAX_SETPOINT_C);
            }
            xSemaphoreGive(g_mutex);
            saveConfigToNVS();
            networkBroadcastWS();
            networkPublishMqttState();
            req->send(200, "application/json", "{\"status\":\"ok\"}");
        });

    server.on("/api/v1/sensors/pair", HTTP_POST, [](AsyncWebServerRequest *req) {
        networkStartPairing(60);
        req->send(200, "application/json", "{\"status\":\"ok\"}");
    });

    server.on("^\\/api\\/v1\\/sensors\\/([0-9]+)$", HTTP_PUT, [](AsyncWebServerRequest *req) {}, nullptr,
        [](AsyncWebServerRequest *req, uint8_t *data, size_t len, size_t index, size_t total) {
            int id = req->pathArg(0).toInt();
            if (id <= 0 || id > MAX_SENSORS || !g_config.sensors[id].paired) {
                req->send(404, "application/json", "{\"error\":\"Not found\"}");
                return;
            }
            JsonDocument doc;
            deserializeJson(doc, data, len);
            xSemaphoreTake(g_mutex, portMAX_DELAY);
            if (doc["name"].is<const char*>()) strlcpy(g_config.sensors[id].name, doc["name"], sizeof(g_config.sensors[id].name));
            if (doc["role"].is<int>()) g_config.sensors[id].role = (sensor_role_t)constrain(doc["role"].as<int>(), 0, 2);
            if (doc["disp_timeout"].is<int>()) g_config.sensors[id].display_timeout_s = constrain(doc["disp_timeout"].as<int>(), 3, 30);
            xSemaphoreGive(g_mutex);
            saveConfigToNVS();
            networkBroadcastWS();
            req->send(200, "application/json", "{\"status\":\"ok\"}");
        });

    server.on("^\\/api\\/v1\\/sensors\\/([0-9]+)$", HTTP_DELETE, [](AsyncWebServerRequest *req) {
        int id = req->pathArg(0).toInt();
        if (id <= 0 || id > MAX_SENSORS || !g_config.sensors[id].paired) {
            req->send(400, "application/json", "{\"error\":\"Invalid sensor\"}");
            return;
        }
        xSemaphoreTake(g_mutex, portMAX_DELAY);
        esp_now_del_peer(g_config.sensors[id].mac);
        memset(&g_config.sensors[id], 0, sizeof(SensorConfig));
        memset(&g_state.sensors[id], 0, sizeof(SensorRuntime));
        xSemaphoreGive(g_mutex);
        saveConfigToNVS();
        networkBroadcastWS();
        req->send(200, "application/json", "{\"status\":\"ok\"}");
    });

    server.on("/api/v1/schedule", HTTP_GET, [](AsyncWebServerRequest *req) {
        JsonDocument doc;
        JsonArray days = doc["days"].to<JsonArray>();
        xSemaphoreTake(g_mutex, portMAX_DELAY);
        for (int d = 0; d < MAX_DAYS; d++) {
            JsonArray sArr = days.add<JsonArray>();
            for (int s = 0; s < MAX_SCHEDULE_SLOTS; s++) {
                JsonObject obj = sArr.add<JsonObject>();
                obj["start_minute"] = g_config.schedule[d][s].start_minute;
                obj["target_temp"]  = g_config.schedule[d][s].target_temp;
                obj["sensor_id"]    = g_config.schedule[d][s].sensor_id;
            }
        }
        xSemaphoreGive(g_mutex);
        String out;
        serializeJson(doc, out);
        req->send(200, "application/json", out);
    });

    server.on("/api/v1/schedule", HTTP_POST, [](AsyncWebServerRequest *req) {}, nullptr,
        [](AsyncWebServerRequest *req, uint8_t *data, size_t len, size_t index, size_t total) {
            JsonDocument doc;
            if (deserializeJson(doc, data, len) || !doc["day"].is<int>()) {
                req->send(400, "application/json", "{\"error\":\"Invalid JSON\"}");
                return;
            }
            int day = constrain(doc["day"].as<int>(), 0, MAX_DAYS - 1);
            JsonArray slots = doc["slots"].as<JsonArray>();

            xSemaphoreTake(g_mutex, portMAX_DELAY);
            int idx = 0;
            for (JsonObject s : slots) {
                if (idx >= MAX_SCHEDULE_SLOTS) break;
                g_config.schedule[day][idx].start_minute = s["start_minute"] | SLOT_EMPTY_MINUTE;
                g_config.schedule[day][idx].target_temp  = constrain(s["target_temp"] | 20.0f, MIN_SETPOINT_C, MAX_SETPOINT_C);
                g_config.schedule[day][idx].sensor_id    = s["sensor_id"] | -1;
                idx++;
            }
            if (doc["copy_to"].is<const char*>()) {
                String target = doc["copy_to"].as<String>();
                for (int d = 0; d < MAX_DAYS; d++) {
                    if (d == day) continue;
                    bool match = (target == "all") ||
                                 (target == "weekdays" && d < 5) ||
                                 (target == "weekend" && d >= 5);
                    if (match) {
                        memcpy(g_config.schedule[d], g_config.schedule[day], sizeof(g_config.schedule[day]));
                    }
                }
            }
            xSemaphoreGive(g_mutex);
            saveConfigToNVS();
            req->send(200, "application/json", "{\"status\":\"ok\"}");
        });

    server.on("/api/v1/settings", HTTP_GET, [](AsyncWebServerRequest *req) {
        req->send(200, "application/json", buildSettingsJson());
    });

    server.on("/api/v1/settings", HTTP_POST, [](AsyncWebServerRequest *req) {}, nullptr,
        [](AsyncWebServerRequest *req, uint8_t *data, size_t len, size_t index, size_t total) {
            JsonDocument doc;
            if (deserializeJson(doc, data, len)) {
                req->send(400, "application/json", "{\"error\":\"Invalid JSON\"}");
                return;
            }
            xSemaphoreTake(g_mutex, portMAX_DELAY);
            if (doc["hub_name"].is<const char*>()) strlcpy(g_config.hub_name, doc["hub_name"], sizeof(g_config.hub_name));
            if (doc["algo"].is<int>()) g_config.algorithm = (control_algo_t)constrain(doc["algo"].as<int>(), 0, 2);
            if (doc["hysteresis"].is<float>()) g_config.static_hyst_c = constrain(doc["hysteresis"].as<float>(), 0.1f, 2.0f);
            if (doc["agg_mode"].is<int>()) g_config.agg_mode = (sensor_agg_t)constrain(doc["agg_mode"].as<int>(), 0, 4);
            if (doc["warm_lockout"].is<float>()) {
                float wl = doc["warm_lockout"].as<float>();
                g_config.warm_lockout_en = (wl > 0.0f);
                if (wl > 0.0f) g_config.warm_lockout_temp = wl;
            }
            if (doc["frost_temp"].is<float>()) g_config.frost_protect_temp = constrain(doc["frost_temp"].as<float>(), 3.0f, 12.0f);
            if (doc["min_cycle_s"].is<int>()) {
                uint16_t mc = constrain(doc["min_cycle_s"].as<int>(), DEFAULT_ANTI_CYCLE_S, 900);
                g_config.min_on_time_s = mc;
                g_config.min_off_time_s = mc;
            }
            if (doc["mqtt_host"].is<const char*>()) {
                strlcpy(g_config.mqtt_host, doc["mqtt_host"], sizeof(g_config.mqtt_host));
                g_config.mqtt_en = (strlen(g_config.mqtt_host) > 0);
            }
            if (doc["mqtt_port"].is<int>()) g_config.mqtt_port = doc["mqtt_port"].as<uint16_t>();
            if (doc["mqtt_user"].is<const char*>()) strlcpy(g_config.mqtt_user, doc["mqtt_user"], sizeof(g_config.mqtt_user));
            if (doc["mqtt_pass"].is<const char*>() && strlen(doc["mqtt_pass"]) > 0) {
                strlcpy(g_config.mqtt_pass, doc["mqtt_pass"], sizeof(g_config.mqtt_pass));
            }
            xSemaphoreGive(g_mutex);
            saveConfigToNVS();
            networkBroadcastWS();
            req->send(200, "application/json", "{\"status\":\"ok\"}");
        });

    server.on("/api/v1/settings/reset-learning", HTTP_POST, [](AsyncWebServerRequest *req) {
        g_control.resetLearnedOvershoot();
        req->send(200, "application/json", "{\"status\":\"ok\"}");
    });

    server.on("/api/v1/wifi", HTTP_POST, [](AsyncWebServerRequest *req) {}, nullptr,
        [](AsyncWebServerRequest *req, uint8_t *data, size_t len, size_t index, size_t total) {
            JsonDocument doc;
            if (deserializeJson(doc, data, len)) {
                req->send(400, "application/json", "{\"error\":\"Invalid JSON\"}");
                return;
            }
            xSemaphoreTake(g_mutex, portMAX_DELAY);
            strlcpy(g_config.wifi_ssid, doc["ssid"] | "", sizeof(g_config.wifi_ssid));
            strlcpy(g_config.wifi_pass, doc["password"] | "", sizeof(g_config.wifi_pass));
            if (strlen(g_config.hostname) == 0) strlcpy(g_config.hostname, "opensmartthermo", sizeof(g_config.hostname));
            xSemaphoreGive(g_mutex);
            saveConfigToNVS();

            WiFi.mode(WIFI_AP_STA);
            WiFi.setHostname(g_config.hostname);
            WiFi.begin(g_config.wifi_ssid, g_config.wifi_pass);

            unsigned long t0 = millis();
            while (WiFi.status() != WL_CONNECTED && millis() - t0 < 12000) {
                delay(250);
            }

            if (WiFi.status() == WL_CONNECTED) {
                MDNS.begin(g_config.hostname);
                configTzTime("EET-2EEST,M3.5.0/3,M10.5.0/4", "pool.ntp.org", "time.google.com");
                pendingApShutdown = true;
                apShutdownAtMs = millis() + 5000;

                String ipStr = WiFi.localIP().toString();
                String resp = "{\"status\":\"ok\",\"connected\":true,\"ip\":\"" + ipStr +
                              "\",\"message\":\"Connected! IP: " + ipStr + "\"}";
                req->send(200, "application/json", resp);
            } else {
                req->send(200, "application/json",
                    "{\"status\":\"error\",\"connected\":false,\"message\":\"Connection failed. Check password.\"}");
            }
        });

    server.on("/api/v1/ota", HTTP_POST,
        [](AsyncWebServerRequest *req) {
            bool ok = !Update.hasError();
            req->send(ok ? 200 : 500, "application/json", ok ? "{\"status\":\"ok\"}" : "{\"error\":\"OTA failed\"}");
            if (ok) { delay(800); ESP.restart(); }
        },
        [](AsyncWebServerRequest *req, String filename, size_t index, uint8_t *data, size_t len, bool final) {
            if (!index) Update.begin(UPDATE_SIZE_UNKNOWN);
            if (len) Update.write(data, len);
            if (final) Update.end(true);
        });

    server.serveStatic("/", LittleFS, "/").setDefaultFile("index.html");
    server.onNotFound([](AsyncWebServerRequest *req) {
        if (LittleFS.exists("/index.html")) req->send(LittleFS, "/index.html", "text/html");
        else req->send(404, "text/plain", "Web UI index.html not uploaded to LittleFS.");
    });
}

// -------------------------------------------------------------------------
// Initialization & Main Network Loop (Core 0)
// -------------------------------------------------------------------------
void networkInit() {
    LittleFS.begin(true);
    espnowRxQueue = xQueueCreate(16, sizeof(rx_queue_item_t));

    if (strlen(g_config.hostname) == 0) {
        strlcpy(g_config.hostname, "opensmartthermo", sizeof(g_config.hostname));
    }

    WiFi.mode(WIFI_STA);
    WiFi.setSleep(false);

    if (strlen(g_config.wifi_ssid) > 0) {
        WiFi.setHostname(g_config.hostname);
        WiFi.begin(g_config.wifi_ssid, g_config.wifi_pass);
        unsigned long startAttempt = millis();
        while (WiFi.status() != WL_CONNECTED && millis() - startAttempt < 20000) {
            delay(250);
        }
    }

    if (WiFi.status() != WL_CONNECTED) {
        networkStartSetupAP(600);
        lastStaRetryMs = millis();
    } else {
        MDNS.begin(g_config.hostname);
        configTzTime("EET-2EEST,M3.5.0/3,M10.5.0/4", "pool.ntp.org", "time.google.com");
    }

    if (esp_now_init() == ESP_OK) {
        esp_now_register_recv_cb(onEspNowRecv);
    }

    setupRestRoutes();
    server.begin();
}

void networkLoop() {
    processEspNowQueue();
    ws.cleanupClients();

    if (g_state.setup_ap_active) {
        dnsServer.processNextRequest();
        if (millis() >= g_state.setup_ap_until_ms) {
            networkStopSetupAP();
        }
    }

    if (pendingApShutdown && millis() >= apShutdownAtMs) {
        pendingApShutdown = false;
        networkStopSetupAP();
    }

    if (strlen(g_config.wifi_ssid) > 0 && WiFi.status() != WL_CONNECTED && !g_state.setup_ap_active) {
        if (millis() - lastStaRetryMs >= 1800000UL) {
            lastStaRetryMs = millis();
            WiFi.begin(g_config.wifi_ssid, g_config.wifi_pass);
            unsigned long t0 = millis();
            while (WiFi.status() != WL_CONNECTED && millis() - t0 < 15000) {
                delay(250);
            }
            if (WiFi.status() != WL_CONNECTED) {
                networkStartSetupAP(600);
            } else {
                MDNS.begin(g_config.hostname);
            }
        }
    }

    if (g_state.pairing_active && getCurrentEpoch() >= g_state.pairing_until_epoch) {
        networkStopPairing();
    }

    if (millis() - lastWsBroadcastMs >= 2000) {
        lastWsBroadcastMs = millis();
        networkBroadcastWS();
    }

    if (g_config.mqtt_en && WiFi.status() == WL_CONNECTED && strlen(g_config.mqtt_host) > 0) {
        if (!mqtt.connected()) {
            if (millis() - lastMqttReconnectMs >= 5000) {
                lastMqttReconnectMs = millis();
                mqtt.setServer(g_config.mqtt_host, g_config.mqtt_port);
                mqtt.setCallback(onMqttMessage);
                mqtt.setBufferSize(1024);
                bool ok = strlen(g_config.mqtt_user) > 0
                    ? mqtt.connect(g_config.hostname, g_config.mqtt_user, g_config.mqtt_pass)
                    : mqtt.connect(g_config.hostname);
                if (ok) {
                    String base = strlen(g_config.mqtt_base_topic) > 0 ? String(g_config.mqtt_base_topic) : "opensmartthermo";
                    mqtt.subscribe((base + "/set/#").c_str());
                    if (!haDiscoveryPublished) publishHaDiscovery();
                    networkPublishMqttState();
                }
            }
        } else {
            mqtt.loop();
            if (millis() - lastMqttPublishMs >= 60000) {
                lastMqttPublishMs = millis();
                networkPublishMqttState();
            }
        }
    }
}

```
