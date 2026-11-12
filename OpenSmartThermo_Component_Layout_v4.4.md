# OpenSmartThermo — Protoboard Component Layouts & Enclosure Guide (v4.4)

This guide defines the physical component placement, thermal isolation barriers, point-to-point wiring connections, and 3D-printable enclosure dimensions for the **Main Thermostat & Hub (`ESP32-S3`)** and the **Wireless Indoor/Outdoor Sensor Nodes (`Seeed Studio XIAO ESP32C3`)**.

---

## 1. Main Thermostat & Hub (`ESP32-S3` on `70mm × 90mm` Protoboard)

### 1.1 Thermal & Electrical Zoning Rules
* **Zone A — High-Voltage `230V AC` & Relay Zone (Top `28mm`):** Houses the `HLK-PM01` (`230V AC -> 5V DC`) isolated power supply, `2A` slow-blow fuse, `10D471K` MOV varistor, and the `5V` optically isolated dry-contact boiler relay (`COM` / `NO` / `NC`). Maintain a strict **`>= 6.0mm` air/creepage gap** (or route a physical isolation slot in the protoboard) between Zone A and all low-voltage `3.3V`/`5V` traces.
* **Zone B — Central Logic, UI & Watchdog Zone (Middle `46mm`):** Houses the `ESP32-S3-DevKitC-1`, `DS3231` RTC module, `TPL5010` hardware watchdog breakout, `ST7567` `128x64` Graphic LCD (mounted on `21.5mm` `M2.5` brass standoffs above the logic area), and the four tactile navigation buttons (`UP`, `DOWN`, `OK`, `BACK`) along the right/bottom UI edge.
* **Zone C — Isolated Sensor Tongue (Bottom-Left `16mm × 22mm`):** Houses the `SHT40` temperature and humidity breakout board. Cut or drill a **`2.0mm` wide U-shaped thermal isolation air slot** around three sides of the `SHT40` corner so heat from the `ESP32-S3` and `HLK-PM01` cannot conduct through the fiberglass board into the temperature sensor.

### 1.2 Top-View Protoboard Layout (`70mm W × 90mm H`)

```text
+-------------------------------------------------------------------+  0 mm
|  [230V AC IN]   [2A FUSE]   [MOV]      [BOILER DRY-CONTACT]       |
|   (L)   (N)                             (COM)  (NO)  (NC)         |
|                                                                   |
|  +-----------------------+             +-----------------------+  |
|  |   HLK-PM01            |             |   5V RELAY MODULE     |  |
|  |   230V AC -> 5V DC    |             |   (Opto-Isolated)     |  |
|  +-----------------------+             +-----------------------+  |
|===================================================================| 28 mm
|  :::::::::::: >= 6.0mm CREEPAGE / ISOLATION BARRIER ::::::::::::  |
|===================================================================| 34 mm
|  +---------------+   +-----------------------------------+        |
|  | TPL5010 WDT   |   |        ESP32-S3-DevKitC-1         |        |
|  | (GPIO 17)     |   |   (Mounted in 2x22 Female Headers)|  [UP]  |
|  +---------------+   |                                   |        |
|                      |   [ST7567 128x64 LCD mounted      |  [DN]  |
|  +---------------+   |    above on 21.5mm M2.5 Standoffs]|        |
|  | DS3231 RTC    |   |                                   |  [OK]  |
|  | (CR2032 Back) |   +-----------------------------------+        |
|  +---------------+                                         [BACK] |
|  +-------------------+                                            | 74 mm
|  | SHT40 SENSOR      |   <-- 2.0mm Routed Air-Gap Slot            |
|  | (Bottom-Left Vent)|   (Keep copper & heat away from SHT40)     |
|  +-------------------+                                            |
+-------------------------------------------------------------------+ 90 mm
0 mm                                                               70 mm
```

### 1.3 Point-to-Point Wiring Reference (Hub)

