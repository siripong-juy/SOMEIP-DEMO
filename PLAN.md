# Implementation Plan: SOMEIP-DEMO

Build order for the CAN → SOME/IP gateway demo described in [README.md](README.md). Each phase ends with a check, so you always have something working before moving on.

## Decisions

| Topic | Decision | Effect |
|---|---|---|
| `cantools` / DBC | Dropped for v1 | Decode with `struct` only |
| SOME/IP-SD | On hold | Plain UDP unicast; subscriber is a listener |
| Multiple subscribers | On hold | One subscriber on `127.0.0.1:30509` |
| `msgpack` | Installed upfront | Avoids an import error from python-can `udp_multicast` |
| `requirements.txt` | Added | Reproducible environment |

## Overview

| Phase | Deliverable | Covers | Needs network? |
|---|---|---|---|
| 0 | Environment | NFR-01..03 | No |
| 1 | `someip.py`, `test_someip.py` | FR-06 | No |
| 2 | `subscriber.py` | FR-05 | UDP loopback |
| 3 | `gateway.py` | FR-02, FR-03, FR-04 | CAN sim + UDP |
| 4 | `can_sender.py` | FR-01 | CAN sim |
| 5 | End-to-end run | All FRs | Yes |
| 6 | Wireshark (optional) | NFR-05 | Yes |
| 7 | Polish | Quality | No |

---

## Phase 0: Environment

- [ ] Create the project folder and a Python 3.10+ virtual environment
- [ ] Install `python-can`, `msgpack`, `pytest`
- [ ] Freeze to `requirements.txt`

```powershell
mkdir someip-demo; cd someip-demo
py -3.10 -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install python-can msgpack pytest
pip freeze > requirements.txt
```

If activation is blocked: `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` (no admin needed).

**Done when:** `python --version` shows 3.10+ and `pytest --version` works inside the venv.

## Phase 1: `someip.py` + `test_someip.py` (pure logic)

Build this first: it needs no networking and everything else depends on it.

- [ ] Header format `">HHIHHBBBB"` (exactly 16 bytes): Service ID, Method/Event ID, Length, Client ID, Session ID, Protocol Version (`0x01`), Interface Version, Message Type (`0x02` = NOTIFICATION), Return Code (`0x00`)
- [ ] `pack(service, event, client, session, payload) -> bytes` with Length = 8 + `len(payload)`
- [ ] `unpack(data) -> (fields, payload)` that validates total size and Length
- [ ] Session counter helper that wraps 65535 → 1 (never 0)

Tests to write:

- [ ] Header size is 16 bytes
- [ ] Length field = 8 + payload length
- [ ] pack/unpack round-trip
- [ ] A known byte string matches expected output
- [ ] Session wraps from 65535 to 1
- [ ] `unpack` rejects short or inconsistent data

**Done when:** `pytest -v` is green.

## Phase 2: `subscriber.py`

- [ ] Bind a UDP socket to `127.0.0.1:30509`
- [ ] Loop on `recvfrom`, call `unpack`, then `struct.unpack(">f", payload)`
- [ ] Print service ID, event ID, session ID and speed (`:.2f`, since float32 otherwise prints noise like `123.44999695`)

**Done when:** a hand-built packet sent from a Python REPL prints correctly.

## Phase 3: `gateway.py`

- [ ] Open `can.Bus(interface="udp_multicast", channel="239.0.0.1")`
- [ ] Loop on `bus.recv()`; skip any ID other than `0x123`
- [ ] Skip frames with fewer than 2 data bytes
- [ ] Decode and re-encode:

```python
raw, = struct.unpack(">H", frame.data[:2])
speed_kmh = raw * 0.01
payload = struct.pack(">f", speed_kmh)
```

- [ ] Wrap in a SOME/IP NOTIFICATION (service `0x1234`, event `0x8001`) with the session counter, and send via UDP to `127.0.0.1:30509`

**Done when:** one manually sent CAN frame shows up in the subscriber with the right speed.

## Phase 4: `can_sender.py`

- [ ] Send ID `0x123` every 100 ms
- [ ] Use a deadline-based loop (`next += 0.1; sleep(next - now)`) so timing does not drift
- [ ] Vary the speed with a ramp or sine wave
- [ ] Encode as `int(round(speed / 0.01))`, clamp to 0..65535, pack with `">H"`

**Done when:** the gateway logs one frame about every 100 ms.

## Phase 5: End-to-end run

Three PowerShell windows, venv activated in each. Start in this order:

1. `python subscriber.py`
2. `python gateway.py`
3. `python can_sender.py`

**Done when:** the subscriber prints a changing speed with an incrementing session ID.

## Phase 6: Wireshark (optional)

- [ ] Install Wireshark + Npcap (installer needs admin; the demo does not)
- [ ] Capture on **Npcap Loopback Adapter**, filter `udp.port == 30509`
- [ ] *Analyze → Decode As… → UDP port 30509 → SOME/IP* (the default dissector port is 30490)
- [ ] Verify header fields against the README

## Phase 7: Polish

- [ ] `if __name__ == "__main__"` entry points
- [ ] `logging` instead of bare `print`
- [ ] Clean Ctrl+C handling (close the bus and sockets)
- [ ] Edge-case checks:
  - [ ] Raw value `0xFFFF` gives 655.35 km/h
  - [ ] A CAN ID other than `0x123` is ignored
  - [ ] Gateway started before the subscriber does not crash
  - [ ] Session ID wraps correctly in a long run

---

## Requirement Traceability

| Requirement | Verified in |
|---|---|
| FR-01 | Phase 4, Phase 5 |
| FR-02 | Phase 3, Phase 7 (wrong ID) |
| FR-03 | Phase 3, Phase 7 (`0xFFFF`) |
| FR-04 | Phase 3, Phase 1 (session wrap) |
| FR-05 | Phase 2, Phase 5 |
| FR-06 | Phase 1, Phase 6 |

## Out of Scope (v1)

- SOME/IP Service Discovery (Offer/Find/Subscribe)
- Multiple subscribers
- DBC / `cantools` decoding
- Real CAN hardware

## Possible Next Steps

1. Decode with a DBC file via `cantools`
2. Minimal SOME/IP-SD
3. Multiple subscribers
4. Request/response method call
