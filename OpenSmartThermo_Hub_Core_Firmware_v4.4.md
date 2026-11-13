# OpenSmartThermo v4.4 — Main Thermostat & Hub Core Firmware (`ESP32-S3`)

This document contains **Part 2A** of the **ESP32-S3 Main Thermostat & Hub** PlatformIO project, updated with **empty schedule slot defaults (`start_minute = -1`)** and **strict chronological schedule evaluation** (ignoring empty slots and out-of-order slots whose start time is earlier than or equal to a preceding valid slot).

---

## 1. `platformio.ini`

```ini
[env:esp32-s3-devkitc-1]
platform = espressif32 @ ^6.5.0
board = esp32-s3-devkitc-1
framework = arduino
monitor_speed = 115200
board_build.filesystem = littlefs
board_build.partitions = default_16MB.csv

build_flags =
    -DARDUINO_USB_MODE=1
    -DARDUINO_USB_CDC_ON_BOOT=1
    -DCORE_DEBUG_LEVEL=1

lib_deps =
    adafruit/Adafruit SHT4x Library @ ^1.0.4
    adafruit/Adafruit BusIO @ ^1.15.0
    adafruit/RTClib @ ^2.1.3
    olikraus/U8g2 @ ^2.35.9
    esphome/ESPAsyncWebServer-esphome @ ^3.1.0
    esphome/AsyncTCP-esphome @ ^2.1.3
    bblanchon/ArduinoJson @ ^7.0.4
    knolleary/PubSubClient @ ^2.8
```

---

## 2. `include/config.h`

