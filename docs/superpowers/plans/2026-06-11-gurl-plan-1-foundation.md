# GURL Plan 1: Foundation + Serial Backend + CLI

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Scaffold the GURL project structure, implement core models and the plugin ABC, migrate the CH340 serial backend, build a registry, and deliver a working `relay` CLI — replacing the existing `relay_control.py` scripts with the new architecture.

**Architecture:** Plugin ABC (`BaseRelayBackend`) defines the contract; `SerialBackend` is the first concrete implementation. `RelayRegistry` auto-detects or loads from config and delegates all commands to the appropriate backend. `typer` CLI wraps the registry.

**Tech Stack:** Python 3.11+, uv, typer[all], pyyaml, pyserial, pytest, pytest-mock, ruff

---

## File Map

```
src/relay_control/__init__.py               exports: RelayState, RelayBoard, RelayRegistry
src/relay_control/core/__init__.py
src/relay_control/core/base.py              RelayState, BackendCapabilities, RelayBoard,
                                            RelayBackendError, BaseRelayBackend
src/relay_control/core/registry.py         RelayRegistry
src/relay_control/backends/__init__.py
src/relay_control/backends/serial_tty.py   SerialBackend
src/relay_control/config.py                load_config, GURLConfig, DeviceConfig, MQTTConfig
src/relay_control/cli/__init__.py
src/relay_control/cli/main.py              typer app — detect, list, set, get, status,
                                            pulse, set-all, play, run, config
tests/__init__.py
tests/conftest.py                          pytest marks: hardware, mqtt
tests/unit/__init__.py
tests/unit/backends/__init__.py
tests/unit/backends/test_serial.py
tests/unit/test_config.py
tests/unit/test_registry.py
pyproject.toml
.python-version
```

---

## Task 1: Project Scaffolding

**Files:**
- Create: `pyproject.toml`
- Create: `.python-version`
- Create: all `__init__.py` stubs listed in the file map
- Create: `tests/conftest.py`

- [ ] **Step 1: Write `.python-version`**

```
3.11
```

- [ ] **Step 2: Write `pyproject.toml`**

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
dev = ["pytest>=8.0", "pytest-asyncio>=0.23", "pytest-mock>=3.12", "ruff>=0.4"]

[project.scripts]
relay = "relay_control.cli.main:app"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/relay_control"]

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-v"

[tool.ruff.lint]
select = ["E", "F", "I"]
```

- [ ] **Step 3: Create all `__init__.py` stubs**

Create empty files (touch is fine) at:
```
src/relay_control/__init__.py
src/relay_control/core/__init__.py
src/relay_control/backends/__init__.py
src/relay_control/cli/__init__.py
tests/__init__.py
tests/unit/__init__.py
tests/unit/backends/__init__.py
```

- [ ] **Step 4: Write `tests/conftest.py`**

```python
import pytest

def pytest_configure(config):
    config.addinivalue_line("markers", "hardware: requires physical relay hardware attached")
    config.addinivalue_line("markers", "mqtt: requires a running MQTT broker")
```

- [ ] **Step 5: Sync deps and verify**

```bash
uv sync --extra serial --extra dev
uv run pytest --collect-only
```

Expected: `no tests ran` (zero tests collected, no errors).

- [ ] **Step 6: Commit**

```bash
git add pyproject.toml .python-version src/ tests/
git commit -m "chore: scaffold GURL project structure"
```

---

## Task 2: Core Models and ABC

**Files:**
- Create: `src/relay_control/core/base.py`
- Create: `tests/unit/test_core.py`

- [ ] **Step 1: Write the failing test**

```python
# tests/unit/test_core.py
import pytest
from relay_control.core.base import (
    RelayState,
    BackendCapabilities,
    RelayBoard,
    RelayBackendError,
    BaseRelayBackend,
)


class TestRelayState:
    def test_on_value(self):
        assert RelayState.ON.value == "ON"

    def test_off_value(self):
        assert RelayState.OFF.value == "OFF"

    def test_unknown_value(self):
        assert RelayState.UNKNOWN.value == "UNKNOWN"


class TestBackendCapabilities:
    def test_defaults_all_false(self):
        caps = BackendCapabilities()
        assert caps.has_state_readback is False
        assert caps.has_bulk_write is False
        assert caps.has_atomic_pulse is False

    def test_explicit_true(self):
        caps = BackendCapabilities(has_state_readback=True)
        assert caps.has_state_readback is True


class TestRelayBoard:
    def test_minimal_construction(self):
        board = RelayBoard(id="test", backend="serial", relay_count=4)
        assert board.id == "test"
        assert board.relay_count == 4
        assert board.state == []
        assert board.meta == {}


class TestBaseRelayBackendIsAbstract:
    def test_cannot_instantiate_directly(self):
        with pytest.raises(TypeError):
            BaseRelayBackend()

    def test_concrete_subclass_must_implement_all_methods(self):
        class Incomplete(BaseRelayBackend):
            def detect(self): ...
            # intentionally missing other methods

        with pytest.raises(TypeError):
            Incomplete()
```

- [ ] **Step 2: Run to confirm failure**

```bash
uv run pytest tests/unit/test_core.py -v
```

Expected: `ImportError` — `relay_control.core.base` does not exist yet.

- [ ] **Step 3: Write `src/relay_control/core/base.py`**

```python
from __future__ import annotations

from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from enum import Enum


class RelayState(Enum):
    ON = "ON"
    OFF = "OFF"
    UNKNOWN = "UNKNOWN"


@dataclass
class BackendCapabilities:
    has_state_readback: bool = False
    has_bulk_write: bool = False
    has_atomic_pulse: bool = False


@dataclass
class RelayBoard:
    id: str
    backend: str
    relay_count: int
    state: list[RelayState] = field(default_factory=list)
    meta: dict = field(default_factory=dict)


class RelayBackendError(Exception):
    pass


class BaseRelayBackend(ABC):

    @abstractmethod
    def detect(self) -> list[RelayBoard]:
        """Find all hardware boards accessible by this backend."""

    @abstractmethod
    def capabilities(self) -> BackendCapabilities:
        """Return what this backend supports."""

    @abstractmethod
    def set(self, board_id: str, relay: int, state: RelayState) -> None:
        """Set a single relay ON or OFF."""

    @abstractmethod
    def set_all(self, board_id: str, states: list[RelayState]) -> None:
        """Set all relays. Loop set() if bulk not supported."""

    @abstractmethod
    def get(self, board_id: str, relay: int) -> RelayState:
        """Read single relay state. Return UNKNOWN if not supported."""

    @abstractmethod
    def get_all(self, board_id: str) -> list[RelayState]:
        """Read all relay states. Return list of UNKNOWN if not supported."""

    @abstractmethod
    def pulse(self, board_id: str, relay: int, duration_ms: int) -> None:
        """Momentary closure. Atomic or set/sleep/set fallback."""

    @abstractmethod
    def close(self) -> None:
        """Release all connections and resources."""
