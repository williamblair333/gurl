# GURL — Grand Unified Relay Logic: Design Spec

**Date:** 2026-06-11  
**Status:** Approved  
**Author:** William Blair  

---

## 1. Purpose

Transform `usb_ch340_relay_control` into **GURL** — a single Python library and CLI that controls ≥80% of commercially available relay hardware through a unified interface. Supported transports: serial/TTY, USB HID, Modbus RTU/TCP, HTTP/REST, MQTT, Sonoff LAN, Raspberry Pi GPIO, Arduino/Firmata, Zigbee2MQTT, and Z-Wave JS.

Home Assistant integration is first-class via MQTT Discovery — every relay appears as a `switch` entity automatically, with no HA config editing required.

---

## 2. Architecture

Three layers, each independently useful:

```
┌─────────────────────────────────────────┐
│  CLI  (relay detect / set / daemon)     │  typer-based, thin wrapper
├─────────────────────────────────────────┤
│  Library  (relay_control)               │  importable, no CLI dependency
│  ┌──────────────┐  ┌─────────────────┐  │
│  │ Registry     │  │ HA MQTT         │  │
│  │ (auto-detect)│  │ Discovery       │  │
│  └──────────────┘  └─────────────────┘  │
│  ┌──────────────────────────────────────┤
│  │ BaseRelayBackend (ABC)               │
│  │  SerialBackend   HIDBackend          │
│  │  ModbusBackend   MQTTBackend         │
│  │  HTTPBackend     SonoffBackend       │
│  │  GPIOBackend     FirmataBackend      │
│  │  ZigbeeBackend   ZWaveBackend        │
│  └──────────────────────────────────────┤
├─────────────────────────────────────────┤
│  Daemon  (relay daemon)                 │  optional persistent MQTT bridge
└─────────────────────────────────────────┘
```

The daemon is the library in long-running mode — not a separate codebase. All three layers share the same `RelayBoard` / `RelayState` models.

---

## 3. Core Models

```python
from enum import Enum
from dataclasses import dataclass, field

class RelayState(Enum):
    ON      = "ON"
    OFF     = "OFF"
    UNKNOWN = "UNKNOWN"

@dataclass
class BackendCapabilities:
    has_state_readback: bool = False
    has_bulk_write:     bool = False
    has_atomic_pulse:   bool = False

@dataclass
class RelayBoard:
    id:           str                   # e.g. "ABCDE", "/dev/ttyUSB0", "esp_shelf"
    backend:      str                   # "serial" | "hid" | "modbus" | ...
    relay_count:  int
    state:        list[RelayState]      = field(default_factory=list)
    meta:         dict                  = field(default_factory=dict)
```

---

## 4. Backend Plugin Interface

```python
from abc import ABC, abstractmethod

class BaseRelayBackend(ABC):

    @abstractmethod
    def detect(self) -> list[RelayBoard]:
        """Find all hardware accessible by this backend."""

    @abstractmethod
    def capabilities(self) -> BackendCapabilities:
        """Advertise what this backend actually supports."""

    @abstractmethod
    def set(self, board_id: str, relay: int, state: RelayState) -> None:
        """Set a single relay ON or OFF."""

    @abstractmethod
    def set_all(self, board_id: str, states: list[RelayState]) -> None:
        """Set all relays in one transaction (or loop if bulk not supported)."""

    @abstractmethod
    def get(self, board_id: str, relay: int) -> RelayState:
        """Read a single relay state. Returns UNKNOWN if not supported."""

    @abstractmethod
    def get_all(self, board_id: str) -> list[RelayState]:
        """Read all relay states. Returns list of UNKNOWN if not supported."""

    @abstractmethod
    def pulse(self, board_id: str, relay: int, duration_ms: int) -> None:
        """Momentary closure. Atomic if has_atomic_pulse, else set/sleep/set."""

    @abstractmethod
    def close(self) -> None:
        """Release all connections and resources."""
```

**Contract:**
- `get()` / `get_all()` on backends without readback return `RelayState.UNKNOWN` — never raise.
- `set_all()` on backends without bulk write loops `set()` — never raise.
- `pulse()` on backends without atomic pulse does `set(ON)` + sleep + `set(OFF)` — never raise.
- All methods that touch hardware wrap exceptions in `RelayBackendError` (defined in `core/base.py`).

---

## 5. Protocol Backends

