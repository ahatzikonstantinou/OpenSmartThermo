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
void networkStartSetupAP();
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
#include <esp_now.h>
#include <esp_wifi.h>
#include <ESPmDNS.h>
#include <ESPAsyncWebServer.h>
#include <ArduinoJson.h>
#include <PubSubClient.h>
#include <LittleFS.h>
#include <Update.h>
#include <Preferences.h>
#include <time.h>

extern SystemState g_state;
extern ThermostatSettings g_settings;
extern SensorInfo g_sensors[MAX_SENSORS];
extern DaySchedule g_schedule[7];
extern SemaphoreHandle_t g_mutex;
extern void saveAllConfigToNVS();

static AsyncWebServer server(80);
static AsyncWebSocket ws("/ws");
static WiFiClient wifiClient;
static PubSubClient mqtt(wifiClient);

static QueueHandle_t espnowRxQueue = nullptr;
static unsigned long lastMqttReconnectMs = 0;
static unsigned long lastMqttPublishMs = 0;
static unsigned long lastWsBroadcastMs = 0;
static bool haDiscoveryPublished = false;

typedef struct {
    uint8_t src_mac[6];
    int8_t rssi;
    espnow_sensor_packet_t pkt;
} rx_queue_item_t;

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
    ack.msg_type = msg_type;
    ack.assigned_id = assigned_id;
    ack.wifi_channel = WiFi.channel();
    ack.report_interval = g_sensors[assigned_id].report_interval_sec;
    ack.temp_offset_x100 = (int16_t)roundf(g_sensors[assigned_id].offset * 100.0f);
    ack.display_timeout_s = g_sensors[assigned_id].display_timeout_s;
    ack.ota_flag = 0;
    strncpy(ack.sensor_name, g_sensors[assigned_id].name, MAX_NAME_LEN);
    ack.sensor_name[MAX_NAME_LEN] = '\0';
    strncpy(ack.hub_name, g_settings.hub_name, MAX_NAME_LEN);
    ack.hub_name[MAX_NAME_LEN] = '\0';

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

        // Check if MAC is already paired (IDs 1..15)
        int found_id = -1;
        for (int i = 1; i < MAX_SENSORS; i++) {
            if (g_sensors[i].active && memcmp(g_sensors[i].mac, item.src_mac, 6) == 0) {
                found_id = i;
                break;
            }
        }

        if (item.pkt.msg_type == 0x02) {
            // PAIR_REQUEST
            if (g_state.pairing_active) {
                if (found_id == -1) {
                    // Find first free slot 1..15
                    for (int i = 1; i < MAX_SENSORS; i++) {
                        if (!g_sensors[i].active) {
                            found_id = i;
                            break;
                        }
                    }
                    if (found_id != -1) {
                        g_sensors[found_id].active = true;
                        g_sensors[found_id].id = found_id;
                        memcpy(g_sensors[found_id].mac, item.src_mac, 6);
                        snprintf(g_sensors[found_id].name, sizeof(g_sensors[found_id].name), "Sensor #%02d", found_id);
                        g_sensors[found_id].role = ROLE_INDOOR;
                        g_sensors[found_id].weight = 1;
                        g_sensors[found_id].offset = 0.0f;
                        g_sensors[found_id].report_interval_sec = 300;
                        g_sensors[found_id].display_timeout_s = 8;
                        needSave = true;
                    }
                }
                if (found_id != -1) {
                    g_sensors[found_id].temp = (item.pkt.temp_x100 / 100.0f) + g_sensors[found_id].offset;
                    g_sensors[found_id].humidity = item.pkt.hum_x100 / 100.0f;
                    g_sensors[found_id].vbat_mv = item.pkt.vbat_mv;
                    g_sensors[found_id].battery_pct = constrain(map(item.pkt.vbat_mv, 2400, 3100, 0, 100), 0, 100);
                    g_sensors[found_id].rssi = item.rssi;
                    g_sensors[found_id].online = true;
                    g_sensors[found_id].last_seen_ms = millis();
                    g_state.last_paired_id = found_id;
                    sendEspNowAck(item.src_mac, 0x82, found_id);
                    needBroadcast = true;
                }
            }
        } else if (item.pkt.msg_type == 0x01 && found_id != -1) {
            // TELEMETRY from known sensor
            g_sensors[found_id].temp = (item.pkt.temp_x100 / 100.0f) + g_sensors[found_id].offset;
            g_sensors[found_id].humidity = item.pkt.hum_x100 / 100.0f;
            g_sensors[found_id].vbat_mv = item.pkt.vbat_mv;
            g_sensors[found_id].battery_pct = constrain(map(item.pkt.vbat_mv, 2400, 3100, 0, 100), 0, 100);
            g_sensors[found_id].rssi = item.rssi;
            g_sensors[found_id].online = true;
            g_sensors[found_id].last_seen_ms = millis();

            sendEspNowAck(item.src_mac, 0x81, found_id);
            needBroadcast = true;
        }

        xSemaphoreGive(g_mutex);

        if (needSave) saveAllConfigToNVS();
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
    DynamicJsonDocument doc(8192);
    xSemaphoreTake(g_mutex, portMAX_DELAY);

    JsonObject dev = doc.createNestedObject("device");
    dev["hostname"] = g_settings.hostname;
    dev["hub_name"] = g_settings.hub_name;
    dev["uptime_sec"] = millis() / 1000UL;
    dev["wifi_connected"] = (WiFi.status() == WL_CONNECTED);
    dev["wifi_ssid"] = g_settings.wifi_ssid;
    dev["ip"] = WiFi.localIP().toString();
    dev["wifi_rssi"] = WiFi.RSSI();
    dev["wifi_channel"] = WiFi.channel();
    dev["mqtt_connected"] = mqtt.connected();
    dev["pairing_active"] = g_state.pairing_active;
    dev["pairing_remaining_sec"] = g_state.pairing_active
        ? (int)max(0L, (long)(g_state.pairing_end_ms - millis()) / 1000L) : 0;
    dev["last_paired_id"] = g_state.last_paired_id;

    struct tm timeinfo;
    if (getLocalTime(&timeinfo, 5)) {
        char iso[32];
        strftime(iso, sizeof(iso), "%Y-%m-%dT%H:%M:%S", &timeinfo);
        dev["rtc_time"] = iso;
    }

    JsonObject th = doc.createNestedObject("thermostat");
    const char *sysModes[] = {"AUTO", "MANUAL", "BOOST", "OFF"};
    const char *ctrlModes[] = {"hysteresis_static", "hysteresis_dynamic", "tpi"};
    const char *aggModes[] = {"single", "average", "weighted", "min", "max"};

    th["system_mode"] = sysModes[g_settings.system_mode];
    th["control_mode"] = ctrlModes[g_settings.control_mode];
    th["agg_mode"] = aggModes[g_settings.agg_mode];
    th["single_sensor_id"] = g_settings.single_sensor_id;
    th["control_temp"] = roundf(g_state.control_temp * 100.0f) / 100.0f;
    th["control_humidity"] = roundf(g_state.control_humidity * 10.0f) / 10.0f;
    th["target_temp"] = g_state.target_temp;
    th["effective_target_temp"] = roundf(g_state.effective_target_temp * 100.0f) / 100.0f;
    th["outdoor_temp"] = isnan(g_state.outdoor_temp) ? JsonVariant() : roundf(g_state.outdoor_temp * 10.0f) / 10.0f;
    th["heating_active"] = g_state.heating_active;
    th["lockout_reason"] = g_state.lockout_reason;
    th["learned_overshoot_c"] = roundf(g_state.learned_overshoot_c * 100.0f) / 100.0f;
    th["min_on_remaining_sec"] = g_state.min_on_remaining_sec;
    th["min_off_remaining_sec"] = g_state.min_off_remaining_sec;
    th["boost_remaining_sec"] = g_state.boost_remaining_sec;
    th["runtime_today_sec"] = g_state.runtime_today_sec;

    JsonArray sArr = doc.createNestedArray("sensors");
    const char *roles[] = {"indoor", "outdoor", "disabled"};
    for (int i = 0; i < MAX_SENSORS; i++) {
        if (!g_sensors[i].active) continue;
        JsonObject s = sArr.createNestedObject();
        s["id"] = g_sensors[i].id;
        s["name"] = g_sensors[i].name;
        s["role"] = roles[g_sensors[i].role];
        s["type"] = (i == 0) ? "onboard" : "wireless";
        if (i > 0) {
            char macStr[18];
            snprintf(macStr, sizeof(macStr), "%02X:%02X:%02X:%02X:%02X:%02X",
                     g_sensors[i].mac[0], g_sensors[i].mac[1], g_sensors[i].mac[2],
                     g_sensors[i].mac[3], g_sensors[i].mac[4], g_sensors[i].mac[5]);
            s["mac"] = macStr;
            s["battery_pct"] = g_sensors[i].battery_pct;
            s["vbat_mv"] = g_sensors[i].vbat_mv;
            s["rssi"] = g_sensors[i].rssi;
            s["report_interval_sec"] = g_sensors[i].report_interval_sec;
            s["display_timeout_s"] = g_sensors[i].display_timeout_s;
        }
        s["temp"] = roundf(g_sensors[i].temp * 100.0f) / 100.0f;
        s["humidity"] = roundf(g_sensors[i].humidity * 10.0f) / 10.0f;
        s["offset"] = g_sensors[i].offset;
        s["weight"] = g_sensors[i].weight;
        s["online"] = g_sensors[i].online;
        s["last_seen_sec"] = (millis() - g_sensors[i].last_seen_ms) / 1000UL;
    }

    xSemaphoreGive(g_mutex);
    String out;
    serializeJson(doc, out);
    return out;
}

