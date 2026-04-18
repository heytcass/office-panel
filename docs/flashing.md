# Flashing

Initial flash and OTA procedure for the office panel firmware.

## Initial flash

To be filled in after Phase 1 firmware exists in `esphome/office_panel.yaml`.

Tom uses the Home Assistant ESPHome app (formerly the "ESPHome add-on") for both initial flash and OTA. There is no local `esphome` CLI on the workstation.

## OTA updates

Once the initial flash is complete and the device is on the IoT VLAN talking to HA, all subsequent updates ship via the HA ESPHome app's OTA flow.
