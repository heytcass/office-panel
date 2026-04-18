# Flashing

How to flash `esphome/office-panel.yaml` onto a reTerminal E1002.

## Prerequisites

- reTerminal E1002, fully charged, USB-C cable for initial flash
- Home Assistant running with the ESPHome add-on (what Seeed calls the
  "ESPHome add-on"; Home Assistant now markets it as the "ESPHome app")
- The `esphome-device-library` branch that adds the E1002 hardware
  package is merged to `main`, OR the firmware is temporarily pointed
  at the feature branch (see "Iterating on the device library" below)
- `esphome/secrets.yaml` exists (copy from `secrets.yaml.example` and
  fill in real values)

## Initial flash

1. Clone this repo onto the machine running the HA ESPHome add-on, or
   copy `esphome/office-panel.yaml` + `esphome/secrets.yaml` into the
   add-on's config directory. (Tom's typical setup: the repo lives
   outside HA and is rsync'd in; pick whichever fits your workflow.)
2. Open the HA ESPHome add-on UI. Click **New device** → point it at
   `office-panel.yaml`, or hit **Adopt** if the device is broadcasting
   via Improv after a prior Seeed factory-firmware flash.
3. Connect the E1002 via USB-C to the HA host.
4. Click **Install → Plug into the computer running ESPHome Dashboard**.
5. Select the serial port and let ESPHome compile + flash. First compile
   pulls device-library packages from GitHub; takes a few minutes.
6. When flashing completes, the panel renders a "Syncing..." screen
   until HA publishes a real `sensor.office_mode` value.

## Iterating on the device library

The firmware imports the hardware package via `packages: url:` pointing
at `heytcass/esphome-device-library@main`. If you're iterating on the
device library before the change is merged:

```yaml
packages:
  device:
    url: https://github.com/heytcass/esphome-device-library
    ref: add-seeed-reterminal-e1002   # ← feature branch, temporary
    files:
      - common/base.yaml
      - common/esp32s3-psram-platform.yaml
      - common/diagnostics.yaml
      - devices/seeed/reterminal-e1002.yaml
    refresh: 0s   # ← force re-fetch on every compile during dev
```

Revert `ref` to `main` and `refresh` to `1d` before flashing a production
build.

## OTA updates

Once the first serial flash completes and the device is on the network,
all subsequent updates ship via the HA ESPHome add-on's OTA flow —
**Install → Wirelessly**. No cables needed.

## Troubleshooting

- **Panel stays on "Syncing..."**: HA isn't publishing `sensor.office_mode`.
  Check `homeassistant/packages/office-panel.yaml` is loaded (reload HA
  configuration or restart), and that `calendar.office` exists (Local
  Calendar integration → add integration named "Office").
- **Compile fails on `Seeed-reTerminal-E1002` model**: you need ESPHome
  2025.11.0 or newer. The HA add-on's ESPHome version is pinned to the
  add-on release; update the add-on if it's out of date.
- **Device reboots during render**: `api: reboot_timeout` in the device
  package is 0s — e-paper refresh blocks 15–30 s and would trip a
  shorter timeout. If you still see reboots, check the ESPHome log for
  the actual cause.
