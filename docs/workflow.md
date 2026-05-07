# Workflow

This document describes how to flash the device, install OTA updates, edit the YAML, set up Home Assistant helpers, and run the Stage 0 verification gate.

## 1. First-time flash (USB, via web.esphome.io)

The HA ESPHome add-on cannot reach USB on the development PC. The first flash uses Chrome's WebSerial API at https://web.esphome.io/.

1. In the HA ESPHome add-on dashboard, paste `weather_station.yaml` from this repo into a new device config (or upload `secrets.yaml` to the add-on's `/config/esphome/` first via Samba/SSH so `!secret` references resolve).
2. Click **Install → Manual download → Modern format**. The dashboard compiles and offers a `.bin` download.
3. Save the `.bin` somewhere accessible.
4. On the development PC, plug the FireBeetle C6 into USB-C **while holding the BOOT button (`GPIO9`)**. This forces ESP32-C6 download mode in case the chip is in an unknown state.
5. Open https://web.esphome.io/ in Chrome. Click **Connect**. Pick the COM port the C6 enumerated as.
6. Click **Install** and select the downloaded `.bin`. Wait for the upload to finish.
7. Press the RESET button (or unplug/replug USB) to boot the new firmware.

## 2. Subsequent flashes (OTA)

Once the device is on Wi-Fi and visible in HA, you no longer need USB:

1. Edit `weather_station.yaml` in this repo.
2. Copy/paste the new contents into the HA ESPHome dashboard editor.
3. Click **Install → Wirelessly**. The dashboard pushes the build over the network using the OTA password from `secrets.yaml`.

## 3. Editing the YAML

The repo at `weather_station_esphome/` is the source of truth. Workflow:

1. Edit `weather_station.yaml` here.
2. Commit changes to git first — the repo is the durable record.
3. Copy/paste into the dashboard when ready to install.

Treat the dashboard like a build/deploy target, not an editor.

## 4. Creating the HA helper toggle

Stage 0 doesn't consume this toggle yet, but creating it now means it's ready when Stage 3 (deep sleep) wires it up.

1. Open Home Assistant → **Settings → Devices & Services → Helpers**.
2. Click **Create Helper → Toggle**.
3. Name: `Weather Station Deep Sleep` (entity ID will become `input_boolean.weather_station_deep_sleep`).
4. Default state: ON.
5. Save.

When wired up in Stage 3: ON = device sleeps normally between wakes; OFF = device stays awake (used for OTA install windows and bench iteration).

## 5. Stage 0 verification gate

Stage 0 is complete when ALL of these are true:

- [ ] `weather_station.yaml` compiles in the HA dashboard with no errors or warnings.
- [ ] Device flashes successfully via web.esphome.io and reboots.
- [ ] Device joins Wi-Fi and appears in HA's ESPHome integration (auto-discovered).
- [ ] `Wi-Fi Signal` and `Uptime` entities show live values in HA.
- [ ] OTA round-trip works: change `friendly_name`, install wirelessly from the dashboard, observe the new name in HA.
- [ ] Onboard LED on `GPIO15` behaves as a status indicator. If it's solid-on when it should be off, flip the `inverted` field in `weather_station.yaml` and re-flash.
- [ ] `weather_station_deep_sleep` HA helper toggle exists (unused for now).