static String buildSettingsJson() {
    DynamicJsonDocument doc(2048);
    xSemaphoreTake(g_mutex, portMAX_DELAY);
    const char *ctrlModes[] = {"hysteresis_static", "hysteresis_dynamic", "tpi"};
    const char *aggModes[] = {"single", "average", "weighted", "min", "max"};

    doc["hub_name"] = g_settings.hub_name;
    doc["hostname"] = g_settings.hostname;
    doc["control_mode"] = ctrlModes[g_settings.control_mode];
    doc["agg_mode"] = aggModes[g_settings.agg_mode];
    doc["single_sensor_id"] = g_settings.single_sensor_id;
    doc["hysteresis_low"] = g_settings.hysteresis_low;
    doc["hysteresis_high"] = g_settings.hysteresis_high;
    doc["tpi_cycle_min"] = g_settings.tpi_cycle_min;
    doc["min_on_time_sec"] = g_settings.min_on_time_sec;
    doc["min_off_time_sec"] = g_settings.min_off_time_sec;
    doc["outdoor_lockout_enabled"] = g_settings.outdoor_lockout_enabled;
    doc["outdoor_lockout_temp"] = g_settings.outdoor_lockout_temp;
    doc["weather_comp_enabled"] = g_settings.weather_comp_enabled;
    doc["weather_comp_slope"] = g_settings.weather_comp_slope;
    doc["frost_protect_temp"] = g_settings.frost_protect_temp;
    doc["open_window_enabled"] = g_settings.open_window_enabled;
    doc["sensor_timeout_min"] = g_settings.sensor_timeout_min;
    doc["pump_exercise_enabled"] = g_settings.pump_exercise_enabled;
    doc["manual_temp"] = g_settings.manual_temp;
    doc["boost_temp"] = g_settings.boost_temp;
    doc["boost_duration_min"] = g_settings.boost_duration_min;
    doc["mqtt_enabled"] = g_settings.mqtt_enabled;
    doc["mqtt_host"] = g_settings.mqtt_host;
    doc["mqtt_port"] = g_settings.mqtt_port;
    doc["mqtt_user"] = g_settings.mqtt_user;
    doc["mqtt_base_topic"] = g_settings.mqtt_base_topic;
    doc["ha_discovery"] = g_settings.ha_discovery;
    xSemaphoreGive(g_mutex);

    String out;
    serializeJson(doc, out);
    return out;
}

