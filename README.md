# Yacht4SRV — Marine Monitoring System

**Open-source lodní monitoring — motory, navigace, vzdálený přístup**  
**Open-source marine monitoring — engines, navigation, remote access**

---

[🇨🇿 Česky](#-česky) · [🇬🇧 English](#-english)

---

## 🇨🇿 Česky

Yacht4SRV čte data z Yanmar motorů (J1939 CAN bus), převádí je do NMEA2000 pro chartplotter a zveřejňuje veškerá lodní data na lokálním webovém dashboardu. Volitelně odesílá data vzdáleně přes LTE a WireGuard VPN do Grafana dashboardu přístupného odkudkoliv.

### Navigace podle role

| Role | Co potřebujete | Dokumentace |
|------|---------------|------------|
| 🛒 **Zajímám se o koupi** | Co to umí, kompatibilita, varianty, cena | [cs/zakaznik.md](cs/zakaznik.md) |
| 🧭 **Skiper / Host na lodi** | Jak použít internet na lodi, jak sledovat lodní data | [cs/skiper.md](cs/skiper.md) |
| 🔧 **Technik lodi** | Co je nainstalováno, troubleshooting | [cs/technik.md](cs/technik.md) |
| 🚢 **Přehled systému** | Architektura, komponenty, jak to funguje | [cs/system.md](cs/system.md) |
| 📦 **Nasazení na novou loď** | Kompletní instalační průvodce | [cs/nasazeni.md](cs/nasazeni.md) |

---

## 🇬🇧 English

Yacht4SRV reads Yanmar engine data (J1939 CAN bus), converts it to NMEA2000 for the chartplotter, and publishes all vessel data to a local web dashboard. Optionally streams data remotely via LTE + WireGuard VPN to a Grafana dashboard accessible from anywhere.

### Navigate by role

| Role | What you need | Documentation |
|------|--------------|--------------|
| 🛒 **Interested in buying** | What it does, compatibility, variants, pricing | [en/customer.md](en/customer.md) |
| 🧭 **Skipper / Guest on board** | How to use the internet, how to view vessel data | [en/skipper.md](en/skipper.md) |
| 🔧 **Boat technician** | What is installed, troubleshooting guide | [en/technician.md](en/technician.md) |
| 🚢 **System overview** | Architecture, components, how it works together | [en/overview.md](en/overview.md) |
| 📦 **Deploy on a new boat** | Complete installation guide | [en/deployment.md](en/deployment.md) |

---

## System at a glance

```
Yanmar engine ECU        NMEA2000 backbone           Internet
(J1939 CAN bus)                │
     │                         │              LTE + WireGuard VPN
     ▼                         │                      │
 Yacht4CAN ─────────────────── ┤              ┌───────▼───────┐
 (gateway)   PGN 127488/9      │              │ Remote Grafana│
                                │              │   dashboard   │
                        ┌──────┤              └───────────────┘
                        │      │                      ▲
                   B&G Zeus3S  │                      │
                   chartplotter│              ┌───────┴──────┐
                   (engine pg) │              │  Yacht4Mon   │
                               │◄─────────────│  /metrics   │
                               │              └──────────────┘
                        Yacht4Mon ──► Local web dashboard
                        (listen-only)   http://192.168.100.3
```

## Compatibility

| Component | Compatible |
|-----------|-----------|
| Engines | Yanmar 4JH, 3JH, 4LHA (J1939 CAN bus, Deutsch 6-pin DT connector) |
| Chartplotter | B&G Zeus, Garmin GPSMAP, Simrad NSS/NSO (NMEA2000 network) |
| NMEA2000 bus | Standard Micro-C / Simnet / DeviceNet |
| LTE router | Teltonika RUTX11 (recommended) or any router with WireGuard |
| Remote monitoring | Prometheus + Grafana (self-hosted) |

## Reference installation — Lagoon 450 "Soulmate"

This system was developed and runs in production on a Lagoon 450 catamaran with 2× Yanmar 4JH engines and B&G Zeus3S chartplotter.

| Component | Status |
|-----------|--------|
| Yacht4CAN — J1939 → NMEA2000 gateway | ✅ Production (on boat June 2026) |
| Yacht4Mon — NMEA2000 monitor + web | ✅ Production (on boat July 2026) |
| Dual-engine J1939 reception | ✅ Verified on boat |
| NMEA2000 output on Zeus3S | ✅ Verified on boat |
| Remote monitoring via LTE + Grafana | ✅ Running continuously |
| SD card engine data logging | ✅ Verified |

## Hardware

The system uses two custom/off-the-shelf boards:

- **Yacht4CAN** — custom PCB (ESP32-S2 + 4× galvanically isolated MCP2515 CAN)
- **Yacht4Mon** — SH-ESP32 rev2 by [hatlabs.fi](https://hatlabs.fi) (ESP32 + integrated CAN transceiver)
- **RUTX11** — Teltonika commercial LTE router with WireGuard VPN support

If you are interested in the hardware or a pre-built board, open an issue.

---

*Photos: see [img/](img/)*
