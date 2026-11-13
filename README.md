# OpenSmartThermo (v4.4)

**Source-Available Wi-Fi + ESP-NOW Smart Thermostat, Integrated Wireless Hub & Ultra-Low-Power Room/Outdoor Sensors**

OpenSmartThermo is a complete, cloud-free smart heating controller and multi-room environmental monitoring system built on the **ESP32-S3** (Wall Thermostat & Integrated Hub) and **Seeed Studio XIAO ESP32C3** (Wireless Indoor & Outdoor Sensor Nodes). It requires **zero SMD soldering**, **zero custom PCB fabrication**, and **zero cloud subscriptions**, while offering full local Web SPA control, REST API, MQTT with Home Assistant Auto-Discovery, and 2–5+ year battery life on wireless nodes.

---

## Key Features

* **Voltage-Free Dry Contact Output (`COM` / `NO` / `NC`):** Galvanically isolated `5V` monostable relay safely controls `24V DC` modulating boilers, heat pumps, and `230V AC` zone valves.
* **Integrated ESP-NOW Wireless Hub:** Supports up to **15 wireless indoor/outdoor thermometers** plus **1 onboard sensor** with automatic Wi-Fi channel tracking (`CH 1–13`) and custom 24-character room names.
* **Ultra-Low-Power Wireless Nodes (`< 25 µA` Sleep):** Powered directly via the `3V3` pin from `2x AA` batteries (`2.5–5+ years` lifespan) with a power-gated `0.96"` I2C display, single-button `PAIR / SCREEN` control, and horizontal marquee text scrolling.
* **Advanced Heating Control Algorithms:**
  * **Dynamic Self-Learning Hysteresis:** Tracks 20-minute post-cutoff radiator overshoot using an Exponential Moving Average (EMA) and cuts off early to prevent overheating.
  * **Static Hysteresis & TPI (Time Proportional & Integral):** Configurable control modes for all boiler types.
  * **Universal Anti-Short-Cycle Guard:** Enforces configurable minimum ON and OFF durations (default `240s` / 4 minutes) across all algorithms.
* **Smart Weekly Scheduler & Multi-Sensor Aggregation:** 7-day × 6-slot schedule with empty-by-default (`--:--`) entries, strict chronological filtering, per-slot room overrides, and `Single`, `Average`, `Weighted`, `Min`, or `Max` indoor sensor aggregation.
* **Continuous Year-Round Operation (Summer Mode):** Setting the thermostat to `OFF` disables only the heating relay (retaining frost protection `< 6.0°C`); Wi-Fi, ESP-NOW, LCD, Web UI, REST API, and MQTT remain active 24/7.
* **Responsive Smartphone & Desktop Web UI:** Self-contained SPA served from `ESP32-S3` LittleFS with automatic Google Material 3 Light / Dark Slate mode, WebSocket live updates, and a fixed mobile bottom navigation bar.
* **On-Demand Captive Portal Wi-Fi Setup (`OpenSmartThermo-Setup`):**
  * Automatically launches an open `WIFI_AP_STA` hotspot (`OpenSmartThermo-Setup` at `192.168.4.1`) with an integrated DNS Captive Portal for 10 minutes on first boot or if the home router is unreachable after 20 seconds.
  * Can be toggled ON/OFF anytime by holding the **`BACK`** button (`GPIO 38`) on the wall thermostat for **3 seconds** (LCD top-left status bar displays `AP: 192.168.4.1`).
  * Performs live non-destructive credential verification (`Connected! IP: x.x.x.x` or `Connection failed. Check password.`) before closing the setup hotspot after 5 seconds.

---

## System Architecture