static bool applySettingsJson(JsonVariantConst root) {
    xSemaphoreTake(g_mutex, portMAX_DELAY);

    if (root.containsKey("hub_name")) {
        strncpy(g_settings.hub_name, root["hub_name"] | "Thermo", MAX_NAME_LEN);
        g_settings.hub_name[MAX_NAME_LEN] = '\0';
    }
    if (root.containsKey("hostname")) {
        strncpy(g_settings.hostname, root["hostname"] | "opensmartthermo", 31);
    }
    if (root.containsKey("control_mode")) {
        String cm = root["control_mode"].as<String>();
        if (cm == "hysteresis_static") g_settings.control_mode = CTRL_HYST_STATIC;
        else if (cm == "hysteresis_dynamic") g_settings.control_mode = CTRL_HYST_DYNAMIC;
        else if (cm == "tpi") g_settings.control_mode = CTRL_TPI;
    }
    if (root.containsKey("agg_mode")) {
        String am = root["agg_mode"].as<String>();
        if (am == "single") g_settings.agg_mode = AGG_SINGLE;
        else if (am == "average") g_settings.agg_mode = AGG_AVERAGE;
        else if (am == "weighted") g_settings.agg_mode = AGG_WEIGHTED;
        else if (am == "min") g_settings.agg_mode = AGG_MIN;
        else if (am == "max") g_settings.agg_mode = AGG_MAX;
    }
    if (root.containsKey("single_sensor_id")) g_settings.single_sensor_id = constrain((int)root["single_sensor_id"], 0, MAX_SENSORS - 1);
    if (root.containsKey("hysteresis_low")) g_settings.hysteresis_low = constrain((float)root["hysteresis_low"], 0.1f, 3.0f);
    if (root.containsKey("hysteresis_high")) g_settings.hysteresis_high = constrain((float)root["hysteresis_high"], 0.1f, 3.0f);
    if (root.containsKey("tpi_cycle_min")) g_settings.tpi_cycle_min = constrain((int)root["tpi_cycle_min"], 10, 30);
    if (root.containsKey("min_on_time_sec")) g_settings.min_on_time_sec = constrain((int)root["min_on_time_sec"], 60, 900);
    if (root.containsKey("min_off_time_sec")) g_settings.min_off_time_sec = constrain((int)root["min_off_time_sec"], 60, 900);
    if (root.containsKey("outdoor_lockout_enabled")) g_settings.outdoor_lockout_enabled = root["outdoor_lockout_enabled"];
    if (root.containsKey("outdoor_lockout_temp")) g_settings.outdoor_lockout_temp = constrain((float)root["outdoor_lockout_temp"], 5.0f, 35.0f);
    if (root.containsKey("weather_comp_enabled")) g_settings.weather_comp_enabled = root["weather_comp_enabled"];
    if (root.containsKey("weather_comp_slope")) g_settings.weather_comp_slope = constrain((float)root["weather_comp_slope"], 0.0f, 0.25f);
    if (root.containsKey("frost_protect_temp")) g_settings.frost_protect_temp = constrain((float)root["frost_protect_temp"], 4.0f, 12.0f);
    if (root.containsKey("open_window_enabled")) g_settings.open_window_enabled = root["open_window_enabled"];
    if (root.containsKey("sensor_timeout_min")) g_settings.sensor_timeout_min = constrain((int)root["sensor_timeout_min"], 5, 120);
    if (root.containsKey("pump_exercise_enabled")) g_settings.pump_exercise_enabled = root["pump_exercise_enabled"];
    if (root.containsKey("manual_temp")) g_settings.manual_temp = constrain((float)root["manual_temp"], 5.0f, 30.0f);
    if (root.containsKey("boost_temp")) g_settings.boost_temp = constrain((float)root["boost_temp"], 5.0f, 30.0f);
    if (root.containsKey("boost_duration_min")) g_settings.boost_duration_min = constrain((int)root["boost_duration_min"], 15, 240);
    if (root.containsKey("mqtt_enabled")) g_settings.mqtt_enabled = root["mqtt_enabled"];
    if (root.containsKey("mqtt_host")) strncpy(g_settings.mqtt_host, root["mqtt_host"] | "", 63);
    if (root.containsKey("mqtt_port")) g_settings.mqtt_port = root["mqtt_port"] | 1883;
    if (root.containsKey("mqtt_user")) strncpy(g_settings.mqtt_user, root["mqtt_user"] | "", 31);
    if (root.containsKey("mqtt_pass") && strlen(root["mqtt_pass"] | "") > 0) strncpy(g_settings.mqtt_pass, root["mqtt_pass"], 31);
    if (root.containsKey("mqtt_base_topic")) strncpy(g_settings.mqtt_base_topic, root["mqtt_base_topic"] | "opensmartthermo", 31);
    if (root.containsKey("ha_discovery")) g_settings.ha_discovery = root["ha_discovery"];

    xSemaphoreGive(g_mutex);
    saveAllConfigToNVS();
    return true;
}

