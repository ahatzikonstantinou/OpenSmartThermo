# OpenSmartThermo — Project Folder Structure & PlatformIO Build/Flashing Checklist (v4.4)

This guide organizes all generated firmware and Web UI files into two standalone PlatformIO projects and provides a step-by-step build, filesystem upload, flashing, and initial verification checklist for both the **Main Thermostat & Hub (`ESP32-S3`)** and the **Wireless Sensor Nodes (`Seeed Studio XIAO ESP32C3`)**.

---

## 1. Complete Project Directory Structure

Organize your workspace into two separate PlatformIO projects so each microcontroller compiles cleanly with its own dependencies, board definition, and partition table:

```text
OpenSmartThermo_Firmware/
├── 1_Thermostat_Hub_ESP32S3/
│   ├── platformio.ini               <-- From Part 2A (OpenSmartThermo_Hub_Core_Firmware_v4.4.md)
│   ├── include/
│   │   └── config.h                 <-- From Part 2A
│   ├── src/
│   │   ├── main.cpp                 <-- From Part 2A
│   │   ├── control_engine.h         <-- From Part 2A
│   │   ├── control_engine.cpp       <-- From Part 2A
│   │   ├── lcd_ui.h                 <-- From Part 2A
│   │   ├── lcd_ui.cpp               <-- From Part 2A
│   │   ├── network_manager.h        <-- From Part 2B-1 (OpenSmartThermo_Hub_Network_API_v4.4.md)
│   │   └── network_manager.cpp      <-- From Part 2B-1
│   └── data/
│       └── index.html               <-- From Part 2B-2 (OpenSmartThermo_Hub_WebUI_v4.4.md)
│
└── 2_Wireless_Node_XIAO_ESP32C3/
    ├── platformio.ini               <-- From Part 1 (OpenSmartThermo_Node_Firmware_v4.4.md)
    └── src/
        └── main.cpp                 <-- From Part 1
```

### Source File Mapping Reference

| Target Device | Project Folder | File Path | Source Blueprint Document |
| :--- | :--- | :--- | :--- |
| **Thermostat & Hub (`ESP32-S3`)** | `1_Thermostat_Hub_ESP32S3/` | `platformio.ini` | `OpenSmartThermo_Hub_Core_Firmware_v4.4.md` (Part 2A) |
| **Thermostat & Hub (`ESP32-S3`)** | `1_Thermostat_Hub_ESP32S3/` | `include/config.h` | `OpenSmartThermo_Hub_Core_Firmware_v4.4.md` (Part 2A) |
| **Thermostat & Hub (`ESP32-S3`)** | `1_Thermostat_Hub_ESP32S3/` | `src/main.cpp` | `OpenSmartThermo_Hub_Core_Firmware_v4.4.md` (Part 2A) |
| **Thermostat & Hub (`ESP32-S3`)** | `1_Thermostat_Hub_ESP32S3/` | `src/control_engine.h` & `.cpp` | `OpenSmartThermo_Hub_Core_Firmware_v4.4.md` (Part 2A) |
| **Thermostat & Hub (`ESP32-S3`)** | `1_Thermostat_Hub_ESP32S3/` | `src/lcd_ui.h` & `.cpp` | `OpenSmartThermo_Hub_Core_Firmware_v4.4.md` (Part 2A) |
| **Thermostat & Hub (`ESP32-S3`)** | `1_Thermostat_Hub_ESP32S3/` | `src/network_manager.h` & `.cpp` | `OpenSmartThermo_Hub_Network_API_v4.4.md` (Part 2B-1) |
| **Thermostat & Hub (`ESP32-S3`)** | `1_Thermostat_Hub_ESP32S3/` | `data/index.html` | `OpenSmartThermo_Hub_WebUI_v4.4.md` (Part 2B-2) |
| **Wireless Node (`XIAO ESP32C3`)** | `2_Wireless_Node_XIAO_ESP32C3/` | `platformio.ini` | `OpenSmartThermo_Node_Firmware_v4.4.md` (Part 1) |
| **Wireless Node (`XIAO ESP32C3`)** | `2_Wireless_Node_XIAO_ESP32C3/` | `src/main.cpp` | `OpenSmartThermo_Node_Firmware_v4.4.md` (Part 1) |

---

## 2. Build & Flashing Checklist: Main Thermostat & Hub (`ESP32-S3`)

> **CRITICAL SAFETY WARNING:** Never connect `230V AC` mains power to the `HLK-PM01` power module while the `ESP32-S3` is connected to your computer via USB-C. Complete all firmware and filesystem flashing on the bench using USB-C power only.

