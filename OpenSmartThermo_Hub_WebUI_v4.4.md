# OpenSmartThermo v4.4 — Main Thermostat & Hub Embedded Web UI (`data/index.html`)

This file contains the complete, self-contained single-page web application (`data/index.html`) served directly from the **ESP32-S3** LittleFS partition.

## Key Features
* **Automatic Light & Dark Mode:** Follows system preference (`@media (prefers-color-scheme: dark)`) with Google Material 3 clean light surfaces by day and high-contrast slate surfaces by night, plus a manual `☀️ / 🌙` toggle button in the top header saved to `localStorage`.
* **Smartphone-Optimized Bottom Tab Bar:** Automatically switches to a fixed bottom navigation bar with safe-area padding on screens `<= 680px`.
* **Empty-Slot & Chronological Weekly Schedule:** Supports leaving unused schedule slots completely blank (`--:--`), visually flags out-of-order times (`<= previous valid slot`) that the firmware ignores, and provides a one-tap **Clear (`×`)** button per slot.
* **Zero External Dependencies:** No CDN links, fonts, or external JS bundles—loads instantaneously offline over your local network or AP setup hotspot.

---

## `data/index.html`

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0, viewport-fit=cover" />
  <title>OpenSmartThermo</title>
  <style>
    :root {
      --bg: #f8fafc;
      --surface: #ffffff;
      --surface-alt: #f1f5f9;
      --border: #e2e8f0;
      --text: #0f172a;
      --text-muted: #64748b;
      --primary: #1a73e8;
      --primary-soft: #e8f0fe;
      --heat: #ea580c;
      --heat-soft: #ffedd5;
      --ok: #16a34a;
      --ok-soft: #dcfce7;
      --warn: #d97706;
      --warn-soft: #fef3c7;
      --danger: #dc2626;
      --danger-soft: #fee2e2;
      --shadow: 0 1px 3px rgba(15, 23, 42, 0.08), 0 1px 2px rgba(15, 23, 42, 0.04);
      --radius: 16px;
    }

    @media (prefers-color-scheme: dark) {
      :root:not([data-theme="light"]) {
        --bg: #0f172a;
        --surface: #1e293b;
        --surface-alt: #0f172a;
        --border: #334155;
        --text: #f8fafc;
        --text-muted: #94a3b8;
        --primary: #38bdf8;
        --primary-soft: rgba(56, 189, 248, 0.15);
        --heat: #f97316;
        --heat-soft: rgba(249, 115, 22, 0.16);
        --ok: #22c55e;
        --ok-soft: rgba(34, 197, 94, 0.16);
        --warn: #f59e0b;
        --warn-soft: rgba(245, 158, 11, 0.16);
        --danger: #ef4444;
        --danger-soft: rgba(239, 68, 68, 0.16);
        --shadow: 0 2px 8px rgba(0, 0, 0, 0.35);
      }
    }

    :root[data-theme="dark"] {
      --bg: #0f172a;
      --surface: #1e293b;
      --surface-alt: #0f172a;
      --border: #334155;
      --text: #f8fafc;
      --text-muted: #94a3b8;
      --primary: #38bdf8;
      --primary-soft: rgba(56, 189, 248, 0.15);
      --heat: #f97316;
      --heat-soft: rgba(249, 115, 22, 0.16);
      --ok: #22c55e;
      --ok-soft: rgba(34, 197, 94, 0.16);
      --warn: #f59e0b;
      --warn-soft: rgba(245, 158, 11, 0.16);
      --danger: #ef4444;
      --danger-soft: rgba(239, 68, 68, 0.16);
      --shadow: 0 2px 8px rgba(0, 0, 0, 0.35);
    }

    * { box-sizing: border-box; -webkit-tap-highlight-color: transparent; }
    body {
      margin: 0;
      font-family: system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
      background: var(--bg);
      color: var(--text);
      line-height: 1.45;
      padding-bottom: 84px;
    }

    header {
      position: sticky;
      top: 0;
      z-index: 20;
      background: var(--surface);
      border-bottom: 1px solid var(--border);
      padding: 12px 16px;
      display: flex;
      align-items: center;
      justify-content: space-between;
      gap: 12px;
    }

    .brand {
      display: flex;
      align-items: center;
      gap: 10px;
      font-weight: 700;
      font-size: 1.05rem;
    }

    .status-dot {
      width: 10px;
      height: 10px;
      border-radius: 50%;
      background: var(--ok);
      box-shadow: 0 0 0 3px var(--ok-soft);
    }
    .status-dot.offline {
      background: var(--danger);
      box-shadow: 0 0 0 3px var(--danger-soft);
    }

    .header-actions {
      display: flex;
      align-items: center;
      gap: 8px;
    }

    .icon-btn {
      background: var(--surface-alt);
      border: 1px solid var(--border);
      color: var(--text);
      border-radius: 10px;
      padding: 7px 10px;
      font-size: 0.9rem;
      cursor: pointer;
    }

    nav.top-nav {
      display: flex;
      gap: 6px;
      padding: 10px 16px;
      background: var(--surface);
      border-bottom: 1px solid var(--border);
      overflow-x: auto;
    }

    nav.top-nav button {
      border: none;
      background: transparent;
      color: var(--text-muted);
      font-weight: 600;
      padding: 8px 14px;
      border-radius: 999px;
      cursor: pointer;
      white-space: nowrap;
      font-size: 0.9rem;
    }

    nav.top-nav button.active {
      background: var(--primary-soft);
      color: var(--primary);
    }

    nav.bottom-nav {
      display: none;
    }

    @media (max-width: 680px) {
      nav.top-nav { display: none; }
      nav.bottom-nav {
        position: fixed;
        bottom: 0;
        left: 0;
        right: 0;
        z-index: 30;
        display: grid;
        grid-template-columns: repeat(6, 1fr);
        background: var(--surface);
        border-top: 1px solid var(--border);
        padding: 6px 4px calc(6px + env(safe-area-inset-bottom));
      }
      nav.bottom-nav button {
        border: none;
        background: transparent;
        color: var(--text-muted);
        display: flex;
        flex-direction: column;
        align-items: center;
        gap: 3px;
        font-size: 0.7rem;
        font-weight: 600;
        padding: 4px 2px;
        cursor: pointer;
      }
      nav.bottom-nav button.active {
        color: var(--primary);
      }
      nav.bottom-nav svg {
        width: 20px;
        height: 20px;
        stroke: currentColor;
        fill: none;
        stroke-width: 2;
      }
    }

    main {
      max-width: 960px;
      margin: 0 auto;
      padding: 16px;
    }

    .page { display: none; }
    .page.active { display: block; }

    .grid {
      display: grid;
      grid-template-columns: repeat(auto-fit, minmax(290px, 1fr));
      gap: 16px;
    }

    .card {
      background: var(--surface);
      border: 1px solid var(--border);
      border-radius: var(--radius);
      padding: 18px;
      box-shadow: var(--shadow);
    }

    .card h2, .card h3 {
      margin: 0 0 12px 0;
      font-size: 1rem;
      font-weight: 700;
    }

    .hero-temp {
      display: flex;
      align-items: baseline;
      justify-content: space-between;
      margin: 8px 0 16px;
    }

    .big-reading {
      font-size: 3.1rem;
      font-weight: 800;
      letter-spacing: -0.03em;
    }

    .badge {
      display: inline-flex;
      align-items: center;
      gap: 6px;
      padding: 5px 11px;
      border-radius: 999px;
      font-size: 0.8rem;
      font-weight: 700;
    }
    .badge.heat { background: var(--heat-soft); color: var(--heat); }
    .badge.idle { background: var(--surface-alt); color: var(--text-muted); }
    .badge.ok { background: var(--ok-soft); color: var(--ok); }
    .badge.warn { background: var(--warn-soft); color: var(--warn); }
    .badge.danger { background: var(--danger-soft); color: var(--danger); }

    .setpoint-controls {
      display: flex;
      align-items: center;
      justify-content: space-between;
      background: var(--surface-alt);
      border: 1px solid var(--border);
      border-radius: 14px;
      padding: 10px 14px;
      margin-bottom: 14px;
    }

    .step-btn {
      width: 48px;
      height: 48px;
      border-radius: 12px;
      border: 1px solid var(--border);
      background: var(--surface);
      color: var(--text);
      font-size: 1.5rem;
      font-weight: 700;
      cursor: pointer;
    }
    .step-btn:active { transform: scale(0.96); }

    .mode-grid {
      display: grid;
      grid-template-columns: repeat(4, 1fr);
      gap: 8px;
    }

    .mode-btn {
      padding: 10px 6px;
      border-radius: 12px;
      border: 1px solid var(--border);
      background: var(--surface-alt);
      color: var(--text);
      font-weight: 600;
      font-size: 0.82rem;
      cursor: pointer;
    }
    .mode-btn.active {
      background: var(--primary);
      border-color: var(--primary);
      color: #ffffff;
    }

    .row {
      display: flex;
      align-items: center;
      justify-content: space-between;
      gap: 10px;
      padding: 10px 0;
      border-bottom: 1px solid var(--border);
    }
    .row:last-child { border-bottom: none; }

    label { font-size: 0.88rem; color: var(--text-muted); display: block; margin-bottom: 4px; }
    input, select {
      width: 100%;
      padding: 9px 11px;
      border-radius: 10px;
      border: 1px solid var(--border);
      background: var(--surface-alt);
      color: var(--text);
      font-size: 0.92rem;
    }

    .btn {
      display: inline-flex;
      align-items: center;
      justify-content: center;
      gap: 6px;
      padding: 10px 16px;
      border-radius: 11px;
      border: none;
      background: var(--primary);
      color: #ffffff;
      font-weight: 600;
      font-size: 0.9rem;
      cursor: pointer;
    }
    .btn.secondary {
      background: var(--surface-alt);
      color: var(--text);
      border: 1px solid var(--border);
    }
    .btn.danger {
      background: var(--danger-soft);
      color: var(--danger);
    }
    .btn.sm { padding: 6px 10px; font-size: 0.8rem; border-radius: 8px; }

    .day-pills {
      display: flex;
      gap: 6px;
      overflow-x: auto;
      margin-bottom: 14px;
    }
    .day-pills button {
      padding: 8px 14px;
      border-radius: 10px;
      border: 1px solid var(--border);
      background: var(--surface-alt);
      color: var(--text);
      font-weight: 600;
      cursor: pointer;
    }
    .day-pills button.active {
      background: var(--primary);
      border-color: var(--primary);
      color: #fff;
    }

    .slot-card {
      display: grid;
      grid-template-columns: 58px 1.1fr 0.9fr 1.1fr 38px;
      gap: 8px;
      align-items: center;
      padding: 10px;
      border: 1px solid var(--border);
      border-radius: 12px;
      margin-bottom: 8px;
      background: var(--surface-alt);
    }
    .slot-card.empty { opacity: 0.72; }
    .slot-card.out-of-order {
      border-color: var(--warn);
      background: var(--warn-soft);
    }

    @media (max-width: 600px) {
      .slot-card {
        grid-template-columns: 1fr 1fr;
      }
      .slot-header-mobile {
        grid-column: 1 / -1;
        display: flex;
        justify-content: space-between;
        align-items: center;
      }
    }

    .toast {
      position: fixed;
      bottom: 86px;
      left: 50%;
      transform: translateX(-50%);
      background: #0f172a;
      color: #f8fafc;
      padding: 10px 18px;
      border-radius: 999px;
      font-size: 0.88rem;
      font-weight: 600;
      box-shadow: 0 6px 20px rgba(0, 0, 0, 0.25);
      opacity: 0;
      pointer-events: none;
      transition: opacity 0.2s ease;
      z-index: 50;
    }
    .toast.show { opacity: 1; }
  </style>