```cpp
#pragma once
#include <Arduino.h>
#include <time.h>

// ============================================================================
// HARDWARE PINOUT (ESP32-S3-DevKitC-1)
// ============================================================================
#define PIN_I2C_SDA           8
#define PIN_I2C_SCL           9

// ST7567 128x64 SPI Graphic LCD
#define PIN_LCD_MOSI          11
#define PIN_LCD_SCLK          12
#define PIN_LCD_CS            10
#define PIN_LCD_DC            13
#define PIN_LCD_RST           14
#define PIN_LCD_BL            21   // PWM Backlight

// 4 Tactile Pushbuttons (Active LOW with internal pull-ups)
#define PIN_BTN_UP            4
#define PIN_BTN_DOWN          5
#define PIN_BTN_OK            6
#define PIN_BTN_BACK          7

// Dry-Contact Boiler Relay & External Hardware Watchdog
#define PIN_RELAY_BOILER      16   // Active HIGH (Opto-isolated relay module)
#define PIN_WDT_DONE          17   // TPL5010 DONE heartbeat pulse

// ============================================================================
// SYSTEM CONSTANTS & LIMITS
// ============================================================================
#define MAX_SENSORS           15
#define MAX_DAYS              7
#define MAX_SCHEDULE_SLOTS    6
#define SLOT_EMPTY_MINUTE     (-1) // Sentinel for empty/unconfigured slot
#define MIN_SETPOINT_C        5.0f
#define MAX_SETPOINT_C        35.0f
#define DEFAULT_ANTI_CYCLE_S  240  // 4-minute universal compressor/boiler protection

// ============================================================================
// ENUMS & DATA STRUCTURES
// ============================================================================
enum thermo_mode_t : uint8_t {
    MODE_AUTO   = 0,
    MODE_MANUAL = 1,
    MODE_BOOST  = 2,
    MODE_OFF    = 3
};

enum control_algo_t : uint8_t {
    ALGO_DYNAMIC_HYST = 0,
    ALGO_STATIC_HYST  = 1,
    ALGO_TPI          = 2
};

enum sensor_role_t : uint8_t {
    ROLE_DISABLED = 0,
    ROLE_INDOOR   = 1,
    ROLE_OUTDOOR  = 2
};

enum sensor_agg_t : uint8_t {
    AGG_SINGLE   = 0,
    AGG_AVERAGE  = 1,
    AGG_WEIGHTED = 2,
    AGG_MIN      = 3,
    AGG_MAX      = 4
};

// ESP-NOW Wireless Node Telemetry Packet (24 bytes)
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

// ESP-NOW Hub ACK + Piggybacked Config Sync (56 bytes)
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

struct SensorConfig {
    bool          paired;
    uint8_t       mac[6];
    char          name[25];
    sensor_role_t role;
    uint8_t       weight;            // 1..10 for AGG_WEIGHTED
    float         cal_offset_c;
    uint16_t      report_interval_s;
    uint8_t       display_timeout_s;
};

struct SensorRuntime {
    bool     online;
    float    temp_c;
    float    humidity_pct;
    uint16_t batt_mv;
    uint8_t  batt_pct;
    int8_t   rssi;
    uint32_t last_seen_epoch;
};

struct ScheduleSlot {
    int16_t start_minute; // -1 = empty (--:--), 0..1439 = exact minute of day
    float   target_temp;  // 5.0 .. 35.0 C
    int8_t  sensor_id;    // -1 = global default, 0 = hub onboard, 1..15 = wireless node
};

struct SystemConfig {
    char           hub_name[21];
    control_algo_t algorithm;
    float          static_hyst_c;       // e.g., 0.4 C (+/- 0.2 C)
    float          learned_overshoot_c; // EMA tracked post-cutoff overshoot
    uint8_t        tpi_cycles_per_hr;   // e.g., 3 cycles/hr
    uint16_t       min_on_time_s;       // >= 240s
    uint16_t       min_off_time_s;      // >= 240s
    sensor_agg_t   agg_mode;
    int8_t         primary_sensor_id;   // 0 = hub onboard, 1..15 = wireless node
    float          onboard_cal_c;
    bool           warm_lockout_en;
    float          warm_lockout_temp;   // e.g., 18.0 C outdoor
    bool           cold_comp_en;
    float          cold_comp_slope;     // e.g., 0.05 C boost per C below 5 C
    float          frost_protect_temp;  // e.g., 5.0 C
    bool           open_window_en;
    bool           pump_exercise_en;
    SensorConfig   sensors[MAX_SENSORS + 1]; // Index 0 = Hub onboard, 1..15 = Wireless
    ScheduleSlot   schedule[MAX_DAYS][MAX_SCHEDULE_SLOTS];
    char           mqtt_host[64];
    uint16_t       mqtt_port;
    char           mqtt_user[32];
    char           mqtt_pass[64];
    bool           mqtt_en;
    char           wifi_ssid[32];
    char           wifi_pass[64];
    char           hostname[32];
    char           mqtt_base_topic[32];
    bool           ha_discovery;    
};

struct SystemState {
    thermo_mode_t mode;
    float         manual_setpoint;
    float         active_setpoint;
    float         control_temp_c;
    float         outdoor_temp_c;
    bool          outdoor_valid;
    bool          heating_active;
    uint32_t      last_relay_change_epoch;
    uint32_t      boost_until_epoch;
    bool          pairing_active;
    uint32_t      pairing_until_epoch;
    bool          lockout_warm_weather;
    bool          lockout_open_window;
    bool          lockout_sensor_fault;
    uint32_t      last_heat_off_epoch;
    float         temp_at_heat_off;
    float         peak_post_heat_temp;
    bool          tracking_overshoot;
    SensorRuntime sensors[MAX_SENSORS + 1];
    bool          setup_ap_active;
    uint32_t      setup_ap_until_ms;
    int8_t        last_paired_id;
    char          lockout_reason[32];
};

extern SystemConfig g_config;
extern SystemState  g_state;

#include <freertos/semphr.h>
extern SemaphoreHandle_t g_mutex;

void saveConfigToNVS();
void loadConfigFromNVS();
```

---

## 3. `src/control_engine.h` & `src/control_engine.cpp`

### `src/control_engine.h`

```cpp
#pragma once
#include "config.h"

class ControlEngine {
public:
    void begin();
    void evaluate(uint32_t now_epoch, const struct tm& timeinfo);
    ScheduleSlot getActiveScheduleSlot(const struct tm& timeinfo, int* out_slot_idx = nullptr);
    void resetLearnedOvershoot();

private:
    float computeIndoorTemp(int8_t sensor_override);
    float computeOutdoorTemp(bool& valid);
    void  setRelayState(bool on, uint32_t now_epoch);
    void  updateOvershootLearning(float current_temp, uint32_t now_epoch);
};

extern ControlEngine g_control;
```

