# office_panel

Battery-powered, wall-mounted color e-paper panel that shows the live booking status of the home office. Mounted in the hallway outside the office door.

Hardware: Seeed Studio reTerminal E1002 (7.3" Spectra 6 ACeP, ESP32-S3R8).
Backend: Home Assistant Local Calendar + ESPHome firmware.

The panel answers one question, visible from across the hallway: **can I knock on this door right now?**

## Status

In development. See [`SPEC.md`](SPEC.md) for the full specification and [`CLAUDE.md`](CLAUDE.md) for contributor / Claude Code operating instructions.

Current phase: **Phase 1 — basic room status** (see SPEC §12).

## Layout

```
esphome/                  Firmware config and fonts
homeassistant/            HA package, dashboards, voice intents
docs/                     Install, flashing, development notes
```

## Related repos

The hardware-level ESPHome package for the reTerminal E1002 lives in
[`heytcass/esphome-device-library`](https://github.com/heytcass/esphome-device-library)
under `devices/seeed/reterminal_e1002/` and is imported by the firmware in
this repo via ESPHome's `packages:` syntax.

## Setup

Setup instructions land here once Phase 1 is implemented. For now, see
[`docs/flashing.md`](docs/flashing.md) and [`docs/development.md`](docs/development.md).

## License

MIT — see [`LICENSE`](LICENSE).
