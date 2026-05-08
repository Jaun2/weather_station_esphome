# Weather Station (ESPHome)

DIY off-grid weather station that publishes data to Home Assistant. Permanent outdoor deployment in Vereeniging, South Africa (Highveld, ~1500 m altitude). Solar-powered, battery-backed, designed to run unattended for years.

## What it is

**Hardware:** DFRobot FireBeetle 2 ESP32-C6 (DFR1075), BME280 (T/H/P), tipping-bucket rain gauge with reed switch on `GPIO4`, 5000 mAh LiPo + 6 V solar panel with onboard MPPT. 3D-printed inner enclosure inside the rain gauge body for the electronics; Stevenson screen for the BME280.

**Software:** ESPHome firmware on the C6, ESPHome native API → Home Assistant. Derived metrics (dew point, heat index, rain intensity, feels-like, etc.) are computed in firmware so they appear on the device card alongside the raw sensors. HA-side companion package handles the things that genuinely need historical data — pressure / humidity / rainfall trend gradients and a daily-reset rainfall total.

**Power architecture:** 5-minute deep sleep cycle. Reed-switch closures wake the chip out of cycle so rain tips are recorded promptly. Battery voltage is monitored on every wake; below 3.3 V the firmware forces a 24-hour deep sleep to protect the LiPo from over-discharge. Average current ~4 mA → essentially indefinite runtime on solar.

For the architectural decisions, hardware rationale, and full multi-stage build roadmap, see [`CLAUDE.md`](./CLAUDE.md).

## Repository layout

```
weather_station.yaml               # ESPHome firmware config (the device)
secrets.example.yaml               # template; copy to secrets.yaml and fill in
secrets.yaml                       # local secrets (gitignored)
home_assistant/
  packages/
    weather_station.yaml           # HA package — drop into /config/packages/
docs/
  workflow.md                      # detailed flashing / OTA workflow + verification gates
CLAUDE.md                          # project context, hardware specs, full roadmap
```

## Setup

Assumes Home Assistant with the ESPHome add-on already installed.

### 1. Create the device in the ESPHome dashboard

In the dashboard:

1. **+ NEW DEVICE → Continue → Skip wizard.**
2. Pick **ESP32-C6** as the chip type.
3. Click **EDIT** on the new device card.
4. Replace its contents with [`weather_station.yaml`](./weather_station.yaml) from this repo. **SAVE.**

### 2. Add secrets

In the dashboard, top-right **⋮ menu → Secrets**. Add these keys (use [`secrets.example.yaml`](./secrets.example.yaml) as a template):

```yaml
wifi_ssid: "..."
wifi_password: "..."
ap_password: "..."          # 8+ chars; for fallback hotspot
ota_password: "..."         # 32+ chars; generate with `openssl rand -base64 32`
api_encryption_key: "..."   # 32-byte base64; ESPHome dashboard has a "Generate" button
```

Save.

### 3. Create the HA helper toggle (required from Stage 3 onwards)

The deep-sleep firmware reads a Home Assistant helper toggle to decide whether to sleep or stay awake. Without it, the device will sleep normally — but you won't have a clean way to interrupt the cycle for OTA flashing.

**Settings → Devices & Services → Helpers → Create Helper → Toggle.**

- Name: `Weather Station Deep Sleep`
- Default state: ON

ON = device sleeps normally. OFF = device stays awake (used for OTA install and bench debugging).

### 4. First flash (USB, via web.esphome.io)

The HA add-on can't reach USB on your PC, so the first flash uses Chrome's WebSerial.

1. In the dashboard, **INSTALL → Manual download → Modern format**. Wait for compile (~10–15 min the first time, much faster after — toolchain is cached).
2. Save the `.bin` file somewhere accessible.
3. Plug the FireBeetle C6 into USB-C **while holding the BOOT button (`GPIO9`)**. Release after USB enumerates.
4. Open https://web.esphome.io/ in Chrome. Click **CONNECT**, pick the COM port.
5. Click **INSTALL** and select the `.bin` you downloaded.
6. After the upload finishes, press RESET on the board (or unplug/replug USB).

> ⚠️ **Trap to avoid:** web.esphome.io has a "**Prepare for first use**" / "Quick start" option. **Do NOT click it.** It flashes a generic ESPHome bootstrap firmware (named `esphome-web-XXXXXX`) instead of your config. Always pick the **INSTALL** / **Choose your own file** path that takes a local `.bin`.

The device joins Wi-Fi and announces itself as `weather-station` via mDNS. Home Assistant's ESPHome integration will offer to add it; click **CONFIGURE → ADD** and paste the API encryption key when prompted.

### 5. Subsequent flashes (OTA — with the deep-sleep dance)

Because the device is in deep sleep most of the time, OTA updates require briefly disabling sleep so the device stays online long enough to receive the new firmware:

