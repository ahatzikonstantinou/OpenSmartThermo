# OpenSmartThermo — Hardware Bench-Testing & Multimeter Verification Checklist (v4.4)

**Target Devices:**
1. **Main Thermostat & Hub (`ESP32-S3`)**
2. **Wireless Indoor/Outdoor Thermometer Nodes (`Seeed Studio XIAO ESP32C3`)**

**Purpose:** Step-by-step pre-installation multimeter, power, safety, and functional bench verification checklist to validate electrical safety, dry-contact isolation, watchdog fail-safes, and microamp deep-sleep performance before wall mounting.

---

## 1. Required Test Equipment
* **Digital Multimeter (DMM):** Capable of continuity buzzer, DC voltage (`0–20V`), resistance (`Ω` / `MΩ`), and microamp (`µA`) DC current measurement.
* **USB-C Power Source:** `5V` computer USB port or bench supply (`5.0V`).
* **Fresh Batteries:** `2x AA` Alkaline (`~3.1–3.2V` total) or `2x AA` Energizer Ultimate Lithium (`~3.3–3.5V` total) in a 2-cell series holder.
* **Optional Logic Probe / Oscilloscope:** Helpful for verifying `100µs` watchdog pulses on `GPIO 17`, though not mandatory.

---

## 2. Main Thermostat & Hub (`ESP32-S3`) — Unpowered Continuity & Isolation Checks

> **CRITICAL SAFETY WARNING:** Perform every check in Section 2 with **NO `230V AC` mains connected** and **NO `USB-C` cable plugged in**.

### 2.1 Galvanic Isolation of Dry Contact Output (`COM` / `NO` / `NC`)
1. Set your multimeter to **Continuity / Resistance (`Ω`)**.
2. **Verify Isolation from Mains (`L` & `N`):**
   * Measure between `L` and `COM`, `L` and `NO`, `L` and `NC`. Expected: **Open Line (`OL` / `> 10 MΩ`)**.
   * Measure between `N` and `COM`, `N` and `NO`, `N` and `NC`. Expected: **Open Line (`OL` / `> 10 MΩ`)**.
3. **Verify Isolation from Low-Voltage Logic (`3.3V` / `5V` / `GND`):**
   * Measure between `ESP32-S3 GND` and `COM`, `NO`, `NC`. Expected: **Open Line (`OL` / `> 10 MΩ`)**.
   * Measure between `5V` rail and `COM`, `NO`, `NC`. Expected: **Open Line (`OL` / `> 10 MΩ`)**.
4. **Verify Fail-Safe De-Energized Relay State:**
   * Measure between `COM` and `NO`. Expected: **Open Line (`OL`, no beep)**.
   * Measure between `COM` and `NC`. Expected: **Short Circuit (`< 0.5 Ω`, continuous beep)**.

### 2.2 Low-Voltage Power Rail Short-Circuit Checks
1. Measure resistance between **`3.3V`** and **`GND`** on the `ESP32-S3`. Expected: **`> 500 Ω`** (never `< 10 Ω`).
2. Measure resistance between **`5V`** (output of `HLK-PM01`) and **`GND`**. Expected: **`> 500 Ω`**.
3. Verify polarity and orientation of the **`1N4148` flyback diode** across the `Omron G5LE-1` relay coil (cathode stripe connected to `+5V`, anode connected to `2N7002` MOSFET drain).
4. Verify the **`10kΩ` pull-down resistor** between `GPIO 16` (`RELAY_CTRL`) and `GND` measures **`~10.0 kΩ`**.

---

## 3. Main Thermostat & Hub (`ESP32-S3`) — Bench Power-Up & Functional Verification

> **Bench Testing Rule:** Power the `ESP32-S3` via **USB-C (`5V`)** on the workbench first. Do not connect `230V AC` until all low-voltage tests pass.

### 3.1 Power Rails & Boot Diagnostics
1. Plug `USB-C` into the `ESP32-S3` and open the PlatformIO Serial Monitor (`115200 baud`).
2. Measure voltage between `3.3V` pin and `GND`. Expected: **`3.28V – 3.33V`**.
3. Confirm Serial Monitor prints clean initialization for:
   * `SHT40` onboard sensor (`I2C 0x44`)
   * `DS3231` hardware RTC (`I2C 0x68`)
   * `ST7567` 128x64 Graphic LCD (`SPI`)
   * `LittleFS` filesystem mounted (`index.html` present)

