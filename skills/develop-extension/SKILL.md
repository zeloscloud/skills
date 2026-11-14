---
name: develop-extension
description: Build a Zelos extension that reads a device or data feed and streams its decoded signals live into the Zelos app, fast. Use whenever someone wants their own hardware or data in Zelos and no marketplace extension reads it - a custom or proprietary protocol, a serial or USB adapter with its own framing, a TCP or UDP telemetry feed, a lab instrument - or pastes a hand-off prompt from the Zelos assistant, or says "Zelos extension", "get my data into Zelos", "decode this in Zelos".
compatibility: Needs the Zelos app 26.0.10 or newer running on the machine the device is attached to, the zelos CLI, and uv. Windows, macOS, Linux.
---

# Develop a Zelos extension

The person wants to see **their own data moving in Zelos**. Get there in
minutes: scaffold, decode, log, check. This is a proof, so build only what the
first live signal needs. Skip settings-form polish, extra actions, extra
tests, a README. Offer those afterwards (step 4).

Take the connection details, the protocol and the wanted values from the
request. Ask only if you cannot connect or decode without the answer.

## 1. Scaffold

```bash
zelos agent info                               # Status OK means the Zelos app is open. If not, ask the user to open it.
zelos extensions create <name> --with agent    # kebab-case, named for the device: acme-bms
cd <name>
```

This renders a working example, installs its dependencies, and installs it into
the app as `local.<name>`. If the agent is not on `localhost:2300`, set
`ZELOS_AGENT_URL=host:port` for every command. Run Python with `uv run`, never
the system interpreter. On Windows call `uv` and `zelos` directly; the
project's `just` recipes need bash.

## 2. Replace the example with your reader

Four edits. Leave everything else as generated.

