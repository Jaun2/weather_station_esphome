# Weather Station (ESPHome)

DIY off-grid weather station that publishes data to Home Assistant. Permanent outdoor deployment in Vereeniging, South Africa (Highveld, ~1500 m altitude). Solar-powered, battery-backed, designed to run unattended for years.

## What it is

**Hardware:** DFRobot FireBeetle 2 ESP32-C6 (DFR1075), BME280 (T/H/P), 5000 mAh LiPo + 6 V solar panel with onboard MPPT. Tipping-bucket rain gauge with reed switch (Stage 2+). 3D-printed inner enclosure inside the rain gauge body for the electronics; Stevenson screen for the BME280.

**Software:** ESPHome firmware on the C6, ESPHome native API → Home Assistant. Derived metrics (dew point, heat index, etc.) are computed in firmware so they appear on the device card alongside the raw sensors. HA-side companion package handles the things that genuinely need historical data (pressure / humidity trend gradients).

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

In the dashboard, top-right **⋮ menu → Secrets**. The ESPHome add-on stores secrets in a single `secrets.yaml` shared across all your ESPHome devices.

Add these keys (use [`secrets.example.yaml`](./secrets.example.yaml) as a template):

```yaml
wifi_ssid: "..."
wifi_password: "..."
ap_password: "..."          # 8+ chars; for fallback hotspot
ota_password: "..."         # 32+ chars; generate with `openssl rand -base64 32`
api_encryption_key: "..."   # 32-byte base64; ESPHome dashboard has a "Generate" button
```

Save.

### 3. First flash (USB, via web.esphome.io)

The HA add-on can't reach USB on your PC, so the first flash uses Chrome's WebSerial.

1. In the dashboard, **INSTALL → Manual download → Modern format**. Wait for compile (~10–15 min the first time, much faster after — toolchain is cached).
2. Save the `.bin` file somewhere accessible.
3. Plug the FireBeetle C6 into USB-C **while holding the BOOT button (`GPIO9`)**. Release after USB enumerates.
4. Open https://web.esphome.io/ in Chrome. Click **CONNECT**, pick the COM port.
5. Click **INSTALL** and select the `.bin` you downloaded.
6. After the upload finishes, press RESET on the board (or unplug/replug USB).

> ⚠️ **Trap to avoid:** web.esphome.io has a "**Prepare for first use**" / "Quick start" option. **Do NOT click it.** It flashes a generic ESPHome bootstrap firmware (named `esphome-web-XXXXXX`) instead of your config. Always pick the **INSTALL** / **Choose your own file** path that takes a local `.bin`.

The device joins Wi-Fi and announces itself as `weather-station` via mDNS. Home Assistant's ESPHome integration will offer to add it; click **CONFIGURE → ADD** and paste the API encryption key when prompted.

### 4. Subsequent flashes (OTA)

Once the device is on Wi-Fi and visible in HA, you don't need USB:

1. Edit `weather_station.yaml` in this repo.
2. Copy/paste the new contents into the ESPHome dashboard editor.
3. **INSTALL → Wirelessly.** The dashboard pushes the build over the network using the OTA password.

### 5. Install the HA package

The package adds two `derivative` sensors that compute pressure/humidity trend gradients from HA's recorder history. The firmware subscribes to these via the native API and uses them as inputs to the **Rain Likelihood** lambda — that's how a firmware-side metric can incorporate trend data without keeping any history on the device.