### 3.2 LCD Screen, Backlight & 4-Button Navigation
1. Confirm **Screen 0 (Home Screen)** renders crisp text without flickering or inverted contrast.
2. Short-press **`BACK`** (`GPIO 38`) to verify the LED backlight wakes up and dims automatically after inactivity.
3. Test all 4 tactile buttons (`UP`, `DOWN`, `OK`, `BACK`):
   * Short-press `UP` / `DOWN` on Screen 0 -> target temperature increments/decrements by `0.5°C`.
   * Short-press `OK` on Screen 0 -> cycles `AUTO` -> `MANUAL` -> `BOOST` -> `OFF`.
   * Hold `OK` (`1.5s`) -> opens **Main Menu (Screen 1)**; short-press `BACK` -> immediately returns to **Screen 0**.
   * Hold `BACK` (`>= 3.0s`) on **Screen 0** -> toggles the **`OpenSmartThermo-Setup`** Wi-Fi Access Point (`192.168.4.1`) ON for 10 minutes (top-left status bar displays **`AP: 192.168.4.1`**); hold `BACK` (`>= 3.0s`) again -> closes the Setup AP immediately.

### 3.3 Dry-Contact Relay Actuation & Anti-Short-Cycle Guard
1. Set thermostat mode to **`MANUAL`** and raise target temperature `2.0°C` above room temperature.
2. Wait for the `min_off_time_sec` countdown (`240s` default, or temporarily set to `10s` in Web UI `/control` during bench testing).
3. Listen for a crisp mechanical click from the `Omron G5LE-1` relay and confirm the flame icon appears on the LCD.
4. Measure continuity across **`COM`** and **`NO`**:
   * Expected when heating is `ON`: **`< 0.5 Ω` (closed circuit)**.
   * Expected across `COM` and `NC`: **`OL` (open circuit)**.
5. Lower target temperature below room temperature (after `min_on_time` elapses) and confirm `COM`–`NO` returns to **`OL` (open circuit)**.

### 3.4 External Hardware Watchdog (`TPL5010`) Fail-Safe Test
1. Verify `GPIO 17` (`WDT_DONE`) pulses periodically to keep the `TPL5010` satisfied during normal operation.
2. **Simulated Firmware Hang Test:** Temporarily disconnect the jumper wire between `GPIO 17` and `TPL5010 DONE` (or hold `BOOT` button to stall execution) while the relay is `ON`.
3. Wait up to **`60 seconds`**:
   * Expected: The `TPL5010` triggers a hardware reset on `ESP32-S3 EN`, immediately de-energizing the relay (`COM`–`NO` opens to `OL`) and rebooting the thermostat cleanly.
4. Reconnect the `GPIO 17` jumper wire after verification.

### 3.5 On-Demand Setup AP, DNS Captive Portal & Live Wi-Fi Verification
1. **Trigger Setup AP:** Either boot with blank NVS or hold **`BACK`** (`GPIO 38`) for **3 seconds** on Screen 0 until the top-left status bar displays **`AP: 192.168.4.1`**.
2. **Verify DNS Captive Portal Auto-Popup:** Connect a smartphone to the open **`OpenSmartThermo-Setup`** Wi-Fi network.
   * Expected: The phone's Captive Portal browser opens the OpenSmartThermo Web SPA automatically (or loads cleanly at `http://192.168.4.1`) with zero external internet connection.
3. **Verify Non-Destructive Live Wi-Fi Verification:**
   * Navigate to the **Network** tab and submit an **incorrect** Wi-Fi password.
   * Expected: After a short check (`<= 12s`), an error toast (`Connection failed. Check password.`) appears and **`OpenSmartThermo-Setup` remains active** so you are not locked out.
   * Submit your **valid** home Wi-Fi SSID and password.
   * Expected: A success toast (`Connected! IP: <LAN_IP>`) displays the assigned router IP address, and the Setup AP shuts down gracefully **5 seconds** later.

---

## 4. Wireless Sensor Nodes (`Seeed Studio XIAO ESP32C3`) — Hardware & Deep-Sleep Verification

> **CRITICAL HARDWARE WARNING:** Never exceed **`3.6V`** on the `3V3` pin, and **always disconnect the `2x AA` battery pack** before connecting USB-C to the `XIAO ESP32C3`.

