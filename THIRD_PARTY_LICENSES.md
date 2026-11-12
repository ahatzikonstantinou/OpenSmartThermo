# Third-Party Open-Source Licenses & Attributions

OpenSmartThermo incorporates and links against several open-source software libraries and frameworks. While OpenSmartThermo itself is licensed under the **PolyForm Noncommercial License 1.0.0** (see `LICENSE`), the third-party components listed below remain under their respective open-source licenses.

All listed third-party licenses permit commercial and non-commercial redistribution, provided their copyright notices and license conditions are preserved when distributing source code, compiled firmware binaries (`.bin`), or pre-flashed hardware units.

---

## Summary of Third-Party Components

| Component / Library | Author / Copyright Holder | License | Repository / Source |
| :--- | :--- | :--- | :--- |
| **Arduino-ESP32 & ESP-IDF** | Espressif Systems (Shanghai) Co., Ltd. | Apache License 2.0 / MIT | https://github.com/espressif/arduino-esp32 |
| **U8g2** | Oliver Kraus (`olikraus`) | BSD 2-Clause License | https://github.com/olikraus/u8g2 |
| **Adafruit SHT4x Library** | Adafruit Industries | BSD 3-Clause License | https://github.com/adafruit/Adafruit_SHT4x |
| **Adafruit BusIO** | Adafruit Industries | MIT License | https://github.com/adafruit/Adafruit_BusIO |
| **RTClib** | Adafruit Industries & JeeLabs | MIT License | https://github.com/adafruit/RTClib |
| **ESPAsyncWebServer** | Hristo Gochkov (`me-no-dev`) / ESP32Async | LGPL-3.0 | https://github.com/ESP32Async/ESPAsyncWebServer |
| **AsyncTCP** | Hristo Gochkov (`me-no-dev`) / ESP32Async | LGPL-3.0 | https://github.com/ESP32Async/AsyncTCP |
| **ArduinoJson** | Benoit Blanchon | MIT License | https://github.com/bblanchon/ArduinoJson |
| **PubSubClient** | Nick O'Leary (`knolleary`) | MIT License | https://github.com/knolleary/pubsubclient |

---

## License Texts & Notices

### 1. MIT License (`Adafruit BusIO`, `RTClib`, `ArduinoJson`, `PubSubClient`)

```text
Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

---

### 2. BSD 2-Clause License (`U8g2`)

```text
Universal 8bit Graphics Library (https://github.com/olikraus/u8g2/)
Copyright (c) 2016, olikraus@gmail.com
All rights reserved.

Redistribution and use in source and binary forms, with or without modification,
are permitted provided that the following conditions are met:

* Redistributions of source code must retain the above copyright notice, this
  list of conditions and the following disclaimer.

* Redistributions in binary form must reproduce the above copyright notice, this
  list of conditions and the following disclaimer in the documentation and/or
  other materials provided with the distribution.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND
ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED
WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE FOR
ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES
(INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES;
LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON
ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
(INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS
SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
```

---

### 3. BSD 3-Clause License (`Adafruit SHT4x Library`)

```text
Software License Agreement (BSD License)
Copyright (c) 2020, Adafruit Industries
All rights reserved.

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are met:
1. Redistributions of source code must retain the above copyright
   notice, this list of conditions and the following disclaimer.
2. Redistributions in binary form must reproduce the above copyright
   notice, this list of conditions and the following disclaimer in the
   documentation and/or other materials provided with the distribution.
3. Neither the name of the copyright holders nor the
   names of its contributors may be used to endorse or promote products
   derived from this software without specific prior written permission.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS ''AS IS'' AND ANY
EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED
WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER BE LIABLE FOR ANY
DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES
(INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES;
LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND
ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
(INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS
SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
```

---

### 4. Apache License 2.0 (`Arduino-ESP32` & `ESP-IDF`)

```text
Copyright (c) 2015-2026 Espressif Systems (Shanghai) CO LTD

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```

---

### 5. GNU Lesser General Public License v3.0 (`ESPAsyncWebServer` & `AsyncTCP`)

`ESPAsyncWebServer` and `AsyncTCP` are licensed under the **GNU Lesser General Public License v3.0 (LGPL-3.0)**.

* Full license text: https://www.gnu.org/licenses/lgpl-3.0.en.html
* Source repositories:
  * https://github.com/ESP32Async/ESPAsyncWebServer
  * https://github.com/ESP32Async/AsyncTCP
* **Device Relinking Notice (LGPL-3.0 Section 4):** End users who receive compiled firmware binaries or hardware pre-flashed with OpenSmartThermo can upload modified versions of these LGPL-3.0 libraries onto the ESP32-S3 device using the built-in Web UI Over-The-Air (OTA) firmware update tool (`/system`) or via USB-C using PlatformIO / `esptool`.
