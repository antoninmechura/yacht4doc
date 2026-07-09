# Yacht4SRV — System Overview

## What the system does

Yacht4SRV connects Yanmar engines to the NMEA2000 network, a local web dashboard, and optionally to remote monitoring via mobile internet. It consists of three devices:

```
Yanmar engine (J1939)         NMEA2000 network            Internet
      │                            │
      ▼                            │         LTE + WireGuard VPN
 [Yacht4CAN] ──────────────────────┤─────────────────────────────►
  J1939→NMEA2000                   │                         Grafana
  gateway                          │                         dashboard
                                   │
                             [B&G Zeus3S]
                             displays
                             engine data
                                   │
                          [Yacht4Mon] ◄── all instruments:
                          listens          GPS, wind, depth,
                          to NMEA2000      heel, temperature
                               │
                               ▼
                     http://192.168.100.3
                     local web dashboard
                     (phone/tablet on board)
```

---

## Components

### Yacht4CAN — J1939 → NMEA2000 gateway

**Hardware:** Custom PCB — ESP32-S2 + 4× MCP2515 (galvanically isolated CAN channels)

**What it does:**
- Reads J1939 frames from Yanmar ECU (Deutsch 6-pin DT connector)
- Parses PGNs: RPM (61444), temperature (65262), oil (65263), fuel (65266), battery (65271), engine hours (65253), DM1 fault codes (65226)
- Generates NMEA2000 PGN 127488 (Engine Rapid) and 127489 (Engine Dynamic)
- Transmits on NMEA2000 backbone → chartplotter displays engine data
- Logs J1939 data to SD card (CSV, rotate at 50 MB, max 100 files ≈ 250 h of engine data)
- Local web dashboard on port 80 (engine data, CAN counters, SD log download)

**CAN channel assignment:**

| Channel | Role |
|---------|------|
| CAN2 | J1939 input — STBD (starboard) engine |
| CAN3 | J1939 input — PORT (port) engine |
| CAN4 | NMEA2000 output → NMEA2000 backbone |

**Configuration:** `config.json` on SD card (editable without reflashing):
```json
{
  "engines": 2,
  "engine0_can": 2,
  "engine1_can": 3,
  "ip": "192.168.100.2",
  "sd_log": true
}
```

---

### Yacht4Mon — NMEA2000 listen-only monitor

**Hardware:** SH-ESP32 rev2 ([hatlabs.fi](https://hatlabs.fi)) — ESP32 with integrated SN65HVD230 CAN transceiver

**What it does:**
- Passively listens to the NMEA2000 network (transmits nothing — zero impact on the network)
- Decodes 14 PGN types: navigation, wind, depth, heel, engines, batteries, environment
- Local web dashboard — all vessel data on a phone/tablet without internet
- Prometheus `/metrics` endpoint — remote scraping every 30 s
- TCP client for B&G Zeus3S (NMEA 0183 port 10110) — supplements sea temperature and wind data
- `/api/eng` endpoint for display devices (compact JSON with engine data)

**PGNs monitored:**

| PGN | Data |
|-----|------|
| 127488, 127489 | Engines — RPM, temperature, oil, fuel, battery, hours |
| 129025, 129026 | Navigation — position, SOG, COG |
| 127250, 128259 | Heading, water speed |
| 128267 | Depth |
| 130306 | Wind (apparent + true) |
| 127257 | Heel and trim |
| 127508, 127506 | Battery — voltage, current, SoC |

**Addresses:**
- Web dashboard: `http://192.168.100.3`
- Prometheus: `http://192.168.100.3/metrics`
- Engine API: `http://192.168.100.3/api/eng`

---

### RUTX11 — LTE router

**Hardware:** Teltonika RUTX11 (industrial-grade LTE router)

**What it does:**
- Boat WiFi network for IoT devices (hidden SSID)
- LTE mobile internet — dual SIM (SIM1 = guest/skipper, SIM2 = owner fallback)
- WireGuard VPN tunnel → remote server (Prometheus + Grafana)
- Automatic SIM switching: SIM1 inserted → system switches; SIM1 removed → fallback to SIM2 within 90 s
- MWAN3 failover: SIM1 (priority 1) → SIM2 (priority 2)

**Critical setting:** GTK rekey interval = **65535** (ESP32 devices disconnect at lower values)

---

## Network architecture

```
Boat LAN 192.168.100.0/24

  192.168.100.1  RUTX11 (gateway + WiFi AP + LTE)
  192.168.100.2  Yacht4CAN (J1939 gateway)
  192.168.100.3  Yacht4Mon (NMEA2000 monitor)

WireGuard VPN tunnel (192.168.31.0/24)
  RUTX11 ◄──── LTE ────► VPN server (public IP)
                              │
                         Prometheus (scrape 30s)
                              │
                           Grafana (remote dashboard)
```

---

## Data flow

```
Yanmar ECU
  ↓ J1939 @ 250 kbps (Deutsch 6-pin DT)
Yacht4CAN CAN2/CAN3
  ↓ parse PGN 61444/65262/65263/65266/65271/65253/65226
  ↓ encode NMEA2000 PGN 127488 (rapid, 0.1s) + PGN 127489 (dynamic, 0.5s)
CAN4 → NMEA2000 backbone
  ↓
B&G Zeus3S (engine page) + Yacht4Mon (PGN handler)
  ↓
Yacht4Mon web dashboard + /metrics endpoint
  ↓ WireGuard VPN (LTE)
Prometheus scrape @ 30s
  ↓
Grafana dashboard (historical data, charts, geomap)
```

---

## Project status

| Component | Status | Verified |
|-----------|--------|----------|
| J1939 reception (1 engine) | ✅ Production | on boat June 2026 |
| J1939 reception (2 engines) | ✅ Production | on boat June 2026 |
| NMEA2000 output (Zeus3S) | ✅ Production | on boat June 2026 |
| SD card logging | ✅ Production | on boat July 2026 |
| Web dashboard (Yacht4CAN) | ✅ Production | on boat July 2026 |
| NMEA2000 monitoring (Yacht4Mon) | ✅ Production | on boat July 2026 |
| Remote Grafana dashboard | ✅ Production | verified July 2026 |
| Prometheus metrics | ✅ Production | verified July 2026 |
| Battery monitoring (PGN 127508/127506) | ⏳ Pending verification | boat electrics were off during test |

---

*[← Back to README](../README.md)*