| Module / Component | Pin | Connects To (`ESP32-S3` / Power) | Notes |
| :--- | :--- | :--- | :--- |
| **`HLK-PM01` Output** | `+Vo` / `-Vo` | `5V` Rail / `GND` Rail | Add `100µF` electrolytic + `100nF` ceramic decoupling caps |
| **`SHT40` & `DS3231`** | `SDA` / `SCL` | `GPIO 8` / `GPIO 9` | Shared I2C bus (`3.3V` logic, `4.7kΩ` pull-ups) |
| **`ST7567` LCD** | `SCK` / `MOSI` / `CS` / `DC` / `RST` / `BL` | `GPIO 12` / `11` / `10` / `13` / `14` / `21` | Hardware SPI + PWM backlight (`GPIO 21`) |
| **`5V` Relay Module** | `IN` / `VCC` / `GND` | `GPIO 16` / `5V` Rail / `GND` | Active-HIGH trigger; `COM`/`NO` isolated from PCB |
| **`TPL5010` Watchdog** | `DONE` / `RST` / `VDD` | `GPIO 17` / `EN` (`RST`) / `3.3V` | Pet every `30s`; hardware-resets ESP32-S3 if frozen `> 60s` |
| **Nav Buttons (`4x`)** | `UP` / `DN` / `OK` / `BACK` | `GPIO 4` / `5` / `6` / `7` | Active-LOW to `GND` (internal `INPUT_PULLUP` enabled) |

---

## 2. Wireless Sensor Node (`Seeed Studio XIAO ESP32C3` on `55mm × 68mm` Protoboard)

### 2.1 Component Placement & Battery Integration Rules
* **Front Face (`55mm × 68mm`):**
  * **Top RF Keep-Out Zone (`10mm`):** Mount the `Seeed Studio XIAO ESP32C3` at the top-left edge with its `U.FL` antenna connector facing upward. Route the `U.FL` coaxial pigtail to the stick-on internal antenna mounted along the top inner wall—away from the rear AA battery metal springs.
  * **Center Display & Button Zone:** Mount the `0.96"` (`128x64`) I2C OLED/LCD display in a `2.54mm` 4-pin female header socket centered on the front face, with the single `PAIR / SCREEN` tactile pushbutton positioned directly below it for intuitive thumb operation.
  * **Battery Voltage Divider (`D0 / GPIO 2`):** Place the two `1MΩ` (`1%`) resistors and `100nF` filter capacitor beside the `XIAO ESP32C3` (`BAT+` -> `1MΩ` -> `D0` -> `1MΩ` -> `GND`).
  * **Bottom Sensor Island:** Mount the `SHT40` breakout at the bottom-center edge, isolated by a `2.0mm` air-gap slot right above the bottom enclosure intake vents.
* **Rear Face (`55mm × 68mm`):**
  * **Vertical `2x AA` Battery Holder (`32mm W × 58mm H × 15mm D`):** Mount the `2x AA` (`3.0V`) battery holder vertically along the center-back of the protoboard using two `M2.5` nylon screws/nuts (insulated with Kapton tape inside the holder). Pass the red (`+3.0V`) and black (`GND`) leads through two `1.5mm` strain-relief holes directly to the front `3V3` and `GND` rails.

### 2.2 Front & Rear Protoboard Layouts (`55mm W × 68mm H`)

```text
         FRONT FACE (Component Side)                     REAR FACE (Battery Side)
+-------------------------------------------+   +-------------------------------------------+  0 mm
| [U.FL ANT KEEPOUT ZONE - NO COPPER/METAL] |   |                                           |
|  +---------------+   +-----------------+  |   |       +---------------------------+       |
|  | XIAO ESP32C3  |   | 2x 1MΩ Divider  |  |   |       |                           |       |
|  | (Top-Left)    |   | + 100nF -> D0   |  |   |       |     2x AA BATTERY HOLDER  |       |
|  +---------------+   +-----------------+  |   |       |     (32mm W x 58mm H x    |       |
|  +-------------------------------------+  |   |       |      15mm D, Vertical)    |       |
|  |     0.96" 128x64 I2C DISPLAY        |  |   |       |                           |       |
|  |     (Power-Gated via D3 / GPIO 5)   |  |   |       |     [+] Battery (+) ----> o (3V3) |
|  +-------------------------------------+  |   |       |     [-] Battery (-) ----> o (GND) |
|             [ PAIR / SCREEN ]             |   |       |                           |       |
|             (Button -> D1 / GPIO 3)       |   |       +---------------------------+       |
|  ======== 2.0mm Air-Gap Slot ===========  |   |                                           |
|             +---------------+             |   |       (Clear of SHT40 Bottom Vent)        |
|             | SHT40 SENSOR  |             |   |                                           |
+-------------------------------------------+   +-------------------------------------------+ 68 mm
0 mm                                       55 mm 0 mm                                       55 mm
```

### 2.3 Point-to-Point Wiring Reference (Wireless Node)