static bool applyScheduleJson(JsonVariantConst root) {
    if (!root.containsKey("day") || !root.containsKey("slots")) return false;
    int day = constrain((int)root["day"], 0, 6);
    JsonArrayConst slots = root["slots"].as<JsonArrayConst>();

    xSemaphoreTake(g_mutex, portMAX_DELAY);
    int idx = 0;
    for (JsonVariantConst s : slots) {
        if (idx >= SLOTS_PER_DAY) break;
        g_schedule[day].slots[idx].enabled = s["enabled"] | false;
        String tStr = s["time"] | "08:00";
        int colon = tStr.indexOf(':');
        if (colon > 0) {
            g_schedule[day].slots[idx].hour = constrain(tStr.substring(0, colon).toInt(), 0, 23);
            g_schedule[day].slots[idx].minute = constrain(tStr.substring(colon + 1).toInt(), 0, 59);
        }
        g_schedule[day].slots[idx].target_temp = constrain((float)(s["target_temp"] | 20.0f), 5.0f, 30.0f);
        String am = s["agg_mode"] | "default";
        if (am == "single") g_schedule[day].slots[idx].agg_mode_override = AGG_SINGLE;
        else if (am == "average") g_schedule[day].slots[idx].agg_mode_override = AGG_AVERAGE;
        else if (am == "weighted") g_schedule[day].slots[idx].agg_mode_override = AGG_WEIGHTED;
        else if (am == "min") g_schedule[day].slots[idx].agg_mode_override = AGG_MIN;
        else if (am == "max") g_schedule[day].slots[idx].agg_mode_override = AGG_MAX;
        else g_schedule[day].slots[idx].agg_mode_override = -1;

        g_schedule[day].slots[idx].target_sensor_id = constrain((int)(s["target_sensor_id"] | 0), 0, MAX_SENSORS - 1);
        idx++;
    }

    if (root.containsKey("copy_to_days")) {
        for (JsonVariantConst d : root["copy_to_days"].as<JsonArrayConst>()) {
            int targetDay = d.as<int>();
            if (targetDay >= 0 && targetDay < 7 && targetDay != day) {
                memcpy(&g_schedule[targetDay], &g_schedule[day], sizeof(DaySchedule));
            }
        }
    }
    xSemaphoreGive(g_mutex);
    saveAllConfigToNVS();
    return true;
}

