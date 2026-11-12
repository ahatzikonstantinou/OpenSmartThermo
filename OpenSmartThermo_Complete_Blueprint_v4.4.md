# OpenSmartThermo — Complete System Architecture, Hardware, Firmware & User Manual (v4.3)

**Document Version:** 4.3 (Full Merged Reference Specification — Seeed Studio XIAO ESP32C3 Edition)  
**Target Platform:** ESP32-S3 (Thermostat & Integrated Hub) + Seeed Studio XIAO ESP32C3 (Ultra-Low-Power Wireless Sensors)  
**Communication Protocols:** Wi-Fi (IEEE 802.11 b/g/n), ESP-NOW (Peer-to-Peer 2.4 GHz), HTTP/REST, WebSockets, MQTT (with Home Assistant Auto-Discovery)  
**Purpose:** Self-contained engineering blueprint enabling full end-to-end reconstruction of hardware, embedded firmware, web application, APIs, and end-user documentation from scratch without SMD soldering.

---

## Table of Contents
1. [System Overview & Dry Contact Theory](#1-system-overview--dry-contact-theory)
2. [System Architecture & Summer Continuous Operation](#2-system-architecture--summer-continuous-operation)
3. [Hardware Design — Thermostat & Hub (ESP32-S3)](#3-hardware-design--thermostat--hub-esp32-s3)
4. [Hardware Design — Wireless Indoor & Outdoor Thermometers (Seeed Studio XIAO ESP32C3)](#4-hardware-design--wireless-indoor--outdoor-thermometers-seeed-studio-xiao-esp32c3)
5. [Heating Control Algorithms, Dynamic Hysteresis & Safety Protections](#5-heating-control-algorithms-dynamic-hysteresis--safety-protections)
6. [Weekly Scheduling Engine & Multi-Sensor Aggregation](#6-weekly-scheduling-engine--multi-sensor-aggregation)
7. [Firmware Architecture & Data Structures](#7-firmware-architecture--data-structures)
8. [Web Interface (SPA) Complete Specification](#8-web-interface-spa-complete-specification)
9. [REST API Complete Reference (JSON Request/Response Payloads)](#9-rest-api-complete-reference-json-requestresponse-payloads)
10. [MQTT API & Home Assistant Auto-Discovery Specification](#10-mqtt-api--home-assistant-auto-discovery-specification)
11. [128x64 Graphic LCD Screen Map & 4-Button Navigation Specification](#11-128x64-graphic-lcd-screen-map--4-button-navigation-specification)
12. [Complete End-User Manual (Hardware, LCD Screens, Web UI & Troubleshooting)](#12-complete-end-user-manual-hardware-lcd-screens-web-ui--troubleshooting)
13. [Hardware Construction & Solder-Free Assembly Guide (Step-by-Step)](#13-hardware-construction--solder-free-assembly-guide-step-by-step)


---

## 1. System Overview & Dry Contact Theory

### 1.1 What is a Dry Contact (Voltage-Free / Potential-Free Contact)?
A **dry contact** (also called a *voltage-free* or *potential-free* contact) is a mechanical relay switch whose output terminals (`COM`, `NO`, `NC`) are **galvanically isolated** from the thermostat's internal power supply (`230V AC` mains or `5V DC` logic).

* **Zero Voltage Injection:** The thermostat neither outputs `230V AC` nor `24V DC` onto the boiler control wires. Instead, it acts strictly as a clean physical switch bridging two wires provided by the heating appliance.
* **Universal Compatibility:**
  * **Modern Modulating Gas Boilers / Heat Pumps:** Typically use a `24V DC` or `3.3V–12V DC` low-voltage control loop (`TA` / `RT` terminals). Injecting `230V AC` into these terminals permanently destroys the boiler's main PCB. A dry contact safely closes this low-voltage loop.
  * **Zone Motorized Valves / Circulation Pumps:** Typically switch `230V AC` phase (`L`) to open a zone valve or energize a pump contactor. The dry contact handles up to `250V AC / 5A` directly.
* **Terminal Definitions:**
  * **`COM` (Common):** Input terminal for the boiler control loop wire.
  * **`NO` (Normally Open):** Open circuit when heating is `OFF` or power is lost (fail-safe); shorts to `COM` when heating is `ON`.
  * **`NC` (Normally Closed):** Shorted to `COM` when heating is `OFF`; opens when heating is `ON` (used for specialized cooling or normally-open thermal actuators).

### 1.2 Core System Capabilities
* **Integrated Hub Architecture:** The wall thermostat (`ESP32-S3`) simultaneously operates as a Wi-Fi client/AP and an **ESP-NOW wireless hub**, eliminating the need for an external gateway or cloud server.
* **2+ Year Battery Life on Wireless Nodes:** Wireless indoor/outdoor thermometers (`Seeed Studio XIAO ESP32C3` + `SHT40`) remain in deep sleep (`~15–25 µA` via direct `3V3` pin rail) and wake every 5 minutes (or on button press) to transmit a 2ms ESP-NOW packet, achieving **2.5 to 5+ years** on `2x AA` alkaline/lithium batteries.
* **Continuous Year-Round Sensor Access (Summer Mode):** Setting the thermostat mode to `OFF` disables only the heating relay (except emergency frost protection `< 6.0°C`). The ESP-NOW hub, Wi-Fi, LCD, Web UI, REST API, and MQTT telemetry remain **100% active 24/7/365**, allowing continuous monitoring of indoor and outdoor temperatures during summer.
* **Custom Sensor Naming & Role Assignment:** Up to **15 wireless sensors** plus **1 onboard sensor** (16 total) can be assigned custom 24-character names (e.g., `"Living Room"`, `"Master Bedroom"`, `"North Balcony"`) and roles (`indoor`, `outdoor`, `disabled`).
* **Advanced Heating Control:** Supports **Static Hysteresis**, **Dynamic Self-Learning Hysteresis** (automatic overshoot compensation via Exponential Moving Average), and **TPI (Time Proportional & Integral)** control—all protected by a universal configurable **Anti-Short-Cycle Delay** (default: `240s` / 4 minutes).

---

## 2. System Architecture & Summer Continuous Operation

### 2.1 Topology Diagram
```
+-----------------------------------------------------------------------------------+
|                              HOME LOCAL NETWORK (LAN)                             |
|   [ Smartphone / PC Browser ] <---> HTTP / WebSockets (Port 80)                   |
|   [ Home Assistant / Node-RED ] <---> REST API / MQTT Broker (Port 1883)          |
+-----------------------------------------+-----------------------------------------+
                                          | Wi-Fi (2.4 GHz, Locked Channel CH_X)
                                          v
+-----------------------------------------------------------------------------------+
|                   OPENSMARTTHERMO WALL UNIT (ESP32-S3-WROOM-1)                    |
|  +-----------------------+  +-----------------------+  +-----------------------+  |
|  | Web Server & REST API |  |   Heating Engine      |  |    ESP-NOW Hub RX/TX  |  |
|  | (AsyncWebServer + WS) |  | (DynHyst / TPI / WDT) |  |  (Zero-Router Mesh)   |  |
|  +-----------------------+  +-----------+-----------+  +-----------+-----------+  |
|  |  128x64 ST7567 LCD    |              |                          ^              |
|  |  4 Tactile Buttons    |     +--------v--------+                 |              |
|  |  DS3231 RTC + CR2032  |     |  5V Non-Latch   |                 |              |
|  |  SHT40 Onboard Sensor |     | Dry Contact RLY |                 |              |
|  +-----------------------+     +--------+--------+                 |              |
+-----------------------------------------|--------------------------|--------------+
                                          | COM / NO                 | ESP-NOW
                                          v                          | (2.4 GHz, CH_X)
                             [ Boiler / Heat Pump ]      +-----------+-----------+
                                                         |                       |
                                              +----------v---------+  +----------v---------+
                                              | Wireless Node #1..N|  | Outdoor Node       |
                                              | (XIAO ESP32C3+SHT40|  | (XIAO ESP32C3+SHT40|
                                              | Role: Indoor       |  | Role: Outdoor      |
                                              | 2x AA (2-5+ Yrs)   |  | IP65 Enclosure     |
                                              +--------------------+  +--------------------+
```

### 2.2 Wi-Fi Channel Synchronization Mechanism
ESP32 hardware shares a single 2.4 GHz radio between Wi-Fi and ESP-NOW. Therefore, both the thermostat and all wireless nodes must transmit on the exact same Wi-Fi channel (`1–13`):
1. When the thermostat connects to the home Wi-Fi router, it reads the router's assigned channel (`WiFi.channel()`, e.g., Channel 6).
2. During **Pairing Mode**, the thermostat broadcasts beacon/ACK packets containing its MAC address and active `wifi_channel`.
3. The wireless node saves `hub_mac` and `wifi_channel` into **RTC Slow Memory** (preserved across deep sleep) and **NVS Flash** (preserved across battery changes).
4. If the home router later changes its Wi-Fi channel, a wireless node that fails to receive an ESP-NOW ACK after 3 retries performs a fast channel scan (`1–13`), rediscovers the thermostat, updates its stored `wifi_channel`, and resumes normal operation automatically.

---

## 3. Hardware Design — Thermostat & Hub (ESP32-S3)

### 3.1 Bill of Materials (BOM)
| Component | Part Number / Specification | Purpose & Engineering Notes |
| :--- | :--- | :--- |
| **MCU Module** | `ESP32-S3-WROOM-1-N16R8` | Dual-core 240 MHz, 16MB Flash (LittleFS + OTA), 8MB PSRAM, Wi-Fi + ESP-NOW. |
| **Graphic LCD** | `128x64 SPI Monochrome LCD (ST7567)` | Zero burn-in, daylight readable with backlight off, software-controlled PWM backlight. |
| **Onboard Temp/RH** | `Sensirion SHT40-AD1B` (or `SHT41`) | `±0.2°C` accuracy, I2C (`0x44`), mounted on thermally isolated PCB tongue. |
| **Real-Time Clock** | `Maxim DS3231MZ+` + `CR2032` holder | Temperature-compensated RTC (`±2 ppm`); keeps weekly schedule accurate during Wi-Fi/power outages. |
| **Dry Contact Relay** | `Omron G5LE-1-VD 5DC` (SPDT, 5A/250VAC) | Monostable (non-latching) relay ensures fail-safe `NO` open state on power/MCU failure. |
| **Relay Driver** | `2N7002` N-MOSFET + `1N4148` flyback diode | Isolates GPIO from inductive coil kickback; `10kΩ` pull-down ensures relay stays OFF during boot. |
| **External Watchdog** | `TI TPL5010DDCR` | Hardware watchdog timer; resets `ESP32-S3 EN` pin if firmware hangs for `> 60 seconds`. |
| **Tactile Buttons** | `4x 6x6mm Soft-Touch Tactile Switches` | Dedicated `UP`, `DOWN`, `OK`, `BACK` buttons with `100nF` hardware debounce capacitors. |
| **AC/DC Power Supply** | `Hi-Link HLK-PM01` (230V AC to 5V DC, 3W) | Isolated step-down module + `AMS1117-3.3` LDO + `470µF` bulk capacitor + `2A` slow-blow fuse + `10D471K` MOV varistor. |
| **Terminals** | `5.08mm Pitch Screw Terminals (2P + 3P)` | `2P` for `L / N` (100–240V AC input); `3P` for `COM / NO / NC` dry contact output. |

### 3.2 Complete GPIO Pinout (ESP32-S3)
| GPIO Pin | Signal Name | Direction | Connected Peripheral / Function |
| :--- | :--- | :--- | :--- |
| `GPIO 4` | `I2C_SDA` | Bi-dir | Shared I2C Bus (`SHT40` @ `0x44` + `DS3231` @ `0x68`), `4.7kΩ` pull-ups to `3.3V` |
| `GPIO 5` | `I2C_SCL` | Output | Shared I2C Clock (`400 kHz`), `4.7kΩ` pull-up to `3.3V` |
| `GPIO 10` | `LCD_CS` | Output | ST7567 Graphic LCD Chip Select |
| `GPIO 11` | `LCD_MOSI` | Output | ST7567 SPI Data (`SDA`) |
| `GPIO 12` | `LCD_SCK` | Output | ST7567 SPI Clock (`SCK`) |
| `GPIO 13` | `LCD_DC` | Output | ST7567 Data/Command (`A0` / `RS`) |
| `GPIO 14` | `LCD_RST` | Output | ST7567 Hardware Reset |
| `GPIO 15` | `LCD_BL` | Output (PWM) | ST7567 LED Backlight control (`LEDC` PWM channel 0) |
| `GPIO 16` | `RELAY_CTRL` | Output | Dry Contact Relay Gate (`10kΩ` pull-down to `GND`; active HIGH) |
| `GPIO 17` | `WDT_DONE` | Output | Pulse HIGH for `100µs` every 10s to pet external `TPL5010` watchdog |
| `GPIO 35` | `BTN_UP` | Input | Tactile Button `UP` (`INPUT_PULLUP`, active LOW) |
| `GPIO 36` | `BTN_DOWN` | Input | Tactile Button `DOWN` (`INPUT_PULLUP`, active LOW) |
| `GPIO 37` | `BTN_OK` | Input | Tactile Button `OK` (`INPUT_PULLUP`, active LOW) |
| `GPIO 38` | `BTN_BACK` | Input | Tactile Button `BACK` (`INPUT_PULLUP`, active LOW) |

### 3.3 PCB Thermal Isolation Guidelines
* **Bottom-Corner Sensor Placement:** Place `SHT40` in the bottom-left corner of the PCB next to bottom enclosure intake vents.
* **Milled Thermal Cutout Slots:** Route `1.5mm` wide U-shaped routing slots around three sides of the `SHT40` footprint so heat from the `HLK-PM01` power supply and `ESP32-S3` (placed at the top of the enclosure near top exhaust vents) cannot conduct through copper planes.
* **Calibration Offset:** Firmware provides a user-configurable `temp_offset` (`-5.0°C` to `+5.0°C`, default `-0.4°C`) per sensor.

---

## 4. Hardware Design — Wireless Indoor & Outdoor Thermometers (Seeed Studio XIAO ESP32C3)

### 4.1 Solder-Free Module Selection & Power Budget (2+ Year Battery Life)
To eliminate SMD soldering and custom PCB fabrication while maintaining ultra-low quiescent current, the wireless sensor nodes use the **Seeed Studio XIAO ESP32C3** (available with pre-soldered 2.54mm through-hole headers, breadboard/protoboard compatible, and containing zero USB-UART bridge chips or always-on power LEDs).

* **Battery Source:** `2x AA Energizer Ultimate Lithium (L91)` (`3000 mAh`, `1.5V` nominal, `-40°C` to `+60°C` operating range — ideal for both indoor and outdoor nodes) OR `2x AA Alkaline` (`2500 mAh`).
* **Direct `3V3` Pin Rail (Bypassing Onboard LDO):** Connect the `2x AA` battery pack positive (`+`, `2.4V–3.2V`) **directly to the XIAO `3V3` pin** and negative (`-`) to **`GND`**.
  * *Important Hardware Rule:* **Do NOT connect `2x AA` batteries to the bottom `BAT` pads** (those pads expect a `3.7V–4.2V` LiPo cell and route through the onboard LDO regulator, wasting quiescent current). Feeding `3.0V` directly into `3V3` powers both the `ESP32-C3` (`2.3V–3.6V` safe range) and `SHT40` (`1.08V–3.6V`) with zero LDO loss. *(Never exceed `3.6V` on the `3V3` pin; disconnect batteries before plugging in USB-C).*
* **Zero-Leakage Display Power Gating (`D3 / GPIO 5`):** The ultra-low-power I2C monochrome OLED/LCD (`0.96"` `128x64` SSD1306/ST7567 recommended, or `0.91"` `128x32`, drawing `~3–6 mA` when active) is **never connected to the permanent `3V3` rail**. Instead, its `VCC` pin is powered directly from `D3` (`GPIO 5`) configured as a digital push-pull output (or via a low-gate-threshold P-MOSFET switch for larger panels). During automatic 5-minute background telemetry cycles and throughout deep sleep, `GPIO 5` is driven `LOW` (`0V` + `gpio_hold_en`), guaranteeing **strictly `0.0 µA` display quiescent draw**.
* **Average Current Calculation (5-Minute Reporting Interval = 300s):**
  * Deep Sleep on XIAO ESP32C3 via `3V3` pin (`299.95s` @ `~18 µA`, Display gated OFF = `0.0 µA`): `5,399.1 µA·s`
  * Active Wake + SHT40 Single-Shot Measurement + ESP-NOW TX/ACK (`50 ms` @ `85 mA`, Display remains OFF during background sync): `4,250.0 µA·s`
  * Occasional User Screen Inspection (e.g., 2 presses/day × `8s` @ `~8 mA` total): `~1.5 µA` amortized daily average
  * Total Charge per 300s Cycle: `9,649.1 µA·s` -> **Average Current = `~33.7 µA`**
  * **Battery Life (`2500 mAh` Alkaline @ 85% usable capacity):** `2125 mAh / 0.0337 mA = 63,056 hours` = **7.2 years theoretical (~3.0 to 5.0 years real-world accounting for self-discharge and cold weather)**.
  * *(Optional USB-C Mains Powered Mode: Connecting 5V USB-C powers the board continuously without batteries).*

### 4.2 Bill of Materials, Pinout & On-Node Display Layout (Seeed Studio XIAO ESP32C3 Node)
| Component | Specification | Notes |
| :--- | :--- | :--- |
| **MCU Module** | `Seeed Studio XIAO ESP32C3` (Pre-Soldered) | No SMD soldering required; attach external U.FL antenna included in box for maximum range. |
| **Temp/RH Sensor** | `Sensirion SHT40 Breakout Board` (I2C `0x44`) | Powered from permanent `3V3`; includes onboard `4.7kΩ` I2C pull-ups; `±0.2°C` accuracy, `0.4 µA` idle current. |
| **Ultra-Low-Power Display** | `0.96" (128x64) I2C Monochrome OLED/LCD` (`0x3C`) | Recommended `128x64` (also supports `0.91" 128x32`); `VCC` connected to `D3` (`GPIO 5`) for `0.0 µA` deep-sleep hardware power gating; shares `D4`/`D5` I2C bus. |
| **Multi-Function Button** | `1x 2-Pin Tactile Pushbutton (SCREEN / SYNC / PAIR)` | Connected between `D1` (`GPIO 3`, RTC wake-up capable) and `GND`. Replaces separate LED & second button. |
| **Battery Monitor** | `2x 1MΩ (1%)` voltage divider + `100nF` cap | Connected `3V3` -> `1MΩ` -> `D0 (GPIO 2)` -> `1MΩ` -> `GND` (`1.5 µA` constant draw). |
| **Enclosure** | Ventilated Wall Case (Indoor) / IP65 Radiation Shield (Outdoor) | Use PTFE breathable membrane over SHT40 and clear viewing window over OLED/LCD for outdoor installations. |

| XIAO Pin | ESP32-C3 GPIO | Signal Name | Function & Wiring |
| :--- | :--- | :--- | :--- |
| `3V3` | `3.3V Rail` | `VCC_BAT` | `2x AA` Battery `+` (`3.0V` nominal, max `3.6V`) & `SHT40 VIN` |
| `GND` | `GND` | `GND` | Common Ground (`Battery -`, `SHT40 GND`, `Display GND`, `Button Pin 2`) |
| `D0` | `GPIO 2` | `VBAT_ADC` | Battery voltage divider midpoint (`ADC1_CH2`) |
| `D1` | `GPIO 3` | `BTN_MAIN` | **Multi-Function Button (`SCREEN ON/OFF` / `SYNC` / `PAIR`)** — RTC GPIO wake-up (`INPUT_PULLUP`, active LOW) |
| `D2` | `GPIO 4` | `SPARE_IO` | Unassigned / Reserved for future expansion |
| `D3` | `GPIO 5` | `DISP_PWR` | **Display Hardware Power Gate** (`HIGH` = Display ON, `LOW` = `0.0 µA` Display OFF) |
| `D4` | `GPIO 6` | `I2C_SDA` | Shared I2C Data (`SHT40 SDA` + `Display SDA`) |
| `D5` | `GPIO 7` | `I2C_SCL` | Shared I2C Clock (`SHT40 SCL` + `Display SCL`) |

#### On-Node Ultra-Low-Power Display Layout & Auto-Scrolling Behavior (`0.96" 128x64`)
When woken by the user, the node asserts `D3 (GPIO 5) = HIGH`, initializes the I2C display (`~12ms`), and renders a structured 4-row layout (`128px` width = `21` characters per line using a crisp `6x8` pixel font for status/labels and large `16x24` digits for temperature):

```
+----------------------------------+
| #01 [LINK OK] ->Thermo           |  <- Row 1 (8px): Static Node ID + Link/Pairing Badge + Hub Name
| Master Bedroom                   |  <- Row 2 (10px): Custom Sensor Name (auto-scrolls if > 21 chars)
|   21.4°C           RH: 54.2%     |  <- Row 3 (24px): Live SHT40 Temperature & Relative Humidity
| BAT: 94% (3.04V)        [OFF: 8s]|  <- Row 4 (8px): Battery % & Voltage + Auto-Off Countdown
+----------------------------------+
```

**Pairing Mode Screen (Active while searching for Thermostat Hub):**
```
+----------------------------------+
| #-- [PAIRING]  <-- blinks 500ms  |  <- Row 1: Static ID + Blinking [PAIRING] indicator
| Unpaired Node                    |  <- Row 2: Current Node Name (auto-scrolls if > 21 chars)
|   21.4°C           RH: 54.2%     |  <- Row 3: Live Temperature & Humidity remain visible!
| BAT: 94% (3.04V)      [WAIT: 48s]|  <- Row 4: Battery % & Voltage + Remaining Pairing Window
+----------------------------------+
```

#### Field Sizing Recommendations & Automatic Horizontal Marquee Scrolling
Full-length names (up to **24 characters**) are supported in the Thermostat Web UI, REST API, and MQTT payloads without restriction. On the `128px`-wide node LCD/OLED, the following viewport widths apply:

| Display Element | Position | Recommended Static Width | Behavior When Longer Than Static Width |
| :--- | :--- | :--- | :--- |
| **Static Node ID** | Row 1, Left | **3 chars** (`#01`..`#15`, or `#--`) | Always pinned statically for instant hardware debugging. |
| **Link / Pairing Status** | Row 1, Center | **9 chars** (`[LINK OK]`, `[NO LINK]`, or blinking `[PAIRING]`) | Always pinned statically; `[PAIRING]` blinks every `500ms` during discovery. |
| **Thermostat Hub Name** | Row 1, Right | **Up to 6 chars** after `->` (e.g., `->Thermo`) | If `hub_name` exceeds `6 chars` (up to `24 chars`), its viewport smoothly **scrolls horizontally (marquee)** so the full thermostat name is readable. |
| **Thermometer Custom Name** | Row 2, Full Width | **Up to 21 chars** (e.g., `Master Bedroom`) | If `sensor_name` exceeds `21 chars` (up to `24 chars`), Row 2 smoothly **scrolls horizontally (marquee)** with a `1s` pause at start and end. |
| **Temp & Humidity** | Row 3, Full Width | Fixed format (`21.4°C  RH: 54.2%`) | Always remains visible on screen, including during Pairing Mode. |
| **Battery & Countdown** | Row 4, Full Width | Fixed format (`BAT: 94% (3.04V) [OFF: 8s]`) | Displays remaining battery (`%` & `V`) and live auto-off countdown (`s`). |

*(Note: If a smaller `0.91" 128x32` display is fitted instead of `0.96" 128x64`, firmware automatically alternates Row 1 every 2 seconds between `#01 Master Bedroom` (with marquee scroll if > 17 chars) and `[LINK OK] ->Thermo` so zero information is lost).*

### 4.3 Deep-Sleep Optimization Rules (Firmware Implementation)
To guarantee sub-`25 µA` sleep current and maximum battery longevity on the **Seeed Studio XIAO ESP32C3**, the sensor firmware must execute the following optimizations on every wake cycle:
1. **Never Initialize Full Wi-Fi Stack:** Call `WiFi.mode(WIFI_STA)` followed immediately by `esp_now_init()`. Never call `WiFi.begin()` during normal telemetry cycles so the radio skips router scanning and DHCP overhead (keeping total background wake duration `< 55 ms`).
2. **RTC Slow Memory State Caching:** Store `hub_mac`, `wifi_channel`, `sensor_id`, `sensor_name`, `hub_name`, `display_timeout_s`, `report_interval_sec`, `last_link_ok`, and `seq_num` in `RTC_DATA_ATTR` variables. Reading/writing NVS Flash is only performed on cold boot or when a new configuration (`sensor_name`, `hub_name`, `display_timeout_s`, `report_interval`, `temp_offset`) arrives in the Hub ACK packet.
3. **SHT40 Single-Shot High-Precision Mode:** Trigger a single measurement (`~8.2 ms`), read temperature/humidity, and let the `SHT40` automatically return to its `0.4 µA` sleep state.
4. **Hardware Display Power-Down & Single-GPIO RTC Wake-Up Sequence:** Before entering deep sleep, explicitly send the display sleep command, drive `D3 (GPIO 5)` `LOW` and latch it via `gpio_hold_en()` for `0.0 µA` leakage, shut down Wi-Fi/ESP-NOW, configure `GPIO 3` (`D1` — `BTN_MAIN`) for low-level RTC wake-up, and power down unused RTC peripherals:
   ```cpp
   // Exact Deep-Sleep Entry Sequence for Seeed Studio XIAO ESP32C3
   // 1. Cut power to I2C Display via D3 (GPIO 5) for strictly 0.0 uA display sleep current
   digitalWrite(5, LOW);
   gpio_hold_en((gpio_num_t)5);

   // 2. Shut down ESP-NOW & Wi-Fi radio
   esp_now_deinit();
   WiFi.disconnect(true);
   WiFi.mode(WIFI_OFF);

   // 3. Enable wake-up via Timer AND Single Multi-Function Button on D1 (GPIO 3)
   gpio_hold_en((gpio_num_t)3);
   esp_deep_sleep_enable_gpio_wakeup(1ULL << 3, ESP_GPIO_WAKEUP_GPIO_LOW);
   esp_sleep_enable_timer_wakeup((uint64_t)report_interval_sec * 1000000ULL);

   // 4. Power down unused RTC domains to minimize quiescent leakage
   esp_sleep_pd_config(ESP_PD_DOMAIN_RTC_PERIPH, ESP_PD_OPTION_OFF);
   esp_sleep_start();
   ```
5. **Brownout Voltage Configuration:** Set brownout detector threshold to `2.43V` (`CONFIG_ESP_BROWNOUT_DET_LVL_SEL_2`) so the sensor continues operating reliably until the `2x AA` batteries are nearly depleted (`1.2V` per cell).

### 4.4 Wireless Node Operating Modes (Single Multi-Function Button & Auto-Off Display)
All user interactions on the wireless thermometer are handled by a **single multi-function pushbutton (`D1 / GPIO 3`)** paired with the GPIO-gated I2C display (`D3 / GPIO 5`):

* **Automatic Timer Wake-Up (Silent Background Sync):** Every `report_interval_sec` (default `300s`), the node wakes silently with the display kept **powered OFF (`D3 = LOW`, `0.0 µA`)**, reads `SHT40` + battery voltage, transmits an ESP-NOW telemetry packet, updates cached `sensor_name`, `hub_name`, `display_timeout_s`, and `[LINK OK]` / `[NO LINK]` status from the Hub ACK, and returns immediately to deep sleep in `< 55 ms`.
* **Short Press (`< 1 second`) when Display is OFF (Screen ON + Instant Sync):**
  * Wakes the node from deep sleep and powers ON the I2C display (`D3 = HIGH`).
  * Measures fresh `SHT40` temperature/humidity and battery voltage, and immediately transmits an ESP-NOW sync packet to the thermostat to fetch any updated `sensor_name`, `hub_name`, or `display_timeout_s`.
  * Displays the static `#ID` (`#01`..`#15`), `[LINK OK]` / `[NO LINK]` status, connected thermostat `hub_name`, thermometer `sensor_name` (auto-scrolling if > 21 chars), temperature, humidity, and battery (`%` & `V`) for the configured **`display_timeout_s` (`5–10 seconds`, default `8s`)**.
  * When the countdown expires, the node automatically cuts display power (`D3 = LOW`, `0.0 µA`) and returns to deep sleep.
* **Short Press (`< 1 second`) while Display is Already ON (Instant Toggle-Off Override):**
  * Pressing the button again while the display is active **immediately turns the display OFF (`D3 = LOW`)** without waiting for `display_timeout_s` to expire, and returns the node straight to deep sleep to conserve battery.
* **Long Press (`>= 2 seconds`) from Any State (Pairing Mode):**
  * Enters **Pairing Broadcast Mode** for `60 seconds` with the display powered ON (`D3 = HIGH`).
  * Temperature and humidity remain visible in Row 3 while **`[PAIRING]` blinks every `500ms`** in Row 1 alongside a `60s` countdown in Row 4, broadcasting pairing requests across Wi-Fi channels `1–13`.
  * As soon as the thermostat accepts the pairing request, `[PAIRING]` stops blinking and changes to **`[LINK OK] -> <hub_name>`**, Row 1 updates with the newly assigned static `#ID`, Row 2 displays the synced `sensor_name`, and the display automatically powers OFF after `display_timeout_s` seconds.

---

## 5. Heating Control Algorithms, Dynamic Hysteresis & Safety Protections

### 5.1 Universal Anti-Short-Cycle Protection (Applies to ALL Control Algorithms)
To protect boiler heat exchangers, motorized zone valves, and heat pump compressors from destructive rapid cycling, **every heating control algorithm** passes its output through a universal **Anti-Short-Cycle State Guard**:
* **`min_on_time_sec` (Default & Recommended Minimum: `240` seconds / 4 minutes; Range: `60–900s`):** Once the relay turns `ON`, it cannot turn `OFF` until at least `240s` have elapsed (unless an emergency over-temperature or outdoor lockout occurs).
* **`min_off_time_sec` (Default & Recommended Minimum: `240` seconds / 4 minutes; Range: `60–900s`):** Once the relay turns `OFF`, it cannot turn `ON` again until at least `240s` have elapsed.
* **Why Universal:** Even in `TPI` mode, if low heat demand calculates a short pulse (e.g., 90 seconds), the Anti-Short-Cycle guard suppresses the pulse until cumulative demand reaches at least `min_on_time_sec`.

### 5.2 Supported Control Algorithms (`control_mode`)

#### Mode A: `hysteresis_static` (Standard Hysteresis with Configurable Time Delay — Factory Default)
* **Why Factory Default:** 100% compatible with every heating system ever made (older oil boilers, pellet stoves with 10-minute ignition cycles, standard gas boilers, and heat pumps).
* **Logic:**
  * Relay turns **ON** when `T_control <= (T_target - hysteresis_low)` (default `hysteresis_low = 0.3°C`).
  * Relay turns **OFF** when `T_control >= (T_target + hysteresis_high)` (default `hysteresis_high = 0.2°C`).
  * Governed by `min_on_time_sec = 240s` and `min_off_time_sec = 240s`.

#### Mode B: `hysteresis_dynamic` (Smart Self-Learning Hysteresis — Recommended for Radiators)
* **Problem Solved:** Cast-iron and aluminum radiators retain large amounts of hot water after the boiler shuts off, causing room temperature to continue rising (`overshoot` of `+0.5°C` to `+1.2°C`), wasting fuel and causing discomfort.
* **Self-Learning Mechanism:**
  1. **Turn-ON Threshold:** Relay turns `ON` at `T_control <= (T_target - dynamic_cutin_offset)` (default `0.25°C`).
  2. **Post-Cutoff Peak Tracking:** Whenever the relay transitions from `ON` to `OFF` at temperature `T_cutoff`, the firmware starts a **20-minute thermal inertia window**, tracking the maximum temperature reached (`T_peak`).
  3. **Overshoot Calculation & Validation:** If the heating cycle ran for at least `6 minutes` and `T_peak > T_cutoff`, the measured overshoot is `Overshoot_meas = clamp(T_peak - T_cutoff, 0.0, 2.0)`.
  4. **Exponential Moving Average (EMA):** The learned overshoot stored in NVS Flash is updated smoothly:
     `Overshoot_learned = (0.70 * Overshoot_learned) + (0.30 * Overshoot_meas)`
  5. **Predictive Early Cutoff:** On the next heating cycle, the relay turns `OFF` early at:
     `T_off_threshold = T_target - (0.80 * Overshoot_learned)`
     Residual radiator heat then coasts the room smoothly onto `T_target` (`±0.1°C`).
  6. **Reset Capability:** Users can clear learned inertia parameters anytime via Web UI (`Reset Learned Data`) or `POST /api/v1/settings/reset-learning`.

#### Mode C: `tpi` (Time Proportional & Integral Control — Highest Efficiency for Gas/Modulating Systems)
* **Logic:** Divides time into fixed cycles (`tpi_cycle_min`, default `15 minutes` = `900s`).
* Computes proportional error `E = T_target - T_control` plus integral trim `I` (eliminates steady-state droop), yielding a duty cycle `D` (`0%` to `100%`).
* `On_Duration = D * 900s`. If `0 < On_Duration < min_on_time_sec (240s)`, the pulse is skipped and integral error accumulates until the next cycle exceeds `240s`.

### 5.3 Outdoor Weather Rules (Requires a Sensor with `role: "outdoor"`)
1. **Warm-Weather Lockout (`outdoor_lockout_enabled`):**
   * If `T_outdoor >= outdoor_lockout_temp` (default `18.5°C`), heating is forcibly locked `OFF` (display shows `ECO LOCK`).
   * **Lockout Hysteresis (`1.0°C`):** Heating re-enables only when `T_outdoor <= (outdoor_lockout_temp - 1.0°C)` (e.g., `17.5°C`) to prevent boundary oscillation.
2. **Cold-Weather Compensation (`weather_comp_enabled`):**
   * When `T_outdoor < 5.0°C`, increases `T_target` slightly using `T_boost = (5.0 - T_outdoor) * weather_comp_slope` (default slope `0.05`, capped at `+1.0°C`) to counteract cold exterior wall radiation.

### 5.4 Mandatory Safety & Fail-Safe Protections
* **Hardware Watchdog (`TPL5010`):** Cuts power and resets `ESP32-S3` if main loop hangs for `> 60s`, immediately opening the `COM–NO` dry contact.
* **Wireless Sensor Timeout Fail-Safe (`sensor_timeout_min = 30`):** If an active indoor sensor stops reporting for `> 30 minutes`, the thermostat flags a `SENSOR LOST` alert on LCD/Web/MQTT and falls back to the onboard `SHT40` sensor (or turns `OFF` if onboard fallback is disabled).
* **Emergency Frost Protection (`frost_protect_temp = 6.0°C`):** Even in `OFF` mode or `Warm-Weather Lockout`, if indoor temperature drops below `6.0°C`, heating turns `ON` until temperature reaches `7.0°C`.
* **Open Window Detection (`open_window_enabled`):** If indoor temperature drops faster than `1.5°C` within `5 minutes`, heating pauses for `20 minutes` (`WINDOW PAUSE`).
* **Summer Pump / Valve Anti-Seize Exercise (`pump_exercise_enabled`):** If the relay has not turned `ON` for `14 consecutive days`, it energizes for `120 seconds` at `12:00 PM` on Sunday to prevent circulator pump seizure.

---

## 6. Weekly Scheduling Engine & Multi-Sensor Aggregation

### 6.1 Multi-Sensor Aggregation Modes (`sensor_agg_mode`)
Users can select how indoor control temperature `T_control` is computed from active `indoor` sensors:
1. **`single`:** Tracks one designated sensor (e.g., Onboard Sensor `#0` or Wireless Sensor `#2 "Bedroom"`).
2. **`average`:** Arithmetic mean of all included active indoor sensors.
3. **`weighted`:** Weighted average (`sum(T_i * w_i) / sum(w_i)`), allowing priority rooms (e.g., Living Room `weight=2`, Hallway `weight=1`).
4. **`min` (Coldest Room Priority):** Uses the lowest temperature among active indoor sensors—ensures no bedroom ever gets cold.
5. **`max` (Warmest Room Priority):** Uses the highest temperature among active indoor sensors—prevents overheating sun-facing rooms.

### 6.2 Weekly Schedule Structure (7 Days × 6 Time Slots)
* **Days:** `0 = Sunday, 1 = Monday, 2 = Tuesday, 3 = Wednesday, 4 = Thursday, 5 = Friday, 6 = Saturday`.
* **Slots per Day (`0–5`):** Each slot defines:
  * `enabled` (`bool`): Whether slot is active.
  * `time` (`"HH:MM"`): Start time in 24-hour format (e.g., `"06:30"`).
  * `target_temp` (`float`): Setpoint temperature (`5.0°C` to `30.0°C`, `0.5°C` step).
  * `agg_mode` (`string`, optional): Override aggregation mode per slot (e.g., use `"average"` during the day, switch to `"single"` on `"Master Bedroom"` at `"22:30"`).
  * `target_sensor_id` (`int`, optional): Sensor ID (`0–15`) used when `agg_mode == "single"`.

### 6.3 Operating Modes (`system_mode`)
* **`AUTO`:** Follows the active Weekly Schedule slot.
* **`MANUAL`:** Holds a fixed user-defined `manual_temp` indefinitely until changed.
* **`BOOST`:** Temporarily overrides setpoint to `boost_temp` (default `22.5°C`) for `boost_duration_min` (default `60 min`), then automatically reverts to the previous mode (`AUTO` or `MANUAL`).
* **`OFF`:** Heating disabled (except Frost Protection `< 6.0°C`). **Hub, Wi-Fi, LCD, Web UI, REST API, and MQTT remain 100% online.**

---

## 7. Firmware Architecture & Data Structures

### 7.1 Dual-Core FreeRTOS Task Assignment (`ESP32-S3`)
* **Core 0 (Protocol Core):** Wi-Fi stack, ESP-NOW ISR & RX/TX Queue, `ESPAsyncWebServer`, WebSocket broadcast task, `PubSubClient` MQTT loop, NTP time sync.
* **Core 1 (Control & UI Core):** `ControlLoopTask` (runs every `1000ms`), `SHT40` + `DS3231` I2C polling, `ST7567` LCD rendering (`U8g2`), 4-button debouncer & state machine, `TPL5010` hardware watchdog petter, NVS & LittleFS persistence.

### 7.2 ESP-NOW Binary Packet Definitions (`__attribute__((packed))`)
```cpp
// Telemetry Packet: Wireless Sensor (Seeed Studio XIAO ESP32C3) -> Thermostat Hub (ESP32-S3)
typedef struct __attribute__((packed)) {
    uint8_t  msg_type;        // 0x01 = TELEMETRY, 0x02 = PAIR_REQUEST
    uint8_t  sensor_id;       // 0xFF if unpaired, 1..15 if paired
    uint8_t  mac[6];          // Sender STA MAC address
    int16_t  temp_x100;       // Temperature in 0.01 C (e.g., 2145 = 21.45 C)
    uint16_t hum_x100;        // Humidity in 0.01 %RH (e.g., 5420 = 54.20 %)
    uint16_t vbat_mv;         // Battery voltage in mV (e.g., 3050 = 3.05 V)
    uint8_t  wake_reason;     // 0 = Timer, 1 = Physical Button Short Press, 2 = Long Press Pair
    uint32_t seq_num;         // Monotonic packet counter
} espnow_sensor_packet_t;

// ACK / Config Piggyback Packet: Thermostat Hub -> Wireless Sensor
typedef struct __attribute__((packed)) {
    uint8_t  msg_type;          // 0x81 = TELEMETRY_ACK, 0x82 = PAIR_ACCEPT
    uint8_t  assigned_id;       // Assigned Sensor ID (1..15)
    uint8_t  wifi_channel;      // Current Hub Wi-Fi channel (1..13)
    uint16_t report_interval;   // Sleep duration in seconds (default 300)
    int16_t  temp_offset_x100;  // Calibration offset in 0.01 C
    char     sensor_name[25];   // Null-terminated custom sensor name (up to 24 chars; auto-scrolls on node LCD if > 21 chars)
    char     hub_name[25];      // Null-terminated thermostat hub name (up to 24 chars; auto-scrolls on node LCD if > 6 chars)
    uint8_t  display_timeout_s; // On-node I2C screen auto-off timeout in seconds (5..10s, default 8s)
    uint8_t  ota_flag;          // 1 = Stay awake & connect Wi-Fi for OTA update
} espnow_hub_ack_t;
```

---

## 8. Web Interface (SPA) Complete Specification

Served directly from `ESP32-S3` LittleFS flash at `http://opensmartthermo.local` (mDNS) or `http://<device-ip>` with real-time **WebSocket (`/ws`)** live state updates (zero page reloads).

### 8.1 Web Application Pages & Features
1. **Page 1: Dashboard (`/`)**
   * Large circular/digital temperature gauge displaying active `T_control`, current `T_target`, active mode badge (`AUTO`, `MANUAL`, `BOOST`, `OFF`), and live flame/relay status indicator (`HEATING ON` / `IDLE`).
   * Quick mode selector pills (`AUTO`, `MANUAL`, `BOOST +1h`, `OFF`), `+0.5°C` / `-0.5°C` setpoint buttons, and active schedule slot banner.
   * **Live Sensor Cards Grid:** Shows every paired sensor's custom name (e.g., `"Master Bedroom"`, `"North Balcony"`), role badge (`INDOOR` / `OUTDOOR`), temperature, humidity, battery percentage, signal RSSI, and `"Last seen Xm ago"` timestamp.
   * **Outdoor Weather Banner:** Displays outdoor temperature and active weather rule badges (`WARM LOCKOUT ACTIVE` or `COLD BOOST +0.4°C`).
   * **24-Hour Telemetry Chart:** Interactive dual-axis chart (built with lightweight embedded canvas/SVG) showing indoor temp, outdoor temp, target temp, and boiler runtime bars.
2. **Page 2: Sensors & Pairing (`/sensors`)**
   * **`+ Pair New Sensor` Button:** Opens a 60-second pairing modal with live countdown timer and WebSocket listener. When a sensor's button is held for 2 seconds, it immediately appears in the modal with its assigned static `#ID` and an inline text input to enter its **Custom Name** and select its **Role** (`Indoor` / `Outdoor`).
   * **Global Wireless Node Display Timeout Setting:** Numeric selector (`5` to `10` seconds, default `8s`) controlling how long the wireless thermometers' I2C displays stay illuminated after a button press before automatically powering off (`0.0 µA`). Synced automatically to each node on its next wake-up or button press.
   * **Sensor Management Table:** Lists Onboard Sensor (`ID 0`) and Wireless Sensors (`ID 1–15`). Allows inline editing of:
     * **Custom Name** (up to 24 chars; automatically synced to the wireless thermometer's display via ESP-NOW ACK—names up to 21 chars display statically, while longer names scroll smoothly across the node's screen).
     * **Role** (`Indoor`, `Outdoor`, `Disabled`).
     * **Aggregation Weight** (`1–5`, used in `weighted` mode).
     * **Calibration Offset** (`-5.0°C` to `+5.0°C`).
     * **Report Interval** (`60s` to `1800s`, synced on next sensor wake-up).
     * **Delete / Unpair** button.
3. **Page 3: Weekly Schedule (`/schedule`)**
   * Day tabs (`Mon`, `Tue`, `Wed`, `Thu`, `Fri`, `Sat`, `Sun`) with 6 configurable time slots per day.
   * Each slot row has: `Enabled` toggle, `Time` picker (`HH:MM`), `Target Temp` (`°C`), and `Sensor Rule` (`Use Global Default` or override to `Single Sensor: <Custom Name>`).
   * **Bulk Copy Tools:** One-click buttons to `Copy Day to Mon–Fri`, `Copy Day to Sat–Sun`, or `Copy Day to All 7 Days`, plus `Save Schedule` button.
4. **Page 4: Heating Control & Algorithms (`/control`)**
   * **Sensor Aggregation Selector:** `Single Sensor` (dropdown of named indoor sensors), `Average`, `Weighted Average`, `Minimum (Coldest)`, `Maximum (Warmest)`.
   * **Control Algorithm Selector:**
     * `Static Hysteresis`: Configure `Hysteresis Low (°C)` and `Hysteresis High (°C)`.
     * `Dynamic Self-Learning Hysteresis`: Displays live **Learned Thermal Overshoot (`°C`)**, sample count, and **`Reset Learned Data`** button.
     * `TPI (Time Proportional & Integral)`: Configure `Cycle Time (10–30 min)`.
   * **Universal Anti-Short-Cycle Delay:** Configure `Min Boiler ON Time (sec)` (default `240`) and `Min Boiler OFF Time (sec)` (default `240`).
   * **Outdoor Weather Rules:** Toggle `Warm-Weather Lockout` + threshold slider (`10°C–25°C`), and toggle `Cold-Weather Compensation` + slope factor.
   * **Safety Toggles:** Frost Protection (`5°C–10°C`), Open Window Detection, Sensor Timeout Fallback, Summer Pump Exercise.
5. **Page 5: Network & Integrations (`/network`)**
   * **Wi-Fi Setup:** Scan nearby SSIDs, enter password, configure static IP or DHCP, set mDNS hostname.
   * **MQTT & Home Assistant:** Enable/disable MQTT, broker host/port, username/password, base topic, and toggle `Home Assistant Auto-Discovery`.
   * **System Time & RTC:** View `DS3231` hardware RTC time, NTP server, timezone POSIX string, and `Sync RTC Now` button.
6. **Page 6: System, Logs & Firmware (`/system`)**
   * Boiler runtime statistics (Today, This Week, This Month, Total Cycles).
   * System diagnostics (Uptime, Free Heap, Wi-Fi RSSI, Watchdog reset counter).
   * **Backup / Restore JSON Configuration** (`Export Config` / `Import Config`) and **OTA Firmware Update** (`.bin` upload).

---

## 9. REST API Complete Reference (JSON Request/Response Payloads)

Base URL: `http://<thermostat-ip>/api/v1` | Content-Type: `application/json`  
Optional Header: `X-API-Key: <token>` (if authentication is enabled in `/network`).

### 9.1 System Status & Mode Control

#### `GET /api/v1/status`
Returns complete live state of thermostat, relay, weather rules, and all sensors.
```json
{
  "device": {
    "hostname": "opensmartthermo",
    "uptime_sec": 482910,
    "wifi_rssi": -58,
    "wifi_channel": 6,
    "rtc_time": "2026-11-12T12:30:00+02:00",
    "pairing_active": false,
    "pairing_remaining_sec": 0
  },
  "thermostat": {
    "system_mode": "AUTO",
    "control_mode": "hysteresis_dynamic",
    "agg_mode": "average",
    "control_temp": 20.65,
    "target_temp": 21.0,
    "effective_target_temp": 21.2,
    "outdoor_temp": 11.4,
    "heating_active": true,
    "lockout_reason": "NONE",
    "learned_overshoot_c": 0.42,
    "min_on_remaining_sec": 145,
    "min_off_remaining_sec": 0,
    "active_schedule_slot": {
      "day": 4,
      "slot_index": 1,
      "time": "08:00",
      "target_temp": 21.0
    },
    "boost_remaining_sec": 0
  },
  "sensors": [
    {
      "id": 0,
      "name": "Hallway Wall",
      "role": "indoor",
      "type": "onboard",
      "temp": 20.5,
      "humidity": 52.1,
      "offset": -0.4,
      "weight": 1,
      "online": true,
      "last_seen_sec": 1
    },
    {
      "id": 1,
      "name": "Master Bedroom",
      "role": "indoor",
      "type": "wireless",
      "mac": "AA:BB:CC:11:22:33",
      "temp": 20.8,
      "humidity": 54.0,
      "offset": 0.0,
      "weight": 2,
      "battery_pct": 94,
      "vbat_mv": 3040,
      "rssi": -64,
      "online": true,
      "last_seen_sec": 112
    },
    {
      "id": 2,
      "name": "North Balcony",
      "role": "outdoor",
      "type": "wireless",
      "mac": "AA:BB:CC:44:55:66",
      "temp": 11.4,
      "humidity": 68.5,
      "offset": 0.0,
      "weight": 1,
      "battery_pct": 91,
      "vbat_mv": 2990,
      "rssi": -71,
      "online": true,
      "last_seen_sec": 85
    }
  ]
}
```

#### `POST /api/v1/control`
Sets operating mode (`AUTO`, `MANUAL`, `BOOST`, `OFF`) or manual/boost setpoint.
* **Request:**
```json
{
  "system_mode": "BOOST",
  "target_temp": 22.5,
  "boost_duration_min": 60
}
```
* **Response (`200 OK`):**
```json
{
  "status": "ok",
  "system_mode": "BOOST",
  "target_temp": 22.5,
  "boost_remaining_sec": 3600
}
```

### 9.2 Sensor Pairing, Naming & Role Configuration

#### `POST /api/v1/sensors/pair`
Starts or cancels 60-second ESP-NOW pairing mode from Web UI or API.
* **Request:**
```json
{
  "action": "start",
  "duration_sec": 60
}
```
* **Response (`200 OK`):**
```json
{
  "status": "ok",
  "pairing_active": true,
  "expires_in_sec": 60
}
```

#### `PUT /api/v1/sensors/{id}`
Updates a sensor's custom name, role (`indoor`, `outdoor`, `disabled`), weight, calibration offset, or wake interval.
* **Request (`PUT /api/v1/sensors/1`):**
```json
{
  "name": "Kids Bedroom",
  "role": "indoor",
  "weight": 2,
  "offset": -0.2,
  "report_interval_sec": 300
}
```
* **Response (`200 OK`):**
```json
{
  "status": "ok",
  "sensor_id": 1,
  "name": "Kids Bedroom",
  "role": "indoor",
  "pending_sync": true
}
```

#### `DELETE /api/v1/sensors/{id}`
Unpairs and deletes wireless sensor `1..15` (`id: 0` onboard sensor cannot be deleted, only set to `role: "disabled"`).
* **Response (`200 OK`):** `{"status": "ok", "deleted_id": 1}`

### 9.3 Weekly Schedule API

#### `GET /api/v1/schedule` & `POST /api/v1/schedule`
Reads or updates one day or the entire 7-day schedule (`0=Sun` .. `6=Sat`).
* **Request (`POST /api/v1/schedule`):**
```json
{
  "day": 1,
  "copy_to_days": [2, 3, 4, 5],
  "slots": [
    {"enabled": true, "time": "06:30", "target_temp": 21.5, "agg_mode": "average", "target_sensor_id": 0},
    {"enabled": true, "time": "08:30", "target_temp": 18.5, "agg_mode": "average", "target_sensor_id": 0},
    {"enabled": true, "time": "17:00", "target_temp": 21.5, "agg_mode": "average", "target_sensor_id": 0},
    {"enabled": true, "time": "22:30", "target_temp": 19.5, "agg_mode": "single", "target_sensor_id": 1},
    {"enabled": false, "time": "23:30", "target_temp": 18.0, "agg_mode": "average", "target_sensor_id": 0},
    {"enabled": false, "time": "23:59", "target_temp": 18.0, "agg_mode": "average", "target_sensor_id": 0}
  ]
}
```
* **Response (`200 OK`):**
```json
{
  "status": "ok",
  "updated_days": [1, 2, 3, 4, 5]
}
```

### 9.4 Control Algorithm, Weather Rules & Wi-Fi Setup APIs

#### `GET /api/v1/settings` & `POST /api/v1/settings`
* **Request (`POST /api/v1/settings`):**
```json
{
  "control_mode": "hysteresis_dynamic",
  "agg_mode": "average",
  "single_sensor_id": 0,
  "hysteresis_low": 0.3,
  "hysteresis_high": 0.2,
  "tpi_cycle_min": 15,
  "min_on_time_sec": 240,
  "min_off_time_sec": 240,
  "outdoor_lockout_enabled": true,
  "outdoor_lockout_temp": 18.5,
  "weather_comp_enabled": true,
  "weather_comp_slope": 0.05,
  "frost_protect_temp": 6.0,
  "open_window_enabled": true,
  "pump_exercise_enabled": true
}
```
* **Response (`200 OK`):** `{"status": "ok", "saved": true}`

#### `POST /api/v1/settings/reset-learning`
Resets the learned thermal overshoot EMA in `hysteresis_dynamic` mode back to default (`0.20°C`).
* **Response (`200 OK`):** `{"status": "ok", "learned_overshoot_c": 0.20}`

#### `POST /api/v1/wifi`
Updates Wi-Fi credentials and triggers reconnection.
* **Request:**
```json
{
  "ssid": "MyHomeWiFi",
  "password": "MySecretPassword123",
  "hostname": "opensmartthermo",
  "dhcp": true
}
```
* **Response (`200 OK`):** `{"status": "ok", "message": "Wi-Fi credentials saved. Reconnecting in 2 seconds."}`

---

## 10. MQTT API & Home Assistant Auto-Discovery Specification

Base Topic (configurable): `opensmartthermo`

### 10.1 Published State Topics (Hub -> Broker)
* `opensmartthermo/state` (JSON payload published on every change and every 60s):
  ```json
  {
    "mode": "AUTO",
    "action": "heating",
    "control_temp": 20.65,
    "target_temp": 21.0,
    "outdoor_temp": 11.4,
    "control_mode": "hysteresis_dynamic",
    "learned_overshoot": 0.42,
    "lockout_reason": "NONE"
  }
  ```
* `opensmartthermo/sensor/{id}/state` (Published whenever sensor `0..15` updates):
  ```json
  {
    "id": 1,
    "name": "Master Bedroom",
    "role": "indoor",
    "temp": 20.8,
    "humidity": 54.0,
    "battery": 94,
    "online": true
  }
  ```

### 10.2 Subscribed Command Topics (Broker -> Hub)
* `opensmartthermo/set/mode`: Payload `"AUTO"`, `"MANUAL"`, `"BOOST"`, or `"OFF"`.
* `opensmartthermo/set/target_temp`: Payload float string, e.g. `"21.5"`.
* `opensmartthermo/set/pairing`: Payload `{"action": "start", "duration_sec": 60}` or `"stop"`.
* `opensmartthermo/set/sensor/{id}`: Payload `{"name": "Office", "role": "indoor", "offset": -0.2}`.
* `opensmartthermo/set/schedule`: Full day JSON object identical to `POST /api/v1/schedule`.
* `opensmartthermo/set/settings`: Partial or full JSON object identical to `POST /api/v1/settings`.
* `opensmartthermo/set/wifi`: JSON `{"ssid": "...", "password": "..."}`.

### 10.3 Home Assistant MQTT Auto-Discovery
On MQTT connection, the thermostat automatically publishes retained discovery configs to:
* `homeassistant/climate/opensmartthermo/config` (exposes full Thermostat Climate card with modes `off`, `heat`, `auto`, current temperature, target temperature, and presets `none` / `boost`).
* `homeassistant/sensor/opensmartthermo_s{id}_temp/config` and `_hum/config` and `_bat/config` using each sensor's **Custom Name** (e.g., `Master Bedroom Temperature`).

---

## 11. 128x64 Graphic LCD Screen Map & 4-Button Navigation Specification

### 11.1 Physical Button Mapping (`UP`, `DOWN`, `OK`, `BACK`)
* **`UP`:** Increase value / move cursor up / previous item.
* **`DOWN`:** Decrease value / move cursor down / next item.
* **`OK`:** Confirm selection / enter submenu / cycle mode (`AUTO` -> `MANUAL` -> `BOOST` -> `OFF`).
* **`BACK`:** Immediate cancel / return to parent menu / exit Pairing or Wi-Fi Setup back to Home Screen (`Screen 0`).

### 11.2 Complete 128x64 Pixel Screen Layouts (ASCII Wireframes)

#### Screen 0: Home Screen (Default Idle View)
```
+----------------------------------+
| AUTO   12:30  [W]  OUT: 11.4°C   |  <- Top Status Bar: Mode, Time, Wi-Fi icon, Outdoor Temp
|----------------------------------|
|  CURRENT          TARGET         |
|   20.6°C  -->     21.0°C         |  <- Large 16x24px Font for instant legibility
|   [FLAME] HEATING   RH: 52%      |  <- Relay state icon + Control sensor humidity
|----------------------------------|
| Src: Avg(2)   Slot: 08:00 21.0°  |  <- Active sensor source & current schedule slot
+----------------------------------+
```
* **Button Actions on Screen 0:**
  * `UP` / `DOWN`: Adjust `Target Temp` by `±0.5°C` (switches to `MANUAL` if in `AUTO`, or adjusts `BOOST` temp).
  * `OK` (Short Press): Cycles `System Mode` (`AUTO` -> `MANUAL` -> `BOOST` -> `OFF`).
  * `OK` (Long Press 1.5s): Opens **Main Menu (`Screen 1`)**.
  * `BACK`: Wakes LCD backlight for 15s (or activates `BOOST` if pressed twice quickly).

#### Screen 1: Main Menu
```
+----------------------------------+
|           MAIN MENU              |
|----------------------------------|
| > 1. Sensor Readings & Roles     |
|   2. Weekly Schedule             |
|   3. Sensor Aggregation          |
|   4. Pair Wireless Sensor        |
|   5. Heating Algorithm & Rules   |
|   6. Wi-Fi & Network Info        |
+----------------------------------+
```
* **Button Actions:** `UP`/`DOWN` moves `>` cursor; `OK` enters selected submenu (`Screens 2–7`); `BACK` returns immediately to `Screen 0`.

#### Screen 2: Sensor Readings & Roles (`Submenu 1`)
```
+----------------------------------+
| SENSORS (1/4)            [IN]    |
|----------------------------------|
| Name: Hallway Wall (ID 0)        |
| Temp: 20.5°C     Hum: 52.1%      |
| Role: INDOOR     Bat: MAINS      |
|----------------------------------|
| [UP/DN] Next   [OK] Toggle Role  |
+----------------------------------+
```
* **Button Actions:** `UP`/`DOWN` scrolls through all paired sensors (`ID 0..15`); `OK` toggles sensor role (`INDOOR` -> `OUTDOOR` -> `DISABLED`); `BACK` exits to `Screen 1`.

#### Screen 3: Weekly Schedule Editor (`Submenu 2`)
```
+----------------------------------+
| SCHEDULE: MON   Slot [1/6]: ON   |
|----------------------------------|
|   Start Time :  [06:30]          |
|   Target Temp:   21.5°C          |
|   Sensor Src :   Avg(All)        |
|----------------------------------|
| [UP/DN] Edit  [OK] Next  [BK] Ex |
+----------------------------------+
```
* **Button Actions:** `OK` steps focus through `Day (Mon–Sun)` -> `Slot (1–6)` -> `Enabled (ON/OFF)` -> `Hour` -> `Minute` -> `Target Temp` -> `Sensor Source`; `UP`/`DOWN` modifies focused field; `BACK` saves to NVS and returns to `Screen 1`.

#### Screen 4: Sensor Aggregation Mode (`Submenu 3`)
```
+----------------------------------+
|     SENSOR AGGREGATION MODE      |
|----------------------------------|
|   ( ) Single Sensor              |
|   (*) Average (All Indoor)       |
|   ( ) Weighted Average           |
|   ( ) Minimum (Coldest Room)     |
|   ( ) Maximum (Warmest Room)     |
+----------------------------------+
```
* **Button Actions:** `UP`/`DOWN` highlights mode; `OK` selects mode (if `Single Sensor` is chosen, prompts to pick which named indoor sensor); `BACK` returns to `Screen 1`.

#### Screen 5: Wireless Sensor Pairing (`Submenu 4`)
```
+----------------------------------+
|     PAIR WIRELESS SENSOR         |
|----------------------------------|
|   Scanning ESP-NOW...   [48s]    |
|   Hold PAIR button on sensor     |
|   for 2 seconds.                 |
|   Found: ID #3 (21.1°C)          |
|   Press [BACK] to Exit           |
+----------------------------------+
```
* **Button Actions:** Automatically opens 60s pairing window upon entry. Displays newly paired Sensor ID immediately. Pressing `BACK` at any time aborts pairing and exits cleanly to `Screen 1`.

#### Screen 6: Heating Algorithm & Safety Rules (`Submenu 5`)
```
+----------------------------------+
|     HEATING CONTROL RULES        |
|----------------------------------|
| > Algo : Dyn. Hysteresis (0.42°) |
|   Min On/Off Delay : 240 sec     |
|   Outdoor Lockout  : 18.5°C      |
|   Frost Protection :  6.0°C      |
|   Reset Learned Overshoot        |
+----------------------------------+
```
* **Button Actions:** `UP`/`DOWN` selects parameter; `OK` edits value (`Static Hyst` / `Dynamic Hyst` / `TPI`, delay seconds, lockout threshold, or resets learned overshoot); `BACK` saves and returns to `Screen 1`.

#### Screen 7: Wi-Fi & Network Info (`Submenu 6`)
```
+----------------------------------+
|       WI-FI & NETWORK            |
|----------------------------------|
| SSID: MyHomeWiFi (CH 6)          |
| IP  : 192.168.1.145              |
| MQTT: Connected                  |
| > [Start Wi-Fi Setup AP]         |
|   [BACK] Return                  |
+----------------------------------+
```
* **Button Actions:** Pressing `OK` on `[Start Wi-Fi Setup AP]` launches temporary `OpenSmartThermo-Setup` AP (`192.168.4.1`) for phone browser configuration. Pressing `BACK` stops Setup AP (if active) and returns to `Screen 1`.

---

## 12. Complete End-User Manual (Hardware, LCD Screens, Web UI & Troubleshooting)

### 12.1 Physical Installation & Safety Warning
> **ELECTRICAL HAZARD WARNING:** Disconnect mains power at the circuit breaker before opening the wall box or wiring `L / N` terminals.
1. **Mount Wall Backplate:** Attach the thermostat rear housing to a standard EU/US wall junction box away from direct sunlight, radiators, or drafts.
2. **Wire Mains Power (`100–240V AC`):** Connect Live (`L`) and Neutral (`N`) to the 2-pin power terminal.
3. **Wire Boiler Dry Contact (`COM` & `NO`):** Connect the two thermostat control wires from your boiler, zone valve, or heat pump to **`COM`** and **`NO`**. **Never bridge `L` to `COM` unless your zone valve explicitly requires `230V AC` switching.**
4. **Snap Front Panel & Restore Power:** The LCD illuminates, displays the boot splash screen, and enters Home Screen (`Screen 0`).

### 12.2 Initial Wi-Fi & Browser Setup
1. On first boot (or via `Main Menu -> 6. Wi-Fi & Network Info -> Start Wi-Fi Setup AP`), the thermostat creates an open Wi-Fi hotspot named **`OpenSmartThermo-Setup`**.
2. Connect with your smartphone or laptop and browse to **`http://192.168.4.1`**.
3. Select your home Wi-Fi network, enter the password, and click **Save & Connect**.
4. Once connected, access the full Web Interface anytime from any browser on your network at **`http://opensmartthermo.local`** (or the IP address shown on LCD `Screen 7`).

### 12.3 Pairing, Naming & Assigning Roles to Wireless Thermometers
1. **Insert Batteries:** Install `2x AA` batteries into the wireless thermometer (wired directly to the `3V3` and `GND` pins on the Seeed Studio XIAO ESP32C3).
2. **Open Pairing Mode (2 Ways):**
   * **From Web UI:** Open **`http://opensmartthermo.local`**, go to the **Sensors (`/sensors`)** page, and click **`+ Pair New Sensor`**.
   * **From Thermostat Buttons:** Hold **`OK`** (1.5s) -> scroll to **`4. Pair Wireless Sensor`** -> press **`OK`**.
3. **Trigger Wireless Sensor:** Press and hold the **`PAIR / SYNC`** button (`D1 / GPIO 3`) on the wireless thermometer for **2 seconds** until its green LED begins blinking every 0.5s.
4. **Confirm & Customize:** Within 2–3 seconds, the LED glows solid for 1 second to confirm pairing. In the Web UI popup (or on the Sensors table), enter a **Custom Name** (e.g., `"Living Room"` or `"Outdoor North"`) and select its **Role**:
   * **`Indoor`:** Included in indoor room temperature calculations.
   * **`Outdoor`:** Used exclusively for **Outdoor Warm-Weather Lockout** and **Cold-Weather Compensation** (never averaged into indoor temperature).
   * **`Disabled`:** Monitored on the dashboard without affecting heating control.

### 12.4 Step-by-Step Guide to Using the 4 Thermostat Buttons
* **Quick Temperature Adjustment:** On the Home Screen, press **`UP`** or **`DOWN`** to nudge the target temperature by `0.5°C`.
* **Changing Operating Mode:** On the Home Screen, press **`OK`** repeatedly to cycle between **`AUTO`** (follows weekly schedule), **`MANUAL`** (holds fixed temperature), **`BOOST`** (heats to `22.5°C` for 1 hour), and **`OFF`** (heating off; sensors remain active).
* **Navigating Menus:** Hold **`OK`** for 1.5 seconds to open the Main Menu. Use **`UP`/`DOWN`** to move the cursor, **`OK`** to select/edit, and **`BACK`** at any time to step backward or return immediately to the Home Screen.

### 12.5 Step-by-Step Guide to Each Web UI Page
* **Dashboard (`/`):** Monitor current vs. target temperature, active boiler flame status, outdoor weather lockouts, 24-hour temperature/runtime history chart, and all named indoor/outdoor sensor cards. Switch between `AUTO`, `MANUAL`, `BOOST`, and `OFF` with one click.
* **Sensors (`/sensors`):** Click `+ Pair New Sensor` to add wireless thermometers, assign custom names (`"Bedroom"`, `"Balcony"`), designate which sensor is `Outdoor`, set calibration offsets, or adjust battery report intervals (`60s–1800s`).
* **Weekly Schedule (`/schedule`):** Select any day (`Mon–Sun`), enable up to 6 time slots, set start time, target temperature, and choose whether that time slot uses all indoor sensors or locks onto a specific room (e.g., `"Master Bedroom"` at night). Use **Copy Day** buttons to replicate schedules across weekdays or weekends in one click.
* **Heating Control (`/control`):** Choose between **Dynamic Self-Learning Hysteresis** (recommended for radiators—automatically learns thermal overshoot), **Static Hysteresis**, or **TPI** (recommended for modern gas boilers). Configure the **Anti-Short-Cycle Delay** (keep at `240s` / 4 minutes minimum), **Outdoor Warm-Weather Lockout** (e.g., `18.5°C`), **Frost Protection** (`6.0°C`), and **Open Window Detection**.
* **Network (`/network`):** Update Wi-Fi credentials, configure MQTT broker details for Home Assistant integration, or manually synchronize the `DS3231` hardware real-time clock.
* **System (`/system`):** Inspect cumulative boiler runtime hours, export/import a JSON backup of your settings and schedule, or upload new `.bin` firmware via OTA.

### 12.6 Troubleshooting & Diagnostic Codes
| LCD / Web Alert | Root Cause | Resolution |
| :--- | :--- | :--- |
| **`ECO LOCK`** | Outdoor sensor temperature is at or above `outdoor_lockout_temp` (e.g., `>= 18.5°C`). | Normal summer/warm-day behavior. Heating resumes automatically when outdoor temp drops `1.0°C` below threshold. |
| **`DELAY 240s`** | Anti-Short-Cycle protection active (`min_on_time` or `min_off_time` countdown). | Normal boiler protection. Relay will switch automatically when countdown reaches `0s`. |
| **`SENSOR LOST`** | Active wireless sensor has not reported for `> 30 minutes`. | Check sensor batteries or press its `PAIR / SYNC` button. Thermostat automatically falls back to onboard sensor. |
| **`FROST GUARD`** | Indoor temperature dropped below `6.0°C` while thermostat was `OFF`. | Emergency anti-freeze heating engaged until room reaches `7.0°C`. |
| **`WINDOW PAUSE`** | Sudden temperature drop (`> 1.5°C` in 5 min) detected. | Heating paused for 20 minutes while window is open. Press `OK` to override immediately. |

### 12.7 Wireless Thermometer Display & Button Operation
Each wireless thermometer (`Seeed Studio XIAO ESP32C3`) features an ultra-low-power I2C screen (`0.0 µA` sleep draw) and **one multi-function button** (`D1 / GPIO 3`) that controls screen wake-up, manual synchronization, and pairing:

* **Turning the Screen ON & Instant Sync (Short Press `< 1s` when Screen is OFF):**
  * Press the button briefly to wake the thermometer. The screen turns ON immediately, measures fresh temperature/humidity/battery data, and syncs with the thermostat hub.
  * **Reading the Screen:**
    * **Top Row:** Shows the thermometer's permanent static ID (`#01`–`#15`, or `#--` if unpaired), connection status (**`[LINK OK]`** when connected to the thermostat, or **`[NO LINK]`** if out of range/unpaired), and the connected thermostat's name (`->Thermo`, which scrolls horizontally if longer than 6 characters).
    * **Second Row:** Displays the custom name assigned to the thermometer in the thermostat's Web UI (e.g., `Master Bedroom`). Names up to 21 characters appear statically; longer names (up to 24 characters) scroll smoothly across the display.
    * **Third Row:** Displays the current room/outdoor temperature (`°C`) and relative humidity (`%RH`).
    * **Bottom Row:** Displays remaining battery percentage (`%`), pack voltage (`V`), and the remaining auto-off countdown (`[OFF: 8s]`).
* **Automatic Screen Shutoff & Instant Manual Shutoff (Short Press `< 1s` while Screen is ON):**
  * To preserve multi-year battery life, the screen automatically powers off after the configured timeout (`5–10 seconds`, default `8s`, adjustable on the thermostat's `/sensors` Web UI page).
  * If you finish reading the screen early, **press the button briefly a second time while the screen is still ON** to turn it off immediately without waiting for the timer to expire.
* **Pairing Mode (Long Press `>= 2s`):**
  * Press and hold the button for **2 seconds** until **`[PAIRING]`** appears blinking in the top row of the display (temperature and humidity remain visible below it).
  * The thermometer searches for the thermostat for up to `60 seconds`. Once paired, `[PAIRING]` changes to **`[LINK OK]`** followed by the thermostat's name, the newly assigned static `#ID` and custom name appear on screen, and the display powers off automatically after the timeout expires.

---

## 13. Hardware Construction & Solder-Free Assembly Guide (Step-by-Step)

This section provides complete, hands-on assembly instructions for building both the **Main Wall Thermostat & Hub (`ESP32-S3`)** and the **Wireless Indoor/Outdoor Thermometers (`Seeed Studio XIAO ESP32C3`)** using off-the-shelf modules, through-hole protoboards, and standard DuPont/screw-terminal wiring—with **zero SMD soldering or custom PCB fabrication required**.

---

### 13.1 Required Tools & Prototyping Supplies
* **Soldering Iron (30W–60W) & Rosin-Core Solder (0.8mm):** Used strictly for standard `2.54mm` through-hole header pins and protoboard point-to-point wires.
* **70x90mm (or 50x70mm) Double-Sided Through-Hole Protoboard (`2.54mm` pitch):** One board for the Main Thermostat, and one compact `30x70mm` protoboard per Wireless Thermometer.
* **2.54mm Female Pin Header Strips:** Allows plugging in the `ESP32-S3`, `Seeed Studio XIAO ESP32C3`, `SHT40`, `DS3231`, and display modules so they remain removable and replaceable.
* **22 AWG Solid/Stranded Hookup Wire (Low-Voltage Logic)** and **18 AWG Insulated Mains Wire (`230V AC` / `COM–NO` relay loop)**.
* **Heat-Shrink Tubing & Hot Glue / Nylon M2.5 Standoffs:** For mechanical strain relief and electrical insulation inside the enclosures.
* **Digital Multimeter:** Mandatory for verifying `3.3V` / `5.0V` rails and checking continuity before applying mains power or inserting batteries.

---

### 13.2 Building the Wireless Thermometer Nodes (`Seeed Studio XIAO ESP32C3`)

Follow this exact sequence to assemble each battery-powered indoor or outdoor thermometer:

#### Step 1: Prepare the XIAO ESP32C3 & Antenna
1. Snap the included **external U.FL antenna** firmly onto the micro-coaxial connector on top of the `Seeed Studio XIAO ESP32C3` until you hear a distinct click. *(Never power on the radio without the antenna attached).*
2. Solder two `2.54mm` female header strips onto your `30x70mm` protoboard spaced to accept the XIAO ESP32C3, and plug the XIAO module into the headers.

#### Step 2: Wire the I2C Bus (`SHT40` Sensor & `0.96"`/`0.91"` Display)
1. **Shared I2C Data & Clock:**
   * Connect XIAO **`D4` (`GPIO 6`)** to both `SHT40 SDA` and `Display SDA`.
   * Connect XIAO **`D5` (`GPIO 7`)** to both `SHT40 SCL` and `Display SCL`.
   * *(Note: Both the SHT40 breakout and I2C OLED/LCD modules already include onboard `4.7kΩ` pull-up resistors to their respective supply pins, so no external pull-ups are needed).*
2. **Sensor Power (Always-On Ultra-Low Idle):**
   * Connect `SHT40 VIN` directly to XIAO **`3V3`** and `SHT40 GND` to **`GND`**.
3. **Display Power Gating (`0.0 µA` Deep-Sleep Cut-Off):**
   * Connect `Display VCC` directly to XIAO **`D3` (`GPIO 5`)** (do **not** connect `Display VCC` to `3V3`).
   * Connect `Display GND` to **`GND`**.

#### Step 3: Wire the Single Multi-Function Button (`D1 / GPIO 3`)
1. Mount a tactile pushbutton on the protoboard (or panel-mount it through the enclosure lid).
2. Connect one leg of the button to XIAO **`D1` (`GPIO 3`)** and the opposite leg to **`GND`**. *(Internal `INPUT_PULLUP` is enabled in firmware; `D2 / GPIO 4` remains unconnected as `SPARE_IO`)*.

#### Step 4: Assemble the Battery Voltage Divider (`D0 / GPIO 2`)
1. Solder two matched **`1MΩ` (1% tolerance)** resistors in series between XIAO **`3V3`** and **`GND`**.
2. Connect the midpoint between the two `1MΩ` resistors to XIAO **`D0` (`GPIO 2`)**.
3. Solder a **`100nF` ceramic capacitor** between **`D0` (`GPIO 2`)** and **`GND`** to stabilize ADC readings during Wi-Fi/ESP-NOW bursts.

#### Step 5: Connect the `2x AA` Battery Holder (Critical Polarity & Rail Rule)
1. Verify with a multimeter that your `2x AA` battery pack measures **`3.0V–3.2V`** (Alkaline or Energizer Ultimate Lithium `L91`).
2. Connect Battery **`+` (Red)** directly to XIAO **`3V3`** and Battery **`-` (Black)** to **`GND`**.
   * **CRITICAL WARNING:** Do **not** solder batteries to the bottom `BAT` pads, and **always remove the `2x AA` batteries before plugging a USB-C cable** into the XIAO ESP32C3 for firmware flashing.

#### Step 6: Enclosure Mounting (Indoor vs. Outdoor)
* **Indoor Node:** Mount the protoboard inside a ventilated ABS wall enclosure (`~80x50x25mm`). Cut a rectangular window for the `0.96"` screen, drill a hole for the `PAIR / SCREEN` pushbutton, and ensure ventilation slots sit directly below the `SHT40` sensor.
* **Outdoor Node:** House the assembly inside an **IP65 weatherproof junction box** mounted in a shaded north-facing location (or inside a Stevenson screen radiation shield). Drill a `6mm` breathing hole at the bottom of the box covered with a **hydrophobic PTFE adhesive membrane** directly over the `SHT40` sensor so humidity equalizes without allowing liquid water entry.

---

### 13.3 Building the Main Wall Thermostat & Hub (`ESP32-S3`)

> **HIGH-VOLTAGE SAFETY WARNING:** The Main Thermostat contains exposed `230V AC` mains traces on the `HLK-PM01` input and relay terminals. Keep at least **`6mm` physical creepage/clearance distance** between all `230V AC` conductors and low-voltage (`3.3V` / `5V`) logic wires, and insulate the underside of the AC section with a Kapton/plastic barrier sheet.

#### Step 1: Assemble the Isolated `230V AC` to `5V DC` / `3.3V DC` Power Stage
1. Mount a `2-pin 5.08mm screw terminal` (`L` and `N`) at the top-left corner of the protoboard.
2. Wire `L` through a **`2A` slow-blow fuse** into the `AC L` input pin of the **`Hi-Link HLK-PM01`** module, and `N` to `AC N`. Place a **`10D471K` MOV varistor** across `L` and `N` after the fuse for surge protection.
3. On the `HLK-PM01` low-voltage output (`+5V` and `0V/GND`), install a **`470µF / 16V` electrolytic capacitor** and a **`100nF` ceramic capacitor** across `+5V` and `GND`.
4. Feed `+5V` into an **`AMS1117-3.3` LDO breakout board** (with `10µF` input/output capacitors) to generate a clean **`3.3V` logic rail**.

#### Step 2: Wire the Fail-Safe Dry Contact Relay Stage (`GPIO 16`)
1. Place the **`Omron G5LE-1-VD 5DC`** relay and a `3-pin 5.08mm screw terminal` (`COM`, `NO`, `NC`) at the top-right corner of the protoboard (away from the `SHT40` sensor).
2. Connect one coil pin of the relay to **`+5V`**.
3. Connect the other coil pin to the Drain (`D`) of a **`2N7002` N-MOSFET** (or `2N2222` NPN transistor with a `1kΩ` base resistor), and connect the Source (`S`) to **`GND`**.
4. Solder a **`1N4148` flyback diode** directly across the two relay coil pins (cathode stripe toward `+5V`).
5. Connect the MOSFET Gate (`G`) to `ESP32-S3` **`GPIO 16`**, and place a **`10kΩ` pull-down resistor** between `GPIO 16` and `GND` so the relay stays strictly `OFF` during boot/reset.
6. Wire the relay's three switch contacts (`COM`, `NO`, `NC`) to the `3-pin 5.08mm screw terminal` using heavy `18 AWG` wire (keep these three terminals **100% isolated** from `L`, `N`, `5V`, and `GND`).

#### Step 3: Wire the External Hardware Watchdog (`TI TPL5010`)
1. Connect `TPL5010 VDD` to `3.3V` and `GND` to `GND`.
2. Connect `TPL5010 DELAY` pin to `GND` via a **`47kΩ` resistor** (sets the hardware timeout window to `~60 seconds`).
3. Connect `TPL5010 DRV` (or `RSTn` via inverter/transistor as per breakout) to the `ESP32-S3 EN` pin, and connect `TPL5010 DONE` to `ESP32-S3` **`GPIO 17`**.

#### Step 4: Wire the `128x64 ST7567` SPI LCD & 4 Tactile Buttons
1. **SPI LCD Connections:**
   * `LCD VCC` -> `3.3V`, `LCD GND` -> `GND`
   * `LCD CS` -> `GPIO 10`
   * `LCD MOSI (SDA)` -> `GPIO 11`
   * `LCD SCK` -> `GPIO 12`
   * `LCD DC (A0/RS)` -> `GPIO 13`
   * `LCD RST` -> `GPIO 14`
   * `LCD BL (Backlight)` -> `GPIO 15`
2. **4 Tactile Buttons (`UP`, `DOWN`, `OK`, `BACK`):**
   * Wire each button between its designated GPIO (`UP = GPIO 35`, `DOWN = GPIO 36`, `OK = GPIO 37`, `BACK = GPIO 38`) and **`GND`**.
   * Solder a **`100nF` ceramic capacitor** across each button for hardware switch debouncing.

#### Step 5: Wire the Shared I2C Bus (`DS3231` RTC & Thermally Isolated `SHT40`)
1. Connect **`GPIO 4` (`I2C_SDA`)** and **`GPIO 5` (`I2C_SCL`)** to both the `DS3231` RTC module (insert a fresh `CR2032` coin cell) and the `SHT40` sensor breakout.
2. **Thermal Isolation Mounting:** Mount the `SHT40` breakout at the **bottom-left corner** of the wall enclosure using `15mm` flexible DuPont jumper wires so it sits directly inside the lower intake vent slot—thermally decoupled from the `HLK-PM01` power supply, relay coil, and `ESP32-S3` mounted in the upper half of the housing.

---

### 13.4 Pre-Power-Up Inspection & Commissioning Checklist
Before closing the enclosures or connecting `230V AC` mains power, complete every verification step below:

1. **Continuity & Short-Circuit Test (Multimeter in Buzzer Mode):**
   * Verify **infinite resistance (`OL`)** between `L` and `N`, between `3.3V` and `GND`, and between `5V` and `GND`.
   * Verify **complete galvanic isolation (`OL`)** between the dry contact terminals (`COM`, `NO`, `NC`) and all power/logic rails (`L`, `N`, `5V`, `3.3V`, `GND`).
2. **USB-C Bench Bring-Up (Before Mains Connection):**
   * Flash the firmware to `ESP32-S3` via USB-C while disconnected from `230V AC`.
   * Confirm the `128x64 LCD` boots to `Screen 0`, all 4 buttons (`UP`, `DOWN`, `OK`, `BACK`) respond cleanly, the `SHT40` reports room temperature, and the `DS3231` RTC retains the time.
   * Test relay actuation: raise `Target Temp` above `Current Temp` and verify with a multimeter that continuity closes (`0 Ω`) between **`COM` and `NO`** after the anti-short-cycle guard clears.
3. **Wireless Node Bench Test (Before Closing Case):**
   * Flash firmware to the `Seeed Studio XIAO ESP32C3` via USB-C, **unplug USB-C**, and connect the `2x AA` battery pack to `3V3` and `GND`.
   * Press the single button (`D1 / GPIO 3`) briefly: verify the `0.96"` screen turns on, displays `#-- [NO LINK] -> HUB: None`, shows live temperature/humidity/battery voltage, and turns off automatically after `8 seconds` (or immediately if pressed again).
   * Hold the button for `>= 2 seconds` while the thermostat is in Pairing Mode: verify `[PAIRING]` blinks on the top line and transitions to `#01 [LINK OK]` once paired.
4. **Final Wall Installation:**
   * Switch off the home circuit breaker, connect `L / N` and the boiler `COM / NO` wires, snap the front bezel onto the wall backplate, and restore power.