```

- [ ] **Step 4: Update `src/relay_control/__init__.py`**

```python
from .core.base import (
    RelayState,
    RelayBoard,
    BackendCapabilities,
    RelayBackendError,
)
from .core.registry import RelayRegistry

__all__ = [
    "RelayState",
    "RelayBoard",
    "BackendCapabilities",
    "RelayBackendError",
    "RelayRegistry",
]
```

(RelayRegistry doesn't exist yet — that's fine, the import will fail at runtime until Task 5. The `__init__.py` is the target state.)

Revert to just the base exports for now:

```python
from .core.base import (
    RelayState,
    RelayBoard,
    BackendCapabilities,
    RelayBackendError,
)

__all__ = [
    "RelayState",
    "RelayBoard",
    "BackendCapabilities",
    "RelayBackendError",
]
```

- [ ] **Step 5: Run tests and confirm they pass**

```bash
uv run pytest tests/unit/test_core.py -v
```

Expected: all 7 tests pass.

- [ ] **Step 6: Commit**

```bash
git add src/relay_control/core/base.py src/relay_control/__init__.py tests/unit/test_core.py
git commit -m "feat: add core models and BaseRelayBackend ABC"
```

---

## Task 3: Config Loader

**Files:**
- Create: `src/relay_control/config.py`
- Create: `tests/unit/test_config.py`

- [ ] **Step 1: Write failing tests**

```python
# tests/unit/test_config.py
import pytest
from pathlib import Path
from relay_control.config import load_config, GURLConfig, DeviceConfig, MQTTConfig


def test_missing_file_returns_empty_config(tmp_path):
    config = load_config(tmp_path / "nonexistent.yaml")
    assert config == GURLConfig()


def test_empty_file_returns_empty_config(tmp_path):
    f = tmp_path / "config.yaml"
    f.write_text("")
    config = load_config(f)
    assert config == GURLConfig()


def test_device_loaded(tmp_path):
    f = tmp_path / "config.yaml"
    f.write_text(
        "devices:\n"
        "  - id: basement\n"
        "    backend: serial\n"
        "    port: /dev/ttyUSB0\n"
        "    baud: 9600\n"
        "    relay_count: 4\n"
    )
    config = load_config(f)
    assert len(config.devices) == 1
    d = config.devices[0]
    assert d.id == "basement"
    assert d.backend == "serial"
    assert d.extras["port"] == "/dev/ttyUSB0"
    assert d.extras["relay_count"] == 4


def test_mqtt_block_loaded(tmp_path):
    f = tmp_path / "config.yaml"
    f.write_text(
        "mqtt:\n"
        "  broker: 192.168.1.10\n"
        "  port: 1883\n"
        "  ha_discovery: true\n"
    )
    config = load_config(f)
    assert config.mqtt is not None
    assert config.mqtt.broker == "192.168.1.10"
    assert config.mqtt.ha_discovery is True


def test_env_var_interpolated(tmp_path, monkeypatch):
    monkeypatch.setenv("RELAY_PWD", "s3cr3t")
    f = tmp_path / "config.yaml"
    f.write_text('mqtt:\n  broker: localhost\n  password: "${RELAY_PWD}"\n')
    config = load_config(f)
    assert config.mqtt.password == "s3cr3t"


def test_missing_env_var_raises(tmp_path):
    f = tmp_path / "config.yaml"
    f.write_text('mqtt:\n  broker: localhost\n  password: "${NO_SUCH_VAR}"\n')
    with pytest.raises(ValueError, match="NO_SUCH_VAR"):
        load_config(f)


def test_multiple_devices(tmp_path):
    f = tmp_path / "config.yaml"
    f.write_text(
        "devices:\n"
        "  - id: a\n    backend: serial\n    port: /dev/ttyUSB0\n"
        "  - id: b\n    backend: hid\n    serial: ABCDE\n"
    )
    config = load_config(f)
    assert len(config.devices) == 2
    assert config.devices[1].extras["serial"] == "ABCDE"
```

- [ ] **Step 2: Run to confirm failure**

```bash
uv run pytest tests/unit/test_config.py -v
```

Expected: `ImportError` — `relay_control.config` does not exist yet.

- [ ] **Step 3: Write `src/relay_control/config.py`**

```python
from __future__ import annotations

import os
import re
from dataclasses import dataclass, field
from pathlib import Path
from typing import Optional

import yaml

_ENV_RE = re.compile(r"\$\{(\w+)\}")


def _interpolate(value: str) -> str:
    def _replace(m: re.Match) -> str:
        var = m.group(1)
        result = os.environ.get(var)
        if result is None:
            raise ValueError(f"Environment variable {var!r} is not set")
        return result

    return _ENV_RE.sub(_replace, value)


def _walk(obj: object) -> object:
    if isinstance(obj, str):
        return _interpolate(obj)
    if isinstance(obj, dict):
        return {k: _walk(v) for k, v in obj.items()}
    if isinstance(obj, list):
        return [_walk(i) for i in obj]
    return obj


@dataclass
class DeviceConfig:
    id: str
    backend: str
    extras: dict = field(default_factory=dict)


@dataclass
class MQTTConfig:
    broker: str
    port: int = 1883
    username: Optional[str] = None
    password: Optional[str] = None
    ha_discovery: bool = False
    discovery_prefix: str = "homeassistant"
    state_poll_interval_s: int = 5


@dataclass
class GURLConfig:
    devices: list[DeviceConfig] = field(default_factory=list)
    mqtt: Optional[MQTTConfig] = None


def load_config(path: Path | str | None = None) -> GURLConfig:
    if path is None:
        path = Path.home() / ".config" / "relay-control" / "config.yaml"
    path = Path(path)
    if not path.exists():
        return GURLConfig()

    with open(path) as f:
        raw = yaml.safe_load(f)

    if raw is None:
        return GURLConfig()

    raw = _walk(raw)

    devices: list[DeviceConfig] = []
    for entry in raw.get("devices", []):
        entry = dict(entry)
        device_id = entry.pop("id")
        backend = entry.pop("backend")
        devices.append(DeviceConfig(id=device_id, backend=backend, extras=entry))

    mqtt: Optional[MQTTConfig] = None
    if "mqtt" in raw:
        m = raw["mqtt"]
        mqtt = MQTTConfig(
            broker=m["broker"],
            port=m.get("port", 1883),
            username=m.get("username"),
            password=m.get("password"),
            ha_discovery=m.get("ha_discovery", False),
            discovery_prefix=m.get("discovery_prefix", "homeassistant"),
            state_poll_interval_s=m.get("state_poll_interval_s", 5),
        )

    return GURLConfig(devices=devices, mqtt=mqtt)
```

- [ ] **Step 4: Run tests and confirm they pass**

```bash
uv run pytest tests/unit/test_config.py -v
```

Expected: all 7 tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/relay_control/config.py tests/unit/test_config.py
git commit -m "feat: add config loader with YAML and env var interpolation"
```

---

## Task 4: SerialBackend