// -------------------------------------------------------------------------
// MQTT & Home Assistant Auto-Discovery
// -------------------------------------------------------------------------
static void publishHaDiscovery() {
    if (!g_settings.ha_discovery || !mqtt.connected()) return;
    String base = String(g_settings.mqtt_base_topic);

    DynamicJsonDocument doc(1024);
    doc["name"] = String(g_settings.hub_name) + " Thermostat";
    doc["uniq_id"] = "opensmartthermo_climate";
    doc["mode_cmd_t"] = base + "/set/ha_mode";
    doc["mode_stat_t"] = base + "/state";
    doc["mode_stat_tpl"] = "{{ 'off' if value_json.mode == 'OFF' else ('auto' if value_json.mode == 'AUTO' else 'heat') }}";
    doc["temp_cmd_t"] = base + "/set/target_temp";
    doc["temp_stat_t"] = base + "/state";
    doc["temp_stat_tpl"] = "{{ value_json.target_temp }}";
    doc["curr_temp_t"] = base + "/state";
    doc["curr_temp_tpl"] = "{{ value_json.control_temp }}";
    doc["act_t"] = base + "/state";
    doc["act_tpl"] = "{{ 'heating' if value_json.heating_active else 'idle' }}";
    JsonArray modes = doc.createNestedArray("modes");
    modes.add("off");
    modes.add("heat");
    modes.add("auto");
    doc["min_temp"] = 5.0;
    doc["max_temp"] = 30.0;
    doc["temp_step"] = 0.5;

    String payload;
    serializeJson(doc, payload);
    mqtt.publish("homeassistant/climate/opensmartthermo/config", payload.c_str(), true);

    // Individual sensor entities
    for (int i = 0; i < MAX_SENSORS; i++) {
        if (!g_sensors[i].active) continue;
        DynamicJsonDocument sDoc(512);
        sDoc["name"] = String(g_sensors[i].name) + " Temperature";
        sDoc["uniq_id"] = "opensmartthermo_s" + String(i) + "_temp";
        sDoc["stat_t"] = base + "/sensor/" + String(i) + "/state";
        sDoc["val_tpl"] = "{{ value_json.temp }}";
        sDoc["dev_cla"] = "temperature";
        sDoc["unit_of_meas"] = "°C";
        String sPay;
        serializeJson(sDoc, sPay);
        mqtt.publish(("homeassistant/sensor/opensmartthermo_s" + String(i) + "_temp/config").c_str(), sPay.c_str(), true);
    }
    haDiscoveryPublished = true;
}

static void onMqttMessage(char *topic, byte *payload, unsigned int length) {
    String msg;
    msg.reserve(length + 1);
    for (unsigned int i = 0; i < length; i++) msg += (char)payload[i];

    String t = String(topic);
    String base = String(g_settings.mqtt_base_topic);

    if (t == base + "/set/mode" || t == base + "/set/ha_mode") {
        msg.toUpperCase();
        xSemaphoreTake(g_mutex, portMAX_DELAY);
        if (msg == "AUTO") g_settings.system_mode = MODE_AUTO;
        else if (msg == "MANUAL" || msg == "HEAT") g_settings.system_mode = MODE_MANUAL;
        else if (msg == "BOOST") {
            g_settings.system_mode = MODE_BOOST;
            g_state.boost_end_ms = millis() + (g_settings.boost_duration_min * 60000UL);
        } else if (msg == "OFF") g_settings.system_mode = MODE_OFF;
        xSemaphoreGive(g_mutex);
        saveAllConfigToNVS();
        networkBroadcastWS();
        networkPublishMqttState();
    } else if (t == base + "/set/target_temp") {
        float val = constrain(msg.toFloat(), 5.0f, 30.0f);
        xSemaphoreTake(g_mutex, portMAX_DELAY);
        if (g_settings.system_mode == MODE_AUTO) g_settings.system_mode = MODE_MANUAL;
        if (g_settings.system_mode == MODE_BOOST) g_settings.boost_temp = val;
        else g_settings.manual_temp = val;
        xSemaphoreGive(g_mutex);
        saveAllConfigToNVS();
        networkBroadcastWS();
        networkPublishMqttState();
    } else if (t == base + "/set/pairing") {
        if (msg.indexOf("start") >= 0) networkStartPairing(60);
        else networkStopPairing();
    } else if (t == base + "/set/settings") {
        DynamicJsonDocument doc(2048);
        if (!deserializeJson(doc, msg)) applySettingsJson(doc.as<JsonVariantConst>());
    } else if (t == base + "/set/schedule") {
        DynamicJsonDocument doc(2048);
        if (!deserializeJson(doc, msg)) applyScheduleJson(doc.as<JsonVariantConst>());
    }
}

void networkPublishMqttState() {
    if (!g_settings.mqtt_enabled || !mqtt.connected()) return;
    String base = String(g_settings.mqtt_base_topic);

    DynamicJsonDocument doc(512);
    const char *sysModes[] = {"AUTO", "MANUAL", "BOOST", "OFF"};
    const char *ctrlModes[] = {"hysteresis_static", "hysteresis_dynamic", "tpi"};

    xSemaphoreTake(g_mutex, portMAX_DELAY);
    doc["mode"] = sysModes[g_settings.system_mode];
    doc["heating_active"] = g_state.heating_active;
    doc["control_temp"] = roundf(g_state.control_temp * 100.0f) / 100.0f;
    doc["target_temp"] = g_state.target_temp;
    if (!isnan(g_state.outdoor_temp)) doc["outdoor_temp"] = roundf(g_state.outdoor_temp * 10.0f) / 10.0f;
    doc["control_mode"] = ctrlModes[g_settings.control_mode];
    doc["learned_overshoot"] = roundf(g_state.learned_overshoot_c * 100.0f) / 100.0f;
    doc["lockout_reason"] = g_state.lockout_reason;
    xSemaphoreGive(g_mutex);

    String payload;
    serializeJson(doc, payload);
    mqtt.publish((base + "/state").c_str(), payload.c_str(), true);

    for (int i = 0; i < MAX_SENSORS; i++) {
        if (!g_sensors[i].active) continue;
        DynamicJsonDocument sDoc(256);
        sDoc["id"] = i;
        sDoc["name"] = g_sensors[i].name;
        sDoc["temp"] = roundf(g_sensors[i].temp * 100.0f) / 100.0f;
        sDoc["humidity"] = roundf(g_sensors[i].humidity * 10.0f) / 10.0f;
        sDoc["battery"] = g_sensors[i].battery_pct;
        sDoc["online"] = g_sensors[i].online;
        String sPay;
        serializeJson(sDoc, sPay);
        mqtt.publish((base + "/sensor/" + String(i) + "/state").c_str(), sPay.c_str(), true);
    }
}