### 4.1 Pre-Power Continuity & Voltage Divider Checks
1. Verify the external **U.FL antenna** is firmly snapped onto the `XIAO ESP32C3` antenna connector.
2. With batteries removed and USB unplugged, measure resistance across the `2x AA` battery holder leads (`+` to `-`). Expected: **`~2.0 MΩ`** (through the `1MΩ + 1MΩ` battery monitor divider).
3. Measure voltage of your `2x AA` battery pack before connecting it to the `3V3` pin:
   * `2x AA` Alkaline: **`3.0V – 3.25V`** (Safe)
   * `2x AA` Energizer Ultimate Lithium: **`3.2V – 3.55V`** (Safe, verify `< 3.60V`)
   * **Never use a `3.7V / 4.2V` Li-Ion battery on the `3V3` pin.**

### 4.2 Deep-Sleep Current Measurement (`µA` Check — Crucial for 2+ Year Battery Life)
1. Set your multimeter to the **`µA` (Microamp)** range (move the red probe to the `µA/mA` jack if required by your meter).
2. Connect the multimeter **in series** between the battery pack `+` wire and the `XIAO ESP32C3` **`3V3` pin**.
3. Wait `10–15 seconds` for the node to complete its boot measurement and auto-off display timeout (`8s`) and enter deep sleep.
4. Read the steady-state deep-sleep current on the multimeter:
   * **Target Pass Criterion:** **`15 µA – 30 µA`** (typically `~18–22 µA`).
   * **Troubleshooting if `> 50 µA`:**
     * Check that display `VCC` is wired to **`D3 (GPIO 5)`** and **NOT** permanently wired to `3V3`.
     * Confirm no floating pull-up resistors are back-powering the display over `SDA`/`SCL`.
     * Confirm the battery `+` lead is connected to **`3V3`** and **NOT** to the bottom `BAT` pad.

### 4.3 Unified Button (`D1 / GPIO 3`), Display & Marquee Scrolling Test
1. **Wake & Immediate Sync (Short Press `< 1s`):**
   * Press the button once while the screen is `OFF`.
   * Expected: Screen turns on within `< 0.5s`, displays `#ID`, `[LINK OK]` (if paired) or `[NO LINK]`, temperature, humidity, and battery `%` / voltage.
2. **Immediate Manual Screen Off (Second Short Press):**
   * While the screen is still illuminated, press the button a second time.
   * Expected: Screen turns off immediately (`0.0 µA` display draw) and node returns to deep sleep.
3. **Auto-Off Timeout Verification:**
   * Press the button once to wake the screen and leave it untouched.
   * Expected: Screen turns off automatically after the configured `display_timeout_s` (`8 seconds` default).
4. **Pairing & Long-Name Marquee Scroll Test:**
   * Open `+ Pair New Sensor` on the Thermostat Web UI (`http://opensmartthermo.local/sensors`).
   * Hold the wireless sensor button for **`>= 2 seconds`** until `[PAIRING]` blinks on the display.
   * Confirm pairing succeeds within `2–3 seconds` and displays `[LINK OK]`.
   * In the Web UI, assign a long name (`> 21 characters`, e.g., `"Master Bedroom North Wall"`) and short-press the node button.
   * Expected: The node syncs the new name and smoothly scrolls it horizontally across Line 2 while keeping `#ID`, `[LINK OK]`, temperature, and battery pinned.

---

## 5. Final Pre-Wall-Mounting Sign-Off Table

| Check ID | Verification Item | Expected Result | Pass / Fail |
| :--- | :--- | :--- | :--- |
| **CHK-01** | `COM/NO/NC` to `L/N` & `GND` isolation | `OL` (`> 10 MΩ`) | `[ ]` |
| **CHK-02** | Relay state when power is removed | `COM–NO` Open (`OL`) | `[ ]` |
| **CHK-03** | Relay state when heating is `ON` | `COM–NO` Closed (`< 0.5 Ω`) | `[ ]` |
| **CHK-04** | `TPL5010` hardware watchdog reset test | Reboots & opens relay within `60s` | `[ ]` |
| **CHK-05** | `DS3231` RTC time retention across power loss | Keeps accurate time with `CR2032` | `[ ]` |
| **CHK-06** | Wireless node deep-sleep current (`3V3` pin) | `15 µA – 30 µA` | `[ ]` |
| **CHK-07** | Wireless node screen on / manual off / auto-off | Wakes, toggles off, or times out cleanly | `[ ]` |
| **CHK-08** | Wireless range test at final room location | Reliable `[LINK OK]` & RSSI `> -85 dBm` | `[ ]` |
| **CHK-09** | 3-sec `BACK` AP toggle & Captive Portal | `AP: 192.168.4.1` toggles ON/OFF & verifies Wi-Fi live | `[ ]` |