```text
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

---

## Complete Project Files & Documentation Index

All project documentation, hardware layouts, testing procedures, and embedded source code bundles are organized in the following Markdown files:

| File Name | Category | Description |
| :--- | :--- | :--- |
| **`OpenSmartThermo_Complete_Blueprint_v4.4.md`** | Master Blueprint | Complete v4.4 system architecture, BOMs, GPIO pinouts, heating algorithms, weekly scheduler rules, REST API, MQTT/Home Assistant specs, 128×64 LCD screen maps, user manual, and solder-free construction guide. |
| **`OpenSmartThermo_Component_Layout_v4.4.md`** | Hardware & Enclosures | Top/rear perfboard component placement diagrams, `>= 6.0mm` AC mains creepage barriers, point-to-point wiring tables, and verified flush-LCD 3D-printable enclosure dimensions (`32.0mm` Hub with `21.5mm` LCD standoffs; `32.1mm` Sensor Node with rear `2x AA` holder). |
| **`OpenSmartThermo_Build_Flashing_Checklist_v4.4.md`** | Build & Flash Guide | Step-by-step PlatformIO directory setup, LittleFS Web UI packaging (`buildfs` / `uploadfs`), firmware compilation (`upload`), and initial Wi-Fi captive portal pairing steps. |
| **`OpenSmartThermo_Bench_Testing_Checklist_v4.4.md`** | QA & Safety Testing | Pre-wall-mount multimeter verification for dry-contact galvanic isolation, `TPL5010` hardware watchdog heartbeat tests, and in-series microamp (`15–30 µA`) deep-sleep current checks. |
| **`OpenSmartThermo_Node_Firmware_v4.4.md`** | Firmware Source (Node) | Complete PlatformIO C++ source code (`platformio.ini` and `src/main.cpp`) for the **Seeed Studio XIAO ESP32C3** wireless indoor/outdoor thermometer node. |
| **`OpenSmartThermo_Hub_Core_Firmware_v4.4.md`** | Firmware Source (Hub 1/3) | Core PlatformIO C++ source code (`platformio.ini`, `include/config.h`, `src/control_engine.*`, `src/lcd_ui.*`, and `src/main.cpp`) for the **ESP32-S3** thermostat hub. |
| **`OpenSmartThermo_Hub_Network_API_v4.4.md`** | Firmware Source (Hub 2/3) | Networking PlatformIO C++ source code (`src/network_manager.h` and `src/network_manager.cpp`) implementing ESP-NOW Hub RX/TX, REST API (`/api/v1/*`), WebSockets (`/ws`), and MQTT + Home Assistant Auto-Discovery. |
| **`OpenSmartThermo_Hub_WebUI_v4.4.md`** | Firmware Source (Hub 3/3) | Complete single-file embedded Web SPA (`data/index.html`) with automatic Light/Dark mode, mobile bottom navigation bar, live WebSocket telemetry, and empty-slot (`--:--`) schedule editor. |

---

## How to Extract Source Code & Build the PlatformIO Projects

Because the firmware source files are packaged inside self-contained Markdown documentation files for easy reading and version control, follow these steps to extract the code into two PlatformIO projects before compiling.

### 1. Create the Target Folder Structure

Create a workspace directory (e.g., `OpenSmartThermo_Firmware/`) with the exact layout below:

```text
OpenSmartThermo_Firmware/
├── 1_Thermostat_Hub_ESP32S3/
│   ├── platformio.ini
│   ├── include/
│   │   └── config.h
│   ├── src/
│   │   ├── main.cpp
│   │   ├── control_engine.h
│   │   ├── control_engine.cpp
│   │   ├── lcd_ui.h
│   │   ├── lcd_ui.cpp
│   │   ├── network_manager.h
│   │   └── network_manager.cpp
│   └── data/
│       └── index.html
│
└── 2_Wireless_Node_XIAO_ESP32C3/
    ├── platformio.ini
    └── src/
        └── main.cpp
```

### 2. Copy Each Code Block into Its Target Source File

Open the four firmware Markdown files in any text editor or Markdown viewer, locate each labeled code block (` ```ini `, ` ```cpp `, or ` ```html `), and copy its contents (excluding the triple-backtick fence lines) into the corresponding file:

#### Project A: Main Thermostat & Hub (`1_Thermostat_Hub_ESP32S3/`)
1. From **`OpenSmartThermo_Hub_Core_Firmware_v4.4.md`**:
   * Copy the `platformio.ini` block -> `1_Thermostat_Hub_ESP32S3/platformio.ini`
   * Copy the `include/config.h` block -> `1_Thermostat_Hub_ESP32S3/include/config.h`
   * Copy the `src/control_engine.h` block -> `1_Thermostat_Hub_ESP32S3/src/control_engine.h`
   * Copy the `src/control_engine.cpp` block -> `1_Thermostat_Hub_ESP32S3/src/control_engine.cpp`
   * Copy the `src/lcd_ui.h` block -> `1_Thermostat_Hub_ESP32S3/src/lcd_ui.h`
   * Copy the `src/lcd_ui.cpp` block -> `1_Thermostat_Hub_ESP32S3/src/lcd_ui.cpp`
   * Copy the `src/main.cpp` block -> `1_Thermostat_Hub_ESP32S3/src/main.cpp`
2. From **`OpenSmartThermo_Hub_Network_API_v4.4.md`**:
   * Copy the `src/network_manager.h` block -> `1_Thermostat_Hub_ESP32S3/src/network_manager.h`
   * Copy the `src/network_manager.cpp` block -> `1_Thermostat_Hub_ESP32S3/src/network_manager.cpp`
3. From **`OpenSmartThermo_Hub_WebUI_v4.4.md`**:
   * Copy the `data/index.html` HTML block -> `1_Thermostat_Hub_ESP32S3/data/index.html`

#### Project B: Wireless Sensor Node (`2_Wireless_Node_XIAO_ESP32C3/`)
1. From **`OpenSmartThermo_Node_Firmware_v4.4.md`**:
   * Copy the `platformio.ini` block -> `2_Wireless_Node_XIAO_ESP32C3/platformio.ini`
   * Copy the `src/main.cpp` block -> `2_Wireless_Node_XIAO_ESP32C3/src/main.cpp`

---

## Flashing & Quick Start

> **Electrical Safety Rule:** Never connect `230V AC` mains power to the `ESP32-S3` Hub while connected via USB-C, and always disconnect the `2x AA` battery pack from the `XIAO ESP32C3` `3V3` pin before plugging in USB-C.

### 1. Flash the Main Thermostat & Hub (`ESP32-S3`)
Open `1_Thermostat_Hub_ESP32S3/` in VS Code with **PlatformIO IDE** and run:
```bash
# 1. Build and upload the embedded Web SPA (data/index.html) to LittleFS
pio run --target buildfs
pio run --target uploadfs

# 2. Compile and upload the ESP32-S3 firmware
pio run --target upload
```
* **First-Time Wi-Fi Setup & 3-Second `BACK` Shortcut:** On first power-up (or anytime by holding the **`BACK`** button on the wall thermostat for **3 seconds** until the status bar shows **`AP: 192.168.4.1`**), connect your smartphone or PC to wifi with SSID **`OpenSmartThermo-Setup`**. Your browser opens the Web SPA automatically via DNS Captive Portal (or browse to `http://192.168.4.1`). Enter your home Wi-Fi credentials on the **Network** tab; the Hub verifies the connection live, displays its assigned LAN IP, and closes the setup hotspot 5 seconds later. Afterward, access the Web UI at `http://opensmartthermo.local`.


### 2. Flash the Wireless Sensor Nodes (`Seeed Studio XIAO ESP32C3`)
Attach the external `U.FL` antenna, open `2_Wireless_Node_XIAO_ESP32C3/` in VS Code / PlatformIO, and run:
```bash
pio run --target upload
```
* **Pairing a Sensor:** Disconnect USB-C, connect the `2x AA` battery holder to `3V3` and `GND`, click **`+ Pair New Sensor`** in the Hub Web UI (or select `4. Pair Wireless Sensor` on the wall LCD), and hold the node's single button (`D1 / GPIO 3`) for **2+ seconds** until `[PAIRING]` appears on its screen.

---

## REST API & MQTT Quick Reference

* **Web UI / WebSockets:** `http://opensmartthermo.local/` and `ws://opensmartthermo.local/ws`
* **Status Endpoint:** `GET /api/v1/status`
* **Mode & Setpoint Control:** `POST /api/v1/control` (`{"system_mode": "AUTO"|"MANUAL"|"BOOST"|"OFF", "target_temp": 21.5}`)
* **MQTT State Topic:** `opensmartthermo/state` & `opensmartthermo/sensor/{id}/state`
* **MQTT Command Topics:** `opensmartthermo/set/mode`, `opensmartthermo/set/target_temp`, `opensmartthermo/set/pairing`
* **Home Assistant:** Publishes retained MQTT Auto-Discovery payloads automatically under `homeassistant/climate/opensmartthermo/config` and `homeassistant/sensor/opensmartthermo_s{id}_*/config`.

---

## License & Commercial Rights

* **Project License (`LICENSE`):** Licensed under the **PolyForm Noncommercial License 1.0.0** [cite: 2]. Individuals may view, build, and modify this project for personal, non-commercial home use. **All commercial rights (manufacturing, selling hardware units, or offering paid commercial services) are reserved exclusively by the author [cite: 2].**
* **Third-Party Libraries (`THIRD_PARTY_LICENSES.md`):** Open-source firmware dependencies (`Arduino-ESP32`, `U8g2`, `Adafruit SHT4x`, `RTClib`, `ESPAsyncWebServer`, `AsyncTCP`, `ArduinoJson`, and `PubSubClient`) are governed by their respective permissive/LGPL licenses (`Apache 2.0`, `MIT`, `BSD`, `LGPL-3.0`). See `THIRD_PARTY_LICENSES.md` for full copyright notices and attribution details.

---

## Hardware Safety Disclaimer

This project involves switching `100–240V AC` mains electricity and controlling heating equipment. Always disconnect mains power at the circuit breaker prior to installation, verify galvanic isolation between `L/N` and `COM/NO/NC` using a multimeter before connecting any boiler or zone valve, and comply with all local electrical codes.
