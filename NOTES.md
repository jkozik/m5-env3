# Build Notes — m5-env3

A running record of how `m5-env3.yaml` got to its current state. Not a line-by-line diff — see backup files for exact snapshots — just the path taken and why.

## Starting point

Came in with a minimal working config: M5StickC PLUS2 + ENV III unit (SHT30 + QMP6988) reading indoor temperature/humidity/pressure over I2C, publishing to MQTT (`weewxrtlsdr/indoor/state`) every 30s for a weewx instance to pick up. No display usage, no OTA visibility, no HTTP interface — just sensor plumbing.

## Round 1 — visibility features

Goal: use the StickC Plus2's screen and add basic operational visibility, without risking the board's modest (but plenty sufficient) RAM/flash budget.

1. Added `text_sensor: platform: version` — exposes the running ESPHome build version (answers "how do I know what firmware is on here").
2. Added `web_server: port 80, version: 3` — ESPHome's built-in lightweight REST API + dashboard, no custom code needed.
3. Added the display block for the ST7789V screen: heartbeat dot, WiFi status dot, and temp/humidity/pressure text.
   - First attempt used `platform: st7789v` with a borrowed `model: TTGO TDisplay 135x240` preset (found via a generic ESPHome device page). This **failed validation** — that model preset silently claims GPIO4 internally, which collided with the existing `power_hold` switch (GPIO4 must stay high or the Plus2 powers itself off on battery).
   - Found a config written specifically for this exact board (community repo, see README credits) using the newer `platform: mipi_spi` with `model: st7789v`, explicit `dimensions:` (height/width/offset_height/offset_width), and `invert_colors: true`. Switched to that — validated clean, no pin conflicts.
   - Also corrected backlight output polarity (`inverted: false`, not `true`) to match that same proven reference.
4. Validated with `esphome config` (schema/pin checks) **and** `esphome compile` (catches lambda C++ errors) before ever touching the hardware — caught the GPIO4 conflict this way without needing a physical flash attempt.
5. First USB flash: booted clean, WiFi/MQTT connected, sensors reading real values — but the **heartbeat dot never appeared**.
6. Diagnosed: the heartbeat lambda derived its blink state from `(millis() / 500) % 2`, but the display only redraws once per second (`update_interval: 1s`). Since 1000ms is an exact multiple of the 500ms toggle period, every redraw sampled the same phase — a classic aliasing bug, not a hardware issue. Fixed by using a `static bool` that flips on every redraw instead of deriving state from wall-clock time.
7. Added a left text margin (x: 0 → x: 10) per feedback that the numbers were flush to the screen edge.
8. Reflashed — heartbeat confirmed blinking, WiFi dot confirmed green, text readable with margin.

## Round 2 — WiFi icon + real health metrics

1. Replaced the plain WiFi status dot with an actual WiFi glyph, using ESPHome's `image:` component pulling a Material Design Icon (`mdi:wifi`) rendered as a `BINARY` bitmap, tinted red/green at draw time via the same `it.image(x, y, id, Color(...))` call used for the dot.
   - Hit a schema surprise: the installed ESPHome version (2026.6.5) rejects `platform: file` under `image:` even though the published docs still show it — removed the `platform` key to match what this version's compiler actually accepts.
2. Added diagnostic sensors for external monitoring (Zabbix was the stated use case): `uptime`, `wifi_signal`, and the built-in `debug` component's `free` heap sensor. Each is automatically exposed as its own REST endpoint via `web_server` (e.g. `GET /sensor/uptime` → `{"id":..., "state":..., "value": <number>}`), so no custom endpoint code was needed — just point Zabbix HTTP-agent Items at the per-sensor URLs with `$.value` JSONPath preprocessing.
3. Compiled (RAM ~15.5%, flash ~24.3% used — comfortable headroom), reflashed via USB, confirmed on-device: WiFi icon renders and turns green, new sensors appear on the dashboard with sane values.

## Safety net along the way

- `m5-env3.pre-display-changes.yaml.bak` — exact content before any of this session's changes.
- `m5-env3.WORKING-confirmed-2026-09-27.yaml.bak` — first confirmed-working display build (heartbeat bug still present at that snapshot).
- No git repository yet — deliberately deferred until the config has sat stable for a few days. Revisit once confidence is high.

## Known open items

- `logger:` is left at the ESPHome default (`DEBUG`) — chatty but harmless; can be dropped to `INFO`/`WARN` later purely to quiet the log stream, no functional effect.
- Long-term plan: replicate the outdoor half of this weather setup (Davis Vantage Pro2, received via an RTL-SDR Blog V3 dongle and the `rtldavis`/`weewx-rtldavis` toolchain) on the Dell Ubuntu server, as a second independent weewx instance alongside the existing Envoy-based one. That work is tracked separately from this ESPHome project.