**Files:**
- Create: `src/relay_control/backends/serial_tty.py`
- Create: `tests/unit/backends/test_serial.py`

- [ ] **Step 1: Write failing tests**

```python
# tests/unit/backends/test_serial.py
import pytest
from unittest.mock import MagicMock, patch, call
from relay_control.backends.serial_tty import SerialBackend, _make_cmd
from relay_control.core.base import RelayState, RelayBackendError, BackendCapabilities


class TestMakeCmd:
    def test_relay_1_on(self):
        assert _make_cmd(1, True) == bytes([0xA0, 0x01, 0x01, 0xA2])

    def test_relay_1_off(self):
        assert _make_cmd(1, False) == bytes([0xA0, 0x01, 0x00, 0xA1])

    def test_relay_4_on(self):
        assert _make_cmd(4, True) == bytes([0xA0, 0x04, 0x01, 0xA5])

    def test_relay_4_off(self):
        assert _make_cmd(4, False) == bytes([0xA0, 0x04, 0x00, 0xA4])

    def test_checksum_formula(self):
        # checksum = (0xA0 + relay + state) & 0xFF
        cmd = _make_cmd(3, True)
        assert cmd[3] == (0xA0 + 3 + 1) & 0xFF


class TestSerialBackendCapabilities:
    def test_no_state_readback(self):
        b = SerialBackend(port="/dev/ttyUSB0")
        caps = b.capabilities()
        assert caps.has_state_readback is False

    def test_no_bulk_write(self):
        b = SerialBackend(port="/dev/ttyUSB0")
        assert b.capabilities().has_bulk_write is False

    def test_no_atomic_pulse(self):
        b = SerialBackend(port="/dev/ttyUSB0")
        assert b.capabilities().has_atomic_pulse is False


class TestSerialBackendSet:
    def _make_backend(self):
        return SerialBackend(port="/dev/ttyUSB0", relay_count=4)

    @patch("relay_control.backends.serial_tty.serial")
    def test_set_on_writes_correct_bytes(self, mock_serial):
        mock_conn = MagicMock()
        mock_conn.is_open = True
        mock_serial.Serial.return_value = mock_conn

        b = self._make_backend()
        b.set("/dev/ttyUSB0", 1, RelayState.ON)

        mock_conn.write.assert_called_once_with(bytes([0xA0, 0x01, 0x01, 0xA2]))

    @patch("relay_control.backends.serial_tty.serial")
    def test_set_off_writes_correct_bytes(self, mock_serial):
        mock_conn = MagicMock()
        mock_conn.is_open = True
        mock_serial.Serial.return_value = mock_conn

        b = self._make_backend()
        b.set("/dev/ttyUSB0", 1, RelayState.OFF)

        mock_conn.write.assert_called_once_with(bytes([0xA0, 0x01, 0x00, 0xA1]))

    def test_relay_0_raises(self):
        b = self._make_backend()
        with pytest.raises(RelayBackendError, match="out of range"):
            b.set("/dev/ttyUSB0", 0, RelayState.ON)

    def test_relay_too_high_raises(self):
        b = self._make_backend()
        with pytest.raises(RelayBackendError, match="out of range"):
            b.set("/dev/ttyUSB0", 5, RelayState.ON)

    @patch("relay_control.backends.serial_tty.serial")
    def test_serial_exception_wrapped(self, mock_serial):
        mock_conn = MagicMock()
        mock_conn.is_open = True
        mock_conn.write.side_effect = Exception("port gone")
        mock_serial.Serial.return_value = mock_conn

        b = self._make_backend()
        with pytest.raises(RelayBackendError, match="Write failed"):
            b.set("/dev/ttyUSB0", 1, RelayState.ON)


class TestSerialBackendGet:
    @patch("relay_control.backends.serial_tty.serial")
    def test_returns_unknown_before_any_set(self, mock_serial):
        b = SerialBackend(port="/dev/ttyUSB0", relay_count=4)
        assert b.get("/dev/ttyUSB0", 1) == RelayState.UNKNOWN

    @patch("relay_control.backends.serial_tty.serial")
    def test_returns_last_set_state(self, mock_serial):
        mock_conn = MagicMock()
        mock_conn.is_open = True
        mock_serial.Serial.return_value = mock_conn

        b = SerialBackend(port="/dev/ttyUSB0", relay_count=4)
        with patch("relay_control.backends.serial_tty.time"):
            b.set("/dev/ttyUSB0", 2, RelayState.ON)
        assert b.get("/dev/ttyUSB0", 2) == RelayState.ON

    @patch("relay_control.backends.serial_tty.serial")
    def test_get_all_returns_all_states(self, mock_serial):
        mock_conn = MagicMock()
        mock_conn.is_open = True
        mock_serial.Serial.return_value = mock_conn

        b = SerialBackend(port="/dev/ttyUSB0", relay_count=4)
        with patch("relay_control.backends.serial_tty.time"):
            b.set("/dev/ttyUSB0", 1, RelayState.ON)
            b.set("/dev/ttyUSB0", 3, RelayState.OFF)

        states = b.get_all("/dev/ttyUSB0")
        assert states[0] == RelayState.ON
        assert states[1] == RelayState.UNKNOWN
        assert states[2] == RelayState.OFF
        assert states[3] == RelayState.UNKNOWN


class TestSerialBackendPulse:
    @patch("relay_control.backends.serial_tty.serial")
    def test_pulse_sends_on_then_off(self, mock_serial):
        mock_conn = MagicMock()
        mock_conn.is_open = True
        mock_serial.Serial.return_value = mock_conn

        b = SerialBackend(port="/dev/ttyUSB0", relay_count=4)
        with patch("relay_control.backends.serial_tty.time.sleep"):
            b.pulse("/dev/ttyUSB0", 1, 500)

        assert mock_conn.write.call_count == 2
        calls = mock_conn.write.call_args_list
        assert calls[0] == call(bytes([0xA0, 0x01, 0x01, 0xA2]))  # ON
        assert calls[1] == call(bytes([0xA0, 0x01, 0x00, 0xA1]))  # OFF

    @patch("relay_control.backends.serial_tty.serial")
    def test_pulse_sleeps_correct_duration(self, mock_serial):
        mock_conn = MagicMock()
        mock_conn.is_open = True
        mock_serial.Serial.return_value = mock_conn

        b = SerialBackend(port="/dev/ttyUSB0", relay_count=4)
        with patch("relay_control.backends.serial_tty.time.sleep") as mock_sleep:
            b.pulse("/dev/ttyUSB0", 1, 750)

        # sleep is called for the pulse duration (0.75s) plus the two 0.1s write delays
        # The pulse duration sleep is 0.75
        pulse_sleep = [c.args[0] for c in mock_sleep.call_args_list if c.args[0] == pytest.approx(0.75)]
        assert len(pulse_sleep) == 1


class TestSerialBackendClose:
    @patch("relay_control.backends.serial_tty.serial")
    def test_close_closes_connection(self, mock_serial):
        mock_conn = MagicMock()
        mock_conn.is_open = True
        mock_serial.Serial.return_value = mock_conn

        b = SerialBackend(port="/dev/ttyUSB0", relay_count=4)
        with patch("relay_control.backends.serial_tty.time"):
            b.set("/dev/ttyUSB0", 1, RelayState.ON)
        b.close()

        mock_conn.close.assert_called_once()

    def test_close_without_connect_is_safe(self):
        b = SerialBackend(port="/dev/ttyUSB0")
        b.close()  # must not raise


class TestSerialBackendDetect:
    @patch("relay_control.backends.serial_tty.serial.tools.list_ports.comports")
    def test_detect_finds_ch340_by_vendor_id(self, mock_comports):
        port_info = MagicMock()
        port_info.vid = 0x1A86
        port_info.device = "/dev/ttyUSB0"
        mock_comports.return_value = [port_info]

        b = SerialBackend()
        boards = b.detect()

        assert len(boards) == 1
        assert boards[0].id == "/dev/ttyUSB0"
        assert boards[0].backend == "serial"

    @patch("relay_control.backends.serial_tty.serial.tools.list_ports.comports")
    def test_detect_ignores_non_ch340(self, mock_comports):
        port_info = MagicMock()
        port_info.vid = 0x0403  # FTDI, not CH340
        port_info.device = "/dev/ttyUSB0"
        mock_comports.return_value = [port_info]

        b = SerialBackend()
        boards = b.detect()
        assert boards == []

    def test_detect_with_explicit_port_returns_that_port(self):
        b = SerialBackend(port="/dev/ttyUSB0", relay_count=4)
        boards = b.detect()
        assert len(boards) == 1
        assert boards[0].id == "/dev/ttyUSB0"
        assert boards[0].relay_count == 4

    @patch("relay_control.backends.serial_tty.serial.tools.list_ports.comports")
    def test_detect_no_devices_returns_empty(self, mock_comports):
        mock_comports.return_value = []
        b = SerialBackend()
        assert b.detect() == []
```