### `src/control_engine.cpp`

```cpp
#include "control_engine.h"
#include <math.h>

ControlEngine g_control;

void ControlEngine::begin() {
    pinMode(PIN_RELAY_BOILER, OUTPUT);
    digitalWrite(PIN_RELAY_BOILER, LOW);
    g_state.heating_active = false;
    g_state.last_relay_change_epoch = 0;
}

// Evaluates a single day's slots in chronological order (0..5).
// Ignores empty slots (start_minute < 0) and out-of-order slots
// (start_minute <= previous valid slot's start_minute).
static int findLatestValidSlotForDay(int day_idx, int max_minute, int* out_slot_idx = nullptr) {
    int16_t prev_valid_min = -1;
    int     chosen_idx     = -1;

    for (int s = 0; s < MAX_SCHEDULE_SLOTS; s++) {
        int16_t m = g_config.schedule[day_idx][s].start_minute;
        if (m < 0 || m > 1439) {
            continue; // Ignore empty/unconfigured slot
        }
        if (m <= prev_valid_min) {
            continue; // Ignore out-of-order slot earlier than or equal to preceding slot
        }
        prev_valid_min = m;
        if (m <= max_minute) {
            chosen_idx = s;
        }
    }
    if (out_slot_idx) {
        *out_slot_idx = chosen_idx;
    }
    return chosen_idx;
}

ScheduleSlot ControlEngine::getActiveScheduleSlot(const struct tm& timeinfo, int* out_slot_idx) {
    // Convert tm_wday (0=Sun..6=Sat) to 0=Mon..6=Sun
    int today = (timeinfo.tm_wday == 0) ? 6 : (timeinfo.tm_wday - 1);
    int now_min = timeinfo.tm_hour * 60 + timeinfo.tm_min;

    // 1. Search today's chronologically valid slots up to now_min
    int idx = findLatestValidSlotForDay(today, now_min, out_slot_idx);
    if (idx >= 0) {
        return g_config.schedule[today][idx];
    }

    // 2. Wrap backward up to 7 days to find the most recent active valid slot
    for (int offset = 1; offset <= MAX_DAYS; offset++) {
        int prev_day = (today - offset + MAX_DAYS) % MAX_DAYS;
        int prev_idx = findLatestValidSlotForDay(prev_day, 1439, nullptr);
        if (prev_idx >= 0) {
            if (out_slot_idx) *out_slot_idx = -1; // Inherited from previous day
            return g_config.schedule[prev_day][prev_idx];
        }
    }

    // 3. Fallback if all 42 weekly slots are completely empty
    if (out_slot_idx) *out_slot_idx = -1;
    ScheduleSlot fallback;
    fallback.start_minute = SLOT_EMPTY_MINUTE;
    fallback.target_temp  = g_state.manual_setpoint;
    fallback.sensor_id    = -1;
    return fallback;
}

float ControlEngine::computeIndoorTemp(int8_t sensor_override) {
    if (sensor_override >= 0 && sensor_override <= MAX_SENSORS) {
        if (g_state.sensors[sensor_override].online) {
            return g_state.sensors[sensor_override].temp_c;
        }
    }

    if (g_config.agg_mode == AGG_SINGLE) {
        int8_t id = g_config.primary_sensor_id;
        if (id >= 0 && id <= MAX_SENSORS && g_state.sensors[id].online) {
            return g_state.sensors[id].temp_c;
        }
        // Fail-safe fallback to onboard Hub sensor
        return g_state.sensors[0].temp_c;
    }

    float sum = 0.0f;
    float weight_sum = 0.0f;
    float min_t = 999.0f;
    float max_t = -999.0f;
    int count = 0;

    for (int i = 0; i <= MAX_SENSORS; i++) {
        if (!g_config.sensors[i].paired && i != 0) continue;
        if (g_config.sensors[i].role != ROLE_INDOOR) continue;
        if (!g_state.sensors[i].online) continue;

        float t = g_state.sensors[i].temp_c;
        float w = (g_config.agg_mode == AGG_WEIGHTED) ? max((uint8_t)1, g_config.sensors[i].weight) : 1.0f;

        sum += t * w;
        weight_sum += w;
        if (t < min_t) min_t = t;
        if (t > max_t) max_t = t;
        count++;
    }

    if (count == 0) {
        return g_state.sensors[0].temp_c;
    }

    switch (g_config.agg_mode) {
        case AGG_MIN:      return min_t;
        case AGG_MAX:      return max_t;
        case AGG_WEIGHTED: return sum / weight_sum;
        case AGG_AVERAGE:
        default:           return sum / (float)count;
    }
}

float ControlEngine::computeOutdoorTemp(bool& valid) {
    float sum = 0.0f;
    int count = 0;
    for (int i = 1; i <= MAX_SENSORS; i++) {
        if (g_config.sensors[i].paired &&
            g_config.sensors[i].role == ROLE_OUTDOOR &&
            g_state.sensors[i].online) {
            sum += g_state.sensors[i].temp_c;
            count++;
        }
    }
    if (count > 0) {
        valid = true;
        return sum / (float)count;
    }
    valid = false;
    return 0.0f;
}

void ControlEngine::setRelayState(bool on, uint32_t now_epoch) {
    if (on == g_state.heating_active) return;

    uint32_t elapsed = now_epoch - g_state.last_relay_change_epoch;
    uint16_t min_hold = g_state.heating_active ? g_config.min_on_time_s : g_config.min_off_time_s;
    if (min_hold < DEFAULT_ANTI_CYCLE_S) min_hold = DEFAULT_ANTI_CYCLE_S;

    if (g_state.last_relay_change_epoch != 0 && elapsed < min_hold) {
        return; // Enforce universal 240s anti-short-cycle protection
    }

    // Track post-cutoff overshoot when turning heat OFF
    if (g_state.heating_active && !on) {
        g_state.last_heat_off_epoch = now_epoch;
        g_state.temp_at_heat_off    = g_state.control_temp_c;
        g_state.peak_post_heat_temp = g_state.control_temp_c;
        g_state.tracking_overshoot  = true;
    } else if (!g_state.heating_active && on) {
        g_state.tracking_overshoot  = false;
    }

    g_state.heating_active = on;
    g_state.last_relay_change_epoch = now_epoch;
    digitalWrite(PIN_RELAY_BOILER, on ? HIGH : LOW);
}

void ControlEngine::updateOvershootLearning(float current_temp, uint32_t now_epoch) {
    if (!g_state.tracking_overshoot) return;

    if (current_temp > g_state.peak_post_heat_temp) {
        g_state.peak_post_heat_temp = current_temp;
    }

    // Evaluate 20 minutes (1200s) after boiler cutoff
    if ((now_epoch - g_state.last_heat_off_epoch) >= 1200) {
        float measured_overshoot = g_state.peak_post_heat_temp - g_state.temp_at_heat_off;
        if (measured_overshoot < 0.0f) measured_overshoot = 0.0f;
        if (measured_overshoot > 2.5f) measured_overshoot = 2.5f;

        // Exponential Moving Average (alpha = 0.25)
        g_config.learned_overshoot_c = (0.75f * g_config.learned_overshoot_c) + (0.25f * measured_overshoot);
        g_state.tracking_overshoot = false;
        saveConfigToNVS();
    }
}

void ControlEngine::resetLearnedOvershoot() {
    g_config.learned_overshoot_c = 0.20f;
    g_state.tracking_overshoot = false;
    saveConfigToNVS();
}

void ControlEngine::evaluate(uint32_t now_epoch, const struct tm& timeinfo) {
    // 1. Check sensor online timeouts (3x reporting interval + 60s grace)
    g_state.sensors[0].online = true;
    for (int i = 1; i <= MAX_SENSORS; i++) {
        if (!g_config.sensors[i].paired) {
            g_state.sensors[i].online = false;
            continue;
        }
        uint32_t timeout_s = (uint32_t)g_config.sensors[i].report_interval_s * 3 + 60;
        g_state.sensors[i].online = ((now_epoch - g_state.sensors[i].last_seen_epoch) <= timeout_s);
    }

    // 2. Check Boost expiration
    if (g_state.mode == MODE_BOOST && now_epoch >= g_state.boost_until_epoch) {
        g_state.mode = MODE_AUTO;
    }

    // 3. Determine active target setpoint & optional room sensor override
    float target = g_state.manual_setpoint;
    int8_t sensor_override = -1;

    if (g_state.mode == MODE_AUTO) {
        ScheduleSlot slot = getActiveScheduleSlot(timeinfo);
        target = slot.target_temp;
        sensor_override = slot.sensor_id;
    } else if (g_state.mode == MODE_BOOST) {
        target = constrain(g_state.manual_setpoint + 1.5f, MIN_SETPOINT_C, MAX_SETPOINT_C);
    } else if (g_state.mode == MODE_OFF) {
        target = g_config.frost_protect_temp;
    }

    // 4. Compute indoor & outdoor temperatures
    g_state.control_temp_c = computeIndoorTemp(sensor_override);
    g_state.outdoor_temp_c = computeOutdoorTemp(g_state.outdoor_valid);
    updateOvershootLearning(g_state.control_temp_c, now_epoch);

    // 5. Cold-Weather Compensation (only when not OFF)
    if (g_state.mode != MODE_OFF && g_config.cold_comp_en && g_state.outdoor_valid) {
        if (g_state.outdoor_temp_c < 5.0f) {
            target += (5.0f - g_state.outdoor_temp_c) * g_config.cold_comp_slope;
        }
    }
    target = constrain(target, MIN_SETPOINT_C, MAX_SETPOINT_C);
    g_state.active_setpoint = target;

    // 6. Warm-Weather Lockout
    g_state.lockout_warm_weather = false;
    if (g_state.mode != MODE_OFF && g_config.warm_lockout_en && g_state.outdoor_valid) {
        if (g_state.outdoor_temp_c >= g_config.warm_lockout_temp) {
            g_state.lockout_warm_weather = true;
            setRelayState(false, now_epoch);
            return;
        }
    }

    // 7. Hysteresis / TPI Heating Demand Calculation
    bool demand = g_state.heating_active;
    float half_band = g_config.static_hyst_c * 0.5f;

    if (g_config.algorithm == ALGO_DYNAMIC_HYST) {
        float cut_off = target - (g_config.learned_overshoot_c * 0.65f);
        float cut_on  = target - half_band;
        if (g_state.control_temp_c <= cut_on)  demand = true;
        if (g_state.control_temp_c >= cut_off) demand = false;
    } else if (g_config.algorithm == ALGO_TPI) {
        float error = target - g_state.control_temp_c;
        float duty = constrain(error / 1.5f, 0.0f, 1.0f);
        uint32_t period_s = 3600 / max((uint8_t)1, g_config.tpi_cycles_per_hr);
        uint32_t pos_in_cycle = now_epoch % period_s;
        demand = (pos_in_cycle < (uint32_t)(duty * period_s));
    } else {
        // Static Hysteresis
        if (g_state.control_temp_c <= (target - half_band)) demand = true;
        if (g_state.control_temp_c >= (target + half_band)) demand = false;
    }

    setRelayState(demand, now_epoch);
}
```

