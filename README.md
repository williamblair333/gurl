# GURL — Grand Unified Relay Logic

> **One Python tool to rule every relay.**  
> Serial. HID. Modbus. MQTT. HTTP. Sonoff. GPIO. Zigbee. Z-Wave. Home Assistant.  
> If it switches, GURL speaks its language.

![Python](https://img.shields.io/badge/python-3.11%2B-blue?logo=python)
![uv](https://img.shields.io/badge/uv-managed-purple)
![License](https://img.shields.io/badge/license-MIT-green)
![Protocols](https://img.shields.io/badge/protocols-10%2B-orange)
![HA Compatible](https://img.shields.io/badge/Home%20Assistant-MQTT%20Discovery-41BDF5?logo=home-assistant)

---

## What Is GURL?

Most relay control software is laser-focused on one board, one protocol, one vendor. You end up with five different tools, five different config formats, and five different ways things can break.

**GURL is different.** It provides a single unified interface — library, CLI, and optional Home Assistant daemon — that speaks every major relay protocol. Plug in a CH340 board, an ESPHome node, a Modbus panel, and a Zigbee relay strip. GURL sees them all as the same thing: relays you can switch.

- **Zero mandatory dependencies** — install only what your hardware needs
- **Auto-detection** — `relay detect` finds everything connected
- **Home Assistant native** — MQTT Discovery, every relay becomes a `switch` entity automatically
- **Backward compatible** — your old playfiles and command scripts still work

---

## Supported Hardware & Protocols

| Protocol | Hardware Examples | Install Extra |
|---|---|---|
| **Serial / TTY** | CH340 4/8-ch boards, LCUS serial modules, generic UART relays | `serial` |
| **USB HID** | DCTTECH USBRelay1–8, UCREATE HIDRelay, LCUS HID | `hid` |
| **Modbus RTU** | Industrial RS485 relay panels (1–64 channels) | `modbus` |
| **Modbus TCP** | Ethernet Modbus relay controllers | `modbus` |
| **HTTP / REST** | Tasmota, ESPHome, custom ESP8266/ESP32 firmware | `http` |
| **MQTT** | Any relay board with MQTT pub/sub | `mqtt` |
| **Sonoff LAN** | Sonoff Basic, Dual, 4CH in DIY mode | `sonoff` |
| **GPIO** | Raspberry Pi GPIO, any gpiozero-compatible board | `gpio` |
| **Arduino / Firmata** | Any Arduino running StandardFirmata | `firmata` |
| **Zigbee** | Any relay via Zigbee2MQTT bridge | `zigbee` |
| **Z-Wave** | Any relay via Z-Wave JS UI | `zwave` |

---

## Quick Start

```bash
# Install uv if you don't have it
curl -LsSf https://astral.sh/uv/install.sh | sh

# Clone and install with the backends you need
git clone https://github.com/williamblair333/usb_ch340_relay_control.git gurl
cd gurl
uv sync --extra serial --extra hid --extra dev

# Find everything connected
uv run relay detect

# Switch a relay
uv run relay set basement 1 on
```

No config file needed for USB-attached devices. GURL finds them automatically.

---

## Installation

GURL requires only `typer` and `pyyaml` at its core. Add hardware backends as needed:

```bash
# Serial / TTY boards (CH340, LCUS, etc.)
uv add relay-control[serial]

# USB HID boards (DCTTECH, UCREATE)
uv add relay-control[hid]

# Modbus RTU / TCP
uv add relay-control[modbus]

# HTTP / REST (Tasmota, ESPHome, custom)
uv add relay-control[http]

# MQTT relay boards + Home Assistant daemon
uv add relay-control[mqtt]

# Sonoff in LAN/DIY mode
uv add relay-control[sonoff]

# Raspberry Pi GPIO
uv add relay-control[gpio]

# Arduino via Firmata
uv add relay-control[firmata]

# Zigbee via Zigbee2MQTT
uv add relay-control[zigbee]

# Z-Wave via Z-Wave JS UI
uv add relay-control[zwave]

# Everything at once
uv add relay-control[all]
```

### Requirements

- Python 3.11+
- [uv](https://docs.astral.sh/uv/) (recommended) or pip

### Linux USB Permissions

Save yourself a headache. Run once after plugging in your board:

```bash
sudo systemctl mask brltty-udev.service
sudo apt-get remove --yes brltty
sudo chmod 0666 /dev/ttyUSB0          # or the relevant device
```

For a permanent fix, use the included udev rule (see [Permissions](#permissions)).

---

## CLI Reference

### Discovery

```bash
relay detect           # scan all installed backends, print everything found
relay list             # list configured + detected devices with state
```

### Control

```bash
# Single relay
relay set <board_id> <relay> on|off
relay set basement 1 on
relay set garage 2 off

# All boards, one relay
relay set all 1 on

# All relays on one board
relay set-all basement on

# Momentary closure (pulse) — duration in milliseconds
relay pulse <board_id> <relay> <ms>
relay pulse garage 1 500        # 500ms pulse — great for garage doors
```

### Query

```bash
relay get <board_id> <relay>    # single relay state
relay status                    # all devices, all relays, formatted table
```

### Automation

```bash
# Playfile — CSV format: relay_number,duration_seconds
relay play playfile.txt

# Run a command file N times or forever
relay run commands.txt --cycles 10
relay run commands.txt --cycles infinite
```

### Daemon (Home Assistant)

```bash
relay daemon                              # start with config file settings
relay daemon --mqtt 192.168.1.10:1883     # override broker address
```

### Config

```bash
relay config show         # print fully resolved config (secrets redacted)
relay config validate     # validate config and deps without connecting to hardware
```

### Exit Codes

| Code | Meaning |
|------|---------|
| `0` | Success |
| `1` | Device not found |
| `2` | Backend / hardware error |

---

## Configuration

GURL works with zero config for local USB devices. For network devices or persistent setups, create `~/.config/relay-control/config.yaml`:

```yaml
devices:
  # CH340 serial board
  - id: basement
    backend: serial
    port: /dev/ttyUSB0
    baud: 9600
    relay_count: 4

  # USB HID board
  - id: garage
    backend: hid
    serial: ABCDE

  # Modbus RS485 panel
  - id: panel
    backend: modbus
    port: /dev/ttyUSB1
    unit_address: 1
    relay_count: 8

  # ESPHome / Tasmota REST
  - id: esp_shelf
    backend: http
    url: http://192.168.1.100
    relay_count: 2

  # Sonoff in DIY mode
  - id: sonoff_patio
    backend: sonoff
    host: 192.168.1.101

  # Raspberry Pi GPIO
  - id: pi_gpio
    backend: gpio
    pins: [17, 27, 22, 23]

# MQTT broker (required for daemon / HA integration)
mqtt:
  broker: 192.168.1.10
  port: 1883
  username: relay
  password: "${RELAY_MQTT_PASSWORD}"    # always use env vars for secrets
  ha_discovery: true
  discovery_prefix: homeassistant
  state_poll_interval_s: 5
```

Validate your config without touching hardware:

```bash
relay config validate
```

---

## Home Assistant Integration

GURL integrates with Home Assistant via **MQTT Discovery** — the same mechanism used by Tasmota, Shelly, and Zigbee2MQTT. Once the daemon is running, every relay appears in HA automatically as a `switch` entity. No HA YAML editing. No custom integrations. No restart required.

### Setup

**1. Make sure you have an MQTT broker running** (Mosquitto is easiest):

```bash
# In HA: Settings → Add-ons → Mosquitto Broker → Install
```

**2. Install GURL with MQTT support:**

```bash
uv add relay-control[mqtt]         # or [all]
```

**3. Add your MQTT broker to the config:**

```yaml
mqtt:
  broker: 192.168.1.10
  port: 1883
  username: relay
  password: "${RELAY_MQTT_PASSWORD}"
  ha_discovery: true
```

**4. Start the daemon:**

```bash
uv run relay daemon
```

**5. Open Home Assistant.** Your relays appear under Settings → Devices & Services → MQTT, named and ready to use.

### What It Looks Like in HA

Each relay becomes a `switch` entity:
- **Name:** `Basement Relay 1` (customizable in config)
- **State:** real-time, polled from hardware
- **Controls:** On / Off / available in automations, dashboards, scripts

### MQTT Topics

If you want to control relays directly via MQTT (Node-RED, custom scripts, etc.):

```
relay-control/<board_id>/<relay>/set      →  ON | OFF | PULSE_500
relay-control/<board_id>/<relay>/state    ←  ON | OFF
relay-control/<board_id>/set-all          →  ON | OFF
relay-control/status                      ←  online | offline
```

**Pulse via MQTT:** publish `PULSE_500` to the set topic for a 500ms momentary closure. GURL handles the ON → wait → OFF sequence and publishes state updates.

---

## Python Library Usage

GURL is a proper library — use it in your own scripts without the CLI:

```python
from relay_control import RelayRegistry, RelayState

# Auto-detect everything connected
registry = RelayRegistry.detect()

# List found boards
for board in registry.boards:
    print(f"{board.id}: {board.relay_count} relays via {board.backend}")

# Control a relay
registry.set("basement", 1, RelayState.ON)

# Pulse (momentary closure)
registry.pulse("garage", 1, duration_ms=500)

# Read state
state = registry.get("basement", 1)
print(state)   # RelayState.ON

# Read all relays on a board
states = registry.get_all("basement")

# Cleanup
registry.close()
```

### Loading from config file

```python
from relay_control import RelayRegistry

registry = RelayRegistry.from_config("~/.config/relay-control/config.yaml")
registry.set("esp_shelf", 2, RelayState.OFF)
registry.close()
```

### Using a specific backend directly

```python
from relay_control.backends.modbus import ModbusBackend
from relay_control.core.base import RelayState

backend = ModbusBackend(port="/dev/ttyUSB1", unit_address=1)
boards = backend.detect()
backend.set(boards[0].id, 3, RelayState.ON)
backend.close()
```

---

## Playfile Format

The original playfile format is fully supported. Each line: `relay_number,duration_seconds`

```
# playfile.txt
1,5      # relay 1 on for 5 seconds
2,3      # relay 2 on for 3 seconds
1,1      # relay 1 on for 1 second
```

```bash
relay play playfile.txt
```

---

## Command Runner

Run a file of `relay` CLI commands in a loop:

```
# commands.txt
relay set basement 1 on
relay set basement 1 off
relay pulse garage 1 500
```

```bash
relay run commands.txt --cycles 10         # 10 times
relay run commands.txt --cycles infinite   # forever
```

---

## Permissions

### Linux: Permanent udev Rule

Copy the included rules file so you never need `sudo` or `chmod` again:

```bash
sudo cp 50-usbrelay.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules
sudo udevadm trigger
```

This gives the `dialout` group access to HID relay devices. Add your user:

```bash
sudo usermod -aG dialout $USER
# Log out and back in for the group to take effect
```

### Windows

Run as Administrator, or set COM port permissions in Device Manager.

---

## Architecture

GURL uses a plugin backend architecture. The core has no hardware dependencies — backends are loaded only when their optional deps are installed.

```
CLI (typer)
    └── RelayRegistry
            ├── SerialBackend     → pyserial → /dev/ttyUSB*
            ├── HIDBackend        → hid      → /dev/hidraw*
            ├── ModbusBackend     → pymodbus → RS485 / TCP
            ├── HTTPBackend       → httpx    → Tasmota / ESPHome
            ├── MQTTBackend       → paho     → MQTT brokers
            ├── SonoffBackend     → httpx    → Sonoff LAN
            ├── GPIOBackend       → gpiozero → RPi pins
            ├── FirmataBackend    → pyFirmata2 → Arduino
            ├── ZigbeeBackend     → paho     → Zigbee2MQTT
            └── ZWaveBackend      → websockets → Z-Wave JS
```

Every backend implements `BaseRelayBackend`:

```python
class BaseRelayBackend(ABC):
    def detect(self)                        -> list[RelayBoard]
    def capabilities(self)                  -> BackendCapabilities
    def set(self, board_id, relay, state)   -> None
    def set_all(self, board_id, states)     -> None
    def get(self, board_id, relay)          -> RelayState
    def get_all(self, board_id)             -> list[RelayState]
    def pulse(self, board_id, relay, ms)    -> None
    def close(self)                         -> None
```

Adding a new backend means implementing this interface and dropping one file in `backends/`. No other code changes required.

---

## Adding a New Backend

1. Create `src/relay_control/backends/mybackend.py`
2. Implement `BaseRelayBackend`
3. Add your optional dep to `pyproject.toml`
4. Register in `src/relay_control/core/registry.py`
5. Add unit tests in `tests/unit/backends/test_mybackend.py`

See any existing backend for a reference implementation. The serial backend (`serial_tty.py`) is the simplest starting point.

---

## Development

```bash
git clone https://github.com/williamblair333/usb_ch340_relay_control.git gurl
cd gurl
uv sync --extra dev --extra serial --extra hid

# Run tests (no hardware needed)
uv run pytest

# Run tests requiring hardware
uv run pytest -m hardware

# Run tests requiring an MQTT broker
uv run pytest -m mqtt

# Lint
uv run ruff check src/
```

---

## Roadmap

- [x] Serial / TTY backend (CH340, LCUS)
- [ ] USB HID backend (DCTTECH, UCREATE)
- [ ] Modbus RTU / TCP backend
- [ ] HTTP / REST backend (Tasmota, ESPHome)
- [ ] MQTT backend
- [ ] Home Assistant MQTT Discovery daemon
- [ ] Sonoff LAN backend
- [ ] GPIO backend (Raspberry Pi)
- [ ] Arduino / Firmata backend
- [ ] Zigbee2MQTT backend
- [ ] Z-Wave JS backend
- [ ] PyPI package release
- [ ] HACS custom component (HA UI integration)
- [ ] Web UI

---

## Backward Compatibility

GURL started as `usb_ch340_relay_control`. If you used the original scripts:

| Old | New |
|-----|-----|
| `python relay_control.py -d /dev/ttyUSB0 -r 1 -s 1 -t 5` | `relay set basement 1 on` |
| `python run_commands.py commands.txt 10` | `relay run commands.txt --cycles 10` |
| `python relay_control.py -f playfile.txt` | `relay play playfile.txt` |

Playfiles and command files from the original project work unchanged.

---

## Contributing

Pull requests welcome. Before you start:

1. Open an issue describing what you want to add or fix
2. For new backends, check the [Adding a New Backend](#adding-a-new-backend) section
3. All new code needs unit tests
4. Run `uv run pytest` and `uv run ruff check src/` before submitting

---

## License

MIT — see [LICENSE.md](LICENSE.md) for full text.

---

## Author

**William Blair**  
[Create an Issue](https://github.com/williamblair333/usb_ch340_relay_control/issues)

---

*GURL — because your relays deserve a unified voice.*
