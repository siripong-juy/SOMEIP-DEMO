# Implementation Plan: SOMEIP-DEMO

Build order for the CAN → SOME/IP gateway demo described in [README.md](README.md). Each phase ends with a check, so you always have something working before moving on.

Requirements (FR/NFR), scope limits and future work are defined in the README and are not repeated here.

## Decisions

| Topic | Decision | Effect |
|---|---|---|
| `cantools` / DBC | Dropped for v1 | Decode with `struct` only |
| SOME/IP-SD | On hold | Plain UDP unicast; subscriber is a listener |
| Multiple subscribers | On hold | One subscriber on `127.0.0.1:30509` |
| `msgpack` | Installed upfront | Avoids an import error from python-can `udp_multicast` |
| `requirements.txt` | Added | Reproducible environment |
| Speed codec | Pure `decode_speed` / `encode_speed` functions | FR-02/FR-03 edge cases covered by pytest, not manual checks |

## Overview

| Phase | Deliverable | Covers | Needs network? |
|---|---|---|---|
| 0 | Environment + CAN sim smoke test | NFR-01..04 | CAN sim |
| 1 | `someip.py`, speed codec, `test_someip.py` | FR-02, FR-03, FR-06 | No |
| 2 | `subscriber.py` | FR-05 | UDP loopback |
| 3 | `gateway.py` | FR-02, FR-03, FR-04 | CAN sim + UDP |
| 4 | `can_sender.py` | FR-01 | CAN sim |
| 5 | End-to-end run | All FRs | Yes |
| 6 | Wireshark (optional) | NFR-05 | Yes |
| 7 | Polish | Quality | No |

---

## Phase 0: Environment

- [ ] Create a Python 3.10+ virtual environment
- [ ] Install `python-can`, `msgpack`, `pytest`
- [ ] Freeze to `requirements.txt` (later users install with `pip install -r requirements.txt`, see README)
- [ ] Add `.gitignore`: `.venv/`, `__pycache__/`, `.pytest_cache/`

```powershell
py -3 -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install python-can msgpack pytest
pip freeze > requirements.txt
```

If activation is blocked: `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` (no admin needed).

- [ ] Smoke-test the CAN simulation. `udp_multicast` is experimental and is the riskiest dependency, so confirm it works on this machine before building on it. In two REPLs:

```python
# REPL A (receiver)
import can
bus = can.Bus(interface="udp_multicast", channel="239.0.0.1")
print(bus.recv(timeout=5))

# REPL B (sender)
import can
bus = can.Bus(interface="udp_multicast", channel="239.0.0.1")
bus.send(can.Message(arbitration_id=0x123, data=[0x30, 0x39], is_extended_id=False))
```

**Done when:** `python --version` shows 3.10+, `pytest --version` works inside the venv, and REPL A prints the frame. (Windows Firewall may prompt for `python.exe`; dismissing it is fine.)

## Phase 1: `someip.py` + speed codec + `test_someip.py` (pure logic)

Build this first: it needs no networking and everything else depends on it.

- [ ] Header format `">HHIHHBBBB"` (exactly 16 bytes): Service ID, Method/Event ID, Length, Client ID (`0x0000`), Session ID, Protocol Version (`0x01`), Interface Version (`0x01`), Message Type (`0x02` = NOTIFICATION), Return Code (`0x00`)
- [ ] `pack(service, event, client, session, payload) -> bytes` with Length = 8 + `len(payload)`
- [ ] `unpack(data) -> (fields, payload)` that validates total size and Length
- [ ] Session counter helper that wraps 65535 → 1 (never 0)
- [ ] Speed codec as pure functions (no sockets, no bus):
  - `decode_speed(can_id, data) -> float | None`: returns `None` for an ID other than `0x123` or fewer than 2 data bytes, otherwise `raw * 0.01`
  - `encode_speed(kmh) -> bytes`: `int(round(kmh / 0.01))`, clamped to 0..65535, packed `">H"`