- [ ] **Step 2: Run to confirm failure**

```bash
uv run pytest tests/unit/backends/test_serial.py -v
```

Expected: `ImportError` — `serial_tty` does not exist yet.

- [ ] **Step 3: Write `src/relay_control/backends/serial_tty.py`**

```python
from __future__ import annotations

import threading
import time
from typing import Optional

from ..core.base import (
    BackendCapabilities,
    BaseRelayBackend,
    RelayBackendError,
    RelayBoard,
    RelayState,
)

try:
    import serial
    import serial.tools.list_ports
except ImportError:
    serial = None  # type: ignore[assignment]

_CH340_VID = 0x1A86


def _make_cmd(relay: int, on: bool) -> bytes:
    state = 1 if on else 0
    checksum = (0xA0 + relay + state) & 0xFF
    return bytes([0xA0, relay, state, checksum])


class SerialBackend(BaseRelayBackend):
    def __init__(
        self,
        port: Optional[str] = None,
        baud: int = 9600,
        relay_count: int = 4,
    ) -> None:
        self._port = port
        self._baud = baud
        self._relay_count = relay_count
        self._conn: Optional[object] = None
        self._lock = threading.Lock()
        self._state: list[RelayState] = [RelayState.UNKNOWN] * relay_count

    def _connect(self) -> object:
        if serial is None:
            raise RelayBackendError(
                "serial backend requires: uv add relay-control[serial]"
            )
        if self._conn is None or not self._conn.is_open:
            try:
                self._conn = serial.Serial(
                    self._port,
                    baudrate=self._baud,
                    timeout=0.05,
                    write_timeout=0.1,
                )
            except Exception as exc:
                raise RelayBackendError(f"Cannot open {self._port}: {exc}") from exc
        return self._conn

    def detect(self) -> list[RelayBoard]:
        if serial is None:
            return []
        if self._port is not None:
            return [
                RelayBoard(
                    id=self._port,
                    backend="serial",
                    relay_count=self._relay_count,
                    state=[RelayState.UNKNOWN] * self._relay_count,
                    meta={"baud": self._baud},
                )
            ]
        boards = []
        for info in serial.tools.list_ports.comports():
            if info.vid == _CH340_VID:
                boards.append(
                    RelayBoard(
                        id=info.device,
                        backend="serial",
                        relay_count=self._relay_count,
                        state=[RelayState.UNKNOWN] * self._relay_count,
                        meta={"baud": self._baud},
                    )
                )
        return boards

    def capabilities(self) -> BackendCapabilities:
        return BackendCapabilities(
            has_state_readback=False,
            has_bulk_write=False,
            has_atomic_pulse=False,
        )

    def set(self, board_id: str, relay: int, state: RelayState) -> None:
        if not 1 <= relay <= self._relay_count:
            raise RelayBackendError(
                f"Relay {relay} out of range 1-{self._relay_count}"
            )
        conn = self._connect()
        cmd = _make_cmd(relay, state == RelayState.ON)
        with self._lock:
            try:
                conn.write(cmd)
                time.sleep(0.1)
            except Exception as exc:
                raise RelayBackendError(f"Write failed: {exc}") from exc
        self._state[relay - 1] = state

    def set_all(self, board_id: str, states: list[RelayState]) -> None:
        for i, state in enumerate(states, start=1):
            self.set(board_id, i, state)

    def get(self, board_id: str, relay: int) -> RelayState:
        return self._state[relay - 1]

    def get_all(self, board_id: str) -> list[RelayState]:
        return list(self._state)

    def pulse(self, board_id: str, relay: int, duration_ms: int) -> None:
        self.set(board_id, relay, RelayState.ON)
        time.sleep(duration_ms / 1000)
        self.set(board_id, relay, RelayState.OFF)

    def close(self) -> None:
        if self._conn is not None and self._conn.is_open:
            with self._lock:
                self._conn.close()
        self._conn = None
```

- [ ] **Step 4: Run tests and confirm they pass**

```bash
uv run pytest tests/unit/backends/test_serial.py -v
```

Expected: all tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/relay_control/backends/serial_tty.py tests/unit/backends/test_serial.py
git commit -m "feat: add SerialBackend for CH340 and generic TTY relay boards"
```

---

## Task 5: RelayRegistry

**Files:**
- Create: `src/relay_control/core/registry.py`
- Create: `tests/unit/test_registry.py`

- [ ] **Step 1: Write failing tests**

```python
# tests/unit/test_registry.py
import pytest
from unittest.mock import MagicMock, patch
from relay_control.core.registry import RelayRegistry
from relay_control.core.base import RelayState, RelayBoard, RelayBackendError
from relay_control.config import GURLConfig, DeviceConfig


def _make_board(id="board1", backend="serial", relay_count=4):
    return RelayBoard(
        id=id,
        backend=backend,
        relay_count=relay_count,
        state=[RelayState.UNKNOWN] * relay_count,
        meta={},
    )


def _registry_with(board: RelayBoard, backend: MagicMock) -> RelayRegistry:
    r = RelayRegistry()
    r._boards.append(board)
    r._backends.append(backend)
    r._board_to_backend[board.id] = backend
    return r


