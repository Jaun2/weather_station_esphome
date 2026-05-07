# Weather Station Project — Claude Instructions (ESPHome edition)

## Project Overview

Building a DIY off-grid weather station that publishes data to Home Assistant. Permanent outdoor deployment in Vereeniging, South Africa (Highveld, ~1500 m altitude, southern hemisphere), running unattended on solar power for years. Tipping-bucket rain gauge plus BME280 temperature/humidity/pressure in a Stevenson screen, with planned expansion to lightning detection, wind speed/direction, soil moisture, and UV.

**Built on ESPHome.** A previous attempt at custom Arduino/PlatformIO firmware shipped Phases 1–6 (Wi-Fi + MQTT, custom OTA, BME280, reed switch, deep sleep) before being abandoned: the deep-sleep + MQTT + retained-value + OTA interactions accumulated into a debugging swamp that wasn't justified for a hobby project. ESPHome handles all of those concerns natively. The hardware and HA-side template sensor design carry forward unchanged; only the firmware layer changes.

## Reference implementation: SS4H-RG

The 3D-printed enclosure for this project comes from the **Smart Solutions 4 Home rain gauge** ([smartsolutions4home.com/ss4h-rg-rain-gauge](https://smartsolutions4home.com/ss4h-rg-rain-gauge/)). It's a working ESPHome rain gauge with open-source design files, PCB, and YAML config. We can lift several patterns from it directly, but the gauge's architecture differs from ours in important ways — copying blindly will cause confusion later. Documenting the deltas up-front:

**What we adopt directly from SS4H-RG:**

- **OTA toggle pattern** (HA `input_boolean` + ESPHome `binary_sensor` with platform `homeassistant`). Toggling the helper to OFF tells the device to skip deep sleep and stay awake; OTA install completes in that window; toggle back ON to resume normal cycles. See "OTA + dev iteration" below.
- **Battery `substitutions:` block** for `min_battery_voltage` / `max_battery_voltage` so the curve clamps are config-driven, not hardcoded. (Adjust the numbers — our LiPo is 3.0–4.2 V nominal, not their 3.6–6.0 V AAA.)
- **Lambda filter** on the battery ADC sensor that converts raw voltage to a clamped 0–100 % percentage in one place (they use a single `filters: - lambda:` block; cleaner than two derived sensors).
- **`fast_connect: True`** under `wifi:` — speeds up reconnect after every wake by skipping the SSID scan when the saved BSSID is still in range. Worth the latency saving for a deep-sleep device.
- **`setup_priority: -200`** on the battery ADC sensor so the read happens late in setup, after WiFi/API have stabilized. Reduces noise from radio activity coupling into the ADC line.
- **GPIO power-disconnect switch with `restore_mode: ALWAYS_ON`** for any analog rail that should stay live across reboots (their pattern for the battery divider; ours for the 3V3_C rail to the BME280).

**What we adapt (does NOT copy directly):**

- **Tip counting.** SS4H-RG counts tips on the HA side using `binary_sensor: status` flips + the `history_stats` integration. That works ONLY because their device wakes ONLY on a reed-switch closure (no `sleep_duration` configured — pure GPIO wake). Each wake = one tip. Our architecture has BOTH timer wakes (for BME280 every 5 minutes) AND reed wakes (for tips). Counting status flips would inflate tip counts by `wakes_per_hour`. We need to actually distinguish wake causes — see "Deep sleep + interrupt wake" below for the options.
- **Power source.** They run on AAA cells (3.6–6 V); we run on LiPo (3.0–4.2 V) with solar + onboard MPPT. Battery curve clamps differ; charging is automatic on our hardware.
- **Sensor set.** They have just a tipping bucket; we add BME280 (and later: lightning, wind, soil moisture, UV).
- **Board variant.** They use a generic ESP32 (`board: esp32dev`, `framework: arduino`); we use the DFRobot FireBeetle 2 ESP32-C6 (DFR1075). There is no dedicated PlatformIO/ESPHome `board:` definition for the DFR1075, so we use `board: esp32-c6-devkitc-1` as a generic ESP32-C6 stand-in (with `variant: esp32c6` and framework `esp-idf` for best C6 support in ESPHome). The board ID supplies framework / flash-size / partition defaults only — **all DFR1075-specific pin assignments must be set explicitly in the YAML** (I²C `sda`/`scl`, battery ADC pin, 3V3_C control pin, status LED, BOOT pin, deep-sleep wake pins). Don't trust any pin-name aliases the generic devkit board file might imply; the DFR1075's silkscreen and the DFRobot wiki are the source of truth. Wake-source API is also different from the original ESP32 — see below.

**Their full YAML and PCB design files** live at https://download.smartsolutions4home.com/ — useful as a known-good starting point for our own config.

## My Background

I'm a software developer (mainly Laravel, Vue.js, Inertia, Tailwind, shadcn-vue, MySQL). This weather station is a hobby hardware project — I'm comfortable with code and embedded development concepts but not a hardware engineer. I appreciate explanations that respect my existing technical knowledge while being clear about hardware-specific concepts.

## Hardware

### Main electronics

- **MCU**: DFRobot FireBeetle 2 ESP32-C6 (DFR1075) — wiki: [wiki.dfrobot.com/dfr1075](https://wiki.dfrobot.com/dfr1075/) (canonical reference for pinout, schematic, and electrical characteristics — defer to the wiki, not the generic ESP32-C6-DevKitC-1 datasheet, when they disagree)
  - 160 MHz RISC-V single-core
  - 4 MB flash, no PSRAM
  - Wi-Fi 6 (2.4 GHz only), BLE 5, Zigbee 3.0, Thread 1.3
  - 16 µA deep sleep
  - Built-in MPPT solar charge controller
  - PH2.0 LiPo battery connector
  - VIN pin accepts 5 V DC OR 4.5–6 V solar panel
  - `IO1`: built-in battery voltage detection pin (ADC, with onboard divider)
  - `GPIO0` (3V3_C): software-controlled 3.3 V output for power-gating sensors
  - `GPIO19` = SDA, `GPIO20` = SCL — must be mapped explicitly in the i2c bus config; silkscreened on the right edge of the FireBeetle 2 V1.0. `GPIO9` and `GPIO10` are NOT broken out on this board, despite earlier dev-kit notes.
  - `GPIO9` is the labeled `BOOT` button, also the boot-mode strapping pin (LOW at reset → download mode).
  - `GPIO15` is the onboard LED.
  - USB-C for programming and serial monitor (USB-CDC; `usb_serial_jtag` in ESPHome).

- **Battery**: 5000 mAh LiPo (955565 form factor) with PCM protection, PH2.0 connector — plugs directly into FireBeetle.
- **Solar**: 6 V / 4.5 W panel (~750 mA peak at 6 V). Verify Voc < 7 V before connecting; add 1N5817 Schottky diode in series for safety / reverse-current protection.
- **Charging**: External only — no charging built into the rain gauge body. Battery is swappable. Solar handles top-up. USB-C for emergency charging.

### Sensors

- **Rain**: Tipping bucket with reed switch + neodymium magnet (5–6 mm disc), normally-open glass reed switch. Calibrate by dripping a known volume.
- **Temperature / Humidity / Pressure**: BME280 (I²C, address `0x76` or `0x77` — verify genuine via "breath test" before use; humidity should jump 5–10 % within seconds. BMP280 lookalikes have no humidity).

### Mechanical

- 3D-printed inner enclosure inside outer rain gauge body (PETG preferred for outdoor parts).
- 3D-printed Stevenson screen (multi-plate radiation shield design recommended for v1, mounted at 1.5 m on grass, away from walls).
- Outer rain gauge body protects inner electronics from direct rain / UV.
- Inner enclosure sealed with Rust-Oleum Flexi Dip (already owned, applied as 3–4 light coats over lightly sanded print).
- Cable entry pointing downward with drip loop.
- Small vent hole at bottom of inner enclosure for air equalization.
- Solar panel mounted on top of rain gauge body, facing north, tilted ~27° (latitude) for year-round, or ~40° to favor winter.

## Architecture Decisions (already made)

### ESPHome over custom firmware

The custom-firmware path was worked through Phases 1–6 (PlatformIO + Arduino-ESP32 v3.x, PubSubClient for MQTT, WiFiManager for captive portal, custom OTA via retained `ota/command`, custom deep-sleep wake-cause handling, RTC-resident tip counter, etc.). It mostly worked but produced a steady stream of subtle bugs: LWT semantics breaking on deep-sleep, OTA retained-PRESS retry loops, partial publishes from TCP send buffer not draining, cross-boot debounce against a `micros()` clock that resets on wake, duplicate publishes per wake from un-traced MQTT reconnect cycles. Each fix surfaced another corner case. ESPHome's deep-sleep + native API + OTA + sensor framework solve all of these out of the box.

The trade-off: less low-level control, more reliance on a framework's choices. Acceptable for a personal weather station that won't be resold or productized.

**Don't suggest reverting to custom Arduino/PlatformIO firmware.** The decision was deliberate after extensive bench experience.

### Native API over MQTT

ESPHome's `api:` component (native HA API) talks directly to Home Assistant's ESPHome integration. HA auto-discovers the device with no broker round-trip, no retained-message handling, no discovery JSON to write. Lower latency, more robust reconnection, encrypted by default (per-device key in `secrets.yaml`).

MQTT is still available in ESPHome if needed (other consumers besides HA, or for the rare HA-down scenario), but for an HA-only setup, native API is strictly better. Stick with native API unless there's a specific reason.

### Deep sleep + interrupt wake

ESPHome's `deep_sleep` component handles this. The shape we want:

```yaml
deep_sleep:
  id: weather_deep_sleep
  run_duration: 15s          # awake window: connect + publish + grace
  sleep_duration: 5min       # timer wake every 5 minutes
  wakeup_pin:                # for ESP32-C6 LP-IO wake on the reed pin
    number: GPIO4
    inverted: true           # reed pulls to GND when closed
  wakeup_pin_mode: IGNORE    # see notes below
```

(Exact YAML keys / wake-source block depend on ESPHome's deep_sleep component version — the `esp32_ext1_wakeup` form works on the original ESP32 but for the C6's LP-IO domain it may be a different sub-key. Check the ESPHome docs for the version you're on, and verify against the SS4H-RG config as a reference.)

**Tip-counting options** — pick one during Stage 3:

1. **Wake-cause-based tip increment** (matches our dual-wake architecture cleanly). Use a `globals:` int with `restore_value: yes` (persists in flash NVS across deep sleep + power cycles). On boot, check `esp_sleep_get_wakeup_cause()` in a lambda; if it's a GPIO/EXT1 wake, increment the global. Publish the global as a `template` sensor on every wake. Trade-off: edge cases like manual button reset or brownout could miscount; mitigate with a sanity-check window.

2. **`pulse_counter` component** on PIN_REED with the GPIO also configured as a deep-sleep wake source. Treats the reed as a frequency input. Need to verify across-deep-sleep retention semantics for the count — ESPHome may need a `globals` mirror to persist properly (the pulse_counter's internal state is RAM-resident).

3. **HA-side counting via `binary_sensor` + `history_stats`** (the SS4H-RG approach). Doesn't work directly for us because timer wakes also trigger status flips. Could be made to work by publishing a separate `binary_sensor` that's true only when the wake cause is GPIO, then HA counts those.

Option 1 is the simplest given our architecture. Don't over-design it — try the simplest approach first, verify it on the bench with magnet swipes during sleep, only switch approaches if measurement reality shows a problem.

**OTA + dev iteration** — adopt SS4H-RG's pattern verbatim:

In Home Assistant, create a helper toggle: Settings → Devices & Services → Helpers → CREATE HELPER → Toggle, named `weather_station_deep_sleep`. ON = device sleeps normally. OFF = device stays awake (no `deep_sleep.allow:` action fires).

In the YAML:

```yaml
binary_sensor:
  - platform: status
    name: "Weather Station Status"
  - platform: homeassistant
    id: weather_deep_sleep_enable
    entity_id: input_boolean.weather_station_deep_sleep
    publish_initial_state: true
    on_state:
      then:
        if:
          condition:
            lambda: 'return x;'   # x is the new state
          then:
            - deep_sleep.allow: weather_deep_sleep
          else:
            - deep_sleep.prevent: weather_deep_sleep
            - switch.turn_on: status_led    # visible bench-test indicator
```

Workflow: OFF the toggle in HA → device stays awake on next wake → flash new YAML via OTA → ON the toggle → device resumes 5-min sleep cycles. No physical USB needed for routine updates.

We do NOT write RTC-memory tip counters, wake-cause branching in C++, MQTT reconnect logic, or any of the corner-case handling that bit us in the custom firmware. ESPHome handles those.

### Power gating via 3V3_C

`GPIO0` is the 3V3_C rail enable on the DFR1075. ESPHome configures it as a `gpio` `output` (active HIGH); a `switch` component drives it for sensor power-cycling, or a `lambda` in `on_boot` / `before_deep_sleep` controls it directly. The BME280 (and any future I²C sensors that don't need to be always-on) sit on this rail and get powered down between wakes.

Always-on sensors (e.g., AS3935 lightning detector — must catch strikes during sleep) sit on the always-on 3V3 rail, not 3V3_C.

### Power architecture

- VIN pin handles both 5 V DC (USB-C) and 4.5–6 V solar input.
- Onboard MPPT chip optimizes solar harvest.
- Power-path management automatically switches between solar / battery.
- Battery monitoring via `IO1` ADC with the onboard divider (verify ratio against the DFR1075 schematic before trusting absolute values — the lambda multiplier in the YAML reflects this).
- Software low-voltage cutoff at ~3.3 V → permanent deep sleep until USB-C reset (PCM hard cutoff at ~2.75 V is the hardware backstop). Implemented as a `lambda` in `on_value` of the battery voltage sensor that calls `id(weather_deep_sleep).prevent_deep_sleep()` and forces a long sleep.
- Target: years of unattended operation. Realistic expectation: 3–5 years before LiPo replacement (calendar aging dominates over cycle aging given the partial-charge profile).

**Battery monitoring YAML shape** (lifted from SS4H-RG, adjusted for our LiPo voltage range):

```yaml
substitutions:
  min_battery_voltage: '3.0'   # LiPo cutoff
  max_battery_voltage: '4.2'   # LiPo full

sensor:
  - platform: adc
    pin: IO1                   # DFR1075's battery sense pin
    name: "Battery Level"
    id: battery_level
    device_class: battery
    unit_of_measurement: "%"
    accuracy_decimals: 0
    setup_priority: -200       # read after WiFi/API stabilize
    update_interval: never     # only sample on wake (deep-sleep cycle)
    filters:
      - lambda: |-
          float v = x * <DIVIDER_RATIO>;   // verify against DFR1075 schematic
          if (v < $min_battery_voltage) return 0;
          else if (v > $max_battery_voltage) return 100;
          else return ((v - $min_battery_voltage) /
                       ($max_battery_voltage - $min_battery_voltage)) * 100;
```

The LiPo discharge curve is non-linear (4.2 V → 100 %, 3.7 V → ~50 %, 3.4 V → ~10 %). Linear interpolation between min and max is good enough for a "needs charging" indicator. If a more accurate percentage matters, do the lookup table on the HA side as a template sensor — easier to tweak without re-flashing.

### Why these choices (avoid revisiting)

- **C6 over C5**: C5's dual-band Wi-Fi 6 not needed; C6 has explicit MPPT, more mature software ecosystem, slightly better deep sleep.
- **C6 over S3 (camera variant)**: camera and PSRAM are dead weight for this application.
- **5000 mAh LiPo over 2× 18650**: simpler, fewer components, plugs directly into PH2.0.
- **Native API over MQTT**: HA-only setup, native is tighter and more robust.
- **No SD card**: HA stores history; offline buffering is unnecessary for a 5-min sample cadence — short outages are tolerable.
- **External charging only**: removes most dangerous part of LiPo design, simplifies PCB.
- **Reed switch over Hall sensor**: lower power, no continuous draw, well-proven for tipping buckets, dead simple.
- **ESPHome over custom firmware**: see above. Extensive prior bench experience with the custom approach informed this choice.

## Power Budget (for reference)

- ESP32-C6 deep sleep: ~16 µA.
- BME280 powered off via 3V3_C: ~0 µA.
- Wake every 5 min for ~10–15 s at ~80 mA average: ~0.4 mAh / hour.
- ~6 mAh per 15-hour overnight period out of 5000 mAh capacity (~0.12 %).
- Solar 4.5 W panel produces ~2000–3000 mAh on a sunny Vereeniging day vs ~10 mAh daily consumption — vast surplus.
- **Power is NOT a meaningful constraint for this project** with current sensor count.
- The chip could publish every 30 seconds and still have multi-week autonomy without sun.

## Home Assistant Integration

### Sensors published via Native API

**Raw**:
- Temperature (°C)
- Humidity (%)
- Pressure raw (hPa)
- Rainfall tips count (cumulative)
- Battery voltage (V)
- Wi-Fi signal strength (dBm, ESPHome's `wifi_signal` sensor — diagnostic)

**Derived in HA via template sensors** (no firmware changes needed to add or modify):
- Sea-level pressure (uses HA's elevation from `zone.home`)
- Dew point (Magnus formula)
- Heat index (when T ≥ 27 °C and RH ≥ 40 %)
- Pressure trend (3-hour gradient via HA's `trend` sensor)
- Fog probability (uses `sun.sun` for time-of-day awareness)
- Frost warning (pre-sunrise low temp + dew point below 0)
- Battery percentage (LiPo discharge curve lookup, not linear — 4.2 V → 100 %, 3.7 V → ~50 %, 3.4 V → ~10 %)
- Rainfall in mm (tips × calibrated mm-per-tip), plus a daily utility meter for per-day totals. The mm-per-tip starting point depends on the bucket: SS4H-RG with a 110 mm-radius funnel and 6 ml bucket arrives at 0.6314 mm/tip via `(1,000,000 / πr²) × bucket_volume_ml`. Calibrate by pouring a measured volume slowly into the funnel, counting tips, and dividing — the printed-bucket geometry can drift a few percent from the design number.
- Rain likelihood (combines pressure trend + humidity rising + dew-point spread)
- UV estimate (from sun elevation, clear-sky approximation)
- Daylight remaining
- Optional: Zambretti forecaster (HACS integration)
- Optional: VPD (vapor pressure deficit, for gardening)

Some derived metrics (dew point, heat index, sea-level pressure) can also be computed in ESPHome itself via `template` sensors if preferred — but keeping them HA-side means they can be tweaked without re-flashing.

### Use HA's built-in location data

- `state_attr('zone.home', 'latitude')`
- `state_attr('zone.home', 'longitude')`
- Elevation: 1500 m for Vereeniging (hardcode is fine — it doesn't change).
- `sun.sun` integration provides sunrise, sunset, dawn, dusk, current sun elevation / azimuth — use these for time-of-day-aware logic instead of `now().hour`.

## Future Expansion (designed-for, not yet built)

ESPHome has a built-in component for almost everything we'd want to add. The firmware side is trivial; the hardware-side considerations (mounting, power rails, calibration) are where the real work is.

### Lightning detector (AS3935) — high-priority post-deployment expansion

**Hardware:** SparkFun Qwiic AS3935 breakout (SPX-15441 or equivalent). I²C address `0x03` (default) — no conflict with BME280 (`0x76`/`0x77`) or AS5600 (`0x36`). Wires onto the existing I²C bus (GPIO19/GPIO20 on the C6); IRQ to a free LP-IO pin (e.g., GPIO5 — confirm against the C6 LP-pin list when wiring).

**Architectural changes this introduces:**

- **Power rail change.** The AS3935 must sit on the **always-on 3V3 rail**, not the gated 3V3_C, because power-gating it between wakes would defeat the point — strikes that happen while the ESP32 sleeps would be missed. Adds ~60 µA continuous quiescent draw on top of the C6's ~16 µA. Daily cost ~1.8 mAh; negligible vs ~2000–3000 mAh/day solar harvest on a sunny Vereeniging day.
- **Second wake source.** The current dual-wake architecture (timer + reed) becomes triple-wake (timer + reed + AS3935 IRQ). On the C6, ESPHome's `deep_sleep` `wakeup_pin` (or the underlying `esp_deep_sleep_enable_gpio_wakeup` LP-IO API) accepts a bitmask of pins, and ESPHome's `binary_sensor` on the IRQ pin distinguishes which fired via the AS3935's interrupt-source register. Wake-cause branching in our `on_boot` lambda extends from "timer or reed" to "timer or reed or lightning."

**ESPHome integration:** the `as3935_i2c` component (built-in) handles the chip's I²C protocol, event types (lightning strike vs disturber vs noise), distance estimation, and energy reporting. Configure noise floor, watchdog threshold, and spike rejection at the YAML level — surface them as MQTT-set retained config topics or via HA-side service calls so they can be tuned post-deployment without re-flashing.

**LCO antenna calibration:** the AS3935's loop antenna must be tuned within ±3.5 % of 500 kHz for accurate distance estimation. ESPHome's component exposes a calibration routine; run it once per physical install. Re-run if the antenna environment changes significantly (new metal in proximity, etc.).

**Mounting considerations** — these are the load-bearing decisions, not the YAML:

- **Keep the antenna away from the ESP32 itself.** The C6's WiFi radio during TX is the loudest disturber source you'll have on this device. Don't share an enclosure with the MCU.
- **Don't put it inside the Stevenson screen.** Metal mesh proximity messes with the antenna, and you want elevation + an unobstructed RF path.
- **Suggested layout:** small dedicated outdoor-rated PETG enclosure on the same pole as the rain gauge or Stevenson screen, short Qwiic cable run to the I²C bus, same Flexi Dip waterproofing treatment as the rain gauge inner enclosure.

**Risks to budget for:**

- **False positives from local RF.** Load-shedding switching transients, neighbour's pool pump, your own WiFi during TX. Mitigation: disturber-rejection tuning + physical separation from the MCU. Budget an evening of post-installation tuning before trusting any push-notification automation that fires on detected strikes.
- **I²C bus contention if cable run is long.** Keep it short. Add pull-ups only if the Qwiic board's are missing or the bus is already pulled up weakly.

**HA integration:** discovery payloads via native API for last strike distance (km), strike count (cumulative), last strike timestamp, last diagnostic event type. Useful HA automation: trigger on `last_strike_distance_km < 20`, condition `(now - last_notification) > 5 min`, action mobile push notification — gives you ~10–20 min warning before a storm closes in.

### Wind speed (cup anemometer) + direction (AS5600 vane) — high-priority post-deployment expansion

**Hardware:**
- 3D-printed 3-cup head, PETG, lightweight cups (ping-pong-half size on ~50 mm arms), proper miniature shielded ball bearings (MR74ZZ or 623ZZ — Bearing Man Group, Vereeniging). Magnet on cup hub + reed switch in stationary base, same pattern as the tipping bucket.
- 3D-printed vane on a separate shaft *above* the cups (so cups don't shadow the vane), diametrically-magnetized 6 mm magnet glued to the underside of the vane shaft, AS5600 breakout below in the stationary base with ~1–2 mm air gap to the magnet.
- Both shafts in a shared PETG body mounted on the highest practical pole (roof line or dedicated mast).

**Architectural changes this introduces:**

Smaller than the lightning detector — both sensors slot cleanly into existing infrastructure:

- **No new wake sources.** The cup-head reed switch is **not** wired as a wake pin. Wind speed is sampled within the existing 5-min timer-wake window (count pulses for ~10 s while awake anyway), not by waking on every pulse. With cups potentially spinning at 20–50 Hz in moderate wind, ext-wake-per-pulse would torch the power budget; sampling within the existing wake cycle is essentially free. This means the wind expansion does NOT depend on the lightning detector's wake-source extensions.
- **AS5600 on the existing I²C bus.** Address `0x36`, no conflict with BME280 (`0x76`/`0x77`) or AS3935 (`0x03`). Powered from 3V3_C — gated off between wakes alongside the BME280.
- **`PIN_WIND_PULSE` on a free GPIO** for the cup-head reed (suggest GPIO6 on the C6 — confirm against pinout when wiring). Standard ESPHome `pulse_counter` component, only active during the awake sampling window; no wake-source plumbing.

**ESPHome integration:**
- **Wind speed:** `pulse_counter` component on the wind pin, configured to sample over a window (~10 s) within each timer wake. Compute pulses-per-second × per-revolution calibration → m/s. 5 ms software debounce on the ISR (cups can spin to ~50 Hz; period 20 ms, so 5 ms is safe).
- **Gust:** track max pulses-in-any-3-second-window across the 10 s sample (small ring buffer in lambda or via ESPHome's `max` filter on a sub-sampled sensor). Approximates the WMO 3-s gust definition closely enough for amateur use.
- **Vane reading:** built-in `as5600` component on the I²C bus. One read per timer wake. Subtract a `WIND_NORTH_OFFSET` substitution for physical alignment. Surface the AS5600's `MAGD` / `MH` / `ML` magnet-status bits in startup logs so alignment issues are obvious.

**Calibration:**
- **Speed:** drive a car at known speed in still air holding the cup head out the window, OR sit beside a handheld anemometer for an afternoon. Adjust the m/s-per-Hz factor — 3D-printed cups will not match the Davis 6410 nominal of 1.006 m/s/Hz.
- **Direction:** with the head fixed in its final mounting, rotate the vane to point at known compass north and record the AS5600 raw reading; that's `WIND_NORTH_OFFSET`. Decide whether to correct for magnetic declination (Vereeniging is ~−21°) in firmware or in an HA template — either works; pick one and document it.

**Risks to budget for:**

- **Bearing friction is the make-or-break.** If startup wind threshold is high, low-wind readings are useless. Use proper miniature ball bearings (not nylon bushings), balance the cups by mass before assembly, lubricate sparingly with dry PTFE — never WD-40, it attracts dust and fails outdoors within months.
- **AS5600 magnet alignment.** Must be **diametrically magnetized** (not axial), centered over the chip, ~1–2 mm air gap. The chip's magnet-status bits flag too-strong / too-weak / missing magnet — log them at boot and during the first few publishes so a marginal install reveals itself before deployment.
- **Reed contact bounce under fast rotation.** 5 ms software debounce handles it, but if pulses look flaky on a scope, a 10 nF cap across the reed terminals smooths things mechanically.

**HA integration:** discovery payloads for wind_speed (m/s), wind_speed_gust (m/s), wind_direction (°). HA template sensors for cardinal direction (N / NE / … / NW), Beaufort scale (0–12), wind chill (when T < 10 °C), and optionally a "non-WMO mounting height" disclaimer attribute so future-you remembers the data isn't taken at the standard 10 m.

### Lower-priority sensors

These are wired-for in the carrier but not on the active roadmap:

- **Soil temperature** (DS18B20): ESPHome's `dallas_temp` component on a one-wire bus.
- **Soil moisture** (capacitive probe): `adc` component on a free ADC pin, gated on 3V3_C between reads.
- **UV sensor** (VEML6075 or LTR390): I²C component on the existing bus, also on 3V3_C.
- **Light sensor / LDR** (solar performance proxy, day/night detection): `adc` component.

Power budget allows comfortably for all of these on top of the lightning + wind set. Wire layout should leave room. Field deployment should happen with just the BME280 + tipping bucket first; add expansions one at a time after the base station is stable.

## Build Stages (sequential, do not parallelize)

Each stage has its own verification gate. Don't advance until the current one is solid.

1. **Stage 0 — Bring-up.** Get an ESPHome YAML compiling and flashing onto the DFR1075 over USB-C. Because there is no dedicated ESPHome/PlatformIO board definition for the DFRobot FireBeetle 2 ESP32-C6, use `board: esp32-c6-devkitc-1` with `variant: esp32c6` and `framework: type: esp-idf` as the generic ESP32-C6 base — the IDF framework has more mature C6 support in ESPHome than `arduino`. **The board ID is just a framework hint — pin mappings on the DFR1075 differ from the generic dev kit and must be set explicitly per the DFRobot wiki ([wiki.dfrobot.com/dfr1075](https://wiki.dfrobot.com/dfr1075/)).** Specifically: I²C on `GPIO19`/`GPIO20`, status LED on `GPIO15`, BOOT button on `GPIO9`, battery ADC on `IO1`, 3V3_C control on `GPIO0` — see the Hardware section. Confirm flash size is 4 MB in the build. Verify Wi-Fi connects (with `fast_connect: True`), native API talks to HA (device shows up in HA's ESPHome integration), `wifi_signal` and `uptime` sensors visible. Test the OTA path (push a tiny config change, see it install via HA). Set up the `weather_station_deep_sleep` HA helper toggle here so you can stay-awake the device for fast iteration in subsequent stages.
2. **Stage 1 — BME280 over I²C.** Add the `i2c` bus, `bme280_i2c` component on `0x76` or `0x77`. T/H/P sensors appear in HA. Breath test confirms it's a real BME280. Sensible values for Vereeniging at 1500 m (~840 hPa raw).
3. **Stage 2 — Reed switch tip counting.** `pulse_counter` component on the reed pin, configured for total count. Wave a magnet, count goes up by 1 per pass. HA template sensor multiplies by `mm-per-tip` for rainfall in mm; HA utility meter integrates per-day total.
4. **Stage 3 — Deep sleep cycle.** `deep_sleep` component with 5-min `sleep_duration` and reed-pin wake. `run_duration` ~15 s (enough for Wi-Fi + native API + publish). Verify on USB power meter: the C6 should hit ~16 µA during sleep. Tip count survives sleep cycles.
5. **Stage 4 — Battery voltage monitoring.** `adc` sensor on `IO1`, sampled before Wi-Fi connects (use `on_boot` priority). Lambda implements low-voltage cutoff (~3.3 V → indefinite sleep). HA template computes battery percentage from LiPo curve.
6. **Stage 5 — Solar input + Schottky diode.** Hardware-only: 1N5817 in series with the panel, into VIN. Confirm via the battery voltage sensor that charging happens during sun.
7. **Stage 6 — Mechanical assembly.** 3D-printed inner enclosure inside outer rain gauge body, Stevenson screen, Flexi Dip waterproofing, cable runs with drip loop.
8. **Stage 7 — Field deployment + calibration.** Side-by-side BME280 calibration with a reference for 1–2 days. Tipping bucket calibration: pour known volume, count tips, compute mm-per-tip. Mounting: 30 cm rim height for the rain gauge, 1.5 m on grass for the Stevenson screen.
9. **Stage 8 — Lightning detector (post-deployment).** Adds the AS3935 on the always-on 3V3 rail (NOT 3V3_C), wires the IRQ to a free LP-IO pin as a third deep-sleep wake source alongside timer + reed. ESPHome's `as3935_i2c` component handles the chip protocol; LCO antenna calibration runs once per install. Wake-cause branching extends to distinguish lightning from reed. Mounting in a separate enclosure on the same pole, away from the C6's WiFi radio. HA automation fires push notification when last strike distance < 20 km. See "Future Expansion" section above for the architectural deltas, mounting constraints, false-positive risks (load-shedding transients, neighbour pool pumps), and budget for an evening of disturber-rejection tuning post-install.
10. **Stage 9 — Wind speed + direction (post-deployment).** Adds a 3-cup anemometer on a roof-mounted mast and an AS5600 magnetic-encoder vane above it. Wind speed via `pulse_counter` sampling within the existing 10 s awake window — NOT a deep-sleep wake source (cups at 50 Hz would torch the power budget). Wind vane via `as5600` on the existing I²C bus. Calibration is the work: bearing friction must be low (proper miniature ball bearings, dry PTFE only), AS5600 magnet must be diametrically magnetized and well-aligned, m/s-per-Hz factor calibrated against a handheld anemometer or a known-speed drive. See "Future Expansion" section for the full hardware design, magnet-status logging, and HA integration (Beaufort scale, wind chill, mounting-height disclaimer).
11. **Stage 10 — Lower-priority sensors (later, as needed).** Soil temperature (DS18B20), soil moisture (capacitive ADC), UV (VEML6075/LTR390), LDR. All slot onto the existing I²C bus or free ADC pins, all gated on 3V3_C. Each is a YAML-component-add operation; no architectural changes. Defer until the base station + wind + lightning are deployed and stable.

## ESPHome Conventions

- **One YAML per device.** Versioned in git. The full config (including substitutions and includes, but not secrets) lives in the repo.
- **Substitutions** (`substitutions:` block) for device-specific values: device name, friendly name, board variant. The YAML body uses `${name}` so the same YAML can be reused with different substitutions.
- **Secrets** (Wi-Fi password, OTA password, native API encryption key) in `secrets.yaml`. **Gitignored.** Use `!secret <key>` in the YAML.
- **External components** for anything not built-in: declare `external_components:` pointing to a git source or local directory. Don't fork ESPHome; layer custom components on top.
- **Custom logic in `lambda:`** when it can't be expressed via core components. Lambda is C++ with `id()` to access components. Keep them small; if a lambda is more than ~20 lines, write a real custom component instead.
- **Pin syntax**: use `GPIO19` (not `19`) for clarity. ESPHome accepts both but the named form is self-documenting.
- **i2c** bus declared once at top with explicit `sda` / `scl` pins; sensors reference it by `id`.
- **Logger level**: `DEBUG` during dev iteration, `INFO` for deployed devices (less log spam over the API connection).
- **`web_server:`** enabled during dev for inline debugging (browse to the device's IP, see live values, force OTA, etc.); disabled for deployed devices to save flash and CPU.
- **OTA password**: long random string in `secrets.yaml`. Re-flashing without it requires USB.
- **API encryption key**: 32-byte base64, generated once per device, in `secrets.yaml`. Native API connections without it are refused.
- **`fast_connect: True`** under the `wifi:` block. After the first associate, ESPHome remembers the BSSID and skips the SSID scan on subsequent connects — measurable wake-budget saving on a deep-sleep device.
- **`update_interval: never`** on sensors that should sample only when awake (battery ADC, BME280) — the deep-sleep + wake cycle drives sampling, not a polling timer.
- **Lambda filters over derived sensors** when the goal is "raw → human-readable" in one step (e.g., ADC voltage → battery percentage). One lambda is cheaper than two sensors and a template.
- **Captive-portal AP fallback** under the `wifi:` block — if the configured SSID is unavailable, ESPHome's `captive_portal:` lets you reconfigure WiFi without re-flashing.

## What Claude Should Help With

- **ESPHome YAML**: writing, debugging, refactoring; explaining component options and how they interact.
- **Custom components** when a built-in doesn't fit: Python registration + C++ component code following ESPHome's external-component pattern.
- **HA configuration**: template sensors, automations, dashboards (Lovelace), helper entities (utility meter, trend sensor), driving the data we publish into useful UI.
- **Hardware troubleshooting**: power consumption debugging, wiring, sensor calibration, signal integrity.
- **3D printing**: Stevenson screen design, enclosure considerations, material selection (PLA vs PETG vs ASA).
- **Calculations**: dew point, sea-level pressure, heat index, derived weather metrics — both as ESPHome lambdas and as HA templates, depending on which side they belong.
- **Build sequencing**: what to do first, how to test in isolation, how to keep one verification gate solid before moving to the next.
- **Mechanical**: tipping bucket geometry, magnet placement, reed switch mounting.

## What's Out of Scope / Not Needed

- ❌ **Don't suggest revisiting hardware decisions** unless I explicitly ask. The C6, the LiPo, the panel, the swappable-battery design are all decided.
- ❌ **Don't suggest commercial weather stations as alternatives** — the point is to build it.
- ❌ **Don't add SD card storage suggestions** — explicitly rejected, HA is storage.
- ❌ **Don't suggest reverting to custom Arduino / PlatformIO firmware**. We tried it through Phase 6, it was a debugging swamp, ESPHome is the deliberate choice.
- ❌ **Don't disable ESPHome's native OTA** — it actually works (button-press, version-aware, rollback-safe). The custom-firmware OTA mess was a custom-firmware problem; ESPHome's OTA is well-engineered.
- ❌ **Don't over-engineer power optimization** — we have plenty of headroom. Sensible defaults are fine; chasing single-digit microamps isn't worth complexity.
- ❌ **Don't suggest BME680 / BME688 for VOC sensing** — explicitly rejected for outdoor weather; air-quality measurements outside aren't useful.
- ❌ **Don't suggest live application web servers on the ESP32** for end-user UI. ESPHome's `web_server:` is fine for dev but the user-facing dashboard is HA, not the device.

## Communication Style

- **Be direct and honest**, including pushback when I'm wrong about something. Don't just validate.
- **Be willing to say "I don't know, let me check"** rather than confidently making up specs. Reference the DFRobot wiki for board details, ESPHome docs for component details.
- **Concrete YAML / numbers / pin assignments over vague advice** — actual snippets, actual GPIO numbers, actual values. Verify against datasheets and ESPHome's component reference.
- **Explain trade-offs**, not just recommendations — I want to understand why so I can apply the reasoning to future decisions.
- **Acknowledge when something is over-engineered** for the use case, even if it's "more correct" in some abstract sense.
- **Reference the Stage list** when I ask about something — keeps build sequencing in mind.
- **South African context where relevant** — local suppliers (Communica, MicroRobotics, Robotics.org.za, Bearing Man Group), Highveld climate (frost June–August, peak UV summer, occasional load shedding affecting Wi-Fi / HA availability).

## Open Questions / TBD

- BME280 calibration plan — side-by-side comparison with reference for first 1–2 days of deployment.
- Stevenson screen mounting hardware — pole, wall mount, dedicated stand?
- Cable run from rain gauge to Stevenson screen — length, routing, connector choice.
- Rain gauge mounting location and height (meteorological standard is 30 cm rim height above ground, away from obstructions).
- Whether to add a status LED or rely on HA's device-status entity for liveness.
- Final BoM with sourcing for any remaining parts.
