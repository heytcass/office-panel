# Development

How to iterate on the panel without bricking it or burning through the battery.

## Repo layout

See `SPEC.md` §11. Firmware in `esphome/`, HA config in `homeassistant/`, this directory for human-facing notes.

## Cross-repo dependency

Hardware-level ESPHome config (pins, display model, sensors, buttons, buzzer) lives in `heytcass/esphome-device-library` at `devices/seeed/reterminal_e1002/`. The firmware in this repo imports it via ESPHome's `packages:` syntax. Changes to the hardware package and to firmware that depends on it may need to land together.

Local checkout path: `/home/tom/Projects/esphome-device-library`.

## Iteration loop

- Edit `esphome/office_panel.yaml` (or HA config under `homeassistant/`).
- Flash via the HA ESPHome app (initial) or OTA via the same app (subsequent).
- View serial / OTA logs in the HA ESPHome app.
- For HA-side changes, use Developer Tools to fire events, set states, and inspect template sensors before triggering on the panel.

There is no simulator. All firmware iteration is physical.

## Testing

See SPEC §12 "Definition of done" per phase. No automated tests — this is a home-scale project.

End-to-end smoke test: create a short calendar event via `script.book_office`, confirm the panel renders the BOOKED mode within one wake cycle, then end via `script.end_office_booking`.
