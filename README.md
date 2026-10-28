# SOME/IP Demo on Windows (CAN → SOME/IP Gateway in Python)
A small demo showing how a signal-based **CAN** message (Classic Platform style) is converted into a service-based **SOME/IP** notification (Adaptive Platform style). It runs entirely on **Windows 11 with Python**: no Linux, VM or SocketCAN needed.

## Overview
```
┌──────────────┐   CAN frame      ┌──────────────┐   SOME/IP notification   ┌──────────────┐
│ can_sender   │ ───────────────▶ │   gateway    │ ───────────────────────▶ │  subscriber  │
│ (CP node)    │  python-can      │ decode+map   │   UDP 127.0.0.1:30509    │ (AP service) │
└──────────────┘  udp_multicast   └──────────────┘                          └──────────────┘
```

| Component | Role | Real-world equivalent |
|---|---|---|
| `can_sender.py` | Sends vehicle speed on CAN ID `0x123` every 100 ms | Classic Platform ECU |
| `gateway.py` | Decodes the CAN signal, republishes as SOME/IP event | Gateway ECU |
| `subscriber.py` | Receives the event, prints the decoded value | Adaptive Platform service |
| `someip.py` | Packs/unpacks the 16-byte SOME/IP header | SOME/IP stack (e.g. vsomeip) |

---

## Requirements

### Functional requirements
| ID | Requirement |
|---|---|
| FR-01 | The system shall send a CAN frame (ID `0x123`, 2-byte big-endian raw speed, 0.01 km/h resolution) every 100 ms. |
| FR-02 | The gateway shall receive CAN ID `0x123` and ignore all other IDs. |
| FR-03 | The gateway shall convert the raw value to a 32-bit float (km/h). |
| FR-04 | The gateway shall send a SOME/IP NOTIFICATION (service `0x1234`, event `0x8001`) with an incrementing session ID (1..65535, wrapping). |
| FR-05 | The subscriber shall decode and print service ID, event ID, session ID and speed. |
| FR-06 | The SOME/IP header shall be 16 bytes, big-endian, with the Length field = 8 + payload length. |

### Non-functional / environment requirements
| ID | Requirement |
|---|---|
| NFR-01 | Runs on Windows 11, Python 3.10 or newer. |
| NFR-02 | No admin rights or Linux components needed. |
| NFR-03 | Python packages: `python-can`, `cantools`, `pytest`. |
| NFR-04 | Network: UDP multicast `239.0.0.1` (CAN simulation), UDP `127.0.0.1:30509` (SOME/IP). |
| NFR-05 | Optional: Wireshark + Npcap to inspect SOME/IP packets. |

---

## Project Structure
```
someip-demo/
├── someip.py         # SOME/IP header pack/unpack
├── can_sender.py     # simulated CP node
├── gateway.py        # CAN -> SOME/IP bridge
├── subscriber.py     # simulated AP service
├── test_someip.py    # unit tests
└── README.md
```
