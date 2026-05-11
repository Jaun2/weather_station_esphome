# Weather Station (ESPHome)

DIY off-grid weather station that publishes data to Home Assistant. Permanent outdoor deployment in Vanderbijlpark, South Africa (Vaal Triangle, Highveld). Solar-powered, battery-backed, designed to run unattended for years. Altitude is sourced at runtime from Home Assistant's `zone.home` elevation, not hardcoded.

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
  dashboards/
    weather_summary_card.yaml      # Lovelace card for HA dashboard (Mushroom Cards)
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
- One `history_stats` sensor (`Rainfall 24h Tips`) that counts reed-switch tip events in the last 24 hours, plus a `template` sensor (`Rainfall 24h Source`) that converts that count to mm. The firmware subscribes to the mm value and republishes as `Rainfall 24h`. (Counting tip events instead of differencing the cumulative `Rainfall` sensor keeps the 24h total reset-safe — a firmware reflash or Rain Tips Reset doesn't make the value go negative.)
- A `utility_meter` for daily rainfall reset at midnight.

**Steps:**

1. On your HA host, copy [`home_assistant/packages/weather_station.yaml`](./home_assistant/packages/weather_station.yaml) to `/config/packages/weather_station.yaml`. Create the directory if it doesn't exist.
2. **One-time setup** if you don't already use packages — in `configuration.yaml`, add (or extend) the `homeassistant:` block:

   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```

3. **Restart Home Assistant.**

After the restart, five plumbing sensors appear in HA: `Pressure Change Rate 3h`, `Humidity Change Rate 2h`, `Rainfall Rate`, `Rainfall 24h Tips`, `Rainfall 24h Source`. They're internal — the firmware reads them back; you don't need them on any dashboard. Hide them via **Settings → Devices & Services → Entities** if they bother you.

The `Rainfall Today` utility meter is user-facing — useful for daily-rainfall dashboard cards.

### 7. Verify the BME280 is genuine

BMP280 lookalikes are common and have **no humidity sensor** (they report fixed humidity values). Quick test:

1. Note the current humidity reading.
2. Breathe gently on the BME280 from ~5 cm away.
3. Watch the humidity in HA — should jump 5–10 % within 10–20 s, then drift back.

Humidity moves on breath = real BME280. Humidity stays flat = BMP280; replace it.

### 8. Install the dashboard summary card (optional)

A polished Lovelace widget sits in [`home_assistant/dashboards/weather_summary_card.yaml`](./home_assistant/dashboards/weather_summary_card.yaml). It bundles current conditions, system health (battery / Wi-Fi / charging), rainfall, sunrise/sunset, and conditional warning rows into one cohesive card — built using `button-card` with custom CSS-grid layouts, with smart conditional sections so it only shows what's relevant.

**Requires:** [HACS](https://hacs.xyz/) plus two custom frontend cards:

- [`button-card`](https://github.com/custom-cards/button-card) — drives the custom layouts.
- [`stack-in-card`](https://github.com/custom-cards/stack-in-card) — merges sub-sections into one visual block (no internal borders).

**Steps:**

1. **Install both cards via HACS:**
   - HACS → **Frontend** → search **button-card** → **Download**.
   - HACS → **Frontend** → search **stack-in-card** → **Download**.
   - Restart Home Assistant when HACS prompts.

2. **Verify the entity IDs match your HA.** Some entities may have a `_2` suffix in your HA registry if the device was re-added at any point. Filter for `weather_station` in **Developer Tools → States** to confirm IDs. The card YAML uses the IDs that worked for the project's reference setup; if any differ in your HA, search-and-replace before pasting.

3. **Open the card YAML in this repo:** [`home_assistant/dashboards/weather_summary_card.yaml`](./home_assistant/dashboards/weather_summary_card.yaml). Copy everything from the `type: custom:stack-in-card` line down to the end (the comments at the top are documentation, not config).

4. **In Home Assistant:**
   - Open the dashboard you want the card on.
   - Click **⋮ menu (top right) → Edit Dashboard**.
   - Click **+ Add Card** at the bottom of the page.
   - Scroll the card picker to the bottom and pick **Manual**.
   - Paste the copied YAML in. **Save**.

5. The widget appears as one unified card. Tap any section for the entity's more-info popup.

**What you get (top to bottom):**

- **Header bar** — "Weather Station" title with subtitle, plus three system-status icons on the right: charging (green when active, dim when idle), battery (11 fill levels at 10 % granularity, color shifts green → orange → red as charge drops), and Wi-Fi (4-bar strength meter, color reflects signal quality). Hover any icon for the actual % value as a tooltip.
- **Main weather block** — large temperature on the left with a thermometer icon whose color matches the temperature (red when hot, orange/amber for warm, blue when cool, indigo when frosty — visually reinforces the reading without claiming forecast info we don't have). `Feels like` temperature underneath, plus current visibility category. Stats column on the right: humidity %, pressure hPa, rain probability %, fog risk %.
- **Rainfall summary** — Today's mm + last 24 h, with the icon directly beside the label. Dimmed when there's been no rain.
- **Conditional warning rows (only render when applicable):**
  - "It's raining right now" with rain intensity (Drizzle / Moderate / Heavy / Storm), only while actively raining.
  - Heat warning (Caution / Extreme Caution / Danger / Extreme Danger), only when heat index is meaningful (T ≥ 27 °C and RH ≥ 40 %). Background tints orange → red as severity climbs.
  - Frost warning (Light / Hard / Severe Frost), only when frost point applies (T < 0 °C). Background tints cyan → indigo as severity climbs.
- **Sunrise / Sunset row** — pulled from HA's built-in `sun.sun` entity. Always shown, 24-hour format. Tap to open the sun entity.

**Tweaks** (in the YAML):

- Change the subtitle (default "Vanderbijlpark") in the header section.
- Adjust the temperature color thresholds inside the JS templates (search `tempColor`).
- Reorder sections by moving their card blocks within `cards:`.
- Edit the JS templates inside `[[[ ... ]]]` blocks to change content / formatting / colors.
- The whole widget adapts gracefully to narrow dashboard columns.

## Sensors

The `weather-station` device card shows ~21 entities after firmware install + HA package, plus 5 HA-side plumbing entities. Each table below has a **What it means** column explaining the metric in plain terms; implementation details are kept brief.

### Raw measurements

Direct readings from the BME280 and reed switch — the ground truth that everything else is derived from.

| Entity | Unit | What it means |
|---|---|---|
| `Temperature` | °C | Air temperature, straight from the BME280. |
| `Humidity` | % RH | Relative humidity — how saturated the air is with water vapour at the current temperature. 100 % means the air physically cannot hold any more. |
| `Pressure` | hPa | Atmospheric pressure (uncorrected for altitude). Reads lower than sea-level pressure (~1013 hPa) at altitude — at Highveld elevation expect roughly 830–850 hPa. Sea-level-corrected pressure is a planned derived metric that pulls altitude from HA's `zone.home`. The absolute value is location-bound; day-to-day **changes** matter most — falling pressure usually precedes rain. |
| `Rain Bucket` | on/off | Instantaneous reed-switch state. Flips `on` briefly each time the tipping bucket tips. Mostly diagnostic — `Rain Tips` is the useful metric. |
| `Rain Tips` | count | Cumulative bucket tips since the last reset. Multiplied by `mm_per_tip` (default 0.6314) to give rainfall. |
| `Is Raining` | on/off | `On` if at least one tip occurred in the last 5 min, else `Off`. Quick "is it actively raining right now?" flag for automations. Window length is the `raining_window_minutes` substitution. |

### Derived metrics — numeric (firmware lambdas)

Formulas evaluated on-device and published alongside the raw readings, so derived metrics live on the device card without round-tripping through HA.

| Entity | Unit | What it means |
|---|---|---|
| `Dew Point` | °C | The temperature the air would need to cool to before water vapour starts condensing into dew. The closer Dew Point is to current Temperature, the more humid and "sticky" the air feels. Magnus-Tetens formula over water saturation. |
| `Heat Index` | °C | The "feels hotter" temperature when humidity adds to heat stress (NWS Rothfusz regression). Only computed when T ≥ 27 °C **and** RH ≥ 40 % — outside that range the formula is meaningless and the entity reads `Unknown`. |
| `Frost Point` | °C | The temperature surfaces (grass, car windscreens) need to fall to before frost forms. Saturation is computed over ice, not water — different from Dew Point in cold conditions. Only meaningful below 0 °C; reads `Unknown` above. Important for gardeners and pilots in winter. |
| `Fog Probability` | % | Heuristic 0–100 score based on relative humidity and the temperature/dew-point spread. High RH + small spread → fog risk now. Not a forecast — an instantaneous read on whether current conditions favour fog. |
| `Rain Likelihood` | % | Heuristic 0–100 score combining three signals: pressure trend (falling fast → stormy), humidity trend (rising → wet air arriving), and current temperature/dew-point spread (small spread → saturated air). Pressure and humidity trends come from HA's recorder history. Settles to low values in stable conditions. |
| `Rainfall` | mm | Cumulative rainfall since the last `Rain Tips Reset` press. Long-running total — useful for monthly or seasonal rainfall. |
| `Rainfall Session` | mm | Rainfall in the current/most-recent rain event. Resets to zero when 60 minutes pass with no tips, defining the end of one rain event. |
| `Rainfall 24h` | mm | Rolling 24-hour rainfall total — always reflects the last 24 hours from now. Old rain ages out as time passes. Computed by HA's `statistics` integration and round-tripped onto the device card. |
| `Feels Like` | °C | Australian Apparent Temperature (BOM / Steadman). Combines temperature and humidity into a single "what it feels like" number. Unlike Heat Index, it's defined across the full temperature range — works in both warm and cool conditions. Stage 9 will incorporate wind once the anemometer is wired. |

### Derived metrics — categorical (firmware text lambdas)

Plain-English labels for the numbers above — easier to glance at and easier to use as automation triggers.

| Entity | Possible values | What it means |
|---|---|---|
| `Visibility` | `Fog` / `Mist` / `Haze` / `Clear` | Coarser at-a-glance version of `Fog Probability`. Derived from RH and dew-point spread thresholds. |
| `Heat Index Category` | `Caution` / `Extreme Caution` / `Danger` / `Extreme Danger` / `Not Applicable` | NWS-standard alert bands for Heat Index (≥27 / ≥32 / ≥39 / ≥52 °C). `Not Applicable` when conditions are too cool or dry for the formula to be meaningful. |
| `Frost Point Category` | `Light Frost` / `Hard Frost` / `Severe Frost` / `Not Applicable` | Horticulture/agriculture bands for Frost Point: Light (0 to −2 °C), Hard (−2 to −5 °C), Severe (below −5 °C). `Not Applicable` above 0 °C. Drives garden-protection automations. |
| `Rain Intensity` | `No Rain` / `Drizzle` / `Moderate` / `Heavy` / `Storm` | Standard meteorological intensity bands derived from `Rainfall Rate`: Drizzle <2.5 mm/h, Moderate <7.6 mm/h, Heavy <50 mm/h, Storm ≥50 mm/h. |

### Diagnostics

Health and self-test entities — surface in the device card under "Diagnostic" so they don't clutter the main view.

| Entity | Unit | What it means |
|---|---|---|
| `Wi-Fi Signal` | dBm | Raw RSSI. Closer to 0 = stronger. -50 is excellent, -70 is workable, -100 is unusable. Useful when picking a mounting location. |
| `Wi-Fi Signal Strength` | % | Same data on a friendlier 0–100 % scale (-50 dBm ≈ 100 %, -100 dBm = 0 %). |
| `Uptime` | s | Seconds since the last wake from deep sleep. Resets to ~0 every 5 minutes — that's expected, not a fault. |
| `Battery Voltage` | V | Raw LiPo voltage measured on `GPIO0` through the FireBeetle's onboard 2:1 divider. 4.2 V = full, 3.0 V = empty. The 3.3 V software cutoff fires below this. |
| `Battery` | % | State-of-charge derived from `Battery Voltage` via a piecewise LiPo discharge curve (4.2 V → 100 %, 3.7 V → 50 %, 3.4 V → 10 %, 3.0 V → 0 %). More accurate near low charge than a single linear interpolation would be. |
| `Solar Voltage` | V | Voltage at the FireBeetle's `VIN` pin, measured on `GPIO1` through the external 5:1 divider (39 kΩ + 10 kΩ). ~0 V at night, climbs to 5–7 V in sun. Stage 5.5. |
| `Charging` | on/off | `On` when `Solar Voltage` is at or above `${min_charging_voltage}` V (default 4.5 V) — the minimum the FireBeetle's MPPT needs to actually charge the LiPo. Below the threshold the panel isn't producing enough to charge regardless of battery state. Stage 5.5. |

### Controls

| Entity | What it does |
|---|---|
| `Restart` | Soft-reboots the device (button entity in HA). |
| `Rain Tips Reset` | Zeros `Rain Tips`, `Rainfall Session`, and the last-tip timestamp. Use after physical bucket recalibration so the cumulative totals don't include test pours. |

### HA-side plumbing (hidden by default)

These `derivative` / `history_stats` / `template` sensors and one `utility_meter` live in Home Assistant because they need historical data — the device sleeps too much to compute rolling windows itself. Several are subscribed back into the firmware via the native API and feed the device-side derived metrics. Hide the internal ones via **Settings → Devices & Services → Entities** if they bother you.

| Entity | Unit | What it means |
|---|---|---|
| `Pressure Change Rate 3h` | hPa/s | 3-hour pressure trend (linear-regression derivative). Negative = falling. Internal — feeds `Rain Likelihood` on the device. |
| `Humidity Change Rate 2h` | %/s | 2-hour humidity trend. Positive = rising. Internal — also feeds `Rain Likelihood`. |
| `Rainfall Rate` | mm/h | 15-minute rainfall derivative. Internal — feeds `Rain Intensity` categorisation. |
| `Rainfall 24h Tips` | count | Tip events on the reed switch over the last 24 hours (`history_stats` count). Internal — input to `Rainfall 24h Source`. |
| `Rainfall 24h Source` | mm | `Rainfall 24h Tips` × `mm_per_tip` (template sensor). Internal — feeds the device-card `Rainfall 24h` entity. Reset-safe by construction (counts events, not counter deltas). |
| `Rainfall Today` | mm | Daily rainfall total resetting at midnight. **User-facing** — useful for dashboard cards. Stays in HA, doesn't round-trip. |

## Build stages

Sequential — each has its own verification gate. See [`CLAUDE.md`](./CLAUDE.md) § Build Stages for the full roadmap with rationale.

- [x] **Stage 0** — Bring-up (Wi-Fi, native API, OTA, status LED, restart button)
- [x] **Stage 1** — BME280 over I²C + derived metrics + HA trend package
- [x] **Stage 2** — Reed-switch rain-gauge tip counting + session detection + intensity categorisation
- [x] **Stage 3** — Deep sleep + 5-minute wake cycle + reed-wake + helper-toggle for OTA
- [x] **Stage 4** — Battery ADC + piecewise LiPo curve + low-voltage cutoff at 3.3 V
- [ ] **Stage 5** — Solar input + Schottky diode (next; **1N5822** in series with panel +ve into `VIN`)
- [ ] **Stage 5.5** — Solar charging detection (5:1 voltage divider on `GPIO1` for VIN sense; `Solar Voltage` + `Charging` entities)
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

## Wiring reference

Pin assignments on the FireBeetle 2 ESP32-C6 (DFR1075). The silkscreen labels (`IO1`, `A0`, etc.) are DFRobot's own names — the YAML and this section use the chip's GPIO numbers (`GPIO19`, `GPIO20`, etc.); refer to the [DFR1075 wiki](https://wiki.dfrobot.com/dfr1075/) for the silkscreen-to-GPIO mapping if you're following the silkscreen on the board.

| Function | Pin | Rail | Notes |
|---|---|---|---|
| Reed switch (rain bucket) | `GPIO4` ↔ `GND` | — | Internal pullup; deep-sleep wake source |
| BME280 SDA | `GPIO19` | `3V3` (always-on) | I²C bus |
| BME280 SCL | `GPIO20` | `3V3` (always-on) | I²C bus |
| Battery voltage sense | `GPIO0` (silkscreen `IO0`) | onboard 2:1 divider | Built-in to the FireBeetle |
| Solar voltage sense | `GPIO1` (silkscreen `IO1`) | external 5:1 divider | Stage 5.5 |
| Status LED | `GPIO15` | — | On-board, no wiring needed |

> ⚠️ The BME280 sits on the **always-on `3V3` rail**, not `3V3_C`. `3V3_C` is power-gated by `GPIO0` on the DFR1075, but `GPIO0` is repurposed in this build as the battery ADC input, so `3V3_C` is permanently disabled. Anything that needs to be powered must go on `3V3`.

### Tipping-bucket reed switch (Stage 2)

The reed switch is a normally-open glass tube with two leads. A neodymium magnet on the tipping bucket arm closes the switch briefly each time the bucket tips.

```
   GPIO4  ──┬── reed ──── GND
            │
        (internal pull-up to 3V3 inside the C6 — enabled in YAML)
```

- One reed lead to **`GPIO4`**, the other to **`GND`**. Polarity doesn't matter — it's a passive switch.
- **No external pull-up resistor** is needed. The firmware enables the C6's internal pullup on `GPIO4` (see `binary_sensor.Rain Bucket` in `weather_station.yaml`), so the pin idles HIGH and the reed closure pulls it LOW.
- `GPIO4` is **also configured as a deep-sleep wakeup pin** — a tip during sleep wakes the chip out of cycle and the wake-cause check in `on_boot` increments the tip counter.
- Software debounce in the YAML (`delayed_off: 100ms`) handles contact bounce. A small ceramic (e.g., 10 nF across the reed leads) is optional and only worth adding if you see spurious double-counts on a scope.
- Keep the wire run from the reed switch to the FireBeetle short (the reed lives inside the rain gauge body, the FireBeetle in the inner enclosure — typically <15 cm). Twisted-pair or shielded wire is overkill at this length.

A strapping-pin warning on `GPIO4` is expected at boot and harmless — the internal pull-up keeps it HIGH at reset, which is the safe boot mode.

### BME280 (Stage 1)

Standard 4-pin I²C breakout (VIN, GND, SDA, SCL). Most BME280 modules already have on-board pull-up resistors on the I²C lines — if yours doesn't (rare), add **~4.7 kΩ from SDA to 3V3** and **~4.7 kΩ from SCL to 3V3**.

| BME280 pin | FireBeetle pin | Notes |
|---|---|---|
| `VIN` (or `VCC`) | `3V3` (always-on rail) | **NOT `3V3_C`** — see warning above |
| `GND` | `GND` | |
| `SDA` | `GPIO19` | Silkscreened `19` on the right edge |
| `SCL` | `GPIO20` | Silkscreened `20` on the right edge |
| `CSB` | `3V3` | **Tie HIGH explicitly.** Selects I²C mode. Leaving `CSB` floating in this build caused intermittent / missing readings — don't trust the breakout to pull it up internally. |
| `SDO` | `GND` | **Tie LOW explicitly.** Sets the I²C address to `0x76` (matching the YAML). Floating makes the address indeterminate. |

- The YAML uses **address `0x76`**, which is what `SDO → GND` gives. If you tie `SDO` to `3V3` instead, the address is `0x77` — change the `address:` line in `weather_station.yaml` to match. The `i2c.scan: true` setting logs whichever address it finds on boot — check the ESPHome logs during the first wake to confirm.
- The BME280 must sit **inside the Stevenson screen** (3D-printed multi-plate radiation shield) so it reads true ambient air temperature, not the inner enclosure or sun-heated surfaces. The cable run from the inner enclosure to the Stevenson screen is the cable run referenced as TBD in [`CLAUDE.md`](./CLAUDE.md) § Open Questions.
- Keep the BME280 away from the C6 itself — the ESP32-C6 warms up a few °C during WiFi TX, which corrupts temperature readings if the sensor is in thermal contact.
- Confirm it's a genuine BME280 (not a BMP280 lookalike with no humidity sensor) using the breath test in [§ 7. Verify the BME280 is genuine](#7-verify-the-bme280-is-genuine).

### Solar voltage divider (Stage 5.5, planned)

The 6 V solar panel measures **7.4 V open-circuit at room temperature** (cold-morning Voc could rise to ~7.8 V with the −0.3 %/°C temperature coefficient). That's outside DFRobot's published 4.5–6 V `VIN` spec but inside the onboard MPPT IC's absolute-max input. Bench-confirm clean behaviour at full charge in bright sun before permanent install.

To detect charging, Stage 5.5 adds a **5:1 voltage divider** from `VIN` (post-Schottky) to `GPIO1`:

```
   VIN  ───[ 39 kΩ ]───┬───[ 10 kΩ ]───  GND
                       │
                    GPIO1 (ADC)

   Optional: 100 nF ceramic from GPIO1 to GND for noise filtering
   (parallel with the 10 kΩ low-side resistor).
```

- High-side **39 kΩ** between `VIN` and `GPIO1`.
- Low-side **10 kΩ** between `GPIO1` and `GND`.
- `GPIO1` taps the junction. With `attenuation: 12db` the C6's ADC reads up to ~3.3 V; the 5:1 ratio maps 7.8 V → 1.59 V — comfortable headroom.

Lambda multiplier in YAML: **`4.9`** (= `(39 + 10) / 10`). Reports `Solar Voltage` in volts.

A binary template `Charging` derives from `Solar Voltage − Battery Voltage`: when the panel is supplying power, VIN sits ~0.3 V or more above the LiPo's terminal voltage; when the panel is dark or the battery is full and the MPPT is idle, VIN tracks battery voltage closely or falls below it.

## Planned: BME280 satellite device

A planned second deployment moves the BME280 onto a dedicated ESP32-C6 placed in deep shade. The 3D-printed Stevenson screen on the main device introduces an ~8 °C error when direct sun lands on it — multiple revisions of the screen haven't fixed it, and printed PETG isn't equivalent to commercial radiation shields. A shaded satellite sensor sidesteps the radiation-shield problem entirely.

The satellite is battery-only — no solar panel, no charging via the FireBeetle. Cells are charged offline on a bench charger every ~17–20 months. A series Schottky diode between the cells and the FireBeetle's PH2.0 connector physically prevents the onboard charge IC from pushing current back into the cells, so plugging in USB-C (for OTA flashing) is always safe.

### Shopping list

- 1× **DFRobot FireBeetle 2 ESP32-C6** (DFR1075), same model as the main device.
- 2× **protected 21700 LiPo cells**, matched batch (same brand, same date, same starting voltage when paralleled).
- 21700 cell holders — one of:
  - 1× dual holder (Keystone 1042 or equivalent), OR
  - 2× single PCB-mount holders (Keystone 1043 or equivalent), wired in parallel on the carrier PCB.
- 1× **series Schottky diode** (charging interlock):
  - **Through-hole prototype build:** 1N5817 (DO-41, 1 A, ~0.2 V V_F at our load). Easy to hand-solder, plentiful at Communica.
  - **SMD final build:** PMEG2010EH (SOD-323, 1 A, ~0.3 V V_F) or BAT54 (SOT-23, 200 mA — adequate at our few-mA average load). Either works; PMEG has more headroom.
- 1× **PH2.0 male connector** with leads, to mate with the FireBeetle's onboard PH2.0 socket. The cell-carrier PCB output terminates in this plug.
- **BME280 breakout** — the existing bare module (chip + a few resistors + a capacitor, no power LED) is the canonical part for this build. Avoid breakouts with a power LED — they'd dominate the satellite's idle current.
- **PETG or ASA filament** for the enclosure (UV-resistant, opaque). The Stevenson screen is not used for the satellite — the entire premise is that the device sits in shade.
- Custom **cell-carrier PCB**: holds the two cells in parallel, integrates the Schottky, terminates in the PH2.0 plug. User-designed (out of scope for this readme).

South African sourcing: Communica, MicroRobotics, and Robotics.org.za stock the FireBeetle, cells, Keystone holders, and both diode form factors.

### Cell-carrier PCB topology

```
[Cell carrier PCB]                                [FireBeetle 2 ESP32-C6]
  Cell1 (+)──┬──┬── ▶| ──┬──────────────────────  PH2.0 (+)  (BAT)
  Cell2 (+)──┘  │ 1N5817 │
                │  (or SMD equivalent — cathode on FireBeetle side)
  Cell1 (−)──┬──┴─────────────────────────────────  PH2.0 (−)  (GND)
  Cell2 (−)──┘
```

The Schottky's anode goes to the cell side, cathode to the FireBeetle side. Current can only flow cells → FireBeetle; the reverse direction (charge IC → cells) is blocked.

**Trade-off:** the Schottky drops ~0.3 V at our load, so the C6's `Battery Voltage` reads ~0.3 V lower than the actual cell terminal voltage. The 3.3 V software low-voltage cutoff fires when cells are at ~3.6 V instead of ~3.3 V — modest capacity loss (~10–15 %) in exchange for never cycling the cells into the steep bottom of their discharge curve, which is good for calendar life. Estimated runtime: ~17–20 months between charges on 2× 3750 mAh cells.

The full design notes for the satellite (firmware splits, HA package changes, dashboard changes) live in the local-only spec at `docs/superpowers/specs/2026-05-11-bme-satellite-device-design.md` — implementation happens once the parts arrive.