1. On your HA host, copy [`home_assistant/packages/weather_station.yaml`](./home_assistant/packages/weather_station.yaml) to `/config/packages/weather_station.yaml`. Create the directory if it doesn't exist.
2. **One-time setup** if you don't already use packages — in `configuration.yaml`, add (or extend) the `homeassistant:` block:

   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```

3. **Restart Home Assistant.**

After the restart, two new sensors appear in HA: `Pressure Change Rate 3h` and `Humidity Change Rate 2h`. They're plumbing — the firmware reads them back; you don't need them on any dashboard. Hide them via **Settings → Devices & Services → Entities** if they bother you.

### 6. (Optional) HA helper toggle for Stage 3

Used by the deep-sleep stage. Create it now so it's ready when Stage 3 lands:

**Settings → Devices & Services → Helpers → Create Helper → Toggle**

- Name: `Weather Station Deep Sleep`
- Default state: ON

When Stage 3 wires it up: ON = device sleeps normally between wakes; OFF = device stays awake (used for OTA install windows).

### 7. Verify the BME280 is genuine

BMP280 lookalikes are common and have **no humidity sensor** (they report fixed humidity values). Quick test:

1. Note the current humidity reading.
2. Breathe gently on the BME280 from ~5 cm away.
3. Watch the humidity in HA — should jump 5–10 % within 10–20 s, then drift back.

Humidity moves on breath = real BME280. Humidity stays flat = BMP280; replace it.

## Sensors

After firmware install + HA package, the `weather-station` device card shows ~13 entities.

### Raw measurements (from the BME280)

| Entity | Unit | Notes |
|---|---|---|
| `Temperature` | °C | |
| `Humidity` | % RH | |
| `Pressure` | hPa | Raw (~840 hPa at 1500 m altitude) |

### Derived metrics (computed in firmware lambdas)

| Entity | Type | Notes |
|---|---|---|
| `Dew Point` | °C | Magnus-Tetens (water saturation) |
| `Heat Index` | °C | NWS Rothfusz regression; `Unknown` when T < 27 °C OR RH < 40 % |
| `Frost Point` | °C | Magnus-Tetens (ice saturation); `Unknown` above 0 °C |
| `Fog Probability` | % | Heuristic from RH and dew-point spread |
| `Rain Likelihood` | % | Combines pressure trend, humidity trend, dew-point spread |
| `Feels Like` | °C | Australian Apparent Temperature (Steadman / BOM), no-wind variant |
| `Visibility` | text | Fog / Mist / Haze / Clear |
| `Heat Index Category` | text | Caution / Extreme Caution / Danger / Extreme Danger / Not Applicable |
| `Frost Point Category` | text | Light Frost / Hard Frost / Severe Frost / Not Applicable |

### Diagnostics

| Entity | Notes |
|---|---|
| `Wi-Fi Signal` | dBm |
| `Uptime` | seconds since boot |
| `Restart` | button to reboot the device |

### HA-side plumbing (hidden from dashboards)

| Entity | Notes |
|---|---|
| `Pressure Change Rate 3h` | hPa/sec, derivative over 3-hour window — fed to firmware |
| `Humidity Change Rate 2h` | %/sec, derivative over 2-hour window — fed to firmware |

## Build stages

Sequential — each has its own verification gate. See [`CLAUDE.md`](./CLAUDE.md) § Build Stages for the full roadmap with rationale.

- [x] **Stage 0** — Bring-up (Wi-Fi, native API, OTA, status LED, restart button)
- [x] **Stage 1** — BME280 over I²C + derived metrics + HA trend package
- [ ] **Stage 2** — Reed-switch rain-gauge tip counting (next)
- [ ] **Stage 3** — Deep sleep + 5-minute wake cycle
- [ ] **Stage 4** — Battery ADC + low-voltage cutoff
- [ ] **Stage 5** — Solar input + Schottky diode
- [ ] **Stage 6** — Mechanical assembly + waterproofing
- [ ] **Stage 7** — Field deployment + calibration
- [ ] **Stage 8** — Lightning detector (AS3935)
- [ ] **Stage 9** — Wind speed + direction (3-cup head + AS5600 vane)
- [ ] **Stage 10** — Soil moisture, soil temp, UV, LDR

## Troubleshooting

**Compile takes forever the first time.** Expected. ESP-IDF + the C6 toolchain weigh in around 1 GB and PlatformIO has to download and build them. ~10–15 minutes on a Pi 4. Subsequent compiles are ~30 seconds (cached).

**OTA fails with "Error resolving IP address of weather-station.local".** mDNS resolution issue. Check your router's client list for the device's IP and either set it as a static IP in the YAML's `wifi:` block, or trigger the install with the IP entered manually.

**Heat Index reads "Unknown".** That's correct below 27 °C / 40 % RH — heat index is a hot-weather metric, undefined in cooler conditions. Same for Frost Point above 0 °C.

**Strapping pin warning on `GPIO15`.** Expected and harmless on the DFR1075. The board's onboard LED is wired to a strapping pin, but DFRobot's circuit doesn't pull strongly enough to affect boot-time strapping. Warning is left visible deliberately.

## Notes

- `secrets.yaml` is **gitignored**. Never commit real Wi-Fi or OTA credentials.
- The HA package's `Pressure Change Rate 3h` / `Humidity Change Rate 2h` sensors need a few minutes of accumulated history before they produce non-zero gradients. Rain Likelihood will read low at first then settle.
- The mm-per-tip rainfall calibration depends on the bucket geometry — calibrate with a measured pour once Stage 2 is in place.
