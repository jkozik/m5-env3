# m5-env3 — ESPHome indoor environment display

ESPHome firmware config for an M5StickC PLUS2 + ENV III unit (temp/humidity/pressure). It shows readings on the built-in screen, publishes to MQTT for weewx, and exposes HTTP endpoints for Zabbix.

**Start every session by reading `README.md` (how to build, flash, verify) and `NOTES.md` (history, decisions, open items, future plans).**

## Rules

- Use the project venv: `.venv\Scripts\esphome ...` (Windows). If you set up or repair the environment, install only from `requirements.txt` (frozen: ESPHome 2026.6.5, Python 3.11.9). Never run a bare `pip install esphome` or upgrade ESPHome unless the user asks; schema changes between versions have broken this config before.
- Before any flash: run `esphome config m5-env3.yaml`, then `compile`. Flashing is `esphome run m5-env3.yaml --device m5-env3.local` (OTA). Ask before flashing the device.
- **Never reuse GPIO4.** It holds the board's power on (the `power_hold` switch). Watch for display or other presets that claim it implicitly.
- Don't change the MQTT topic `weewxrtlsdr/indoor/state` or its JSON keys; weewx depends on them.
- Before risky config changes, save a dated `.bak` snapshot (or commit, once a git repo exists).
- `secrets.yaml` holds WiFi credentials: never print its contents, commit it, or publish it.
- After each work session, add a dated section to `NOTES.md` covering what changed and why. Update `README.md` if build, verify, or monitoring steps change.