// -------------------------------------------------------------------------
// Public Control Helpers
// -------------------------------------------------------------------------
void networkStartPairing(uint16_t duration_sec) {
    xSemaphoreTake(g_mutex, portMAX_DELAY);
    g_state.pairing_active = true;
    g_state.pairing_end_ms = millis() + (duration_sec * 1000UL);
    g_state.last_paired_id = -1;
    xSemaphoreGive(g_mutex);
    networkBroadcastWS();
}

void networkStopPairing() {
    xSemaphoreTake(g_mutex, portMAX_DELAY);
    g_state.pairing_active = false;
    g_state.pairing_end_ms = 0;
    xSemaphoreGive(g_mutex);
    networkBroadcastWS();
}

void networkStartSetupAP() {
    WiFi.mode(WIFI_AP_STA);
    WiFi.softAP("OpenSmartThermo-Setup");
}

void networkStopSetupAP() {
    WiFi.softAPdisconnect(true);
    WiFi.mode(WIFI_STA);
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
        if (type == WS_EVT_CONNECT) {
            c->text(buildStatusJson());
        }
    });
    server.addHandler(&ws);

    // GET /api/v1/status
    server.on("/api/v1/status", HTTP_GET, [](AsyncWebServerRequest *req) {
        req->send(200, "application/json", buildStatusJson());
    });

    // POST /api/v1/control
    server.on("/api/v1/control", HTTP_POST, [](AsyncWebServerRequest *req) {}, nullptr,
        [](AsyncWebServerRequest *req, uint8_t *data, size_t len, size_t index, size_t total) {
            DynamicJsonDocument doc(512);
            if (deserializeJson(doc, data, len)) {
                req->send(400, "application/json", "{\"error\":\"Invalid JSON\"}");
                return;
            }
            xSemaphoreTake(g_mutex, portMAX_DELAY);
            if (doc.containsKey("system_mode")) {
                String m = doc["system_mode"].as<String>();
                if (m == "AUTO") g_settings.system_mode = MODE_AUTO;
                else if (m == "MANUAL") g_settings.system_mode = MODE_MANUAL;
                else if (m == "BOOST") {
                    g_settings.system_mode = MODE_BOOST;
                    int dur = doc["boost_duration_min"] | g_settings.boost_duration_min;
                    g_state.boost_end_ms = millis() + (dur * 60000UL);
                } else if (m == "OFF") g_settings.system_mode = MODE_OFF;
            }
            if (doc.containsKey("target_temp")) {
                float t = constrain((float)doc["target_temp"], 5.0f, 30.0f);
                if (g_settings.system_mode == MODE_AUTO) g_settings.system_mode = MODE_MANUAL;
                if (g_settings.system_mode == MODE_BOOST) g_settings.boost_temp = t;
                else g_settings.manual_temp = t;
            }
            xSemaphoreGive(g_mutex);
            saveAllConfigToNVS();
            networkBroadcastWS();
            networkPublishMqttState();
            req->send(200, "application/json", "{\"status\":\"ok\"}");
        });

    // POST /api/v1/sensors/pair
    server.on("/api/v1/sensors/pair", HTTP_POST, [](AsyncWebServerRequest *req) {}, nullptr,
        [](AsyncWebServerRequest *req, uint8_t *data, size_t len, size_t index, size_t total) {
            DynamicJsonDocument doc(256);
            deserializeJson(doc, data, len);
            String act = doc["action"] | "start";
            if (act == "start") {
                uint16_t dur = constrain((int)(doc["duration_sec"] | 60), 15, 180);
                networkStartPairing(dur);
                req->send(200, "application/json", "{\"status\":\"ok\",\"pairing_active\":true}");
            } else {
                networkStopPairing();
                req->send(200, "application/json", "{\"status\":\"ok\",\"pairing_active\":false}");
            }
        });

    // PUT & DELETE /api/v1/sensors/{id}
    server.on("^\\/api\\/v1\\/sensors\\/([0-9]+)$", HTTP_PUT, [](AsyncWebServerRequest *req) {}, nullptr,
        [](AsyncWebServerRequest *req, uint8_t *data, size_t len, size_t index, size_t total) {
            int id = req->pathArg(0).toInt();
            if (id < 0 || id >= MAX_SENSORS || !g_sensors[id].active) {
                req->send(404, "application/json", "{\"error\":\"Sensor not found\"}");
                return;
            }
            DynamicJsonDocument doc(512);
            if (deserializeJson(doc, data, len)) {
                req->send(400, "application/json", "{\"error\":\"Invalid JSON\"}");
                return;
            }
            xSemaphoreTake(g_mutex, portMAX_DELAY);
            if (doc.containsKey("name")) {
                strncpy(g_sensors[id].name, doc["name"] | "Sensor", MAX_NAME_LEN);
                g_sensors[id].name[MAX_NAME_LEN] = '\0';
            }
            if (doc.containsKey("role")) {
                String r = doc["role"].as<String>();
                if (r == "indoor") g_sensors[id].role = ROLE_INDOOR;
                else if (r == "outdoor") g_sensors[id].role = ROLE_OUTDOOR;
                else if (r == "disabled") g_sensors[id].role = ROLE_DISABLED;
            }
            if (doc.containsKey("weight")) g_sensors[id].weight = constrain((int)doc["weight"], 1, 5);
            if (doc.containsKey("offset")) g_sensors[id].offset = constrain((float)doc["offset"], -5.0f, 5.0f);
            if (doc.containsKey("report_interval_sec")) g_sensors[id].report_interval_sec = constrain((int)doc["report_interval_sec"], 60, 1800);
            if (doc.containsKey("display_timeout_s")) g_sensors[id].display_timeout_s = constrain((int)doc["display_timeout_s"], 5, 10);
            xSemaphoreGive(g_mutex);

            saveAllConfigToNVS();
            haDiscoveryPublished = false;
            networkBroadcastWS();
            req->send(200, "application/json", "{\"status\":\"ok\",\"pending_sync\":true}");
        });

    server.on("^\\/api\\/v1\\/sensors\\/([0-9]+)$", HTTP_DELETE, [](AsyncWebServerRequest *req) {
        int id = req->pathArg(0).toInt();
        if (id <= 0 || id >= MAX_SENSORS || !g_sensors[id].active) {
            req->send(400, "application/json", "{\"error\":\"Cannot delete sensor\"}");
            return;
        }
        xSemaphoreTake(g_mutex, portMAX_DELAY);
        esp_now_del_peer(g_sensors[id].mac);
        memset(&g_sensors[id], 0, sizeof(SensorInfo));
        xSemaphoreGive(g_mutex);
        saveAllConfigToNVS();
        networkBroadcastWS();
        req->send(200, "application/json", "{\"status\":\"ok\"}");
    });

    // GET & POST /api/v1/schedule
    server.on("/api/v1/schedule", HTTP_GET, [](AsyncWebServerRequest *req) {
        DynamicJsonDocument doc(8192);
        JsonArray days = doc.createNestedArray("days");
        const char *aggModes[] = {"single", "average", "weighted", "min", "max"};
        xSemaphoreTake(g_mutex, portMAX_DELAY);
        for (int d = 0; d < 7; d++) {
            JsonArray sArr = days.createNestedArray();
            for (int s = 0; s < SLOTS_PER_DAY; s++) {
                JsonObject obj = sArr.createNestedObject();
                obj["enabled"] = g_schedule[d].slots[s].enabled;
                char tBuf[6];
                snprintf(tBuf, sizeof(tBuf), "%02d:%02d", g_schedule[d].slots[s].hour, g_schedule[d].slots[s].minute);
                obj["time"] = tBuf;
                obj["target_temp"] = g_schedule[d].slots[s].target_temp;
                int ov = g_schedule[d].slots[s].agg_mode_override;
                obj["agg_mode"] = (ov >= 0 && ov <= 4) ? aggModes[ov] : "default";
                obj["target_sensor_id"] = g_schedule[d].slots[s].target_sensor_id;
            }
        }
        xSemaphoreGive(g_mutex);
        String out;
        serializeJson(doc, out);
        req->send(200, "application/json", out);
    });

    server.on("/api/v1/schedule", HTTP_POST, [](AsyncWebServerRequest *req) {}, nullptr,
        [](AsyncWebServerRequest *req, uint8_t *data, size_t len, size_t index, size_t total) {
            DynamicJsonDocument doc(2048);
            if (deserializeJson(doc, data, len) || !applyScheduleJson(doc.as<JsonVariantConst>())) {
                req->send(400, "application/json", "{\"error\":\"Invalid schedule payload\"}");
                return;
            }
            req->send(200, "application/json", "{\"status\":\"ok\"}");
        });

    // GET & POST /api/v1/settings
    server.on("/api/v1/settings", HTTP_GET, [](AsyncWebServerRequest *req) {
        req->send(200, "application/json", buildSettingsJson());
    });

    server.on("/api/v1/settings", HTTP_POST, [](AsyncWebServerRequest *req) {}, nullptr,
        [](AsyncWebServerRequest *req, uint8_t *data, size_t len, size_t index, size_t total) {
            DynamicJsonDocument doc(2048);
            if (deserializeJson(doc, data, len)) {
                req->send(400, "application/json", "{\"error\":\"Invalid JSON\"}");
                return;
            }
            applySettingsJson(doc.as<JsonVariantConst>());
            networkBroadcastWS();
            req->send(200, "application/json", "{\"status\":\"ok\"}");
        });

    // POST /api/v1/settings/reset-learning
    server.on("/api/v1/settings/reset-learning", HTTP_POST, [](AsyncWebServerRequest *req) {
        controlResetLearning();
        saveAllConfigToNVS();
        networkBroadcastWS();
        req->send(200, "application/json", "{\"status\":\"ok\",\"learned_overshoot_c\":0.20}");
    });

    // GET /api/v1/wifi/scan & POST /api/v1/wifi
    server.on("/api/v1/wifi/scan", HTTP_GET, [](AsyncWebServerRequest *req) {
        int n = WiFi.scanComplete();
        if (n == -2) {
            WiFi.scanNetworks(true);
            req->send(202, "application/json", "{\"status\":\"scanning\"}");
            return;
        }
        DynamicJsonDocument doc(2048);
        JsonArray arr = doc.createNestedArray("networks");
        for (int i = 0; i < n; i++) {
            JsonObject net = arr.createNestedObject();
            net["ssid"] = WiFi.SSID(i);
            net["rssi"] = WiFi.RSSI(i);
            net["secure"] = (WiFi.encryptionType(i) != WIFI_AUTH_OPEN);
        }
        WiFi.scanDelete();
        String out;
        serializeJson(doc, out);
        req->send(200, "application/json", out);
    });

    server.on("/api/v1/wifi", HTTP_POST, [](AsyncWebServerRequest *req) {}, nullptr,
        [](AsyncWebServerRequest *req, uint8_t *data, size_t len, size_t index, size_t total) {
            DynamicJsonDocument doc(512);
            if (deserializeJson(doc, data, len)) {
                req->send(400, "application/json", "{\"error\":\"Invalid JSON\"}");
                return;
            }
            xSemaphoreTake(g_mutex, portMAX_DELAY);
            strncpy(g_settings.wifi_ssid, doc["ssid"] | "", 31);
            strncpy(g_settings.wifi_pass, doc["password"] | "", 63);
            if (doc.containsKey("hostname")) strncpy(g_settings.hostname, doc["hostname"] | "opensmartthermo", 31);
            xSemaphoreGive(g_mutex);
            saveAllConfigToNVS();
            req->send(200, "application/json", "{\"status\":\"ok\",\"message\":\"Rebooting to apply Wi-Fi...\"}");
            delay(800);
            ESP.restart();
        });

    // POST /api/v1/ota (Firmware binary upload)
    server.on("/api/v1/ota", HTTP_POST,
        [](AsyncWebServerRequest *req) {
            bool ok = !Update.hasError();
            req->send(ok ? 200 : 500, "application/json", ok ? "{\"status\":\"ok\"}" : "{\"error\":\"OTA failed\"}");
            if (ok) { delay(800); ESP.restart(); }
        },
        [](AsyncWebServerRequest *req, String filename, size_t index, uint8_t *data, size_t len, bool final) {
            if (!index) {
                Update.begin(UPDATE_SIZE_UNKNOWN);
            }
            if (len) {
                Update.write(data, len);
            }
            if (final) {
                Update.end(true);
            }
        });

    // Serve Embedded Web SPA from LittleFS
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
    espnowRxQueue = xQueueCreate(16, sizeof(rx_queue_item_t));

    WiFi.mode(WIFI_STA);
    WiFi.setSleep(false); // Keep Wi-Fi + ESP-NOW receiver responsive 24/7

    if (strlen(g_settings.wifi_ssid) > 0) {
        WiFi.setHostname(g_settings.hostname);
        WiFi.begin(g_settings.wifi_ssid, g_settings.wifi_pass);
        unsigned long startAttempt = millis();
        while (WiFi.status() != WL_CONNECTED && millis() - startAttempt < 10000) {
            delay(250);
        }
    }

    if (WiFi.status() != WL_CONNECTED) {
        networkStartSetupAP();
    } else {
        MDNS.begin(g_settings.hostname);
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

    // Auto-expire pairing window
    if (g_state.pairing_active && millis() >= g_state.pairing_end_ms) {
        networkStopPairing();
    }

    // Periodic WebSocket push every 2 seconds
    if (millis() - lastWsBroadcastMs >= 2000) {
        lastWsBroadcastMs = millis();
        networkBroadcastWS();
    }

    // MQTT Connection & Periodic State Publish
    if (g_settings.mqtt_enabled && WiFi.status() == WL_CONNECTED && strlen(g_settings.mqtt_host) > 0) {
        if (!mqtt.connected()) {
            if (millis() - lastMqttReconnectMs >= 5000) {
                lastMqttReconnectMs = millis();
                mqtt.setServer(g_settings.mqtt_host, g_settings.mqtt_port);
                mqtt.setCallback(onMqttMessage);
                mqtt.setBufferSize(1024);
                bool ok = strlen(g_settings.mqtt_user) > 0
                    ? mqtt.connect(g_settings.hostname, g_settings.mqtt_user, g_settings.mqtt_pass)
                    : mqtt.connect(g_settings.hostname);
                if (ok) {
                    String base = String(g_settings.mqtt_base_topic);
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