---

## 4. `src/lcd_ui.h` & `src/lcd_ui.cpp`

### `src/lcd_ui.h`

```cpp
#pragma once
#include "config.h"

class LcdUI {
public:
    void begin();
    void loop(uint32_t now_epoch, const struct tm& timeinfo);

private:
    uint8_t  current_screen = 0;
    uint32_t last_input_ms  = 0;
    uint8_t  sched_day_idx  = 0;
    void renderMainScreen(const struct tm& timeinfo);
    void renderScheduleScreen(const struct tm& timeinfo);
    void renderSensorsScreen();
    void renderNetworkScreen();
};

extern LcdUI g_lcd;
```

### `src/lcd_ui.cpp`

```cpp
#include "lcd_ui.h"
#include "control_engine.h"
#include <U8g2lib.h>
#include <WiFi.h>
#include "network_manager.h"

LcdUI g_lcd;
U8G2_ST7567_ENH_DG128064I_F_4W_HW_SPI u8g2(U8G2_R0, PIN_LCD_CS, PIN_LCD_DC, PIN_LCD_RST);

void LcdUI::begin() {
    pinMode(PIN_BTN_UP,   INPUT_PULLUP);
    pinMode(PIN_BTN_DOWN, INPUT_PULLUP);
    pinMode(PIN_BTN_OK,   INPUT_PULLUP);
    pinMode(PIN_BTN_BACK, INPUT_PULLUP);
    pinMode(PIN_LCD_BL,   OUTPUT);
    analogWrite(PIN_LCD_BL, 180);

    u8g2.begin();
    last_input_ms = millis();
}

void LcdUI::loop(uint32_t now_epoch, const struct tm& timeinfo) {
    static uint32_t last_btn_ms = 0;
    if (millis() - last_btn_ms > 180) {
        if (digitalRead(PIN_BTN_UP) == LOW) {
            last_btn_ms = millis();
            last_input_ms = millis();
            if (current_screen == 0) {
                g_state.manual_setpoint = constrain(g_state.manual_setpoint + 0.5f, MIN_SETPOINT_C, MAX_SETPOINT_C);
                g_state.mode = MODE_MANUAL;
                saveConfigToNVS();
            } else if (current_screen == 1) {
                sched_day_idx = (sched_day_idx + 1) % MAX_DAYS;
            }
        } else if (digitalRead(PIN_BTN_DOWN) == LOW) {
            last_btn_ms = millis();
            last_input_ms = millis();
            if (current_screen == 0) {
                g_state.manual_setpoint = constrain(g_state.manual_setpoint - 0.5f, MIN_SETPOINT_C, MAX_SETPOINT_C);
                g_state.mode = MODE_MANUAL;
                saveConfigToNVS();
            } else if (current_screen == 1) {
                sched_day_idx = (sched_day_idx + MAX_DAYS - 1) % MAX_DAYS;
            }
        } else if (digitalRead(PIN_BTN_OK) == LOW) {
            last_btn_ms = millis();
            last_input_ms = millis();
            if (current_screen == 0) {
                g_state.mode = (thermo_mode_t)((g_state.mode + 1) % 4);
                if (g_state.mode == MODE_BOOST) {
                    g_state.boost_until_epoch = now_epoch + 3600;
                }
                saveConfigToNVS();
            }
        } else if (digitalRead(PIN_BTN_BACK) == LOW) {
            uint32_t press_start = millis();
            bool long_pressed = false;
            while (digitalRead(PIN_BTN_BACK) == LOW) {
                if (!long_pressed && (millis() - press_start >= 3000)) {
                    long_pressed = true;
                    last_input_ms = millis();
                    if (g_state.setup_ap_active) {
                        networkStopSetupAP();
                    } else {
                        networkStartSetupAP(600); // 10 λεπτά
                    }
                }
                delay(15);
            }
            last_btn_ms = millis();
            last_input_ms = millis();
            if (!long_pressed) {
                current_screen = (current_screen + 1) % 4;
            }
        }
    }

    // Dim backlight after 20s idle
    analogWrite(PIN_LCD_BL, (millis() - last_input_ms > 20000) ? 20 : 200);

    u8g2.clearBuffer();
    switch (current_screen) {
        case 0: renderMainScreen(timeinfo);     break;
        case 1: renderScheduleScreen(timeinfo); break;
        case 2: renderSensorsScreen();          break;
        case 3: renderNetworkScreen();          break;
    }
    u8g2.sendBuffer();
}

void LcdUI::renderMainScreen(const struct tm& timeinfo) {
    u8g2.setFont(u8g2_font_6x10_tr);
    const char* mode_names[] = {"AUTO", "MANUAL", "BOOST", "OFF"};
    char header[32];
    if (g_state.setup_ap_active) {
        snprintf(header, sizeof(header), "AP: 192.168.4.1");
    } else {
        snprintf(header, sizeof(header), "%02d:%02d  [%s]", timeinfo.tm_hour, timeinfo.tm_min, mode_names[g_state.mode]);
    }
    u8g2.drawStr(0, 9, header);
    if (g_state.heating_active) u8g2.drawStr(104, 9, "HEAT");
    u8g2.drawHLine(0, 12, 128);

    u8g2.setFont(u8g2_font_logisoso24_tr);
    char temp_str[16];
    snprintf(temp_str, sizeof(temp_str), "%.1fC", g_state.control_temp_c);
    u8g2.drawStr(4, 42, temp_str);

    u8g2.setFont(u8g2_font_6x12_tr);
    char set_str[24];
    snprintf(set_str, sizeof(set_str), "Set: %.1fC", g_state.active_setpoint);
    u8g2.drawStr(4, 58, set_str);

    if (g_state.outdoor_valid) {
        char out_str[20];
        snprintf(out_str, sizeof(out_str), "Out:%.1fC", g_state.outdoor_temp_c);
        u8g2.drawStr(72, 58, out_str);
    }
}

void LcdUI::renderScheduleScreen(const struct tm& timeinfo) {
    const char* days[] = {"MON", "TUE", "WED", "THU", "FRI", "SAT", "SUN"};
    u8g2.setFont(u8g2_font_6x10_tr);
    char title[32];
    snprintf(title, sizeof(title), "SCHEDULE: %s (UP/DN)", days[sched_day_idx]);
    u8g2.drawStr(0, 9, title);
    u8g2.drawHLine(0, 11, 128);

    int16_t prev_valid_min = -1;
    for (int s = 0; s < MAX_SCHEDULE_SLOTS; s++) {
        int y = 20 + (s * 8);
        ScheduleSlot slot = g_config.schedule[sched_day_idx][s];
        char line[32];

        if (slot.start_minute < 0 || slot.start_minute > 1439) {
            snprintf(line, sizeof(line), "%d: --:--   --.-C", s + 1);
        } else if (slot.start_minute <= prev_valid_min) {
            snprintf(line, sizeof(line), "%d: %02d:%02d  [IGNORED]",
                     s + 1, slot.start_minute / 60, slot.start_minute % 60);
        } else {
            prev_valid_min = slot.start_minute;
            snprintf(line, sizeof(line), "%d: %02d:%02d   %.1fC",
                     s + 1, slot.start_minute / 60, slot.start_minute % 60, slot.target_temp);
        }
        u8g2.drawStr(0, y, line);
    }
}

void LcdUI::renderSensorsScreen() {
    u8g2.setFont(u8g2_font_6x10_tr);
    u8g2.drawStr(0, 9, "ACTIVE SENSORS");
    u8g2.drawHLine(0, 11, 128);

    int row = 0;
    for (int i = 0; i <= MAX_SENSORS && row < 5; i++) {
        if (i != 0 && !g_config.sensors[i].paired) continue;
        char line[32];
        snprintf(line, sizeof(line), "#%02d %-10.10s %.1fC",
                 i, g_config.sensors[i].name, g_state.sensors[i].temp_c);
        u8g2.drawStr(0, 21 + row * 10, line);
        row++;
    }
}

void LcdUI::renderNetworkScreen() {
    u8g2.setFont(u8g2_font_6x10_tr);
    u8g2.drawStr(0, 9, "NETWORK & SYSTEM");
    u8g2.drawHLine(0, 11, 128);

    char ip_str[32];
    snprintf(ip_str, sizeof(ip_str), "IP: %s", WiFi.localIP().toString().c_str());
    u8g2.drawStr(0, 23, ip_str);
    u8g2.drawStr(0, 35, "Host: opensmartthermo.local");

    char learn_str[32];
    snprintf(learn_str, sizeof(learn_str), "Overshoot: %.2fC", g_config.learned_overshoot_c);
    u8g2.drawStr(0, 47, learn_str);
}
```

