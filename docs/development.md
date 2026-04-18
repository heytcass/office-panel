# Development

How to iterate on the panel without bricking it or burning through the battery.

## Repo layout

See `SPEC.md` §11. Firmware in `esphome/`, HA config in `homeassistant/`, this directory for human-facing notes.

## Cross-repo dependency

Hardware-level ESPHome config (pins, display model, sensors, buttons, buzzer) lives in `heytcass/esphome-device-library` at `devices/seeed/reterminal-e1002.yaml`. The firmware in this repo imports it via ESPHome's `packages:` syntax. Changes to the hardware package and to firmware that depends on it may need to land together.

Local checkout path: `/home/tom/Projects/esphome-device-library`.

Launch Claude Code sessions from `/home/tom/Projects/` (the parent dir) so both repos share one context tree. See `CLAUDE.md` for the working-across-repos rules.

## HA setup — one-time

Before the panel does anything useful, the HA side needs:

1. **Local Calendar integration.** Settings → Devices & Services → Add
   Integration → Local Calendar. Name it exactly `Office` so the entity
   resolves to `calendar.office`.
2. **Package include.** In `configuration.yaml` at the top level:
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
   Then symlink or copy the repo's package into HA's config dir:
   ```
   ln -s <repo>/homeassistant/packages/office_panel.yaml \
         <ha_config>/packages/office_panel.yaml
   ```
3. **Dashboard registration.** See the header of
   `homeassistant/dashboards/office.yaml` for the `lovelace:` config
   snippet to add to `configuration.yaml`, plus the symlink for the
   dashboard file itself.
4. **Restart / reload.** Developer Tools → YAML → Reload "Template
   Entities" and "Scripts", or restart HA. After reload, the
   `sensor.office_mode` entity should appear and show `AVAILABLE`
   (assuming no booking, weekday, daytime).

## Iteration loop

- **Firmware change.** Edit `esphome/office_panel.yaml`. Flash via the
  HA ESPHome add-on (first time over USB, subsequent via OTA). Watch
  the device log in the add-on UI — there is no simulator.
- **HA change.** Edit `homeassistant/packages/office_panel.yaml`.
  Reload the relevant integration via Developer Tools or restart HA.
  Use the Template editor to sanity-check template sensor states before
  checking behavior on the panel.
- **Dashboard change.** Edit `homeassistant/dashboards/office.yaml`.
  Ctrl+F5 in a browser tab on the dashboard to reload.

## Testing

See SPEC §12 "Definition of done" per phase. No automated tests — this is a home-scale project.

**End-to-end smoke test for Phase 1:**

1. Create a 15-minute test booking via the dashboard's calendar card
   starting ~2 minutes from now.
2. Watch `sensor.office_mode` in Developer Tools → States. It should
   flip `AVAILABLE → STARTING_SOON → BOOKED` at the right moments.
3. Wait for the next wake cycle (up to 15 minutes during weekday active
   hours). The panel should render the current mode within one cycle.
4. Delete the test event via the calendar card. Mode returns to
   `AVAILABLE` within a wake cycle.

Manual scripts are available in Developer Tools → Services:

- `script.book_office` (duration_minutes, owner, source)
- `script.book_office_until_next` (owner, source)
- `script.end_office_booking`
- `script.extend_office_booking` (duration_minutes)
- `script.book_office_custom` (owner) — uses
  `input_number.office_booking_custom_minutes`

## Debugging

- **Stale-looking display.** The no-op refresh optimization means the
  panel doesn't re-render if nothing changed. Check
  `sensor.office_panel_last_refresh` — if it's recent, the panel is
  healthy and just has nothing to redraw.
- **Battery drain faster than expected.** Check
  `sensor.office_panel_battery` over a day. SPEC §10 targets 3–6 weeks
  between charges on stock wake cadence. Lots of rapid-fire HA state
  changes (e.g., a misconfigured template recomputing every second)
  will wake the panel unnecessarily — if wake frequency is higher than
  the scheduled cadence, something is waking the ESP externally.
- **Panel falls behind HA.** Wake cycle is up to 15 min weekday, 60 min
  weekend, 2 h overnight. "Up to" because the device wakes on its own
  schedule, not on HA state changes. If you need faster reflection for
  testing, temporarily shorten `sleep_duration` in
  `esphome/office_panel.yaml`.