### Step 1: Prepare the Project & LittleFS Web UI
- [ ] Open VS Code with the **PlatformIO IDE** extension installed.
- [ ] Open the `1_Thermostat_Hub_ESP32S3/` folder as your active project workspace.
- [ ] Confirm that `data/index.html` is placed inside the `data/` subdirectory at the root of `1_Thermostat_Hub_ESP32S3/` (PlatformIO packages all files inside `data/` into the LittleFS flash image).

### Step 2: Enter Bootloader Mode (First-Time USB Flash Only)
- [ ] Connect the `ESP32-S3` USB port to your computer using a data-capable USB-C cable.
- [ ] If your `ESP32-S3` dev board does not automatically enter download mode over USB-CDC, hold the **`BOOT`** button, tap **`RST`**, and release **`BOOT`**.

### Step 3: Build & Upload the LittleFS Filesystem (`data/index.html`)
- [ ] Open the PlatformIO terminal (`Ctrl + ~`) or navigate to **Project Tasks -> esp32-s3-devkitc-1 -> Platform** in the PlatformIO sidebar.
- [ ] Build and flash the LittleFS partition containing the embedded Web SPA:
  ```bash
  pio run --target buildfs
  pio run --target uploadfs
  ```
- [ ] Verify the terminal prints `[SUCCESS]` and confirms the LittleFS binary was written to flash.

### Step 4: Compile & Upload Main Thermostat Firmware
- [ ] Compile the C++ source code and upload the firmware binary:
  ```bash
  pio run --target upload
  ```
- [ ] Press the **`RST`** button on the `ESP32-S3` once after flashing finishes.

### Step 5: Verify Hub Boot & Initial Wi-Fi Setup
- [ ] Open the Serial Monitor at `115200 baud` (`pio device monitor`) and verify that `SHT40`, `DS3231 RTC`, `ST7567 LCD`, `LittleFS`, and `ESP-NOW` initialize without errors.
- [ ] Confirm the `128x64 ST7567 LCD` illuminates and renders the Home Screen (`Screen 0`).
- [ ] On a smartphone or PC, connect to the **`OpenSmartThermo-Setup`** Wi-Fi access point, navigate to `http://192.168.4.1`, enter your home Wi-Fi SSID and password, and click **Save & Connect**.
- [ ] Once connected to your home network, open `http://opensmartthermo.local` (or the IP address shown on LCD `Screen 7`) and verify the 6-page Web SPA loads cleanly.

---

## 3. Build & Flashing Checklist: Wireless Sensor Nodes (`Seeed Studio XIAO ESP32C3`)

> **CRITICAL HARDWARE RULE:** Disconnect the `2x AA` battery pack from the `3V3` pin **before** plugging a USB-C cable into the `Seeed Studio XIAO ESP32C3`. Never power the board from both USB-C (`5V` regulated to `3.3V`) and batteries (`3.0V` on `3V3`) at the same time.

### Step 1: Prepare Hardware & Flash Firmware
- [ ] Open the `2_Wireless_Node_XIAO_ESP32C3/` folder in VS Code / PlatformIO.
- [ ] Snap the external **U.FL antenna** onto the `Seeed Studio XIAO ESP32C3` antenna connector (never transmit RF without the antenna connected).
- [ ] Ensure no batteries are connected to `3V3`, then plug the `XIAO ESP32C3` into your PC via USB-C.
- [ ] Compile and upload the ultra-low-power node firmware:
  ```bash
  pio run --target upload
  ```
- [ ] *(Optional)* Open `pio device monitor` (`115200 baud`) during USB testing to verify SHT40 temperature/humidity readings and display initialization before disconnecting USB.

### Step 2: Battery Power-Up & Wireless Pairing Verification
- [ ] Unplug the USB-C cable completely from the `Seeed Studio XIAO ESP32C3`.
- [ ] Install `2x AA` batteries (`3.0V` nominal) into the battery holder wired directly to `3V3` (`+`) and `GND` (`-`).
- [ ] On the Thermostat Hub Web UI (`http://opensmartthermo.local/sensors`), click **`+ Pair New Sensor`** (or select `4. Pair Wireless Sensor` on the wall thermostat LCD).
- [ ] On the wireless sensor node, press and hold the single button (`D1 / GPIO 3`) for **2+ seconds** until the `0.96"` screen turns on and displays a blinking `[PAIRING]` badge.
- [ ] Within 2–3 seconds, confirm the node screen updates to **`[LINK OK]`**, shows its assigned `#ID` (e.g., `#01`), and automatically turns off the screen after the configured timeout (`8s` default) to enter deep sleep.
- [ ] In the Web UI (`Sensors` page), assign a **Custom Name** (e.g., `"Master Bedroom"` or `"Outdoor North"`), set the **Role** (`Indoor` or `Outdoor`), and save.
- [ ] Short-press the single button (`< 1s`) on the wireless node once and verify that its custom name and hub name sync onto the node's screen (with smooth horizontal marquee scrolling if the name exceeds `21` characters).