class TestRelayRegistryDelegation:
    def test_set_delegates_to_backend(self):
        board = _make_board()
        backend = MagicMock()
        r = _registry_with(board, backend)
        r.set(board.id, 1, RelayState.ON)
        backend.set.assert_called_once_with(board.id, 1, RelayState.ON)

    def test_get_delegates_to_backend(self):
        board = _make_board()
        backend = MagicMock()
        backend.get.return_value = RelayState.ON
        r = _registry_with(board, backend)
        result = r.get(board.id, 1)
        backend.get.assert_called_once_with(board.id, 1)
        assert result == RelayState.ON

    def test_get_all_delegates_to_backend(self):
        board = _make_board()
        backend = MagicMock()
        backend.get_all.return_value = [RelayState.ON] * 4
        r = _registry_with(board, backend)
        result = r.get_all(board.id)
        assert result == [RelayState.ON] * 4

    def test_set_all_delegates_to_backend(self):
        board = _make_board()
        backend = MagicMock()
        r = _registry_with(board, backend)
        states = [RelayState.ON, RelayState.OFF, RelayState.ON, RelayState.OFF]
        r.set_all(board.id, states)
        backend.set_all.assert_called_once_with(board.id, states)

    def test_pulse_delegates_to_backend(self):
        board = _make_board()
        backend = MagicMock()
        r = _registry_with(board, backend)
        r.pulse(board.id, 2, 500)
        backend.pulse.assert_called_once_with(board.id, 2, 500)

    def test_close_calls_all_backends(self):
        b1, b2 = MagicMock(), MagicMock()
        r = RelayRegistry()
        r._backends = [b1, b2]
        r.close()
        b1.close.assert_called_once()
        b2.close.assert_called_once()


class TestRelayRegistryErrors:
    def test_set_unknown_board_raises_key_error(self):
        r = RelayRegistry()
        with pytest.raises(KeyError, match="not found"):
            r.set("nonexistent", 1, RelayState.ON)

    def test_get_unknown_board_raises_key_error(self):
        r = RelayRegistry()
        with pytest.raises(KeyError, match="not found"):
            r.get("nonexistent", 1)


class TestRelayRegistryFromConfig:
    def test_unknown_backend_raises_value_error(self):
        config = GURLConfig(devices=[DeviceConfig(id="x", backend="bogus")])
        with pytest.raises(ValueError, match="Unknown backend: 'bogus'"):
            RelayRegistry.from_config(config)

    def test_missing_dep_raises_import_error_with_hint(self):
        config = GURLConfig(
            devices=[DeviceConfig(id="x", backend="hid", extras={})]
        )
        with patch(
            "relay_control.core.registry._load_backend_class",
            side_effect=ImportError("no module named hid"),
        ):
            with pytest.raises(ImportError, match=r"relay-control\[hid\]"):
                RelayRegistry.from_config(config)

    def test_from_config_builds_board_from_extras(self):
        config = GURLConfig(
            devices=[
                DeviceConfig(
                    id="basement",
                    backend="serial",
                    extras={"port": "/dev/ttyUSB0", "relay_count": 4},
                )
            ]
        )
        fake_backend = MagicMock()
        fake_backend.detect.return_value = []

        fake_class = MagicMock(return_value=fake_backend)
        with patch(
            "relay_control.core.registry._load_backend_class",
            return_value=fake_class,
        ):
            r = RelayRegistry.from_config(config)

        assert len(r.boards) == 1
        assert r.boards[0].id == "basement"
        assert r.boards[0].relay_count == 4


class TestRelayRegistryBoards:
    def test_boards_property_returns_list(self):
        r = RelayRegistry()
        assert r.boards == []

    def test_boards_returns_added_boards(self):
        board = _make_board()
        backend = MagicMock()
        r = _registry_with(board, backend)
        assert r.boards == [board]
```

- [ ] **Step 2: Run to confirm failure**

```bash
uv run pytest tests/unit/test_registry.py -v
```

Expected: `ImportError` — `relay_control.core.registry` does not exist yet.

- [ ] **Step 3: Write `src/relay_control/core/registry.py`**

```python
from __future__ import annotations

import importlib
from concurrent.futures import ThreadPoolExecutor, as_completed
from typing import TYPE_CHECKING

from .base import BaseRelayBackend, RelayBoard, RelayState

if TYPE_CHECKING:
    from ..config import GURLConfig

_BACKEND_MAP: dict[str, str] = {
    "serial":  "relay_control.backends.serial_tty.SerialBackend",
    "hid":     "relay_control.backends.hid.HIDBackend",
    "modbus":  "relay_control.backends.modbus.ModbusBackend",
    "http":    "relay_control.backends.http.HTTPBackend",
    "mqtt":    "relay_control.backends.mqtt.MQTTBackend",
    "sonoff":  "relay_control.backends.sonoff.SonoffBackend",
    "gpio":    "relay_control.backends.gpio.GPIOBackend",
    "firmata": "relay_control.backends.firmata.FirmataBackend",
    "zigbee":  "relay_control.backends.zigbee.ZigbeeBackend",
    "zwave":   "relay_control.backends.zwave.ZWaveBackend",
}


def _load_backend_class(dotted: str) -> type:
    module_path, class_name = dotted.rsplit(".", 1)
    module = importlib.import_module(module_path)
    return getattr(module, class_name)


class RelayRegistry:
    def __init__(self) -> None:
        self._boards: list[RelayBoard] = []
        self._backends: list[BaseRelayBackend] = []
        self._board_to_backend: dict[str, BaseRelayBackend] = {}

    @property
    def boards(self) -> list[RelayBoard]:
        return self._boards

    @classmethod
    def detect(cls) -> RelayRegistry:
        registry = cls()

        def _try(name: str, dotted: str) -> list[tuple[RelayBoard, BaseRelayBackend]]:
            try:
                klass = _load_backend_class(dotted)
                backend = klass()
                return [(board, backend) for board in backend.detect()]
            except (ImportError, Exception):
                return []

        with ThreadPoolExecutor() as pool:
            futures = {
                pool.submit(_try, name, dotted): name
                for name, dotted in _BACKEND_MAP.items()
            }
            for future in as_completed(futures):
                for board, backend in future.result():
                    registry._boards.append(board)
                    if backend not in registry._backends:
                        registry._backends.append(backend)
                    registry._board_to_backend[board.id] = backend

        return registry

    @classmethod
    def from_config(cls, config: GURLConfig) -> RelayRegistry:
        registry = cls()
        for dev in config.devices:
            dotted = _BACKEND_MAP.get(dev.backend)
            if dotted is None:
                raise ValueError(f"Unknown backend: {dev.backend!r}")
            try:
                klass = _load_backend_class(dotted)
            except ImportError as exc:
                raise ImportError(
                    f"{dev.backend} backend requires: "
                    f"uv add relay-control[{dev.backend}]"
                ) from exc
            backend = klass(**dev.extras)
            relay_count = dev.extras.get("relay_count", 4)
            board = RelayBoard(
                id=dev.id,
                backend=dev.backend,
                relay_count=relay_count,
                state=[RelayState.UNKNOWN] * relay_count,
                meta=dev.extras,
            )
            registry._boards.append(board)
            registry._backends.append(backend)
            registry._board_to_backend[dev.id] = backend
        return registry

    def _backend_for(self, board_id: str) -> BaseRelayBackend:
        if board_id not in self._board_to_backend:
            raise KeyError(f"Board {board_id!r} not found")
        return self._board_to_backend[board_id]

    def set(self, board_id: str, relay: int, state: RelayState) -> None:
        self._backend_for(board_id).set(board_id, relay, state)

    def set_all(self, board_id: str, states: list[RelayState]) -> None:
        self._backend_for(board_id).set_all(board_id, states)

    def get(self, board_id: str, relay: int) -> RelayState:
        return self._backend_for(board_id).get(board_id, relay)

    def get_all(self, board_id: str) -> list[RelayState]:
        return self._backend_for(board_id).get_all(board_id)

    def pulse(self, board_id: str, relay: int, duration_ms: int) -> None:
        self._backend_for(board_id).pulse(board_id, relay, duration_ms)

    def close(self) -> None:
        for backend in self._backends:
            backend.close()