1. **Toggle `Weather Station Deep Sleep` to OFF** in HA.
2. Wait for the next wake (within 5 min). Device comes online and stays awake.
3. Edit `weather_station.yaml` in this repo, paste into the ESPHome dashboard editor.
4. **INSTALL → Wirelessly.**
5. Device reboots and stays awake (toggle still OFF).
6. **Toggle `Weather Station Deep Sleep` back to ON.**
7. Device sleeps within 25 s. Production cycle resumes.

If you forget to toggle OFF and try to install while the device is sleeping, the dashboard fails with a connection error. Recovery: toggle OFF, wait up to 5 min for the next wake, retry the install.

### 6. Install the HA package

The package adds:
- Three `derivative` sensors that compute trend gradients (pressure 3h, humidity 2h, rainfall 15min) from HA's recorder history. The firmware subscribes to these via the native API and uses them for `Rain Likelihood` and `Rain Intensity`.
- One `statistics` sensor (`Rainfall 24h Source`) that computes a rolling 24-hour rainfall total. The firmware subscribes to it and republishes as `Rainfall 24h`.
- A `utility_meter` for daily rainfall reset at midnight.

**Steps:**

1. On your HA host, copy [`home_assistant/packages/weather_station.yaml`](./home_assistant/packages/weather_station.yaml) to `/config/packages/weather_station.yaml`. Create the directory if it doesn't exist.
2. **One-time setup** if you don't already use packages — in `configuration.yaml`, add (or extend) the `homeassistant:` block:

   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```

3. **Restart Home Assistant.**

After the restart, four plumbing sensors appear in HA: `Pressure Change Rate 3h`, `Humidity Change Rate 2h`, `Rainfall Rate`, `Rainfall 24h Source`. They're internal — the firmware reads them back; you don't need them on any dashboard. Hide them via **Settings → Devices & Services → Entities** if they bother you.

The `Rainfall Today` utility meter is user-facing — useful for daily-rainfall dashboard cards.

### 7. Verify the BME280 is genuine

BMP280 lookalikes are common and have **no humidity sensor** (they report fixed humidity values). Quick test:

1. Note the current humidity reading.
2. Breathe gently on the BME280 from ~5 cm away.
3. Watch the humidity in HA — should jump 5–10 % within 10–20 s, then drift back.

Humidity moves on breath = real BME280. Humidity stays flat = BMP280; replace it.

## Sensors

The `weather-station` device card shows ~19 entities after firmware install + HA package, organised as follows.

### Raw measurements

| Entity | Unit | Notes |
|---|---|---|
| `Temperature` | °C | BME280 |
| `Humidity` | % RH | BME280 |
| `Pressure` | hPa | BME280 — raw (~840 hPa at 1500 m) |
| `Rain Bucket` | on/off | Reed switch state (mostly diagnostic) |
| `Rain Tips` | count | Cumulative tip count since install |
| `Is Raining` | on/off | True if a tip occurred in the last 5 min (`raining_window_minutes` substitution) |

### Derived metrics — numeric (firmware lambdas)

| Entity | Type | Notes |
|---|---|---|
| `Dew Point` | °C | Magnus-Tetens (water saturation) |
| `Heat Index` | °C | NWS Rothfusz regression; `Unknown` when T < 27 °C OR RH < 40 % |
| `Frost Point` | °C | Magnus-Tetens (ice saturation); `Unknown` above 0 °C |
| `Fog Probability` | % | Heuristic from RH and dew-point spread |
| `Rain Likelihood` | % | Combines pressure trend, humidity trend, dew-point spread |
| `Rainfall` | mm | Cumulative since install (`tips × mm_per_tip`) |
| `Rainfall Session` | mm | Current/most-recent rain event; resets when no tips for 60 min |
| `Rainfall 24h` | mm | Rolling 24-hour total (window from now); subscribes to `Rainfall 24h Source` in HA |
| `Feels Like` | °C | Australian Apparent Temperature (Steadman / BOM), no-wind variant |

### Derived metrics — categorical (firmware text lambdas)

| Entity | Possible values |
|---|---|
| `Visibility` | Fog / Mist / Haze / Clear |
| `Heat Index Category` | Caution / Extreme Caution / Danger / Extreme Danger / Not Applicable |
| `Frost Point Category` | Light Frost / Hard Frost / Severe Frost / Not Applicable |
| `Rain Intensity` | No Rain / Drizzle / Moderate / Heavy / Storm |

### Diagnostics

| Entity | Notes |
|---|---|
| `Wi-Fi Signal` | dBm — raw signal strength |
| `Wi-Fi Signal Strength` | % — human-friendly mapping (-50 dBm ≈ 100 %, -100 dBm = 0 %) |
| `Uptime` | seconds since last wake (resets every cycle) |
| `Battery Voltage` | V — raw LiPo voltage via the FireBeetle's onboard 2:1 divider on `GPIO0` |
| `Battery` | % — piecewise LiPo curve mapping voltage to charge level (4.2 V → 100 %, 3.7 V → 50 %, 3.4 V → 10 %, 3.0 V → 0 %) |

### Controls

| Entity | Notes |
|---|---|
| `Restart` | Reboot the device |
| `Rain Tips Reset` | Zero out cumulative tips, session tips, last-tip timestamp (for calibration) |

### HA-side plumbing (hidden by default)

These compute trend gradients from HA's recorder history and feed them back to the firmware. Hide them in HA's UI if they appear and bother you.

| Entity | Notes |
|---|---|
| `Pressure Change Rate 3h` | hPa/sec, derivative over 3-hour window — feeds Rain Likelihood |
| `Humidity Change Rate 2h` | %/sec, derivative over 2-hour window — feeds Rain Likelihood |
| `Rainfall Rate` | mm/hour, derivative over 15-min window — feeds Rain Intensity |
| `Rainfall 24h Source` | mm, statistics integration over 24-hour window — feeds the device's `Rainfall 24h` entity |
| `Rainfall Today` | mm, utility meter resetting at midnight — useful on dashboards |

## Build stages

Sequential — each has its own verification gate. See [`CLAUDE.md`](./CLAUDE.md) § Build Stages for the full roadmap with rationale.

- [x] **Stage 0** — Bring-up (Wi-Fi, native API, OTA, status LED, restart button)
- [x] **Stage 1** — BME280 over I²C + derived metrics + HA trend package
- [x] **Stage 2** — Reed-switch rain-gauge tip counting + session detection + intensity categorisation
- [x] **Stage 3** — Deep sleep + 5-minute wake cycle + reed-wake + helper-toggle for OTA
- [x] **Stage 4** — Battery ADC + piecewise LiPo curve + low-voltage cutoff at 3.3 V
- [ ] **Stage 5** — Solar input + Schottky diode (next; **1N5822** in series with panel +ve into `VIN`)
- [ ] **Stage 6** — Mechanical assembly + waterproofing
- [ ] **Stage 7** — Field deployment + calibration
- [ ] **Stage 8** — Lightning detector (AS3935)
- [ ] **Stage 9** — Wind speed + direction (3-cup head + AS5600 vane)
- [ ] **Stage 10** — Soil moisture, soil temp, UV, LDR

## Troubleshooting

**Compile takes forever the first time.** Expected. ESP-IDF + the C6 toolchain weigh in around 1 GB and PlatformIO has to download and build them. ~10–15 minutes on a Pi 4. Subsequent compiles are ~30 seconds (cached).

**OTA fails with "Error resolving IP address of weather-station.local".** Either mDNS resolution issue (try the device's IP directly), or — far more likely from Stage 3 onwards — you forgot to toggle `Weather Station Deep Sleep` OFF before installing. The device is asleep and not reachable. Toggle OFF, wait up to 5 min for the next wake, retry.

**The device disappears from HA every 5 minutes.** That's deep sleep working as intended. Each wake cycle is ~25 s during which the device publishes fresh values, then it sleeps for 5 minutes.

**Heat Index reads "Unknown".** That's correct below 27 °C / 40 % RH — heat index is a hot-weather metric, undefined in cooler conditions. Same for Frost Point above 0 °C.

**Strapping pin warnings on `GPIO4` and `GPIO15`.** Expected and harmless on the DFR1075. GPIO4 is the reed switch (normally open, internal pullup keeps it HIGH at reset → safe boot mode), GPIO15 is the onboard LED. Warnings are left visible deliberately.

**Rain tip not registering when I swipe a magnet.** With the device in deep sleep, the swipe needs to actually close the reed switch — too far away and the magnet won't trigger it. The first reliable indicator is `Rain Bucket` flipping briefly to `on` in HA, OR `Rain Tips` incrementing on the next wake. Without a tipping bucket installed, manual swipes are awkward.

**Device disappears for 24 hours and helper toggle won't bring it back.** The battery voltage dropped below 3.3 V, triggering the software low-voltage cutoff. Plug in USB-C; the LiPo will charge. The device will wake on its next 24-hour timer (you can also force-wake via reset). The cutoff is deliberate LiPo protection — over-discharging below 3.3 V ages the cell prematurely.

## Notes

- `secrets.yaml` is **gitignored**. Never commit real Wi-Fi or OTA credentials.
- The HA package's `Rainfall Rate` and `Pressure/Humidity Change Rate` sensors need a few minutes of accumulated history before they produce meaningful gradients. Rain Likelihood and Rain Intensity will read low/idle at first then settle.
- The `mm_per_tip` rainfall calibration depends on bucket geometry. Default is `0.6314` (SS4H-RG reference). Recalibrate at Stage 7 with a measured pour: pour a known volume into the funnel slowly, count tips, divide. Edit the substitution in `weather_station.yaml` and re-flash.
- Rain Session resets when no tips arrive for `session_gap_minutes` (default 60). Tweak the substitution if your rain patterns suggest a different gap.