Tests to write:

- [ ] Header size is 16 bytes
- [ ] Length field = 8 + payload length
- [ ] pack/unpack round-trip
- [ ] A known byte string matches expected output
- [ ] Session wraps from 65535 to 1
- [ ] `unpack` rejects short or inconsistent data
- [ ] `decode_speed` ignores a wrong CAN ID and a short frame
- [ ] `decode_speed(0x123, b"\xff\xff")` gives 655.35
- [ ] `encode_speed` clamps negative and too-large values; round-trips with `decode_speed`

**Done when:** `pytest -v` is green.

## Phase 2: `subscriber.py`

- [ ] Bind a UDP socket to `127.0.0.1:30509`
- [ ] `sock.settimeout(1.0)` and loop on `recvfrom`, ignoring `socket.timeout`. On Windows, Ctrl+C does not interrupt a blocking `recvfrom`; the timeout lets `KeyboardInterrupt` through.
- [ ] Call `unpack`, then `struct.unpack(">f", payload)`
- [ ] Print service ID, event ID, session ID and speed (`:.2f`, since float32 otherwise prints noise like `123.44999695`)

**Done when:** a hand-built packet sent from a Python REPL prints correctly.

## Phase 3: `gateway.py`

- [ ] Open `can.Bus(interface="udp_multicast", channel="239.0.0.1")`
- [ ] Loop on `bus.recv(timeout=1.0)` (same Ctrl+C reason as Phase 2); skip `None`
- [ ] `decode_speed(frame.arbitration_id, frame.data)`; skip when it returns `None`
- [ ] `payload = struct.pack(">f", speed_kmh)`
- [ ] Wrap in a SOME/IP NOTIFICATION (service `0x1234`, event `0x8001`) with the session counter, and send via UDP to `127.0.0.1:30509`
- [ ] Use a send-only UDP socket and catch `ConnectionResetError` around `sendto`. On Windows, sending to a port with no listener (subscriber not started yet) can raise WinError 10054.

**Done when:** a CAN frame sent from REPL B of the Phase 0 smoke test (`data=[0x30, 0x39]` → 123.45 km/h) shows up in the subscriber with the right speed.

## Phase 4: `can_sender.py`

- [ ] Send standard 11-bit ID `0x123`, DLC 2, every 100 ms
- [ ] Use a deadline-based loop (`next += 0.1; sleep(next - now)`) so timing does not drift
- [ ] Vary the speed with a ramp or sine wave
- [ ] Encode with `encode_speed`

**Done when:** the gateway logs one frame about every 100 ms.

## Phase 5: End-to-end run

Follow **How to Run** in the README (subscriber, then gateway, then sender).

**Done when:** the subscriber prints a changing speed with an incrementing session ID.

## Phase 6: Wireshark (optional)

- [ ] Follow **Inspecting with Wireshark** in the README
- [ ] Verify every header field against FR-06

## Phase 7: Polish

- [ ] `if __name__ == "__main__"` entry points
- [ ] `logging` instead of bare `print`
- [ ] Clean Ctrl+C handling (close the bus and sockets)
- [ ] Runtime checks not covered by unit tests:
  - [ ] Gateway started before the subscriber does not crash
  - [ ] Session ID wraps correctly in a long run

---

## Requirement Traceability

| Requirement | Verified in |
|---|---|
| FR-01 | Phase 4, Phase 5 |
| FR-02 | Phase 1 (unit test), Phase 3 |
| FR-03 | Phase 1 (unit test, `0xFFFF`), Phase 3 |
| FR-04 | Phase 1 (session wrap), Phase 3, Phase 7 |
| FR-05 | Phase 2, Phase 5 |
| FR-06 | Phase 1, Phase 6 |
| NFR-01..03 | Phase 0 |
| NFR-04 | Phase 0 (multicast), Phases 3–5 (UDP 30509) |
| NFR-05 | Phase 6 |