| Backend    | Protocols / Hardware                          | Optional dep       | Tier |
|------------|-----------------------------------------------|--------------------|------|
| `serial`   | CH340, generic TTY serial, LCUS serial        | `pyserial`         | 1    |
| `hid`      | DCTTECH (USBRelayX), UCREATE (HIDRelay), LCUS | `hid`              | 1    |
| `modbus`   | Modbus RTU (RS485), Modbus TCP                | `pymodbus`         | 1    |
| `http`     | Tasmota, ESPHome, generic REST                | `httpx`            | 1    |
| `mqtt`     | MQTT pub/sub relay boards                     | `paho-mqtt`        | 1    |
| `sonoff`   | Sonoff LAN (DIY mode)                         | `httpx`            | 2    |
| `gpio`     | Raspberry Pi GPIO, gpiozero                   | `gpiozero`         | 2    |
| `firmata`  | Arduino via Firmata protocol                  | `pyFirmata2`       | 2    |
| `zigbee`   | Zigbee2MQTT bridge                            | `paho-mqtt`        | 3    |
| `zwave`    | Z-Wave JS via WebSocket                       | `websockets`       | 3    |

Install by tier:
```bash
uv add relay-control[serial,hid,modbus,http,mqtt]   # tier 1
uv add relay-control[all]                           # everything
```

---

## 6. Configuration

### Auto-detect (zero config)

`relay detect` scans all installed backends in parallel. Good for first run and single-device setups.

### Config file

Location: `~/.config/relay-control/config.yaml`  
Explicit config wins over auto-detect for the same device.

```yaml
devices:
  - id: basement
    backend: serial
    port: /dev/ttyUSB0
    baud: 9600
    relay_count: 4

  - id: garage
    backend: hid
    serial: ABCDE

  - id: panel
    backend: modbus
    port: /dev/ttyUSB1
    unit_address: 1
    relay_count: 8

  - id: esp_shelf
    backend: http
    url: http://192.168.1.100
    relay_count: 2

  - id: sonoff_patio
    backend: sonoff
    host: 192.168.1.101

  - id: pi_gpio
    backend: gpio
    pins: [17, 27, 22, 23]

mqtt:
  broker: 192.168.1.10
  port: 1883
  username: relay
  password: "${RELAY_MQTT_PASSWORD}"
  ha_discovery: true
  discovery_prefix: homeassistant
  state_poll_interval_s: 5
```

**Rules:**
- Secrets always via env var interpolation (`"${VAR}"`) — never plaintext in file.
- Unknown backend name at startup: clear error with install hint.
- Missing optional dep: `ModbusBackend requires: uv add relay-control[modbus]`.

---

## 7. CLI

Built with `typer`. Exit codes: `0` success, `1` device not found, `2` backend error.

```bash
# Discovery
relay detect                              # scan all backends
relay list                                # list configured + detected devices

# Control
relay set <board_id> <relay> on|off       # single relay
relay set basement 1 on
relay set all 1 on                        # all boards, relay 1
relay pulse <board_id> <relay> <ms>       # momentary closure (ms)
relay pulse garage 1 500
relay set-all <board_id> on|off           # all relays on a board

# Query
relay get <board_id> <relay>              # single relay state
relay status                              # all devices, all relays

# Playfile (backward compatible with original relay_control.py)
relay play playfile.txt                   # run CSV sequence (relay,duration_s)
relay run commands.txt --cycles 10        # loop command file N times or infinite

# Daemon
relay daemon                              # start with config file settings
relay daemon --mqtt 192.168.1.10:1883     # override broker

# Config
relay config show                         # print resolved config
relay config validate                     # validate config + deps, no hardware
```

`relay set` with no config and exactly one detected board uses it automatically. Ambiguous detection is a hard error with a helpful message.

---

## 8. Daemon + Home Assistant MQTT Discovery

On start the daemon:
1. Loads config + auto-detects hardware
2. Publishes retained discovery payloads to HA
3. Subscribes to command topics
4. Polls hardware state on configurable interval and publishes updates

### Topic layout

```
homeassistant/switch/relay_<board_id>_<n>/config   ← discovery (retained)
relay-control/<board_id>/<n>/state                 ← current state (retained)
relay-control/<board_id>/<n>/set                   ← commands: ON | OFF | PULSE_<ms>
relay-control/<board_id>/set-all                   ← bulk: ON | OFF
relay-control/status                               ← online | offline (LWT)
```

### Discovery payload

```json
{
  "name": "Basement Relay 1",
  "unique_id": "relay_basement_1",
  "command_topic": "relay-control/basement/1/set",
  "state_topic": "relay-control/basement/1/state",
  "payload_on": "ON",
  "payload_off": "OFF",
  "availability_topic": "relay-control/status",
  "payload_available": "online",
  "payload_not_available": "offline",
  "device": {
    "identifiers": ["relay_basement"],
    "name": "Basement Relay Board",
    "model": "CH340 4-channel",
    "manufacturer": "GURL"
  }
}
```