```

- [ ] **Step 4: Update `src/relay_control/__init__.py` to export RelayRegistry**

```python
from .core.base import (
    RelayState,
    RelayBoard,
    BackendCapabilities,
    RelayBackendError,
)
from .core.registry import RelayRegistry

__all__ = [
    "RelayState",
    "RelayBoard",
    "BackendCapabilities",
    "RelayBackendError",
    "RelayRegistry",
]
```

- [ ] **Step 5: Run tests and confirm they pass**

```bash
uv run pytest tests/unit/test_registry.py tests/unit/test_core.py -v
```

Expected: all tests pass.

- [ ] **Step 6: Commit**

```bash
git add src/relay_control/core/registry.py src/relay_control/__init__.py tests/unit/test_registry.py
git commit -m "feat: add RelayRegistry with auto-detect and from_config"
```

---

## Task 6: CLI

**Files:**
- Create: `src/relay_control/cli/main.py`
- Create: `tests/unit/test_cli.py`

- [ ] **Step 1: Write failing tests**

```python
# tests/unit/test_cli.py
import pytest
from typer.testing import CliRunner
from unittest.mock import MagicMock, patch
from relay_control.cli.main import app
from relay_control.core.base import RelayState, RelayBoard

runner = CliRunner()


def _make_board(id="/dev/ttyUSB0", relay_count=4):
    return RelayBoard(
        id=id,
        backend="serial",
        relay_count=relay_count,
        state=[RelayState.OFF] * relay_count,
        meta={},
    )


def _mock_registry(boards=None, get_state=RelayState.OFF, get_all_states=None):
    mock = MagicMock()
    mock.boards = boards or [_make_board()]
    mock.get.return_value = get_state
    mock.get_all.return_value = get_all_states or [RelayState.OFF] * 4
    return mock


class TestDetectCommand:
    def test_no_boards_exits_1(self):
        with patch("relay_control.cli.main.RelayRegistry") as MockReg:
            MockReg.detect.return_value.boards = []
            result = runner.invoke(app, ["detect"])
        assert result.exit_code == 1

    def test_found_board_prints_id(self):
        with patch("relay_control.cli.main.RelayRegistry") as MockReg:
            MockReg.detect.return_value.boards = [_make_board(id="/dev/ttyUSB0")]
            result = runner.invoke(app, ["detect"])
        assert "/dev/ttyUSB0" in result.output
        assert result.exit_code == 0


class TestSetCommand:
    def test_set_on_calls_registry(self):
        with patch("relay_control.cli.main._get_registry") as mock_get:
            mock_reg = _mock_registry()
            mock_get.return_value = mock_reg
            result = runner.invoke(app, ["set", "/dev/ttyUSB0", "1", "on"])
        mock_reg.set.assert_called_once_with("/dev/ttyUSB0", 1, RelayState.ON)
        assert result.exit_code == 0

    def test_set_off_calls_registry(self):
        with patch("relay_control.cli.main._get_registry") as mock_get:
            mock_reg = _mock_registry()
            mock_get.return_value = mock_reg
            result = runner.invoke(app, ["set", "/dev/ttyUSB0", "1", "off"])
        mock_reg.set.assert_called_once_with("/dev/ttyUSB0", 1, RelayState.OFF)
        assert result.exit_code == 0

    def test_invalid_state_exits_2(self):
        with patch("relay_control.cli.main._get_registry") as mock_get:
            mock_get.return_value = _mock_registry()
            result = runner.invoke(app, ["set", "/dev/ttyUSB0", "1", "maybe"])
        assert result.exit_code == 2

    def test_unknown_board_exits_1(self):
        with patch("relay_control.cli.main._get_registry") as mock_get:
            mock_reg = _mock_registry()
            mock_reg.set.side_effect = KeyError("board not found")
            mock_get.return_value = mock_reg
            result = runner.invoke(app, ["set", "unknown", "1", "on"])
        assert result.exit_code == 1

    def test_set_all_boards(self):
        board_a = _make_board(id="a")
        board_b = _make_board(id="b")
        with patch("relay_control.cli.main._get_registry") as mock_get:
            mock_reg = _mock_registry(boards=[board_a, board_b])
            mock_get.return_value = mock_reg
            result = runner.invoke(app, ["set", "all", "1", "on"])
        assert mock_reg.set.call_count == 2


class TestGetCommand:
    def test_get_prints_state(self):
        with patch("relay_control.cli.main._get_registry") as mock_get:
            mock_reg = _mock_registry(get_state=RelayState.ON)
            mock_get.return_value = mock_reg
            result = runner.invoke(app, ["get", "/dev/ttyUSB0", "1"])
        assert "ON" in result.output
        assert result.exit_code == 0


class TestPulseCommand:
    def test_pulse_calls_registry(self):
        with patch("relay_control.cli.main._get_registry") as mock_get:
            mock_reg = _mock_registry()
            mock_get.return_value = mock_reg
            result = runner.invoke(app, ["pulse", "/dev/ttyUSB0", "1", "500"])
        mock_reg.pulse.assert_called_once_with("/dev/ttyUSB0", 1, 500)
        assert result.exit_code == 0


class TestSetAllCommand:
    def test_set_all_relays_on(self):
        board = _make_board(relay_count=4)
        with patch("relay_control.cli.main._get_registry") as mock_get:
            mock_reg = _mock_registry(boards=[board])
            mock_get.return_value = mock_reg
            result = runner.invoke(app, ["set-all", board.id, "on"])
        mock_reg.set_all.assert_called_once_with(
            board.id, [RelayState.ON] * 4
        )
        assert result.exit_code == 0