| Module / Component | Pin | Connects To (`XIAO ESP32C3`) | Notes |
| :--- | :--- | :--- | :--- |
| **`2x AA` Battery Pack** | `+3.0V` / `GND` | `3V3` Pin / `GND` Pin | Bypasses LDO for `< 25 µA` deep sleep; disconnect before USB-C |
| **Battery Divider** | Midpoint | `D0 / GPIO 2` | `BAT+` -> `1MΩ` -> `D0` -> `1MΩ` -> `GND` (`100nF` across lower `1MΩ`) |
| **`PAIR / SCREEN` Button** | Switch Pins | `D1 / GPIO 3` & `GND` | RTC wake-up pin (`INPUT_PULLUP`); `< 1s` screen toggle, `>= 2s` pair |
| **`0.96"` LCD / OLED** | `VCC` / `GND` | `D3 / GPIO 5` / `GND` | Hardware power-gated (`OUTPUT HIGH` = ON, `INPUT` = `0 µA` sleep) |
| **`0.96"` Display & `SHT40`** | `SDA` / `SCL` | `D4 / GPIO 6` / `D5 / GPIO 7` | Shared I2C bus (`D4`/`D5` set to `INPUT` high-Z before deep sleep) |

---

## 3. 3D-Printable Enclosure Dimensions & Cutout Guide

### 3.1 Main Thermostat & Hub Enclosure (`88.0mm W × 110.0mm H × 32.0mm D`)

* **Print Material & Wall Thickness:** Matte White PETG or ASA (`2.0mm` outer shell walls, `20%` gyroid infill, `0.20mm` layer height).
* **Why `32.0mm` Depth + LCD Standoffs Are Required:** A shallower enclosure (`27.0mm` or less) cannot accommodate the **`18.5mm` tall `5V` relay module** (plus screw-terminal wire bend) or the **`15.0mm` `HLK-PM01` AC-DC PSU** mounted on the main PCB. Therefore, the enclosure must remain **`32.0mm` deep (`28.0mm` inner cavity)**, and **`21.5mm` `M2.5` brass standoffs + stacked `2.54mm` female pin headers** are required to elevate the `ST7567` LCD by `+10.5mm` so its front glass sits completely flush with the outer front plate.
* **Internal Depth Stack-Up (`28.0mm` Inner Cavity):**
  * Rear solder-pin clearance standoffs: `5.0mm`
  * Protoboard PCB thickness: `1.6mm`
  * Front component headroom (fits `18.5mm` relay + `15.0mm` `HLK-PM01` + `13.5mm` `ESP32-S3`): `21.4mm`
  * Total inner cavity: `5.0 + 1.6 + 21.4 = 28.0mm` (`+ 2.0mm` rear wall + `2.0mm` front wall = **`32.0mm` outer depth**).

| Feature / Cutout | Dimensions (`W × H`) | Center / Position Coordinates (from Bottom-Left Outer Corner) | Mechanical & Mounting Notes |
| :--- | :--- | :--- | :--- |
| **Outer Enclosure Shell** | `88.0mm × 110.0mm × 32.0mm` | Origin `(0, 0, 0)` | Two-piece snap-fit + `4x M2.5` corner screws |
| **Internal PCB Standoffs (`4x`)** | `Ø 5.5mm` boss, `5.0mm` tall | `(9.0, 10.0)`, `(79.0, 10.0)`, `(9.0, 100.0)`, `(79.0, 100.0)` | Heat-set `M2.5` brass inserts; centers `70×90mm` PCB |
| **`ST7567` LCD Viewport** | `54.0mm × 29.0mm` | Center `X = 39.0mm`, `Y = 56.0mm` | Includes `0.8mm` inner step-bezel pocket; LCD elevated on `21.5mm` `M2.5` brass standoffs to sit flush with front face |
| **4 Navigation Button Holes** | `4x Ø 6.2mm` round holes | `X = 76.0mm`; `Y = 68.0, 58.0, 48.0, 38.0mm` | Fitted with 3D-printed captive plunger caps (`Ø 5.8mm` shaft, `Ø 8.0mm × 1.2mm` inner retaining flange) |
| **Bottom Cool Air Intake Slots** | `6x (3.0mm × 10.0mm)` slots | Bottom wall `Z = 6.0mm`, `X = 12.0mm to 42.0mm` | Directly under the isolated `SHT40` sensor island |
| **Top Warm Air Exhaust Slots** | `8x (3.0mm × 10.0mm)` slots | Top wall `Z = 18.0mm`, `X = 14.0mm to 74.0mm` | Vents heat from `HLK-PM01` and `ESP32-S3` |
| **Internal `SHT40` Baffle Wall** | `2.0mm` thick × `18.0mm` tall | Encloses `X = 7.0–28.0mm`, `Y = 8.0–26.0mm` | Seals against PCB air-gap slot to block internal heat |
| **Rear Mains Wire Entry** | `14.0mm × 10.0mm` oval | Rear wall `X = 44.0mm`, `Y = 92.0mm` | Aligns with `230V AC` & boiler terminals + EU `Ø 60mm` box holes |