</head>
<body>
  <header>
    <div class="brand">
      <span id="wsDot" class="status-dot"></span>
      <span id="hubTitle">OpenSmartThermo</span>
    </div>
    <div class="header-actions">
      <span id="clockLabel" style="font-size:0.85rem;color:var(--text-muted);font-weight:600;">--:--</span>
      <button class="icon-btn" id="themeToggle" title="Toggle Light/Dark Theme">☀️</button>
    </div>
  </header>

  <nav class="top-nav" id="topNav">
    <button class="active" data-page="dashboard">Dashboard</button>
    <button data-page="sensors">Sensors</button>
    <button data-page="schedule">Schedule</button>
    <button data-page="control">Control</button>
    <button data-page="network">Network</button>
    <button data-page="system">System</button>
  </nav>

  <main>
    <!-- PAGE 1: DASHBOARD -->
    <section id="page-dashboard" class="page active">
      <div class="grid">
        <div class="card">
          <div style="display:flex;justify-content:space-between;align-items:center;">
            <span style="font-size:0.85rem;color:var(--text-muted);font-weight:600;">INDOOR TEMPERATURE</span>
            <span id="relayBadge" class="badge idle">IDLE</span>
          </div>
          <div class="hero-temp">
            <div class="big-reading" id="indoorTemp">--.-°C</div>
            <div style="text-align:right;">
              <div style="font-size:0.85rem;color:var(--text-muted);">Humidity</div>
              <div style="font-weight:700;" id="indoorHum">--%</div>
            </div>
          </div>

          <div class="setpoint-controls">
            <button class="step-btn" onclick="adjustTemp(-0.5)">−</button>
            <div style="text-align:center;">
              <div style="font-size:0.75rem;color:var(--text-muted);font-weight:700;">TARGET SETPOINT</div>
              <div style="font-size:1.65rem;font-weight:800;" id="targetTemp">--.-°C</div>
            </div>
            <button class="step-btn" onclick="adjustTemp(0.5)">+</button>
          </div>

          <div class="mode-grid">
            <button class="mode-btn" id="mode-AUTO" onclick="setMode('AUTO')">AUTO</button>
            <button class="mode-btn" id="mode-MANUAL" onclick="setMode('MANUAL')">MANUAL</button>
            <button class="mode-btn" id="mode-BOOST" onclick="setMode('BOOST')">BOOST +1h</button>
            <button class="mode-btn" id="mode-OFF" onclick="setMode('OFF')">OFF</button>
          </div>
          <div id="lockoutBanner" style="margin-top:12px;display:none;"></div>
        </div>

        <div class="card">
          <h2>Active Sensors & Outdoor</h2>
          <div id="dashboardSensors"></div>
        </div>
      </div>
    </section>

    <!-- PAGE 2: SENSORS & PAIRING -->
    <section id="page-sensors" class="page">
      <div class="card" style="margin-bottom:16px;">
        <div style="display:flex;justify-content:space-between;align-items:center;flex-wrap:wrap;gap:10px;">
          <div>
            <h2 style="margin-bottom:4px;">Wireless Sensor Nodes (ESP-NOW)</h2>
            <div style="font-size:0.85rem;color:var(--text-muted);">Hold PAIR button on sensor for 2s after starting pairing.</div>
          </div>
          <button class="btn" id="pairBtn" onclick="startPairing()">+ Pair New Sensor</button>
        </div>
        <div id="pairStatus" style="margin-top:10px;display:none;" class="badge warn">Pairing window active (60s)...</div>
      </div>
      <div id="sensorList" class="grid"></div>
    </section>

    <!-- PAGE 3: WEEKLY SCHEDULE -->
    <section id="page-schedule" class="page">
      <div class="card">
        <div style="display:flex;justify-content:space-between;align-items:center;flex-wrap:wrap;gap:8px;margin-bottom:12px;">
          <h2 style="margin:0;">Weekly Heating Schedule</h2>
          <div style="display:flex;gap:6px;flex-wrap:wrap;">
            <button class="btn secondary sm" onclick="copySchedule('weekdays')">Copy to Mon–Fri</button>
            <button class="btn secondary sm" onclick="copySchedule('weekend')">Copy to Sat–Sun</button>
            <button class="btn secondary sm" onclick="copySchedule('all')">Copy to All</button>
          </div>
        </div>

        <p style="font-size:0.83rem;color:var(--text-muted);margin-top:0;">
          Leave unused slots blank (<code>--:--</code>). Only filled slots in chronological order are evaluated.
        </p>

        <div class="day-pills" id="dayPills"></div>
        <div id="scheduleSlots"></div>

        <div style="margin-top:14px;display:flex;justify-content:flex-end;">
          <button class="btn" onclick="saveSchedule()">Save Day Schedule</button>
        </div>
      </div>
    </section>

    <!-- PAGE 4: HEATING CONTROL -->
    <section id="page-control" class="page">
      <div class="grid">
        <div class="card">
          <h2>Control Algorithm</h2>
          <div style="margin-bottom:12px;">
            <label>Regulation Mode</label>
            <select id="ctrlAlgo">
              <option value="0">Dynamic Self-Learning Hysteresis</option>
              <option value="1">Static Hysteresis</option>
              <option value="2">TPI (Time Proportional & Integral)</option>
            </select>
          </div>
          <div style="margin-bottom:12px;">
            <label>Hysteresis Band (±°C)</label>
            <input type="number" step="0.1" min="0.1" max="2.0" id="ctrlHyst" />
          </div>
          <div style="margin-bottom:12px;">
            <label>Sensor Aggregation</label>
            <select id="ctrlAgg">
              <option value="0">Single Primary Sensor</option>
              <option value="1">Average All Indoor</option>
              <option value="2">Weighted Indoor Average</option>
              <option value="3">Coldest Room (Min)</option>
              <option value="4">Warmest Room (Max)</option>
            </select>
          </div>
          <div style="display:flex;justify-content:space-between;align-items:center;margin-top:10px;">
            <span style="font-size:0.85rem;color:var(--text-muted);">Learned Overshoot: <strong id="learnedOvershoot">0.00°C</strong></span>
            <button class="btn secondary sm" onclick="resetLearning()">Reset Learning</button>
          </div>
        </div>

        <div class="card">
          <h2>Weather & Fail-Safes</h2>
          <div style="margin-bottom:12px;">
            <label>Outdoor Warm-Weather Lockout (°C, 0 = disabled)</label>
            <input type="number" step="0.5" min="0" max="30" id="ctrlWarmLock" />
          </div>
          <div style="margin-bottom:12px;">
            <label>Frost Protection Floor (°C)</label>
            <input type="number" step="0.5" min="3" max="12" id="ctrlFrost" />
          </div>
          <div style="margin-bottom:12px;">
            <label>Min Boiler ON / OFF Anti-Short-Cycle (sec)</label>
            <input type="number" step="30" min="180" max="900" id="ctrlMinCycle" />
          </div>
          <button class="btn" style="width:100%;" onclick="saveControlSettings()">Save Control Settings</button>
        </div>
      </div>
    </section>

    <!-- PAGE 5: NETWORK -->
    <section id="page-network" class="page">
      <div class="grid">
        <div class="card">
          <h2>Wi-Fi Configuration</h2>
          <p style="font-size:0.83rem;color:var(--text-muted);margin-top:0;">Enter your 2.4 GHz home Wi-Fi name and password below.</p>
          <div style="margin-bottom:12px;">
            <label>Wi-Fi SSID</label>
            <input type="text" id="wifiSsid" placeholder="Home Wi-Fi Name" />
          </div>
          <div style="margin-bottom:12px;">
            <label>Wi-Fi Password</label>
            <input type="password" id="wifiPass" placeholder="••••••••" />
          </div>
          <button class="btn" onclick="saveWifi()">Save & Connect Wi-Fi</button>
        </div>

        <div class="card">
          <h2>MQTT & Home Assistant</h2>
          <div style="margin-bottom:12px;">
            <label>MQTT Broker Host / IP</label>
            <input type="text" id="mqttHost" placeholder="192.168.1.50" />
          </div>
          <div style="margin-bottom:12px;">
            <label>MQTT Port</label>
            <input type="number" id="mqttPort" value="1883" />
          </div>
          <div style="margin-bottom:12px;">
            <label>MQTT Username</label>
            <input type="text" id="mqttUser" />
          </div>
          <div style="margin-bottom:12px;">
            <label>MQTT Password</label>
            <input type="password" id="mqttPass" />
          </div>
          <button class="btn" onclick="saveMqtt()">Save MQTT Settings</button>
        </div>
      </div>
    </section>

    <!-- PAGE 6: SYSTEM & OTA -->
    <section id="page-system" class="page">
      <div class="grid">
        <div class="card">
          <h2>Hub Diagnostics</h2>
          <div class="row"><span>Hub Display Name</span><input style="max-width:170px;" id="hubNameInput" maxlength="20" /></div>
          <div class="row"><span>IP Address</span><strong id="sysIp">--</strong></div>
          <div class="row"><span>Wi-Fi Channel (ESP-NOW)</span><strong id="sysChan">--</strong></div>
          <div class="row"><span>Uptime</span><strong id="sysUptime">--</strong></div>
          <div class="row"><span>Free Heap</span><strong id="sysHeap">--</strong></div>
          <div style="margin-top:12px;">
            <button class="btn" onclick="saveHubName()">Update Hub Name</button>
          </div>
        </div>

        <div class="card">
          <h2>Firmware Update (OTA)</h2>
          <p style="font-size:0.85rem;color:var(--text-muted);">Upload a compiled PlatformIO <code>firmware.bin</code> file.</p>
          <input type="file" id="otaFile" accept=".bin" style="margin-bottom:12px;" />
          <button class="btn" onclick="uploadOta()">Flash Firmware</button>
          <div id="otaProgress" style="margin-top:10px;font-size:0.85rem;font-weight:600;"></div>
        </div>
      </div>
    </section>
  </main>

  <nav class="bottom-nav" id="bottomNav">
    <button class="active" data-page="dashboard">
      <svg viewBox="0 0 24 24"><path d="M3 12l9-9 9 9M5 10v10h14V10"/></svg>
      <span>Home</span>
    </button>
    <button data-page="sensors">
      <svg viewBox="0 0 24 24"><circle cx="12" cy="12" r="3"/><path d="M16.2 7.8a6 6 0 010 8.4M7.8 16.2a6 6 0 010-8.4"/></svg>
      <span>Sensors</span>
    </button>
    <button data-page="schedule">
      <svg viewBox="0 0 24 24"><rect x="3" y="4" width="18" height="18" rx="2"/><path d="M16 2v4M8 2v4M3 10h18"/></svg>
      <span>Schedule</span>
    </button>
    <button data-page="control">
      <svg viewBox="0 0 24 24"><path d="M4 21v-7M4 10V3M12 21v-9M12 8V3M20 21v-5M20 12V3M1 14h6M9 8h6M17 16h6"/></svg>
      <span>Control</span>
    </button>
    <button data-page="network">
      <svg viewBox="0 0 24 24"><path d="M5 12.55a11 11 0 0114.08 0M1.42 9a16 16 0 0121.16 0M8.53 16.11a6 6 0 016.95 0M12 20h.01"/></svg>
      <span>Network</span>
    </button>
    <button data-page="system">
      <svg viewBox="0 0 24 24"><circle cx="12" cy="12" r="3"/><path d="M19.4 15a1.65 1.65 0 00.33 1.82l.06.06a2 2 0 01-2.83 2.83l-.06-.06a1.65 1.65 0 00-1.82-.33 1.65 1.65 0 00-1 1.51V21a2 2 0 01-4 0v-.09A1.65 1.65 0 009 19.4a1.65 1.65 0 00-1.82.33l-.06.06a2 2 0 01-2.83-2.83l.06-.06a1.65 1.65 0 00.33-1.82 1.65 1.65 0 00-1.51-1H3a2 2 0 010-4h.09A1.65 1.65 0 004.6 9a1.65 1.65 0 00-.33-1.82l-.06-.06a2 2 0 012.83-2.83l.06.06a1.65 1.65 0 001.82.33H9a1.65 1.65 0 001-1.51V3a2 2 0 014 0v.09a1.65 1.65 0 001 1.51 1.65 1.65 0 001.82-.33l.06-.06a2 2 0 012.83 2.83l-.06.06a1.65 1.65 0 00-.33 1.82V9c.26.6.85 1 1.51 1H21a2 2 0 010 4h-.09a1.65 1.65 0 00-1.51 1z"/></svg>
      <span>System</span>
    </button>
  </nav>

  <div class="toast" id="toast"></div>

  <script>
    const DAYS = ["Mon", "Tue", "Wed", "Thu", "Fri", "Sat", "Sun"];
    let state = { target_temp: 21.0, mode: "AUTO", sensors: [] };
    let scheduleData = Array.from({ length: 7 }, () =>
      Array.from({ length: 6 }, () => ({ start_minute: -1, target_temp: 20.0, sensor_id: -1 }))
    );
    let selectedDay = 0;
    let ws = null;

    // Theme handling
    const themeBtn = document.getElementById("themeToggle");
    function applyTheme(t) {
      if (t) {
        document.documentElement.setAttribute("data-theme", t);
        localStorage.setItem("ost_theme", t);
      } else {
        document.documentElement.removeAttribute("data-theme");
      }
      const isDark = document.documentElement.getAttribute("data-theme") === "dark" ||
        (!document.documentElement.getAttribute("data-theme") && window.matchMedia("(prefers-color-scheme: dark)").matches);
      themeBtn.textContent = isDark ? "🌙" : "☀️";
    }
    themeBtn.onclick = () => {
      const cur = document.documentElement.getAttribute("data-theme");
      applyTheme(cur === "dark" ? "light" : "dark");
    };
    applyTheme(localStorage.getItem("ost_theme"));

    function toast(msg) {
      const el = document.getElementById("toast");
      el.textContent = msg;
      el.classList.add("show");
      setTimeout(() => el.classList.remove("show"), 4500);
    }

    // Navigation
    function switchPage(page) {
      document.querySelectorAll(".page").forEach(p => p.classList.remove("active"));
      document.getElementById("page-" + page).classList.add("active");
      document.querySelectorAll("[data-page]").forEach(b => {
        b.classList.toggle("active", b.dataset.page === page);
      });
      if (page === "schedule") loadSchedule();
      if (page === "control" || page === "system" || page === "network") loadSettings();
    }
    document.querySelectorAll("[data-page]").forEach(btn => {
      btn.onclick = () => switchPage(btn.dataset.page);
    });

    // WebSocket + Polling fallback
    function connectWs() {
      const proto = location.protocol === "https:" ? "wss:" : "ws:";
      ws = new WebSocket(`${proto}//${location.host}/ws`);
      ws.onopen = () => document.getElementById("wsDot").classList.remove("offline");
      ws.onclose = () => {
        document.getElementById("wsDot").classList.add("offline");
        setTimeout(connectWs, 3000);
      };
      ws.onmessage = (e) => {
        try { updateUI(JSON.parse(e.data)); } catch (_) {}
      };
    }

    async function fetchStatus() {
      try {
        const res = await fetch("/api/v1/status");
        const data = await res.json();
        updateUI(data);
      } catch (_) {}
    }

    function updateUI(d) {
      state = { ...state, ...d };
      if (d.hub_name) document.getElementById("hubTitle").textContent = d.hub_name;
      if (d.time_str) document.getElementById("clockLabel").textContent = d.time_str;
      if (typeof d.indoor_temp === "number") {
        document.getElementById("indoorTemp").textContent = d.indoor_temp.toFixed(1) + "°C";
      }
      if (typeof d.indoor_hum === "number") {
        document.getElementById("indoorHum").textContent = Math.round(d.indoor_hum) + "%";
      }
      if (typeof d.target_temp === "number") {
        document.getElementById("targetTemp").textContent = d.target_temp.toFixed(1) + "°C";
      }

      const rb = document.getElementById("relayBadge");
      if (d.heating_active) {
        rb.textContent = "HEATING ON";
        rb.className = "badge heat";
      } else {
        rb.textContent = "IDLE";
        rb.className = "badge idle";
      }

      document.querySelectorAll(".mode-btn").forEach(b => b.classList.remove("active"));
      if (d.mode) {
        const activeBtn = document.getElementById("mode-" + d.mode);
        if (activeBtn) activeBtn.classList.add("active");
      }

      const lb = document.getElementById("lockoutBanner");
      if (d.lockout_reason) {
        lb.style.display = "block";
        lb.innerHTML = `<span class="badge warn">${d.lockout_reason}</span>`;
      } else {
        lb.style.display = "none";
      }

      if (Array.isArray(d.sensors)) renderSensors(d.sensors);
    }

    function renderSensors(sensors) {
      const dash = document.getElementById("dashboardSensors");
      const list = document.getElementById("sensorList");
      dash.innerHTML = "";
      list.innerHTML = "";

      sensors.forEach(s => {
        const online = s.online !== false;
        const roleLabel = s.role === 2 ? "Outdoor" : (s.role === 0 ? "Disabled" : "Indoor");
        dash.innerHTML += `
          <div class="row">
            <div>
              <strong>${s.name || ("Sensor #" + s.id)}</strong>
              <div style="font-size:0.78rem;color:var(--text-muted);">${roleLabel} • ${s.id === 0 ? "Onboard" : (s.batt_pct + "%")}</div>
            </div>
            <div style="text-align:right;">
              <strong>${online ? s.temp.toFixed(1) + "°C" : "--.-°C"}</strong>
              <div style="font-size:0.78rem;color:var(--text-muted);">${online ? Math.round(s.hum) + "%" : "Offline"}</div>
            </div>
          </div>`;

        if (s.id > 0) {
          list.innerHTML += `
            <div class="card">
              <div style="display:flex;justify-content:space-between;align-items:center;margin-bottom:10px;">
                <strong>#${String(s.id).padStart(2, "0")} • ${s.name}</strong>
                <span class="badge ${online ? "ok" : "danger"}">${online ? "ONLINE" : "OFFLINE"}</span>
              </div>
              <div style="margin-bottom:8px;">
                <label>Custom Name (max 24 chars)</label>
                <input id="sname-${s.id}" maxlength="24" value="${s.name}" />
              </div>
              <div class="grid" style="grid-template-columns:1fr 1fr;gap:8px;margin-bottom:8px;">
                <div>
                  <label>Role</label>
                  <select id="srole-${s.id}">
                    <option value="1" ${s.role === 1 ? "selected" : ""}>Indoor</option>
                    <option value="2" ${s.role === 2 ? "selected" : ""}>Outdoor</option>
                    <option value="0" ${s.role === 0 ? "selected" : ""}>Disabled</option>
                  </select>
                </div>
                <div>
                  <label>Screen Timeout (s)</label>
                  <input type="number" min="3" max="30" id="stime-${s.id}" value="${s.disp_timeout || 8}" />
                </div>
              </div>
              <div style="display:flex;justify-content:space-between;margin-top:10px;">
                <button class="btn danger sm" onclick="deleteSensor(${s.id})">Remove</button>
                <button class="btn sm" onclick="saveSensor(${s.id})">Save</button>
              </div>
            </div>`;
        }
      });
    }

    async function adjustTemp(delta) {
      const next = Math.min(35, Math.max(5, (state.target_temp || 21.0) + delta));
      await fetch("/api/v1/control", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ target_temp: next, mode: state.mode === "BOOST" ? "BOOST" : "MANUAL" })
      });
      fetchStatus();
    }

    async function setMode(mode) {
      await fetch("/api/v1/control", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ mode })
      });
      toast("Mode: " + mode);
      fetchStatus();
    }

    async function startPairing() {
      await fetch("/api/v1/sensors/pair", { method: "POST" });
      const ps = document.getElementById("pairStatus");
      ps.style.display = "inline-flex";
      setTimeout(() => ps.style.display = "none", 60000);
      toast("Pairing window opened for 60s");
    }

    async function saveSensor(id) {
      await fetch(`/api/v1/sensors/${id}`, {
        method: "PUT",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          name: document.getElementById(`sname-${id}`).value,
          role: parseInt(document.getElementById(`srole-${id}`).value, 10),
          disp_timeout: parseInt(document.getElementById(`stime-${id}`).value, 10)
        })
      });
      toast("Sensor updated");
      fetchStatus();
    }

    async function deleteSensor(id) {
      if (!confirm("Unpair sensor #" + id + "?")) return;
      await fetch(`/api/v1/sensors/${id}`, { method: "DELETE" });
      toast("Sensor removed");
      fetchStatus();
    }

    // Schedule Editor (Supports empty --:-- slots & chronological validation)
    function minsToTimeStr(m) {
      if (m === null || m === undefined || m < 0) return "";
      const hh = String(Math.floor(m / 60)).padStart(2, "0");
      const mm = String(m % 60).padStart(2, "0");
      return `${hh}:${mm}`;
    }

    function timeStrToMins(str) {
      if (!str || !str.includes(":")) return -1;
      const [h, m] = str.split(":").map(Number);
      if (isNaN(h) || isNaN(m)) return -1;
      return h * 60 + m;
    }

    async function loadSchedule() {
      const pills = document.getElementById("dayPills");
      pills.innerHTML = DAYS.map((d, i) =>
        `<button class="${i === selectedDay ? "active" : ""}" onclick="selectDay(${i})">${d}</button>`
      ).join("");
      try {
        const res = await fetch("/api/v1/schedule");
        const data = await res.json();
        if (Array.isArray(data.days)) scheduleData = data.days;
      } catch (_) {}
      renderDaySlots();
    }

    function selectDay(d) {
      selectedDay = d;
      loadSchedule();
    }

    function renderDaySlots() {
      const container = document.getElementById("scheduleSlots");
      const slots = scheduleData[selectedDay];
      let prevValidMin = -1;

      container.innerHTML = slots.map((slot, idx) => {
        const isEmpty = slot.start_minute < 0;
        const isOutOfOrder = !isEmpty && slot.start_minute <= prevValidMin;
        if (!isEmpty && !isOutOfOrder) prevValidMin = slot.start_minute;

        const sensorOpts = `<option value="-1" ${slot.sensor_id === -1 ? "selected" : ""}>Default Sensor</option>` +
          (state.sensors || []).filter(s => s.role !== 2 && s.role !== 0).map(s =>
            `<option value="${s.id}" ${slot.sensor_id === s.id ? "selected" : ""}>${s.name || ("#" + s.id)}</option>`
          ).join("");

        return `
          <div class="slot-card ${isEmpty ? "empty" : ""} ${isOutOfOrder ? "out-of-order" : ""}">
            <div class="slot-header-mobile">
              <strong>Slot ${idx + 1}</strong>
              ${isOutOfOrder ? `<span class="badge warn">Ignored (earlier than previous)</span>` : ""}
            </div>
            <div>
              <input type="time" value="${minsToTimeStr(slot.start_minute)}"
                onchange="updateSlot(${idx}, 'start_minute', timeStrToMins(this.value))" />
            </div>
            <div>
              <input type="number" step="0.5" min="5" max="30" placeholder="°C"
                value="${isEmpty ? "" : slot.target_temp.toFixed(1)}"
                onchange="updateSlot(${idx}, 'target_temp', parseFloat(this.value) || 20.0)" />
            </div>
            <div>
              <select onchange="updateSlot(${idx}, 'sensor_id', parseInt(this.value, 10))">${sensorOpts}</select>
            </div>
            <button class="btn secondary sm" title="Clear Slot" onclick="clearSlot(${idx})">×</button>
          </div>`;
      }).join("");
    }

    function updateSlot(idx, field, val) {
      scheduleData[selectedDay][idx][field] = val;
      if (field === "start_minute" && val >= 0 && !scheduleData[selectedDay][idx].target_temp) {
        scheduleData[selectedDay][idx].target_temp = 21.0;
      }
      renderDaySlots();
    }

    function clearSlot(idx) {
      scheduleData[selectedDay][idx] = { start_minute: -1, target_temp: 20.0, sensor_id: -1 };
      renderDaySlots();
    }

    async function saveSchedule() {
      await fetch("/api/v1/schedule", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ day: selectedDay, slots: scheduleData[selectedDay] })
      });
      toast(DAYS[selectedDay] + " schedule saved");
    }

    async function copySchedule(target) {
      await fetch("/api/v1/schedule", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ day: selectedDay, slots: scheduleData[selectedDay], copy_to: target })
      });
      toast("Schedule copied");
      loadSchedule();
    }

    async function loadSettings() {
      try {
        const res = await fetch("/api/v1/settings");
        const s = await res.json();
        document.getElementById("ctrlAlgo").value = s.algo ?? 0;
        document.getElementById("ctrlHyst").value = s.hysteresis ?? 0.3;
        document.getElementById("ctrlAgg").value = s.agg_mode ?? 0;
        document.getElementById("learnedOvershoot").textContent = (s.learned_overshoot ?? 0).toFixed(2) + "°C";
        document.getElementById("ctrlWarmLock").value = s.warm_lockout ?? 18.0;
        document.getElementById("ctrlFrost").value = s.frost_temp ?? 5.0;
        document.getElementById("ctrlMinCycle").value = s.min_cycle_s ?? 240;
        document.getElementById("mqttHost").value = s.mqtt_host || "";
        document.getElementById("mqttPort").value = s.mqtt_port || 1883;
        document.getElementById("mqttUser").value = s.mqtt_user || "";
        document.getElementById("hubNameInput").value = s.hub_name || "OpenSmartThermo";
        document.getElementById("sysIp").textContent = s.ip || "--";
        document.getElementById("sysChan").textContent = s.wifi_channel || "--";
        document.getElementById("sysUptime").textContent = s.uptime || "--";
        document.getElementById("sysHeap").textContent = s.free_heap || "--";
      } catch (_) {}
    }

    async function saveControlSettings() {
      await fetch("/api/v1/settings", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          algo: parseInt(document.getElementById("ctrlAlgo").value, 10),
          hysteresis: parseFloat(document.getElementById("ctrlHyst").value),
          agg_mode: parseInt(document.getElementById("ctrlAgg").value, 10),
          warm_lockout: parseFloat(document.getElementById("ctrlWarmLock").value),
          frost_temp: parseFloat(document.getElementById("ctrlFrost").value),
          min_cycle_s: parseInt(document.getElementById("ctrlMinCycle").value, 10)
        })
      });
      toast("Control settings saved");
    }

    async function resetLearning() {
      await fetch("/api/v1/settings/reset-learning", { method: "POST" });
      toast("Self-learning overshoot reset");
      loadSettings();
    }

    async function saveWifi() {
      toast("Testing Wi-Fi connection...");
      try {
        const res = await fetch("/api/v1/wifi", {
          method: "POST",
          headers: { "Content-Type": "application/json" },
          body: JSON.stringify({
            ssid: document.getElementById("wifiSsid").value,
            password: document.getElementById("wifiPass").value
          })
        });
        const data = await res.json();
        toast(data.message || "Wi-Fi updated");
        if (data.connected && data.ip) {
          document.getElementById("sysIp").textContent = data.ip;
        }
      } catch (_) {
        toast("Connection error");
      }
    }

    async function saveMqtt() {
      await fetch("/api/v1/settings", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          mqtt_host: document.getElementById("mqttHost").value,
          mqtt_port: parseInt(document.getElementById("mqttPort").value, 10),
          mqtt_user: document.getElementById("mqttUser").value,
          mqtt_pass: document.getElementById("mqttPass").value
        })
      });
      toast("MQTT settings saved");
    }

    async function saveHubName() {
      await fetch("/api/v1/settings", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ hub_name: document.getElementById("hubNameInput").value })
      });
      toast("Hub name updated");
      fetchStatus();
    }

    async function uploadOta() {
      const file = document.getElementById("otaFile").files[0];
      if (!file) return toast("Select a .bin file first");
      const form = new FormData();
      form.append("update", file);
      document.getElementById("otaProgress").textContent = "Uploading firmware...";
      const res = await fetch("/api/v1/ota", { method: "POST", body: form });
      document.getElementById("otaProgress").textContent = res.ok ? "Success! Rebooting..." : "OTA Failed";
    }

    fetchStatus();
    connectWs();
    setInterval(fetchStatus, 10000);
  </script>
</body>
</html>
```