HA auto-configures each relay as a `switch` entity. No HA YAML editing required.

**Pulse via MQTT:** `PULSE_500` on command topic triggers 500ms pulse, publishes `ON` then `OFF` state updates automatically.

**State polling** is per-device configurable. Backends without readback (`has_state_readback=False`) are not polled — state is inferred from last command.

---

## 9. Project Structure

```
relay-control/
├── src/
│   └── relay_control/
│       ├── core/
│       │   ├── base.py          # BaseRelayBackend, RelayBoard, RelayState
│       │   ├── registry.py      # auto-detect, parallel backend enumeration
│       │   └── models.py        # BackendCapabilities, config dataclasses
│       ├── backends/
│       │   ├── serial_tty.py
│       │   ├── hid.py
│       │   ├── modbus.py
│       │   ├── http.py
│       │   ├── mqtt.py
│       │   ├── sonoff.py
│       │   ├── gpio.py
│       │   ├── firmata.py
│       │   ├── zigbee.py
│       │   └── zwave.py
│       ├── discovery/
│       │   ├── auto.py          # parallel backend enumeration
│       │   └── ha_mqtt.py       # HA discovery payload builder + publisher
│       ├── daemon/
│       │   └── service.py       # persistent loop, MQTT command bridge
│       ├── config.py            # YAML load, env var interpolation, validation
│       └── cli/
│           └── main.py          # typer app entry point
├── tests/
│   ├── unit/
│   │   ├── backends/
│   │   │   ├── test_serial.py
│   │   │   ├── test_hid.py
│   │   │   ├── test_modbus.py
│   │   │   ├── test_http.py
│   │   │   ├── test_mqtt.py
│   │   │   └── ...
│   │   ├── test_registry.py
│   │   ├── test_config.py
│   │   └── test_ha_mqtt.py
│   └── integration/
│       ├── test_serial_live.py    # @pytest.mark.hardware
│       └── test_daemon_mqtt.py    # @pytest.mark.mqtt
├── docs/
│   └── superpowers/
│       └── specs/
│           └── 2026-06-11-gurl-design.md
├── pyproject.toml
└── .python-version
```

---

## 10. pyproject.toml

```toml
[project]
name = "relay-control"
version = "2.0.0"
description = "GURL — Grand Unified Relay Logic"
requires-python = ">=3.11"
dependencies = [
    "typer[all]>=0.12",
    "pyyaml>=6.0",
]

[project.optional-dependencies]
serial  = ["pyserial>=3.5"]
hid     = ["hid>=1.0"]
modbus  = ["pymodbus>=3.0"]
http    = ["httpx>=0.27"]
mqtt    = ["paho-mqtt>=2.0"]
sonoff  = ["httpx>=0.27"]
gpio    = ["gpiozero>=2.0"]
firmata = ["pyFirmata2>=1.2"]
zigbee  = ["paho-mqtt>=2.0"]
zwave   = ["websockets>=12.0"]
all     = [
    "pyserial>=3.5", "hid>=1.0", "pymodbus>=3.0",
    "httpx>=0.27", "paho-mqtt>=2.0", "gpiozero>=2.0",
    "pyFirmata2>=1.2", "websockets>=12.0",
]
dev     = ["pytest", "pytest-asyncio", "pytest-mock", "ruff"]

[project.scripts]
relay = "relay_control.cli.main:app"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

---

## 11. Testing

**Unit tests** — no hardware required, everything mocked at the OS boundary.

Each backend test covers:
- Correct bytes / payload on `set()`, `set_all()`, `pulse()`
- Correct state parsing in `get()` / `get_all()`
- `RelayState.UNKNOWN` returned (not raised) when readback unsupported
- `close()` releases the connection
- Failure path for silent-fail scenarios (write with no ACK, HTTP 200 wrong body)

**Integration tests** — skipped without hardware or broker:

```bash
uv run pytest                     # unit only (default)
uv run pytest -m hardware         # requires device attached
uv run pytest -m mqtt             # requires broker running
```

**HA discovery tests** verify payload shape against the HA MQTT Discovery schema.

---

## 12. Backward Compatibility

The original `relay_control.py` playfile format (`relay,duration_s` CSV) is preserved via `relay play`. The original `run_commands.py` loop behavior is preserved via `relay run --cycles`. Existing shell scripts that called the old scripts continue to work via the new CLI.

---

## 13. Out of Scope (v1)

- Cloud relay APIs (Tuya cloud, Shelly cloud) — LAN-only for v1
- Web UI
- Authentication / multi-user access control on the daemon
- Windows GPIO support