---

### 3.2 Wireless Sensor Node Enclosure (`72.0mm W × 84.0mm H × 32.1mm D`)

* **Print Material:**
  * **Indoor Nodes:** Matte White PETG or PLA+ (`2.0mm` walls, `20%` infill).
  * **Outdoor Nodes:** UV-stabilized **ASA** (`2.5mm` walls) housed inside a vented Stevenson radiation shield, with a `10mm` adhesive **`ePTFE` hydrophobic membrane** covering the bottom sensor vent.
* **Flush LCD Fitment Verification (No Extra Standoffs Needed):** Reducing the total outer depth from `36.0mm` to **`32.1mm`** (`28.1mm` inner cavity) allows the `0.96"` display (`4.2mm` module mounted in a `2.5mm` `2.54mm` female header socket = `6.7mm` above PCB) to sit **completely flush with the outer front face** (`6.7mm` + `2.0mm` front wall bezel overlap = `8.7mm` front headroom) without requiring extra standoffs, while still leaving comfortable clearance for every other component (`XIAO ESP32C3` + `U.FL` plug = `7.2mm` front height; `2x AA` holder = `15.0mm` rear thickness).
* **Internal Depth Stack-Up (`28.1mm` Inner Cavity):**
  * Rear `2x AA` battery cavity (`15.0mm` holder + `2.5mm` swap clearance): `17.5mm` (`M2.5` rear standoffs)
  * Protoboard PCB thickness: `1.6mm`
  * Front component headroom (`XIAO ESP32C3` `7.2mm`, `SHT40` `4.5mm`, `0.96"` display `6.7mm` + `2.0mm` lid lip): `9.0mm`
  * Total inner cavity: `17.5 + 1.6 + 9.0 = 28.1mm` (`+ 2.0mm` rear wall + `2.0mm` front wall = **`32.1mm` outer depth**).

| Feature / Cutout | Dimensions (`W × H`) | Center / Position Coordinates (from Bottom-Left Outer Corner) | Mechanical & Mounting Notes |
| :--- | :--- | :--- | :--- |
| **Outer Enclosure Shell** | `72.0mm × 84.0mm × 32.1mm` | Origin `(0, 0, 0)` | Rear removable battery cover (`42mm × 64mm` slide/screw hatch) |
| **Internal PCB Standoffs (`4x`)** | `Ø 5.0mm` boss, `17.5mm` tall | `(8.5, 8.0)`, `(63.5, 8.0)`, `(8.5, 76.0)`, `(63.5, 76.0)` | Holds `55×68mm` PCB; `6.5mm` side gap protects `U.FL` coax bend |
| **`0.96"` Display Viewport** | `24.0mm × 13.0mm` | Center `X = 36.0mm`, `Y = 44.0mm` | `0.8mm` inner step-bezel pocket; display sits flush in `2.54mm` socket |
| **`PAIR / SCREEN` Button Hole** | `Ø 6.2mm` round hole | Center `X = 36.0mm`, `Y = 26.0mm` | Fitted with 3D-printed captive plunger cap (`Ø 5.8mm` shaft, `Ø 8.0mm` base) |
| **Side USB-C Service Port** | `10.0mm × 4.5mm` pill slot | Left wall `Y = 66.0mm`, `Z = 21.5mm` | Allows firmware updates (remove AA batteries first) |
| **Convection Airflow Vents** | `5x (2.5mm × 8.0mm)` slots | Bottom wall (`Y = 0`) & lower side walls (`Y = 6–16mm`) | Cross-flow ventilation around isolated bottom `SHT40` island |
| **Rear Wall-Mount Keyhole** | `Ø 8.0mm` -> `Ø 4.0mm` slot | Rear cover center `X = 36.0mm`, `Y = 68.0mm` | Flush screw/nail wall hanging |
