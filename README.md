# m5-env3 — Indoor Environment Display

An ESPHome-based indoor temperature/humidity/barometric-pressure display built on an M5Stack M5StickC PLUS2 with an ENV III sensor unit. It shows live readings on its own screen, publishes to MQTT for ingestion into weewx, and exposes a small HTTP API for external health monitoring (e.g. Zabbix).

This device is the *indoor* half of a home weather setup — the *outdoor* half is a Davis Vantage Pro2 received over RF via an RTL-SDR Blog V3 dongle on a separate Ubuntu server, feeding its own weewx instance. That side is a separate project; this README covers only the M5StickC Plus2 display.

This is a Dev Workstation setup.  The esphome tool is running in this project folder.  The key file is `m5-env3.yaml`.  

<img width="1628" height="902" alt="image" src="https://github.com/user-attachments/assets/91e1303c-9e31-4877-a028-52b7677c3e44" />


## What it does

- Reads indoor temperature, humidity, and barometric pressure from the ENV III unit every 30s.
- Displays them on the built-in screen, along with:
  - A **heartbeat indicator** (blinking dot) confirming the firmware loop is alive, not hung.
  - A **WiFi status icon**, green when connected, red when not.
- Publishes readings to MQTT (`weewxrtlsdr/indoor/state`) for weewx to pick up.
- Supports **OTA firmware updates** (no USB cable needed after the first flash).
- Exposes a lightweight **HTTP interface** (ESPHome's built-in `web_server`) with a small dashboard and per-sensor JSON endpoints suitable for monitoring tools — see [Monitoring / health endpoints](#monitoring--health-endpoints) below.

## Hardware

| Part | Notes |
|---|---|
| [M5Stack M5StickC PLUS2](https://shop.m5stack.com/products/m5stickc-plus2-esp32-mini-iot-development-kit) | ESP32-PICO-V3-02, 8MB flash, 1.14" 135×240 ST7789V2 display |
| [M5Stack ENV III Unit](https://shop.m5stack.com/products/env-iii-unit-with-temperature-humidity-air-pressure-sensor-sht30-qmp6988) | SHT30 (temp/humidity) + QMP6988 (pressure), connects via the Plus2's Grove/I2C port |

Plug the ENV III into the StickC Plus2's Type-C Grove port — no wiring beyond that.

## Software / credits

- **[ESPHome](https://esphome.io/)** — the firmware framework this whole config is written for. (Version in use during development: 2026.6.5; note that the `image:` component's exact schema shifted between that version and the published docs — see [Notes](NOTES.md) if you hit schema errors on a different ESPHome version.)
- Display pin mapping and the working `mipi_spi`/`st7789v` configuration for this exact board were sourced from **[WarC0zes/m5stickc-plus2-esphome](https://github.com/WarC0zes/m5stickc-plus2-esphome)** — credit to that project for the board-specific config that a generic device-database entry got wrong (see [Notes](NOTES.md) for what went wrong and why).
- Font rendered via ESPHome's built-in Google Fonts integration (`gfonts://Roboto`).
- WiFi icon rendered via ESPHome's built-in [Material Design Icons](https://pictogrammers.com/library/mdi/) integration (`mdi:wifi`).

## Overview of the dev workflow: YAML → C++ → device

If you're coming from a Raspberry Pi, this is the surprising part. An ESP32 doesn't run Linux, and you don't copy a program onto it. The whole device runs a single firmware image, which is a small C++ program compiled for the chip. With ESPHome, **you don't write that C++ yourself**. You describe the device in `m5-env3.yaml`: which sensors are on which pins, how often to read them, and where to send the data. ESPHome then generates the C++ for you, compiles it, and flashes it.

Here is the ENV III's SHT30 temperature/humidity sensor as you write it in `m5-env3.yaml`:

```yaml
sensor:
  - platform: sht3xd
    address: 0x44
    temperature:
      name: "Indoor Temperature"
      id: indoor_temp
    humidity:
      name: "Indoor Humidity"
      id: indoor_hum
    update_interval: 30s
```

`esphome compile` turns that into C++ like this, an excerpt from the generated `.esphome/build/m5-env3/src/main.cpp`:

```cpp
new(sht3xd_sht3xdcomponent_id) sht3xd::SHT3XDComponent();
sht3xd_sht3xdcomponent_id->set_update_interval(30000);
sht3xd_sht3xdcomponent_id->set_i2c_address(0x44);
new(indoor_temp) sensor::Sensor();
sht3xd_sht3xdcomponent_id->set_temperature_sensor(indoor_temp);
```

ESPHome never writes the actual sensor driver. That driver is ready-made C++ that ships inside the ESPHome package (`.venv\Lib\site-packages\esphome\components\sht3xd\sht3xd.cpp`). The generated `main.cpp` only creates and wires up these pre-written components. Each `platform:` in the YAML corresponds to one of these component folders.

```
m5-env3.yaml                       (you edit)
   │  esphome config               validate schema + pins
   ▼
main.cpp                           (generated)  + ESPHome component library (.cpp)
   │  esphome compile              PlatformIO / ESP-IDF C++ compiler
   ▼
firmware.bin
   │  esphome upload               USB the first time, OTA over WiFi after
   ▼
M5StickC PLUS2 (ESP32)
```

Things that follow from this:
- **Only edit the YAML.** `main.cpp` is regenerated on every compile, so hand edits disappear. For custom logic, ESPHome lets you put short C++ snippets inside the YAML as `lambda:` blocks. This project's display drawing and MQTT JSON payload work that way.
- **The ESPHome version matters.** The YAML schema and the generated C++ both come from the installed ESPHome release. That's why `requirements.txt` pins it.
- **There's no OS on the device.** Every change means a full rebuild and reflash. `esphome run` does compile, upload and logs in one step.

To learn more, start with the official [Getting Started with ESPHome](https://esphome.io/guides/getting_started_command_line/) guide. The [ESPHome Architectural Overview](https://developers.esphome.io/architecture/overview/) explains how YAML becomes C++ under the hood.

## Installation (from scratch)

1. **Install ESPHome** in a Python virtual environment, using the pinned versions in `requirements.txt` (ESPHome 2026.6.5, Python 3.11.9 — the exact environment this config was built and verified with):
   ```
   python -m venv .venv
   .venv\Scripts\Activate.ps1
   pip install -r requirements.txt
   ```
   `Activate.ps1` is the PowerShell version of Linux's `source .venv/bin/activate`. Your prompt shows `(.venv)` while it's active. All commands below assume the venv is active.

   Don't use a bare `pip install esphome` — that grabs the latest version, which may reject this config (ESPHome changes its YAML schema between releases; see [Notes](NOTES.md)). Upgrade deliberately instead: install the new version, re-run `config`/`compile`, and if it all works, `pip freeze > requirements.txt` to record the new known-good set.
2. **Create `secrets.yaml`** from the template, then edit it with your WiFi name and password:
   ```
   Copy-Item secrets.yaml.example secrets.yaml
   ```
   `secrets.yaml` is gitignored and must never be committed. `secrets.yaml.example` is committed and holds placeholder values only. ESPHome reads only `secrets.yaml`, which is where every `!secret` in `m5-env3.yaml` looks up its value. If you add a new secret, add its key to both files.
3. **Check/update the MQTT broker address** in `m5-env3.yaml` (`mqtt: broker:`) to match your broker's IP.
4. **Validate the config** before touching hardware:
   ```
   esphome config m5-env3.yaml
   ```
   This catches pin conflicts and schema errors without needing the device connected.
5. **Compile** (also catches C++ errors inside display `lambda:` blocks that config validation can't see):
   ```
   esphome compile m5-env3.yaml
   ```
6. **First flash — over USB serial** (required the first time; OTA needs existing ESPHome firmware already on the device):
   - Plug the M5StickC Plus2 into a USB port. It should enumerate as a `CH9102` USB-serial device.
   - Find its COM port (Windows: Device Manager, or `Get-PnpDevice -Class Ports` in PowerShell).
   - Flash:
     ```
     esphome upload m5-env3.yaml --device COM<N>
     ```
   - Watch the boot log to confirm a clean start:
     ```
     esphome logs m5-env3.yaml --device COM<N>
     ```
7. **All subsequent updates can go over OTA** (device must already be on WiFi with this firmware):
   ```
   esphome upload m5-env3.yaml
   ```
   (omit `--device` — ESPHome will find it on the network via mDNS)

### Everyday build sequence (shortcut)

Once the device has been flashed the first time, a normal edit → deploy cycle is just:

```
.venv\Scripts\Activate.ps1                            # once per PowerShell window; prompt shows (.venv)
esphome config m5-env3.yaml                           # optional: fast YAML/pin check, no C++ build
esphome run m5-env3.yaml --device m5-env3.local       # compile → upload (OTA) → stream logs
```

- `run` = `compile` + `upload` + `logs` in one command. `compile`/`run` validate the config too, so `config` is only for catching YAML mistakes quickly.
- `--device` skips the "where to upload?" prompt. Use `m5-env3.local` (or the device IP) for OTA, or `COM<N>` for USB.
- Logs keep streaming after the upload; `Ctrl+C` stops them (the device keeps running). Add `--no-logs` to just flash.
- `deactivate` leaves the venv. If you forget to activate, `esphome` gives "not recognized". If ESPHome is ever installed globally, it instead runs the wrong version without warning, so check for `(.venv)` in the prompt.

### Quick sanity checks after a flash

Not a full troubleshooting guide. These are the places to look first, roughly in order:

| Where | What "healthy" looks like |
|---|---|
| **The screen** | The heartbeat dot blinks, which means the firmware loop is running and not hung. The WiFi icon is green, and readings are filled in. |
| **The logs** (`esphome logs m5-env3.yaml`, or the `m5-env3/debug` topic in MQTT Explorer) | Boot finishes, WiFi and MQTT connect, and sensor readings appear every 30s. Errors and warnings show in red and yellow. |
| **MQTT Explorer** (broker `192.168.100.174:1883`, subscribe `#`) | `m5-env3/status` is `online`, and `weewxrtlsdr/indoor/state` updates every 30s. This confirms what weewx will actually receive. |
| **Browser** (`http://m5-env3.local/`) | The dashboard loads with live values. This proves the device is on the network even if MQTT isn't working. |

Rules of thumb:
- **The build fails?** `esphome config` explains YAML and pin mistakes in plain words. A C++ error during `compile` almost always comes from a `lambda:` block.
- **Can't reach `m5-env3.local`?** mDNS can be flaky on Windows. Get the device's IP from your router and use it in place of `m5-env3.local`.
- **Device unreachable, or rebooting over and over?** Plug in USB and run `esphome logs m5-env3.yaml --device COM<N>`. The serial log works even when WiFi doesn't. ESPHome's built-in safe mode also keeps OTA available after repeated boot failures, so you can reflash a fixed build.
- **A reading is NaN, or the first JSON payload is null?** Wait one 30s cycle. The first publish after boot can come before the first sensor read.

### Coming back later (environment refresher)

`.venv` stays in this folder, so you don't need to set anything up again. Open PowerShell in this folder, activate the venv, and check it still works:

```
.venv\Scripts\Activate.ps1     # PowerShell equivalent of: source .venv/bin/activate
esphome version                # should print 2026.6.5
```

If it prints 2026.6.5, continue with the everyday build sequence above. The PowerShell execution policy is already `RemoteSigned`, so `Activate.ps1` runs without a "scripts disabled" error.

If it errors (for example "No Python at ..." or "not recognized"), the venv is broken. That happens when the folder was moved or renamed, or when the Python it was built from was removed or replaced. This venv is built on the **Microsoft Store Python 3.11**. To rebuild:

```
deactivate                             # if the prompt shows (.venv)
python --version                       # must be 3.11.x; if not, install "Python 3.11" from the Microsoft Store
Remove-Item -Recurse -Force .venv
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r requirements.txt
esphome version
```

The first compile after a rebuild or move does a full clean build, which takes about 12 minutes. Ignore pip's "new release of pip" and PlatformIO's "upgrade" notices.

## Why GPIO4 matters

The M5StickC Plus2 powers itself off on battery unless GPIO4 is held high — this is handled by the `power_hold` switch in the config (`restore_mode: ALWAYS_ON`). **Don't reuse GPIO4 for anything else** — an earlier draft of the display config did this by accident via a borrowed display "model" preset that claimed GPIO4 internally, and it failed config validation as a result. See [Notes](NOTES.md) for the full story.

## Monitoring / health endpoints

The device exposes a small dashboard and REST API at `http://<device-ip>/` (or `http://m5-env3.local/` via mDNS) once on WiFi. Each entity has its own JSON endpoint, e.g.:

**Addressing note for Zabbix specifically:** point Zabbix HTTP-agent Items at a **DHCP reservation or static IP** for this device, not a raw IP you copied once — the IP will change on lease renewal and silently break polling otherwise. The `m5-env3.local` mDNS name is more convenient but only works if the Zabbix server itself can resolve mDNS (many Linux Zabbix installs need `avahi-daemon`/`libnss-mdns` for this, and it's not always installed by default) — worth testing (`ping m5-env3.local` from the Zabbix server) before relying on it, otherwise set a DHCP reservation on your router for this device's MAC address.

```
GET /sensor/indoor_temperature
→ {"id": "sensor-indoor_temperature", "state": "22.4 °C", "value": 22.4}
```

Useful endpoints for external monitoring (e.g. a Zabbix HTTP-agent Item with `$.value` JSONPath preprocessing):

| Endpoint | Metric | Suggested use |
|---|---|---|
| `/sensor/uptime` | Seconds since boot | Trigger on a drop vs. previous poll → device rebooted |
| `/sensor/wifi_signal` | WiFi RSSI (dBm) | Trigger on weak signal (e.g. below -80) |
| `/sensor/heap_free` | Free heap (bytes) | Trigger on sustained downward trend → possible memory leak |
| `/text_sensor/esphome_version` | Running firmware version | Confirm which build is deployed (read the `state` field) |
| `/sensor/indoor_temperature`, `/sensor/indoor_humidity`, `/sensor/indoor_pressure` | Live readings | Same data as MQTT, available without a broker |

## Data flow

Build and deploy (Windows PC → device):

```
m5-env3.yaml + secrets.yaml ──esphome compile──▶ firmware ──first flash: USB serial──▶ m5-env3
                                                          ──later: OTA over WiFi─────▶ (M5StickC PLUS2 + ENV III)
```

Runtime data (device → consumers):

```
ENV III (I2C) → m5-env3 (ESPHome) → MQTT broker 192.168.100.174:1883 → weewx           (weewxrtlsdr/indoor/state)
                                                                     → MQTT Explorer   (subscribe #, sees m5-env3/# too)
                                  → HTTP :80 /sensor/*               → Zabbix / browser
                                  → on-device display
```

To verify the MQTT side live, connect MQTT Explorer to the broker (no username/password) and check that `m5-env3/status` is `online` and `weewxrtlsdr/indoor/state` updates every 30s.

## Repo hygiene

- `secrets.yaml` and `.esphome/` (build cache) are gitignored — don't commit either. The compiled firmware in `.esphome/` contains the WiFi password too. `secrets.yaml.example` is the committed template.
- `.venv/` (the Python virtual environment) should also stay out of git if you initialize a repo here.
- `*.bak` files in this folder are point-in-time safety snapshots taken during development — see [Notes](NOTES.md) for what each one is. Safe to delete once you've committed a stable version to git.
