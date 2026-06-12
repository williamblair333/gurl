# GURL Roadmap

## In Progress

- **Plan 1: Foundation + Serial Backend + CLI** — project scaffold, core models, config loader, SerialBackend, RelayRegistry, typer CLI (`docs/superpowers/plans/2026-06-11-gurl-plan-1-foundation.md`)

## Planned

- **Plan 2: Tier 1 backends** — USB HID (DCTTECH, UCREATE, LCUS), Modbus RTU/TCP, HTTP/REST (Tasmota, ESPHome), MQTT
- **Plan 3: Daemon + HA MQTT Discovery** — persistent daemon, MQTT Discovery payloads, state polling, pulse-via-MQTT
- **Plan 4: Tier 2/3 backends** — Sonoff LAN, Raspberry Pi GPIO, Arduino/Firmata, Zigbee2MQTT, Z-Wave JS
- **PyPI release** — `pip install relay-control[serial]`
- **HACS custom component** — native Home Assistant UI integration
- **Web UI** — lightweight dashboard for relay status and control

## Completed

- Project renamed from `usb_ch340_relay_control` to `gurl`
- Full architecture design spec written and approved
- README overhauled with GURL branding, protocol table, HA guide, CLI reference
- GitHub topics and description added (18 topics)
- Serial/TTY backend design complete (implemented in original `relay_control.py`, to be migrated in Plan 1)