```

- [ ] **Step 2: Run to confirm failure**

```bash
uv run pytest tests/unit/test_cli.py -v
```

Expected: `ImportError` — `relay_control.cli.main` does not exist yet.

- [ ] **Step 3: Write `src/relay_control/cli/main.py`**

```python
from __future__ import annotations

import shlex
import subprocess
import csv
import time
from pathlib import Path
from typing import Optional

import typer
from rich.console import Console
from rich.table import Table

from ..config import load_config
from ..core.base import RelayState
from ..core.registry import RelayRegistry

app = typer.Typer(name="relay", help="GURL — Grand Unified Relay Logic", no_args_is_help=True)
console = Console()


def _get_registry() -> RelayRegistry:
    config = load_config()
    if config.devices:
        return RelayRegistry.from_config(config)
    return RelayRegistry.detect()


@app.command()
def detect() -> None:
    """Scan all installed backends and print found devices."""
    registry = RelayRegistry.detect()
    if not registry.boards:
        console.print("[yellow]No relay boards found.[/yellow]")
        raise typer.Exit(1)
    for board in registry.boards:
        console.print(
            f"[green]{board.id}[/green]  ({board.backend}, {board.relay_count} relays)"
        )
    registry.close()


@app.command()
def list() -> None:
    """List all configured and detected devices."""
    registry = _get_registry()
    if not registry.boards:
        console.print("[yellow]No boards found.[/yellow]")
        raise typer.Exit(1)
    for board in registry.boards:
        console.print(f"[cyan]{board.id}[/cyan]  ({board.backend}, {board.relay_count} relays)")
    registry.close()


@app.command()
def status() -> None:
    """Print state of all relays on all devices."""
    registry = _get_registry()
    if not registry.boards:
        console.print("[yellow]No boards found.[/yellow]")
        raise typer.Exit(1)
    table = Table(title="Relay Status")
    table.add_column("Board", style="cyan")
    table.add_column("Relay", justify="right")
    table.add_column("State")
    for board in registry.boards:
        states = registry.get_all(board.id)
        for i, state in enumerate(states, start=1):
            color = "green" if state == RelayState.ON else "red" if state == RelayState.OFF else "yellow"
            table.add_row(board.id, str(i), f"[{color}]{state.value}[/{color}]")
    console.print(table)
    registry.close()


@app.command(name="set")
def set_relay(
    board_id: str = typer.Argument(..., help="Board ID or 'all'"),
    relay: int = typer.Argument(..., help="Relay number (1-indexed)"),
    state: str = typer.Argument(..., help="on or off"),
) -> None:
    """Set a single relay ON or OFF."""
    try:
        relay_state = RelayState[state.upper()]
    except KeyError:
        console.print(f"[red]State must be 'on' or 'off', got {state!r}[/red]")
        raise typer.Exit(2)

    registry = _get_registry()
    try:
        if board_id == "all":
            for board in registry.boards:
                registry.set(board.id, relay, relay_state)
        else:
            registry.set(board_id, relay, relay_state)
    except KeyError as exc:
        console.print(f"[red]Board not found: {exc}[/red]")
        raise typer.Exit(1)
    except Exception as exc:
        console.print(f"[red]Error: {exc}[/red]")
        raise typer.Exit(2)
    finally:
        registry.close()


@app.command(name="set-all")
def set_all(
    board_id: str = typer.Argument(..., help="Board ID"),
    state: str = typer.Argument(..., help="on or off"),
) -> None:
    """Set all relays on a board to the same state."""
    try:
        relay_state = RelayState[state.upper()]
    except KeyError:
        console.print(f"[red]State must be 'on' or 'off', got {state!r}[/red]")
        raise typer.Exit(2)

    registry = _get_registry()
    try:
        board = next((b for b in registry.boards if b.id == board_id), None)
        if board is None:
            console.print(f"[red]Board {board_id!r} not found[/red]")
            raise typer.Exit(1)
        registry.set_all(board_id, [relay_state] * board.relay_count)
    except typer.Exit:
        raise
    except Exception as exc:
        console.print(f"[red]Error: {exc}[/red]")
        raise typer.Exit(2)
    finally:
        registry.close()


@app.command()
def get(
    board_id: str = typer.Argument(...),
    relay: int = typer.Argument(...),
) -> None:
    """Get the current state of a single relay."""
    registry = _get_registry()
    try:
        state = registry.get(board_id, relay)
        console.print(state.value)
    except KeyError as exc:
        console.print(f"[red]Board not found: {exc}[/red]")
        raise typer.Exit(1)
    except Exception as exc:
        console.print(f"[red]Error: {exc}[/red]")
        raise typer.Exit(2)
    finally:
        registry.close()


@app.command()
def pulse(
    board_id: str = typer.Argument(...),
    relay: int = typer.Argument(...),
    ms: int = typer.Argument(..., help="Pulse duration in milliseconds"),
) -> None:
    """Momentary relay closure."""
    registry = _get_registry()
    try:
        registry.pulse(board_id, relay, ms)
    except KeyError as exc:
        console.print(f"[red]Board not found: {exc}[/red]")
        raise typer.Exit(1)
    except Exception as exc:
        console.print(f"[red]Error: {exc}[/red]")
        raise typer.Exit(2)
    finally:
        registry.close()


@app.command()
def play(
    file: str = typer.Argument(..., help="CSV playfile (relay_number,duration_s per line)"),
) -> None:
    """Run a CSV relay playfile (backward compatible with original relay_control.py)."""
    registry = _get_registry()
    if not registry.boards:
        console.print("[red]No boards found.[/red]")
        raise typer.Exit(1)
    if len(registry.boards) > 1:
        console.print("[red]Multiple boards found. Specify a board in config.[/red]")
        raise typer.Exit(1)
    board_id = registry.boards[0].id
    try:
        with open(file) as f:
            reader = csv.reader(f)
            for row in reader:
                if not row or str(row[0]).startswith("#"):
                    continue
                relay_num = int(row[0])
                duration_s = float(row[1])
                registry.set(board_id, relay_num, RelayState.ON)
                time.sleep(duration_s)
                registry.set(board_id, relay_num, RelayState.OFF)
    except Exception as exc:
        console.print(f"[red]Error: {exc}[/red]")
        raise typer.Exit(2)
    finally:
        registry.close()


@app.command(name="run")
def run_commands(
    file: str = typer.Argument(..., help="File of shell commands to execute"),
    cycles: str = typer.Option("infinite", help="Number of cycles or 'infinite'"),
) -> None:
    """Run a command file N times or infinitely (backward compatible with run_commands.py)."""
    try:
        max_cycles: float = float("inf") if cycles == "infinite" else int(cycles)
    except ValueError:
        console.print("[red]cycles must be an integer or 'infinite'[/red]")
        raise typer.Exit(2)

    with open(file) as f:
        commands = [
            line.strip()
            for line in f
            if line.strip() and not line.strip().startswith("#")
        ]

    count = 0
    while count < max_cycles:
        for cmd in commands:
            subprocess.run(shlex.split(cmd), check=False)
        count += 1