1. `<name_with_underscores>/extension.py`: replace the whole file with the skeleton below, then fill in the names, the schema, `decode()` and `_open()`.
2. `<name_with_underscores>/__init__.py` and `main.py` (in the project root): change `SensorMonitor` to `Reader`, three places in all. Edit the files directly; `sed -i` behaves differently across systems. Touch nothing else in `main.py`: its `init(name=...)` must stay as generated.
3. `config.schema.json`: replace the two example properties with what `_open()` reads, each with a `default` set to the user's actual value, and empty the `required` list. The extension must start with no typing.
4. `tests/test_extension.py`: replace with one test that feeds `decode()` a frame whose values you know (the user's sample) and asserts them. Run `uv run pytest -q`. It takes seconds and catches a wrong byte order before anyone looks at a plot.

```python
"""Reads <device> and logs its values to Zelos."""

import logging
import time
from typing import Any

import zelos_sdk
from zelos_sdk import DataType
from zelos_sdk import TraceEventFieldMetadata as Field

logger = logging.getLogger(__name__)


def decode(buf: bytearray) -> list[tuple[str, dict[str, Any]]]:
    """Take whole frames off the front of buf; return (event, fields) for each.

    Leave a partial frame in buf. Skip bytes that are not a frame. Drop a frame
    that fails its checksum, CRC, length or end byte.
    """
    out: list[tuple[str, dict[str, Any]]] = []
    ...
    return out


class Reader:
    def __init__(self, config: dict[str, Any]) -> None:
        self.config = config
        self.running = False
        self.conn = None
        self.bytes = 0
        self.frames = 0
        self.errors = 0
        self.source = zelos_sdk.TraceSourceCacheLast("acme_bms")  # main.py uses .source
        self.source.add_event("pack", [
            Field("voltage", DataType.Float32, "V"),
            Field("state", DataType.UInt8),
        ])
        self.source.add_value_table("pack", "state", {0: "IDLE", 1: "CHARGE", 2: "DISCHARGE"})

    def start(self) -> None:
        self.running = True

    def stop(self) -> None:
        self.running = False

    def run(self) -> None:
        buf = bytearray()
        last_error = None
        while self.running:
            try:
                if self.conn is None:
                    self.conn = self._open()
                    buf.clear()
                    last_error = None
                    logger.info("Connected")
                chunk = self._read()
                self.bytes += len(chunk)
                buf += chunk
            except OSError as e:
                if str(e) != last_error:  # say it once, not every second
                    logger.warning("Not connected: %s", e)
                    last_error = str(e)
                self._close()
                time.sleep(1.0)
                continue
            try:
                for event, fields in decode(buf):
                    getattr(self.source, event).log(**fields)
                    self.frames += 1
            except Exception:  # one bad frame must not end the extension
                self.errors += 1
                if self.errors <= 3:
                    logger.exception("Frame skipped")
                buf.clear()
        self._close()

    def _open(self):
        import serial  # uv add pyserial

        return serial.Serial(self.config["port"], self.config["baudrate"], timeout=0.2)

    def _read(self) -> bytes:
        return self.conn.read(max(1, self.conn.in_waiting))

    def _close(self) -> None:
        if self.conn is not None:
            self.conn.close()
            self.conn = None

    @zelos_sdk.action("Get Status", "Connection state, bytes read, frames decoded", read_only=True)
    def get_status(self) -> dict[str, Any]:
        return {
            "connected": self.conn is not None,
            "bytes": self.bytes,
            "frames": self.frames,
            "errors": self.errors,
        }
```

Keep `get_status`: `main.py` registers the class for actions and fails at start
if it has none. Opening a serial port asserts DTR and RTS, which resets some
boards; if that matters, build `serial.Serial()` unopened, set
`dtr = rts = False`, then `open()`.

**Other transports.** Only `_open()` and `_read()` change, plus `import socket` at the top:

```python
# TCP, device is the server
def _open(self):
    conn = socket.create_connection((self.config["host"], self.config["port"]), timeout=3)
    conn.settimeout(0.5)
    return conn

def _read(self) -> bytes:
    try:
        data = self.conn.recv(4096)
    except TimeoutError:
        return b""                       # quiet, still connected
    if not data:
        raise ConnectionError("closed by device")
    return data

# UDP, device sends to this machine: bind the port it targets
def _open(self):
    conn = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    conn.bind(("0.0.0.0", self.config["port"]))
    conn.settimeout(0.5)
    return conn

def _read(self) -> bytes:
    try:
        return self.conn.recv(65535)
    except TimeoutError:
        return b""                       # quiet, still listening
```

A UDP datagram is one whole packet, so there `decode()` reads one packet from
`buf` and ends with `buf.clear()`. No scanning or resynchronising is needed.

**Decoding.**

- Find the frame first: a line ending, a start byte and fixed length, or a header with a length field. A byte stream can start mid-frame, so scan for the start, and if the frame does not check out, drop one byte and scan again.
- Use `struct` with an explicit byte order (`<` little, `>` big). Wrong byte order gives values that look almost right, which is what the test in edit 4 is for.
- Scale to engineering units in `decode()`: a field named `voltage` with unit `V` reads volts.
- One flag per `Boolean` field, not a `flags` integer. A named state gets a number and a value table.

**Naming.** Signals appear in the app as `source/event.field`.

- Source: the device (`acme_bms`). Event: one message it sends (`pack`). Field: one value in it (`voltage`).
- Lowercase with underscores. No dots, slashes or spaces. No units in names; the unit has its own argument.
- Values that arrive together go in one event, so they share a timestamp.
- Several identical devices on one link: one event each, named from the id (`motor1_feedback`, `motor2_feedback`), same fields. The request may name one and the bus carry more: declare an event the first time an id appears, and say what you found.
- Types: `Float32` for scaled values, `Float64` for latitude, longitude and counters, the device's integer type for raw counts, `Boolean` for flags.

**Device time.** `log()` stamps the sample on arrival, which is right unless the
user asked to keep the device's timing. Then have `decode()` return the device
time as a third item and add one fixed offset to it. Take the offset from the
first sample and take it again only when it stops matching the wall clock,
which is what a reboot or a wrapped counter looks like (`self.offset = None` in
`__init__`):

```python
for event, fields, device_ns in decode(buf):
    now = time.time_ns()
    if self.offset is None or abs(self.offset + device_ns - now) > 1_000_000_000:
        self.offset = now - device_ns
    getattr(self.source, event).log_at(self.offset + device_ns, **fields)
```

A sample stamped in the future is invisible in a live view, so a wrong offset
looks exactly like "no data".

## 3. Run and check

```bash
uv add pyserial && zelos extensions reinstall local.<name>   # before the first start, if the code imports a new package
zelos extensions start local.<name>
zelos extensions list                              # STATE must be running
zelos live signals                                 # your source, events, fields
zelos live query -s '<source>/*.*' --last 5s       # run it twice
```

Done means all three:

- STATE is `running`, and still is ten seconds later.
- Every value the user asked for is listed, with its unit.
- The values are right: they change between two queries where they should, and one matches a value you know (the user's sample, the device's display).

Check against the user's device only. Never send data of your own into the
running extension, not even the user's sample to "test the path": it lands in
their signals as if it were real and can skew what the extension learned from
the stream. The sample belongs in the test from edit 4.

When something is off, ask the extension first:

```bash
zelos actions execute "<Name>/Reader/get_status"    # <Name> is `name` in extension.toml, e.g. "Acme Bms"
```

`bytes` 0 means nothing is arriving. `bytes` rising with `frames` 0 means
`decode()` rejects everything. `errors` rising means `decode()` or a `log()`
call raises, and the log has the traceback: often an id or event you did not
declare. `frames` rising with nothing in `live query` means samples are logged
with the wrong time.

What trips people:

| You see | Cause | Do |
|---------|-------|----|
| `start` succeeded but STATE is `installed` | The process died right after launch. `start` does not report that. | `zelos extensions logs local.<name> --tail 3000` (bytes) and read the end; the log keeps old runs. |
| `ModuleNotFoundError` | A dependency was added without `reinstall`. | `zelos extensions reinstall local.<name>`, then `start`. |
| `No @action decorated methods found` | The class has no action. | Keep `get_status`. |
| Your edit changed nothing | The process was not restarted. | `zelos extensions stop local.<name>`, then `start`. No reinstall for code edits. |
| Running, no signals | Signals appear with the first logged sample. After 15 s with none: not connected, or `decode()` rejects everything. | `get_status`, then the log. Then look at the raw bytes (below). |
| Port busy, or half the bytes | Another program has the port: the Zelos Serial extension, a vendor tool, a capture script you left running, your own extension. | Close it. A serial port has one owner. |
| A value-table field prints `2` | The CLI prints the number. | Nothing. The app shows the name. |

To look at raw bytes, stop the extension and read the port for three seconds
with the same `_open()` settings, then print `data[:256].hex(" ")` beside the
protocol description. Garbage with no repeating structure means a wrong baud
rate. Nothing at all means a wrong port or a device that is not sending.

## 4. Hand back

Tell the user, briefly: the extension id, the signals now live, the value you
checked them against, anything you assumed, and anything on the wire you did
not decode (other ids, other message types). Then give them this to paste into
the Zelos assistant, which can see the new signals:

> My `<name>` extension is running and streaming `<source>` signals. Build a
> layout that shows them.

Then offer follow-ups, and do none of them unasked: more signals or devices, a
settings form with descriptions, actions to read or write values, tests, a
README, publishing to the marketplace. The guide for all of it is
<https://docs.zeloscloud.io/latest/sdk/how-to/develop-agent-extensions/>, and
the published extensions are the model to copy
(<https://github.com/zeloscloud/zelos-extension-modbus>).