---

## 5. `src/main.cpp`

```cpp
#include <Arduino.h>
#include <Wire.h>
#include <Preferences.h>
#include <Adafruit_SHT4x.h>
#include <RTClib.h>
#include "config.h"
#include "control_engine.h"
#include "lcd_ui.h"
#include "network_manager.h"

SystemConfig g_config;
SystemState  g_state;

SemaphoreHandle_t g_mutex = nullptr;

Preferences    prefs;
Adafruit_SHT4x sht4;
RTC_DS3231     rtc;
bool           rtc_ok = false;

void setDefaultConfig() {
    memset(&g_config, 0, sizeof(SystemConfig));
    strlcpy(g_config.hub_name, "Living Room Hub", sizeof(g_config.hub_name));
    g_config.algorithm         = ALGO_DYNAMIC_HYST;
    g_config.static_hyst_c     = 0.4f;
    g_config.learned_overshoot_c = 0.20f;
    g_config.tpi_cycles_per_hr = 3;
    g_config.min_on_time_s     = DEFAULT_ANTI_CYCLE_S;
    g_config.min_off_time_s    = DEFAULT_ANTI_CYCLE_S;
    g_config.agg_mode          = AGG_SINGLE;
    g_config.primary_sensor_id = 0;
    g_config.warm_lockout_en   = true;
    g_config.warm_lockout_temp = 18.0f;
    g_config.cold_comp_en      = false;
    g_config.cold_comp_slope   = 0.05f;
    g_config.frost_protect_temp = 5.0f;
    g_config.open_window_en    = true;
    g_config.pump_exercise_en  = true;
    g_config.mqtt_port         = 1883;

    // Hub onboard sensor (index 0)
    g_config.sensors[0].paired = true;
    strlcpy(g_config.sensors[0].name, "Hub Onboard", sizeof(g_config.sensors[0].name));
    g_config.sensors[0].role   = ROLE_INDOOR;
    g_config.sensors[0].weight = 5;

    // Initialize all 42 weekly schedule slots as EMPTY (-1 / --:--)
    for (int d = 0; d < MAX_DAYS; d++) {
        for (int s = 0; s < MAX_SCHEDULE_SLOTS; s++) {
            g_config.schedule[d][s].start_minute = SLOT_EMPTY_MINUTE;
            g_config.schedule[d][s].target_temp  = 20.0f;
            g_config.schedule[d][s].sensor_id    = -1;
        }
    }
}

void saveConfigToNVS() {
    prefs.begin("thermo_cfg", false);
    prefs.putBytes("cfg_blob", &g_config, sizeof(SystemConfig));
    prefs.putUChar("mode", (uint8_t)g_state.mode);
    prefs.putFloat("man_set", g_state.manual_setpoint);
    prefs.end();
}

void loadConfigFromNVS() {
    prefs.begin("thermo_cfg", true);
    size_t len = prefs.getBytesLength("cfg_blob");
    if (len == sizeof(SystemConfig)) {
        prefs.getBytes("cfg_blob", &g_config, sizeof(SystemConfig));
    } else {
        setDefaultConfig();
    }
    g_state.mode = (thermo_mode_t)prefs.getUChar("mode", MODE_AUTO);
    g_state.manual_setpoint = prefs.getFloat("man_set", 21.0f);
    prefs.end();
}

void petExternalWatchdog() {
    digitalWrite(PIN_WDT_DONE, HIGH);
    delayMicroseconds(10);
    digitalWrite(PIN_WDT_DONE, LOW);
}

void setup() {
    Serial.begin(115200);
    pinMode(PIN_WDT_DONE, OUTPUT);
    petExternalWatchdog();

    memset(&g_state, 0, sizeof(SystemState));
    loadConfigFromNVS();

    Wire.begin(PIN_I2C_SDA, PIN_I2C_SCL);
    if (sht4.begin()) {
        sht4.setPrecision(SHT4X_HIGH_PRECISION);
    }
    rtc_ok = rtc.begin();

    g_mutex = xSemaphoreCreateMutex();
    g_control.begin();
    g_lcd.begin();
    networkInit();
}

void loop() {
    static uint32_t last_1s_ms = 0;
    uint32_t now_ms = millis();

    // Read current time from DS3231 or system NTP clock
    struct tm timeinfo;
    uint32_t now_epoch = 0;
    if (rtc_ok) {
        DateTime dt = rtc.now();
        now_epoch = dt.unixtime();
        time_t t = (time_t)now_epoch;
        gmtime_r(&t, &timeinfo);
    } else {
        getLocalTime(&timeinfo, 10);
        now_epoch = (uint32_t)time(nullptr);
    }

    if (now_ms - last_1s_ms >= 1000) {
        last_1s_ms = now_ms;
        petExternalWatchdog();

        sensors_event_t hum, temp;
        if (sht4.getEvent(&hum, &temp)) {
            g_state.sensors[0].temp_c = temp.temperature + g_config.onboard_cal_c;
            g_state.sensors[0].humidity_pct = hum.relative_humidity;
            g_state.sensors[0].last_seen_epoch = now_epoch;
        }

        g_control.evaluate(now_epoch, timeinfo);
        networkLoop();
    }

    g_lcd.loop(now_epoch, timeinfo);
    delay(20);
}
```
