# OpenSmartThermo — Part 1: Wireless Sensor Node Firmware (`v4.4`)

**Target Hardware:** Seeed Studio XIAO ESP32C3 + Sensirion SHT40 + 0.96" 128x64 I2C Display  
**Framework:** Arduino / PlatformIO (`espressif32`)

---

## 1. Project Folder Structure

Create a PlatformIO project folder named `OpenSmartThermo_Node` with the following two files:

```text
OpenSmartThermo_Node/
├── platformio.ini
└── src/
    └── main.cpp
```

---

## 2. `platformio.ini`

```ini
[env:seeed_xiao_esp32c3]
platform = espressif32 @ ^6.7.0
board = seeed_xiao_esp32c3
framework = arduino
monitor_speed = 115200

build_flags =
    -DARDUINO_USB_MODE=1
    -DARDUINO_USB_CDC_ON_BOOT=1
    -DCORE_DEBUG_LEVEL=0

lib_deps =
    adafruit/Adafruit SHT4x Library @ ^1.0.4
    adafruit/Adafruit Unified Sensor @ ^1.1.14
    olikraus/U8g2 @ ^2.35.19
```

---

## 3. `src/main.cpp`

```cpp
/**
 * OpenSmartThermo - Wireless Indoor/Outdoor Thermometer Node Firmware (v4.4)
 * Target MCU: Seeed Studio XIAO ESP32C3
 * Peripherals:
 *   - Sensirion SHT40 (I2C: SDA=D4/GPIO6, SCL=D5/GPIO7)
 *   - 0.96" 128x64 I2C OLED/LCD (SSD1306/ST7567 via U8g2, VCC gated by D3/GPIO5)
 *   - Unified PAIR/SCREEN Button (D1/GPIO3, RTC wake-up, INPUT_PULLUP)
 *   - Battery Voltage Divider (2x 1MOhm on D0/GPIO2)
 */

#include <Arduino.h>
#include <Wire.h>
#include <WiFi.h>
#include <esp_now.h>
#include <esp_wifi.h>
#include <esp_sleep.h>
#include <Preferences.h>
#include <Adafruit_SHT4x.h>
#include <U8g2lib.h>

// ============================================================================
// PIN DEFINITIONS (Seeed Studio XIAO ESP32C3)
// ============================================================================
#define PIN_VBAT_ADC    2   // D0 / GPIO 2 (2x 1MOhm divider from 3V3 rail)
#define PIN_BTN_UNIFIED 3   // D1 / GPIO 3 (Unified PAIR / SCREEN button, RTC wake)
#define PIN_DISP_PWR    5   // D3 / GPIO 5 (Display VCC hardware power gate)
#define PIN_I2C_SDA     6   // D4 / GPIO 6 (I2C SDA)
#define PIN_I2C_SCL     7   // D5 / GPIO 7 (I2C SCL)

// ============================================================================
// PROTOCOL & TIMING CONSTANTS
// ============================================================================
#define MSG_TELEMETRY       0x01
#define MSG_PAIR_REQUEST    0x02
#define MSG_TELEMETRY_ACK   0x81
#define MSG_PAIR_ACCEPT     0x82

#define DEFAULT_INTERVAL_S  300     // 5 minutes sleep interval
#define DEFAULT_TIMEOUT_S   8       // 8 seconds screen auto-off
#define PAIRING_DURATION_MS 60000   // 60 seconds pairing window
#define LONG_PRESS_MS       2000    // >= 2 seconds triggers pairing
#define ACK_WAIT_MS         180     // Max wait for Hub ESP-NOW ACK

// ============================================================================
// ESP-NOW PACKET STRUCTURES (Must match ESP32-S3 Hub v4.4 exactly)
// ============================================================================
typedef struct __attribute__((packed)) {
    uint8_t  msg_type;        // 0x01 = TELEMETRY, 0x02 = PAIR_REQUEST
    uint8_t  sensor_id;       // 0 = unpaired, 1..15 = assigned ID
    float    temp_c;
    float    humidity_pct;
    uint16_t batt_mv;
    uint8_t  batt_pct;
    uint8_t  wake_reason;     // 0 = timer, 1 = button
    uint32_t seq_num;
    uint8_t  reserved[6];
} espnow_sensor_packet_t;

typedef struct __attribute__((packed)) {
    uint8_t  msg_type;          // 0x81 = TELEMETRY_ACK, 0x82 = PAIR_ACK
    uint8_t  assigned_id;       // 1..15
    uint8_t  wifi_channel;      // Active Wi-Fi channel (1..13)
    uint16_t report_interval_s; // Wake interval in seconds (default 300)
    float    cal_offset_c;      // Temperature calibration offset
    uint8_t  display_timeout_s; // LCD screen-on duration (5..10s)
    char     sensor_name[25];   // Null-terminated custom sensor name
    char     hub_name[21];      // Null-terminated hub name
} espnow_hub_ack_t;

// ============================================================================
// RTC SLOW MEMORY STATE (Persists across Deep Sleep without NVS wear)
// ============================================================================
RTC_DATA_ATTR bool     rtc_initialized       = false;
RTC_DATA_ATTR bool     rtc_is_paired         = false;
RTC_DATA_ATTR uint8_t  rtc_sensor_id         = 0xFF;
RTC_DATA_ATTR uint8_t  rtc_hub_mac[6]        = {0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF};
RTC_DATA_ATTR uint8_t  rtc_wifi_channel      = 1;
RTC_DATA_ATTR uint16_t rtc_report_interval_s = DEFAULT_INTERVAL_S;
RTC_DATA_ATTR int16_t  rtc_temp_offset_x100  = 0;
RTC_DATA_ATTR uint8_t  rtc_display_timeout_s = DEFAULT_TIMEOUT_S;
RTC_DATA_ATTR char     rtc_sensor_name[25]   = "Unpaired Node";
RTC_DATA_ATTR char     rtc_hub_name[25]      = "None";
RTC_DATA_ATTR uint32_t rtc_seq_num           = 0;
RTC_DATA_ATTR bool     rtc_last_link_ok      = false;

// ============================================================================
// RUNTIME GLOBALS
// ============================================================================
Preferences prefs;
Adafruit_SHT4x sht4 = Adafruit_SHT4x();
// 0.96" 128x64 I2C SSD1306/SH1106 compatible U8g2 driver (change if using ST7567)
U8G2_SSD1306_128X64_NONAME_F_HW_I2C u8g2(U8G2_R0, /* reset=*/ U8X8_PIN_NONE);

volatile bool ack_received = false;
espnow_hub_ack_t last_ack;
const uint8_t BROADCAST_MAC[6] = {0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF};

float    cur_temp_c = 0.0f;
float    cur_hum_rh = 0.0f;
uint16_t cur_vbat_mv = 3000;
uint8_t  cur_bat_pct = 100;
bool     display_active = false;

// Marquee scroll state
int16_t  scroll_x_sensor = 0;
int16_t  scroll_x_hub    = 0;
uint32_t last_scroll_ms  = 0;

// ============================================================================
// NVS PERSISTENCE HELPERS
// ============================================================================
void loadConfigFromNVS() {
    prefs.begin("ost_node", true);
    rtc_is_paired         = prefs.getBool("paired", false);
    rtc_sensor_id         = prefs.getUChar("id", 0xFF);
    rtc_wifi_channel      = prefs.getUChar("ch", 1);
    rtc_report_interval_s = prefs.getUShort("intv", DEFAULT_INTERVAL_S);
    rtc_temp_offset_x100  = prefs.getShort("offs", 0);
    rtc_display_timeout_s = prefs.getUChar("tout", DEFAULT_TIMEOUT_S);
    if (rtc_display_timeout_s < 3 || rtc_display_timeout_s > 30) {
        rtc_display_timeout_s = DEFAULT_TIMEOUT_S;
    }
    prefs.getBytes("hmac", rtc_hub_mac, 6);
    String sname = prefs.getString("sname", "Unpaired Node");
    String hname = prefs.getString("hname", "None");
    strncpy(rtc_sensor_name, sname.c_str(), 24);
    rtc_sensor_name[24] = '\0';
    strncpy(rtc_hub_name, hname.c_str(), 24);
    rtc_hub_name[24] = '\0';
    prefs.end();
    rtc_initialized = true;
}

void saveConfigToNVS() {
    prefs.begin("ost_node", false);
    prefs.putBool("paired", rtc_is_paired);
    prefs.putUChar("id", rtc_sensor_id);
    prefs.putUChar("ch", rtc_wifi_channel);
    prefs.putUShort("intv", rtc_report_interval_s);
    prefs.putShort("offs", rtc_temp_offset_x100);
    prefs.putUChar("tout", rtc_display_timeout_s);
    prefs.putBytes("hmac", rtc_hub_mac, 6);
    prefs.putString("sname", rtc_sensor_name);
    prefs.putString("hname", rtc_hub_name);
    prefs.end();
}

// ============================================================================
// BATTERY & SENSOR MEASUREMENT
// ============================================================================
uint16_t readBatteryVoltageMv() {
    analogReadResolution(12);
    // Discard first dummy reading
    (void)analogReadMilliVolts(PIN_VBAT_ADC);
    uint32_t sum = 0;
    for (int i = 0; i < 8; i++) {
        sum += analogReadMilliVolts(PIN_VBAT_ADC);
    }
    // 2x 1MOhm divider halves the voltage -> multiply by 2
    uint16_t vbat = (uint16_t)((sum / 8) * 2);
    return vbat;
}

uint8_t calculateBatteryPercent(uint16_t mv) {
    // 2x AA curve: 3100mV = 100%, 2400mV = 0%
    if (mv >= 3100) return 100;
    if (mv <= 2400) return 0;
    return (uint8_t)(((mv - 2400) * 100UL) / 700UL);
}

void measureSensors() {
    Wire.begin(PIN_I2C_SDA, PIN_I2C_SCL);
    Wire.setClock(400000);

    if (sht4.begin(&Wire)) {
        sht4.setPrecision(SHT4X_HIGH_PRECISION);
        sht4.setHeater(SHT4X_NO_HEATER);
        sensors_event_t humidity, temp;
        if (sht4.getEvent(&humidity, &temp)) {
            cur_temp_c = temp.temperature + (rtc_temp_offset_x100 / 100.0f);
            cur_hum_rh = humidity.relative_humidity;
        }
    }
    cur_vbat_mv = readBatteryVoltageMv();
    cur_bat_pct = calculateBatteryPercent(cur_vbat_mv);
}

// ============================================================================
// ESP-NOW COMMUNICATION
// ============================================================================
void onEspNowRecv(const uint8_t *mac_addr, const uint8_t *data, int len) {
    if (len < (int)sizeof(espnow_hub_ack_t)) return;
    const espnow_hub_ack_t *ack = (const espnow_hub_ack_t *)data;

    if (ack->msg_type == MSG_TELEMETRY_ACK || ack->msg_type == MSG_PAIR_ACCEPT) {
        memcpy(&last_ack, ack, sizeof(espnow_hub_ack_t));
        memcpy(rtc_hub_mac, mac_addr, 6);
        ack_received = true;
    }
}

void initEspNow(uint8_t channel) {
    WiFi.mode(WIFI_STA);
    WiFi.disconnect();
    esp_wifi_set_promiscuous(true);
    esp_wifi_set_channel(channel, WIFI_SECOND_CHAN_NONE);
    esp_wifi_set_promiscuous(false);

    if (esp_now_init() != ESP_OK) {
        return;
    }
    esp_now_register_recv_cb(onEspNowRecv);

    // Register broadcast peer for pairing & channel discovery
    esp_now_peer_info_t peerInfo = {};
    memcpy(peerInfo.peer_addr, BROADCAST_MAC, 6);
    peerInfo.channel = channel;
    peerInfo.encrypt = false;
    esp_now_add_peer(&peerInfo);

    if (rtc_is_paired) {
        esp_now_peer_info_t hubPeer = {};
        memcpy(hubPeer.peer_addr, rtc_hub_mac, 6);
        hubPeer.channel = channel;
        hubPeer.encrypt = false;
        esp_now_add_peer(&hubPeer);
    }
}

void applyAckConfiguration() {
    bool changed = false;

    if (last_ack.assigned_id >= 1 && last_ack.assigned_id <= 15) {
        if (!rtc_is_paired || rtc_sensor_id != last_ack.assigned_id) {
            rtc_is_paired = true;
            rtc_sensor_id = last_ack.assigned_id;
            changed = true;
        }
    }
    if (last_ack.wifi_channel >= 1 && last_ack.wifi_channel <= 13 &&
        rtc_wifi_channel != last_ack.wifi_channel) {
        rtc_wifi_channel = last_ack.wifi_channel;
        changed = true;
    }
    if (last_ack.report_interval_s >= 60 && last_ack.report_interval_s <= 3600 &&
        rtc_report_interval_s != last_ack.report_interval_s) {
        rtc_report_interval_s = last_ack.report_interval_s;
        changed = true;
    }
    int16_t new_offs_x100 = (int16_t)roundf(last_ack.cal_offset_c * 100.0f);
    if (rtc_temp_offset_x100 != new_offs_x100) {
        rtc_temp_offset_x100 = new_offs_x100;
        changed = true;
    }
    if (last_ack.display_timeout_s >= 3 && last_ack.display_timeout_s <= 30 &&
        rtc_display_timeout_s != last_ack.display_timeout_s) {
        rtc_display_timeout_s = last_ack.display_timeout_s;
        changed = true;
    }

    char clean_sname[25];
    char clean_hname[21];
    strncpy(clean_sname, last_ack.sensor_name, 24);
    clean_sname[24] = '\0';
    strncpy(clean_hname, last_ack.hub_name, 20);
    clean_hname[20] = '\0';

    if (strlen(clean_sname) > 0 && strcmp(rtc_sensor_name, clean_sname) != 0) {
        strncpy(rtc_sensor_name, clean_sname, 25);
        changed = true;
    }
    if (strlen(clean_hname) > 0 && strcmp(rtc_hub_name, clean_hname) != 0) {
        strncpy(rtc_hub_name, clean_hname, 25);
        changed = true;
    }

    if (changed) {
        saveConfigToNVS();
    }
}

bool sendTelemetryAndAwaitAck(uint8_t wake_reason) {
    if (!rtc_is_paired) {
        rtc_last_link_ok = false;
        return false;
    }

    initEspNow(rtc_wifi_channel);

    espnow_sensor_packet_t pkt = {};
    pkt.msg_type     = MSG_TELEMETRY;
    pkt.sensor_id    = rtc_sensor_id;
    pkt.temp_c       = cur_temp_c;
    pkt.humidity_pct = cur_hum_rh;
    pkt.batt_mv      = cur_vbat_mv;
    pkt.batt_pct     = cur_bat_pct;
    pkt.wake_reason  = wake_reason;
    pkt.seq_num      = ++rtc_seq_num;

    ack_received = false;
    esp_now_send(rtc_hub_mac, (uint8_t *)&pkt, sizeof(pkt));

    uint32_t t0 = millis();
    while (!ack_received && (millis() - t0 < ACK_WAIT_MS)) {
        delay(2);
    }

    // Channel-Hopping Self-Healing if router changed Wi-Fi channel
    if (!ack_received) {
        for (uint8_t ch = 1; ch <= 13; ch++) {
            if (ch == rtc_wifi_channel) continue;
            esp_now_deinit();
            initEspNow(ch);
            esp_now_send(rtc_hub_mac, (uint8_t *)&pkt, sizeof(pkt));
            uint32_t tch = millis();
            while (!ack_received && (millis() - tch < 60)) {
                delay(2);
            }
            if (ack_received) {
                rtc_wifi_channel = ch;
                break;
            }
        }
    }

    if (ack_received) {
        rtc_last_link_ok = true;
        applyAckConfiguration();
        return true;
    } else {
        rtc_last_link_ok = false;
        return false;
    }
}

// ============================================================================
// DISPLAY POWER GATING & MARQUEE RENDERING (128x64)
// ============================================================================
void powerOnDisplay() {
    if (display_active) return;
    pinMode(PIN_DISP_PWR, OUTPUT);
    digitalWrite(PIN_DISP_PWR, HIGH);
    delay(25); // Allow display boost cap to stabilize
    Wire.begin(PIN_I2C_SDA, PIN_I2C_SCL);
    u8g2.begin();
    u8g2.setContrast(200);
    display_active = true;
    scroll_x_sensor = 0;
    scroll_x_hub = 0;
    last_scroll_ms = millis();
}

void powerOffDisplay() {
    if (!display_active) return;
    u8g2.setPowerSave(1);
    Wire.end();
    // Prevent parasitic leakage through I2C lines into unpowered display
    pinMode(PIN_I2C_SDA, INPUT);
    pinMode(PIN_I2C_SCL, INPUT);
    digitalWrite(PIN_DISP_PWR, LOW);
    pinMode(PIN_DISP_PWR, OUTPUT);
    display_active = false;
}

void renderDisplayFrame(bool pairing_mode, uint8_t remaining_sec) {
    if (!display_active) return;

    uint32_t now = millis();
    if (now - last_scroll_ms >= 90) {
        last_scroll_ms = now;
        scroll_x_sensor += 2;
        scroll_x_hub += 2;
    }

    u8g2.clearBuffer();
    u8g2.setFont(u8g2_font_6x10_tf);

    // ------------------------------------------------------------------------
    // LINE 1 (y=10): Fixed #ID + Status Badge + Marquee Hub Name
    // ------------------------------------------------------------------------
    char id_buf[6];
    if (rtc_is_paired && rtc_sensor_id <= 15) {
        snprintf(id_buf, sizeof(id_buf), "#%02u", rtc_sensor_id);
    } else {
        snprintf(id_buf, sizeof(id_buf), "#--");
    }
    u8g2.drawStr(0, 10, id_buf); // x = 0..17

    bool blink_on = ((now / 500) % 2) == 0;
    if (pairing_mode) {
        if (blink_on) {
            u8g2.drawStr(21, 10, "[PAIRING]");
        }
    } else if (rtc_last_link_ok) {
        u8g2.drawStr(21, 10, "[LINK OK]");
    } else {
        u8g2.drawStr(21, 10, "[NO LINK]");
    }

    // Hub Name Viewport: x = 78..127 (width = 50px, ~8 chars static, marquee if wider)
    u8g2.drawStr(77, 10, "->");
    int16_t hub_w = u8g2.getStrWidth(rtc_hub_name);
    const int16_t hub_vp_x = 90;
    const int16_t hub_vp_w = 38;
    u8g2.setClipWindow(hub_vp_x, 0, 127, 13);
    if (hub_w <= hub_vp_w) {
        u8g2.drawStr(hub_vp_x, 10, rtc_hub_name);
    } else {
        int16_t span = hub_w + 18;
        int16_t offset = scroll_x_hub % span;
        u8g2.drawStr(hub_vp_x - offset, 10, rtc_hub_name);
        u8g2.drawStr(hub_vp_x - offset + span, 10, rtc_hub_name);
    }
    u8g2.setMaxClipWindow();

    // ------------------------------------------------------------------------
    // LINE 2 (y=23): Custom Sensor Name (Full 128px width, marquee if > 21 chars)
    // ------------------------------------------------------------------------
    int16_t sname_w = u8g2.getStrWidth(rtc_sensor_name);
    if (sname_w <= 128) {
        u8g2.drawStr(0, 23, rtc_sensor_name);
    } else {
        int16_t span = sname_w + 24;
        int16_t offset = scroll_x_sensor % span;
        u8g2.drawStr(-offset, 23, rtc_sensor_name);
        u8g2.drawStr(-offset + span, 23, rtc_sensor_name);
    }

    u8g2.drawHLine(0, 26, 128);

    // ------------------------------------------------------------------------
    // LINE 3 (y=47): Large Temperature & Humidity (Always visible, even in Pairing)
    // ------------------------------------------------------------------------
    char temp_str[16];
    snprintf(temp_str, sizeof(temp_str), "%.1fC", cur_temp_c);
    u8g2.setFont(u8g2_font_logisoso18_tf);
    u8g2.drawStr(0, 48, temp_str);

    char hum_str[16];
    snprintf(hum_str, sizeof(hum_str), "RH:%.0f%%", cur_hum_rh);
    u8g2.setFont(u8g2_font_7x14_tf);
    u8g2.drawStr(80, 46, hum_str);

    u8g2.drawHLine(0, 52, 128);

    // ------------------------------------------------------------------------
    // LINE 4 (y=63): Battery Status & Countdown Timer
    // ------------------------------------------------------------------------
    u8g2.setFont(u8g2_font_6x10_tf);
    char bat_str[22];
    snprintf(bat_str, sizeof(bat_str), "BAT:%u%% %.2fV", cur_bat_pct, cur_vbat_mv / 1000.0f);
    u8g2.drawStr(0, 63, bat_str);

    char timer_str[14];
    if (pairing_mode) {
        snprintf(timer_str, sizeof(timer_str), "[WAIT:%2us]", remaining_sec);
    } else {
        snprintf(timer_str, sizeof(timer_str), "[OFF:%2us]", remaining_sec);
    }
    int16_t tw = u8g2.getStrWidth(timer_str);
    u8g2.drawStr(128 - tw, 63, timer_str);

    u8g2.sendBuffer();
}

// ============================================================================
// PAIRING MODE (60s Channel Sweep with Live Blinking [PAIRING] Display)
// ============================================================================
void runPairingMode() {
    powerOnDisplay();
    uint32_t start_ms = millis();
    uint8_t scan_ch = 1;
    uint32_t last_tx_ms = 0;

    espnow_sensor_packet_t pkt = {};
    pkt.msg_type     = MSG_PAIR_REQUEST;
    pkt.sensor_id    = rtc_is_paired ? rtc_sensor_id : 0;
    pkt.temp_c       = cur_temp_c;
    pkt.humidity_pct = cur_hum_rh;
    pkt.batt_mv      = cur_vbat_mv;
    pkt.batt_pct     = cur_bat_pct;
    pkt.wake_reason  = 1;

    ack_received = false;

    while (millis() - start_ms < PAIRING_DURATION_MS) {
        uint8_t rem_s = (uint8_t)((PAIRING_DURATION_MS - (millis() - start_ms)) / 1000UL);
        renderDisplayFrame(true, rem_s);

        if (millis() - last_tx_ms >= 350) {
            last_tx_ms = millis();
            esp_now_deinit();
            initEspNow(scan_ch);
            pkt.seq_num = ++rtc_seq_num;
            esp_now_send(BROADCAST_MAC, (uint8_t *)&pkt, sizeof(pkt));

            scan_ch++;
            if (scan_ch > 13) scan_ch = 1;
        }

        if (ack_received && last_ack.msg_type == MSG_PAIR_ACCEPT) {
            rtc_last_link_ok = true;
            applyAckConfiguration();
            break;
        }
        delay(20);
    }
}

// ============================================================================
// DEEP-SLEEP ENTRY (< 25 uA Quiescent Target)
// ============================================================================
void enterOptimizedDeepSleep() {
    powerOffDisplay();

    esp_now_deinit();
    WiFi.disconnect(true);
    WiFi.mode(WIFI_OFF);

    // Ensure button pin is cleanly pulled up for RTC GPIO wake
    pinMode(PIN_BTN_UNIFIED, INPUT_PULLUP);
    esp_deep_sleep_enable_gpio_wakeup(1ULL << PIN_BTN_UNIFIED, ESP_GPIO_WAKEUP_GPIO_LOW);
    esp_sleep_enable_timer_wakeup((uint64_t)rtc_report_interval_s * 1000000ULL);

    esp_sleep_pd_config(ESP_PD_DOMAIN_RTC_PERIPH, ESP_PD_OPTION_OFF);
    esp_deep_sleep_start();
}

// ============================================================================
// MAIN SETUP & INTERACTIVE DISPLAY LOOP
// ============================================================================
void setup() {
    // Immediately keep display power OFF on boot until needed
    pinMode(PIN_DISP_PWR, OUTPUT);
    digitalWrite(PIN_DISP_PWR, LOW);
    pinMode(PIN_BTN_UNIFIED, INPUT_PULLUP);

    if (!rtc_initialized) {
        loadConfigFromNVS();
    }

    esp_sleep_wakeup_cause_t wakeup_reason = esp_sleep_get_wakeup_cause();
    bool woke_by_button = (wakeup_reason == ESP_SLEEP_WAKEUP_GPIO) ||
                          (digitalRead(PIN_BTN_UNIFIED) == LOW);

    // 1. Always take a fresh sensor & battery reading first
    measureSensors();

    // 2. If woken by Timer (or cold boot without button), send telemetry and sleep immediately
    if (!woke_by_button) {
        sendTelemetryAndAwaitAck(0);
        enterOptimizedDeepSleep();
        return;
    }

    // 3. Woken by Unified Button (D1 / GPIO 3): Measure press duration
    uint32_t press_start = millis();
    bool long_press_triggered = false;

    // Turn on display immediately on press so user gets instant visual feedback
    powerOnDisplay();
    renderDisplayFrame(false, rtc_display_timeout_s);

    while (digitalRead(PIN_BTN_UNIFIED) == LOW) {
        if (millis() - press_start >= LONG_PRESS_MS) {
            long_press_triggered = true;
            break;
        }
        renderDisplayFrame(false, rtc_display_timeout_s);
        delay(20);
    }

    if (long_press_triggered) {
        // >= 2s Hold: Enter 60s Pairing Mode
        runPairingMode();
        // Wait for button release if still held
        while (digitalRead(PIN_BTN_UNIFIED) == LOW) {
            delay(10);
        }
    } else {
        // < 1s Short Press: Perform immediate sync & keep screen ON
        sendTelemetryAndAwaitAck(1);
    }

    // 4. Keep screen illuminated for rtc_display_timeout_s (5..10s)
    //    If user presses button again while screen is ON:
    //      - Short press -> Turn off screen IMMEDIATELY and sleep
    //      - Long press (>= 2s) -> Enter Pairing Mode
    uint32_t screen_start = millis();
    uint32_t timeout_ms   = (uint32_t)rtc_display_timeout_s * 1000UL;

    while (millis() - screen_start < timeout_ms) {
        uint8_t rem_s = (uint8_t)((timeout_ms - (millis() - screen_start) + 999) / 1000UL);
        renderDisplayFrame(false, rem_s);

        if (digitalRead(PIN_BTN_UNIFIED) == LOW) {
            delay(30); // Debounce
            if (digitalRead(PIN_BTN_UNIFIED) == LOW) {
                uint32_t p2_start = millis();
                bool p2_long = false;
                while (digitalRead(PIN_BTN_UNIFIED) == LOW) {
                    if (millis() - p2_start >= LONG_PRESS_MS) {
                        p2_long = true;
                        break;
                    }
                    delay(15);
                }
                if (p2_long) {
                    runPairingMode();
                    while (digitalRead(PIN_BTN_UNIFIED) == LOW) delay(10);
                    screen_start = millis(); // Reset view timeout after pairing
                } else {
                    // Immediate manual display OFF -> return to deep sleep
                    break;
                }
            }
        }
        delay(20);
    }

    enterOptimizedDeepSleep();
}

void loop() {
    // Never reached; all execution happens in setup() prior to deep sleep
}
```