@app.command(name="config")
def config_cmd(
    action: str = typer.Argument("show", help="show or validate"),
) -> None:
    """Show or validate the current config."""
    cfg = load_config()
    if action == "show":
        import yaml
        data = {
            "devices": [
                {"id": d.id, "backend": d.backend, **d.extras}
                for d in cfg.devices
            ]
        }
        if cfg.mqtt:
            mqtt_dict = {
                "broker": cfg.mqtt.broker,
                "port": cfg.mqtt.port,
                "ha_discovery": cfg.mqtt.ha_discovery,
            }
            if cfg.mqtt.username:
                mqtt_dict["username"] = cfg.mqtt.username
            if cfg.mqtt.password:
                mqtt_dict["password"] = "***"
            data["mqtt"] = mqtt_dict
        console.print(yaml.dump(data, default_flow_style=False))
    elif action == "validate":
        errors = []
        for dev in cfg.devices:
            from ..core.registry import _BACKEND_MAP, _load_backend_class
            if dev.backend not in _BACKEND_MAP:
                errors.append(f"Unknown backend: {dev.backend!r}")
            else:
                try:
                    _load_backend_class(_BACKEND_MAP[dev.backend])
                except ImportError:
                    errors.append(
                        f"Backend {dev.backend!r} not installed: "
                        f"uv add relay-control[{dev.backend}]"
                    )
        if errors:
            for e in errors:
                console.print(f"[red]✗ {e}[/red]")
            raise typer.Exit(1)
        console.print("[green]Config valid.[/green]")
```

- [ ] **Step 4: Run tests and confirm they pass**

```bash
uv run pytest tests/unit/test_cli.py -v
```

Expected: all tests pass.

- [ ] **Step 5: Run the full test suite**

```bash
uv run pytest -v
```

Expected: all tests across all unit test files pass.

- [ ] **Step 6: Commit**

```bash
git add src/relay_control/cli/main.py tests/unit/test_cli.py
git commit -m "feat: add typer CLI with detect, set, get, pulse, play, run"
```

---

## Task 7: Stub Backend Files for Future Plans

Backends referenced in `_BACKEND_MAP` that don't exist yet will cause `ImportError` at detection time. The detection loop swallows those silently (Task 5 handles this), but the files should exist as stubs so linters and future plans can reference them cleanly.

**Files:**
- Create: `src/relay_control/backends/hid.py`
- Create: `src/relay_control/backends/modbus.py`
- Create: `src/relay_control/backends/http.py`
- Create: `src/relay_control/backends/mqtt.py`
- Create: `src/relay_control/backends/sonoff.py`
- Create: `src/relay_control/backends/gpio.py`
- Create: `src/relay_control/backends/firmata.py`
- Create: `src/relay_control/backends/zigbee.py`
- Create: `src/relay_control/backends/zwave.py`

- [ ] **Step 1: Write each stub**

Each stub follows the same pattern. Create all nine files with the appropriate class name:

```python
# src/relay_control/backends/hid.py
from ..core.base import BaseRelayBackend, BackendCapabilities, RelayBoard, RelayState

try:
    import hid as _hid
except ImportError:
    _hid = None  # type: ignore[assignment]


class HIDBackend(BaseRelayBackend):
    """USB HID relay backend. Install: uv add relay-control[hid]"""

    def detect(self) -> list[RelayBoard]:
        raise NotImplementedError("HIDBackend not yet implemented — see Plan 2")

    def capabilities(self) -> BackendCapabilities:
        raise NotImplementedError

    def set(self, board_id: str, relay: int, state: RelayState) -> None:
        raise NotImplementedError

    def set_all(self, board_id: str, states: list[RelayState]) -> None:
        raise NotImplementedError

    def get(self, board_id: str, relay: int) -> RelayState:
        raise NotImplementedError

    def get_all(self, board_id: str) -> list[RelayState]:
        raise NotImplementedError

    def pulse(self, board_id: str, relay: int, duration_ms: int) -> None:
        raise NotImplementedError

    def close(self) -> None:
        pass
```

Repeat for: `ModbusBackend` in `modbus.py`, `HTTPBackend` in `http.py`, `MQTTBackend` in `mqtt.py`, `SonoffBackend` in `sonoff.py`, `GPIOBackend` in `gpio.py`, `FirmataBackend` in `firmata.py`, `ZigbeeBackend` in `zigbee.py`, `ZWaveBackend` in `zwave.py`. Only the class name and install hint change.

- [ ] **Step 2: Run full test suite to confirm nothing broke**

```bash
uv run pytest -v
```

Expected: all tests pass.

- [ ] **Step 3: Commit**

```bash
git add src/relay_control/backends/
git commit -m "chore: add stub backends for Plan 2 implementation"
```

---

## Task 8: Smoke Test with Real Hardware (Optional)

This task requires a CH340 relay board connected via USB. Skip if no hardware available.

- [ ] **Step 1: Detect the board**

```bash
uv run relay detect
```

Expected output (port will vary):
```
/dev/ttyUSB0  (serial, 4 relays)
```

- [ ] **Step 2: Set relay 1 ON**

```bash
uv run relay set /dev/ttyUSB0 1 on
```

Expected: relay 1 clicks on, no error output.

- [ ] **Step 3: Set relay 1 OFF**

```bash
uv run relay set /dev/ttyUSB0 1 off
```

Expected: relay 1 clicks off.

- [ ] **Step 4: Pulse relay 1 for 500ms**

```bash
uv run relay pulse /dev/ttyUSB0 1 500
```

Expected: relay 1 clicks on then off 0.5s later.

- [ ] **Step 5: Run the original playfile**

```bash
uv run relay play playfile.txt
```

Expected: relays toggle per the CSV schedule (same behavior as old `relay_control.py -f playfile.txt`).

- [ ] **Step 6: Mark hardware tests passed and commit if changes were needed**

```bash
git add -p   # only if any fixes were made during hardware testing
git commit -m "fix: hardware smoke test corrections" 2>/dev/null || echo "No fixes needed"
```

---

## Self-Review Checklist

- [x] **Spec coverage:** All spec sections are covered: models (Task 2), config (Task 3), serial backend (Task 4), registry (Task 5), CLI with all commands (Task 6), stub backends (Task 7).
- [x] **No placeholders:** Every task has real code, exact commands, expected output.
- [x] **Type consistency:** `RelayState`, `RelayBoard`, `BackendCapabilities`, `RelayBackendError`, `BaseRelayBackend` — names are consistent across all tasks. `_make_cmd` used in both Task 4 test and implementation. `_BACKEND_MAP` and `_load_backend_class` exposed from registry and imported in CLI `config validate`.
- [x] **Exit codes:** `0` success, `1` device not found, `2` backend error — consistent in CLI (Task 6) and spec (Section 7).
- [x] **Security:** `run_commands` uses `shlex.split(cmd)` with `shell=False` — fixes the original repo's shell injection vulnerability.
- [x] **Backward compat:** `play` and `run` commands preserve original behavior. No shell injection in `run`.
