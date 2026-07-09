# Přehled systému Yacht4SRV

## Co systém dělá

Yacht4SRV propojuje Yanmar motory s NMEA2000 sítí, lokálním webovým dashboardem a vzdáleným monitoringem přes mobilní internet. Skládá se ze tří zařízení:

```
Yanmar motor (J1939)          NMEA2000 síť              Internet
      │                            │
      ▼                            │         LTE + WireGuard VPN
 [Yacht4CAN] ──────────────────────┤─────────────────────────────►
  J1939→NMEA2000                   │                         Grafana
  gateway                          │                         dashboard
                                   │
                             [B&G Zeus3S]
                             zobrazuje
                             data motorů
                                   │
                          [Yacht4Mon] ◄── všechny nástroje:
                          poslouchá        GPS, vítr, hloubka,
                          NMEA2000         náklon, teplota
                               │
                               ▼
                     http://192.168.100.3
                     lokální web dashboard
                     (mobil/tablet na lodi)
```

---

## Komponenty

### Yacht4CAN — J1939 → NMEA2000 gateway

**Hardware:** Vlastní DPS — ESP32-S2 + 4× MCP2515 (galvanicky izolované CAN kanály)

**Co dělá:**
- Čte J1939 rámce z Yanmar ECU (Deutsch 6-pin DT konektor)
- Parsuje PGN: RPM (61444), teplota (65262), olej (65263), palivo (65266), napětí (65271), motohodiny (65253), DM1 chyby (65226)
- Generuje NMEA2000 PGN 127488 (Engine Rapid) a 127489 (Engine Dynamic)
- Vysílá na NMEA2000 backbone → chartplotter zobrazuje data motorů
- Ukládá J1939 data na SD kartu (CSV, rotace po 50 MB, max 100 souborů ≈ 250 h dat)
- Lokální web dashboard na portu 80 (engine data, CAN čítače, SD logy ke stažení)

**CAN kanály:**

| Kanál | Role |
|-------|------|
| CAN2 | J1939 vstup — motor STBD (pravý) |
| CAN3 | J1939 vstup — motor PORT (levý) |
| CAN4 | NMEA2000 výstup → NMEA2000 backbone |

**Konfigurace:** `config.json` na SD kartě (editovatelné bez flashování):
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

**Hardware:** SH-ESP32 rev2 ([hatlabs.fi](https://hatlabs.fi)) — ESP32 s integrovaným SN65HVD230 CAN transceivereem

**Co dělá:**
- Pasivně poslouchá NMEA2000 síť (nevysílá nic — nulový dopad na síť)
- Dekóduje 14 typů PGN: navigace, vítr, hloubka, náklon, motory, baterie, prostředí
- Lokální web dashboard — všechna lodní data na mobilu/tabletu bez internetu
- Prometheus `/metrics` endpoint — vzdálený scraping každých 30 s
- TCP klient na B&G Zeus3S (NMEA 0183 port 10110) — doplňuje data teploty moře a větru
- `/api/eng` endpoint pro display (kompaktní JSON s engine daty)

**Sledované PGN:**

| PGN | Data |
|-----|------|
| 127488, 127489 | Motory — RPM, teplota, olej, palivo, napětí, hodiny |
| 129025, 129026 | Navigace — poloha, SOG, COG |
| 127250, 128259 | Kurz, rychlost vody |
| 128267 | Hloubka |
| 130306 | Vítr (zdánlivý + pravý) |
| 127257 | Náklon a trim |
| 127508, 127506 | Baterie — napětí, proud, SoC |

**Adresy:**
- Web dashboard: `http://192.168.100.3`
- Prometheus: `http://192.168.100.3/metrics`
- Engine API: `http://192.168.100.3/api/eng`

---

### RUTX11 — LTE router

**Hardware:** Teltonika RUTX11 (komerční router pro průmyslové aplikace)

**Co dělá:**
- Lodní WiFi síť pro IoT zařízení (skrytý SSID)
- LTE mobilní internet — dual SIM (SIM1 = host/skiper, SIM2 = záloha majitele)
- WireGuard VPN tunel → vzdálený server (Prometheus + Grafana)
- Automatické přepínání SIM: SIM1 se vloží → systém přepne; SIM1 se vytáhne → záloha SIM2 do 90 s
- MWAN3 failover: SIM1 (priorita 1) → SIM2 (priorita 2)

**Kritické nastavení:** GTK rekey interval = **65535** (ESP32 zařízení se odpojují při nižší hodnotě)

---

## Síťová architektura

```
Lodní LAN 192.168.100.0/24

  192.168.100.1  RUTX11 (gateway + WiFi AP + LTE)
  192.168.100.2  Yacht4CAN (J1939 gateway)
  192.168.100.3  Yacht4Mon (NMEA2000 monitor)

WireGuard VPN tunel (192.168.31.0/24)
  RUTX11 ◄──── LTE ────► VPN server (veřejná IP)
                              │
                         Prometheus (scrape 30s)
                              │
                           Grafana (vzdálený dashboard)
```

---

## Datový tok

```
Yanmar ECU
  ↓ J1939 @ 250 kbps (Deutsch 6-pin DT)
Yacht4CAN CAN2/CAN3
  ↓ parsování PGN 61444/65262/65263/65266/65271/65253/65226
  ↓ NMEA2000 PGN 127488 (rapid, 0.1s) + PGN 127489 (dynamic, 0.5s)
CAN4 → NMEA2000 backbone
  ↓
B&G Zeus3S (engine stránka) + Yacht4Mon (PGN handler)
  ↓
Yacht4Mon web dashboard + /metrics endpoint
  ↓ WireGuard VPN (LTE)
Prometheus scrape @ 30s
  ↓
Grafana dashboard (historická data, grafy, geomap)
```

---

## Stav projektu

| Komponenta | Stav | Ověřeno |
|-----------|------|---------|
| J1939 příjem (1 motor) | ✅ Produkce | na lodi 2026-06 |
| J1939 příjem (2 motory) | ✅ Produkce | na lodi 2026-06 |
| NMEA2000 výstup (Zeus3S) | ✅ Produkce | na lodi 2026-06 |
| SD card logging | ✅ Produkce | na lodi 2026-07 |
| Web dashboard (Yacht4CAN) | ✅ Produkce | na lodi 2026-07 |
| NMEA2000 monitoring (Yacht4Mon) | ✅ Produkce | na lodi 2026-07 |
| Vzdálený Grafana dashboard | ✅ Produkce | ověřeno 2026-07 |
| Prometheus metriky | ✅ Produkce | ověřeno 2026-07 |
| Battery monitoring (PGN 127508/127506) | ⏳ Čeká na ověření | lodní elektrika nebyla při testu zapnutá |

---

*[← Zpět na README](../README.md)*