## Resource check (2026-09-28, ~28.6h uptime on bookshelf)

Live numbers pulled from the device's own `/sensor/*` endpoints, not just build-time estimates:

- **Free heap: 198 KB** — healthy, no sign of a leak or pressure.
- **Uptime: 28.6 hours** with no resets since the last flash — stable.
- **WiFi signal: -46 dBm** — excellent at the bookshelf location.
- **Flash/RAM (build-time): 24.3% / 15.5%** — plenty of headroom.
- Nothing close to maxing out on any axis. No `loop_time`/CPU-percentage sensor is currently exposed (ESPHome doesn't have a simple CPU-utilization metric the way it does heap) — could add the `debug` component's `loop_time` sub-sensor later if a real quantified CPU baseline is ever needed. Not done now since the device is going to be left alone once stable.

## Documentation session (2026-10-14) — MQTT verification, diagram, build sequence

No firmware/config changes this session — documentation only.

1. **Verified MQTT end-to-end with MQTT Explorer** (connect to `192.168.100.174:1883`, no credentials, subscribe `#`). Confirmed working. What to look for:
   - `m5-env3/status` = `online` — retained birth message; flips to `offline` via the broker's last-will if the device drops off. Quickest health check.
   - `weewxrtlsdr/indoor/state` — the custom JSON payload weewx consumes, updating every 30s.
   - `m5-env3/sensor/<name>/state` etc. — ESPHome still publishes per-entity state topics even with `discovery: false` (discovery only controls the Home Assistant `homeassistant/...` config topics).
   - `m5-env3/debug` — the device's log stream over MQTT (no USB/`esphome logs` needed).
   - Inbound test: publishing `{"state":"OFF"}` to `m5-env3/light/display_backlight/command` turns the backlight off — proves broker → device works too.
   - Gotcha: the first JSON message after boot may contain null/NaN — the publish interval and sensor reads both run at 30s but aren't synchronized.
2. **Added a Mermaid diagram to the README** ("Data flow" section) showing build host (Windows PC: YAML source, ESPHome CLI in `.venv`, MQTT Explorer) → device (USB first flash, OTA after) → MQTT broker → weewx, plus HTTP → Zabbix/browser. Chose Mermaid over a screenshot because it's plain text: lives in the README, diffs cleanly, and is easy to update. Renders on GitHub automatically; in VS Code needs the *Markdown Preview Mermaid Support* extension. Kept the old ASCII diagram underneath as a fallback. If the `@{ shape: docs }` line errors on an older renderer, swap it for `SRC[/"..."/]`.
3. **Documented the everyday build sequence** in the README: `esphome config` (optional fast check) then `esphome run --device m5-env3.local` (compile → OTA upload → logs in one command). The longer `compile`/`upload`/`logs` steps stay documented for the first USB flash.
4. **Froze the build environment** in `requirements.txt` (full `pip freeze`: ESPHome 2026.6.5 + all its dependencies, Python 3.11.9). Pinning ESPHome also effectively pins the ESP-IDF/PlatformIO toolchain, since each ESPHome release pins its own platform versions. README install step now uses `pip install -r requirements.txt`. Also added `/.venv/` to `.gitignore`. Remaining unpinned inputs: the Roboto font (`gfonts://`) and `mdi:wifi` icon are downloaded at build time (cached in `.esphome/`) — could be vendored into the repo as local files if a fully offline build is ever needed.
5. **Decided on repo/folder name `m5-env3`** (one repo per device, matches device name and MQTT prefix) instead of the generic `esphome`. Renaming the folder requires recreating `.venv` (Windows venv launchers hard-code absolute paths) and deleting `.esphome/build`; the device itself is unaffected.

## Folder move (2026-10-15) — `esphome/` → `m5-env3/`

No firmware/config changes.

1. Copied the project with `robocopy esphome m5-env3 /E /XD .venv build` (skipping the venv and build dir, as planned on 10-14).
2. Recreated `.venv` with `python -m venv .venv` (the `py` launcher isn't installed on this PC; plain `python` is 3.11.9, which matches the freeze) and `pip install -r requirements.txt`. `pip check` is clean and `esphome version` reports 2026.6.5.
3. `esphome compile m5-env3.yaml` succeeded. ESPHome saw the path change ("Core config or version changed"), cleaned the build files, and did a full rebuild that took about 12 min. Usage is RAM 15.5% / flash 24.3%, the same as before, so the build is the same. No stale references to the old path are left in `.esphome/storage` or `idedata`.
4. Not flashed. The device is still running the build from 2026-09-27, which is functionally identical.
5. The old `projects\esphome\` folder still exists. It's safe to delete once this folder is confirmed as the working copy.
6. Added a "Coming back later" refresher to the README. It covers the one-line health check (`esphome version`), the venv's base Python (Microsoft Store Python 3.11), and how to rebuild the venv. The PowerShell execution policy is `RemoteSigned`, so `Activate.ps1` works.
7. Changed the README to the activate-first style throughout: run `.venv\Scripts\Activate.ps1` (the equivalent of Linux's `source .venv/bin/activate`), then use plain `esphome`/`pip`. That matches how you use Python on Linux. CLAUDE.md still tells Claude to use the `.venv\Scripts\esphome` prefix, because Claude's shells don't keep activation between commands.
8. Removed the Data flow Mermaid diagram (added 10-14) because it doesn't render in the viewer used for this README. Its details (broker IP, MQTT Explorer, USB vs OTA flashing) were merged into two plain-text diagrams. The README is now Mermaid-free; plain-text diagrams are preferred.
9. Added an "Overview of the dev workflow: YAML → C++ → device" section to the README, before Installation. It contains the SHT30 YAML block, the matching excerpt from the generated `main.cpp`, a plain-text block diagram (the Mermaid version didn't render, so it was dropped), and links to the ESPHome Getting Started guide and the developer Architectural Overview.
10. Added a "Quick sanity checks after a flash" table under the everyday build section. It lists the four places to look first (screen, logs, MQTT Explorer, browser) and four rules of thumb. It's deliberately short; it isn't meant to be a full troubleshooting guide.
11. Reviewed secrets and `.gitignore`. All credentials go through `!secret`; there are no literal passwords in the YAML, `.bak` files or `old/`. `.gitignore` covers `secrets.yaml`, `.esphome/` (which holds firmware with the WiFi password) and `.venv/`. No `.env` is needed, because `secrets.yaml` is ESPHome's equivalent. Added `secrets.yaml.example` as the committed template, and README install step 2 now copies it. Known LAN-trust choices, left unchanged: OTA has no password, the web server has no auth, and MQTT has no credentials. An OTA password in `secrets.yaml` would be the easy first hardening step.

## Possible future enhancements

**Adding more sensors to *this* board (I2C hub route — chosen approach, not yet built):**
I2C is a shared bus, so multiple Grove/I2C sensors can technically coexist on one board as long as addresses don't collide (current bus already has SHT30 @ 0x44 and QMP6988 @ 0x70). The StickC Plus2 only has one physical Grove port, though, so a third+ sensor needs either an **I2C Hub unit** (splits one Grove port into several) or a pass-through-capable sensor unit. Decision: try the I2C hub approach when/if a new indoor sensor is wanted; if wiring turns into a rat's nest, fall back to just buying a second M5StickC Plus2 as an independent node instead (simpler, more robust, costs about the same as a hub + cabling anyway).

**Adding sensors in a *new physical location* (e.g. the outdoor UV/solar unit below):**
This is NOT a daisy-chain/hub situation — a sensor in a different location needs its own independent ESPHome device (own ESP32, own WiFi/MQTT identity), same pattern as this device itself. Multiple independent nodes all publishing to the same MQTT broker is the normal way to scale this, with no hard limit from the existing setup.

**Outdoor UV/Solar sensor unit (planned, not yet built):**
Motivation: the Davis Vantage Vue has no UV/solar sensor, and Davis's official add-on path requires a WeatherLink Live + standalone transmitter + UV sensor + solar radiation sensor (~$1,480 total, SKUs 6100/6332/6490/6450) since the Vue console itself can't pair with those standalone sensors directly. Chose the cheaper DIY route instead (~$50-90), reusing this project's proven ESPHome+MQTT pattern:
- **Sensor**: LTR390 I2C UV + ambient-light breakout (confirmed native ESPHome support: `sensor: platform: ltr390`) — gives UV index and a lux reading usable as an approximate (not calibrated) solar-radiation proxy.
- **Power**: solar panel (5-6W, plain 5V USB output, IP65) **plus a separate** USB battery pack rated for pass-through/UPS-style charging (5,000-10,000mAh) — important distinction: most "solar panel for security camera" products are panel-only and expect the camera to supply its own battery, which the StickC's tiny ~200mAh internal battery can't adequately do on its own overnight.
- **Enclosure**: small IP65 project box, cable gland for the one USB power cable, open cutout or thin acrylic window over the UV sensor (needs sky exposure).
- **Firmware changes for this build**: turn the display backlight off by default (biggest power saving; nobody reads a screen mounted outside), otherwise same MQTT/`web_server` pattern as this indoor unit. Deliberately skipping ESP32 deep-sleep for the first version — this board's GPIO4 power-hold requirement makes deep sleep interaction unverified/riskier; revisit only if real-world battery margin proves too tight running always-on.
