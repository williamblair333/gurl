# Changelog

## 2026-06-12 — Project reborn as GURL (Grand Unified Relay Logic)

### Added
- `docs/superpowers/specs/2026-06-11-gurl-design.md` — full architecture design spec: plugin ABC, 10 protocol backends, daemon, HA MQTT Discovery, config format, CLI, testing strategy
- `docs/superpowers/plans/2026-06-11-gurl-plan-1-foundation.md` — TDD implementation plan covering project scaffold, core models, config loader, SerialBackend, RelayRegistry, typer CLI, and stub backends (8 tasks, full code)
- GitHub repo description and 18 topics added (`relay`, `home-assistant`, `mqtt`, `modbus`, `python`, `iot`, `usb-relay`, `ch340`, `automation`, `tasmota`, `esphome`, `sonoff`, `raspberry-pi`, `gpio`, `zigbee`, `zwave`, `hid`, `home-automation`)

### Changed
- `README.md` — complete overhaul: GURL branding, protocol/hardware table, quick start, full CLI reference with playfile and command runner examples inline, HA MQTT Discovery integration guide, Python library usage, architecture diagram, adding-a-backend guide, roadmap, backward compatibility table
- Repo renamed from `usb_ch340_relay_control` to `gurl` on GitHub and locally
- Remote URL updated to `https://github.com/williamblair333/gurl.git`

### Fixed
- `README.md` — removed duplicate Linux USB Permissions block, merged standalone Playfile Format and Command Runner sections into CLI Reference, fixed three stale `usb_ch340_relay_control` URLs
- `relay_set.py` — identified shell injection via `subprocess.run(..., shell=True)` with user-controlled `--device`; fix planned in Plan 1 (`relay run` uses `shlex.split`)
- `run_commands.py` — identified shell injection; Plan 1 implementation uses `shlex.split` + `shell=False`
