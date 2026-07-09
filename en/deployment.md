# Deployment Guide — Marine monitoring system

Complete guide for installing the Yacht4SRV monitoring system on a new boat.  
Based on the reference installation on Lagoon 450 "Soulmate" (2026).

---

## What the system does

```
Engines (J1939)           NMEA2000 network              Internet (VPN)
Yanmar ECU ──► Yacht4CAN ──► NMEA2000 backbone ◄── Yacht4Mon ──► Prometheus → Grafana
  CAN2/CAN3     gateway        (chartplotter)       monitor       (scrape)      remote
  J1939 250kbps  PGN 127488/9  Zeus/Garmin/...      SH-ESP32      30s interval  dashboard
```

---

## Boat requirements

| Requirement | Required | Notes |
|-------------|---------|-------|
| Yanmar engines with J1939 CAN bus | ✅ | Deutsch 6-pin DT connector on the engine |
| NMEA2000 backbone network | ✅ | Yacht4Mon reads navigation data from it |
| 12V power at installation location | ✅ | Yacht4CAN ~200 mA, Yacht4Mon ~200 mA |
| Chartplotter with NMEA2000 | recommended | Zeus, Garmin, Simrad |
| LTE signal at typical anchorage | recommended | For remote access via VPN |
| Server with Prometheus + Grafana + VPN hub | for remote monitoring | See Part 1 |

---

## Hardware list

### On-board components

| Part | Qty | Notes |
|------|-----|-------|
| **Yacht4CAN board** | 1 | Custom PCB: ESP32-S2 + 4× galvanically isolated MCP2515 |
| **SH-ESP32 rev2** (hatlabs.fi) | 1 | For Yacht4Mon; integrated SN65HVD230 CAN transceiver |
| **RUTX11** (Teltonika) | 1 | LTE router, dual SIM, WireGuard VPN |
| **Data SIM card** | 1 | For RUTX11 SIM2 — owner's backup SIM |
| **SD card 8–32 GB** | 1 | FAT32 MBR format; for J1939 engine logs |
| **120 Ω / 0.25 W resistor** | 3 | CAN termination — one per channel: CAN2, CAN3, CAN4 |
| **NMEA2000 Micro-C connector** (male) | 2 | Yacht4CAN CAN4 output + Yacht4Mon input |
| **Deutsch DT 6-pin** (female) | 1–2 | Depending on engine count; for Y-cable to ECU |
| **CAN cable, twisted pair** | by length | CANH/CANL + GND for each engine |
| **12V → 5V USB power adapter** | 1–2 | From switch panel or 12V distribution |

### Server-side components (once, shared across all boats)

| Part | Notes |
|------|-------|
| VPS or home server | Prometheus + Grafana + WireGuard VPN hub |
| Public IP address | For WireGuard endpoint (UDP port 51820) |

---

## Part 1 — Server preparation (once)

### 1.1 WireGuard hub

Add a new peer for each new boat:

1. VPN → WireGuard → Peers → Add
2. **Name:** boat name (e.g., `soulmate2`)
3. **Public key:** key from RUTX11 (see step 2.3)
4. **Allowed IPs:** `192.168.31.X/32, 192.168.100.0/24`
   - `192.168.31.X` = unique WireGuard IP for the new boat
   - `192.168.100.0/24` = boat LAN (same for every boat — VPN tunnels are isolated)

### 1.2 Prometheus — add scrape job

```yaml
scrape_configs:
  - job_name: 'yacht'
    scrape_interval: 30s
    scrape_timeout: 10s
    static_configs:
      - targets:
          - '192.168.100.3:80'
        labels:
          instance: 'boat_name'
```

```bash
systemctl restart prometheus
```

### 1.3 Grafana dashboard

Import `grafana_dashboard.json` from this repository:  
Dashboards → Import → Upload JSON → set Prometheus datasource.

---

## Part 2 — RUTX11 router

### 2.1 Basic setup

1. Log in to `192.168.1.1` (default IP after reset)
2. **System → Administration** → change `admin` password
3. **Network → LAN** → IP: `192.168.100.1`, mask `255.255.255.0`
4. **Network → Mobile** → insert SIM2, configure APN

### 2.2 WiFi — two separate networks

Configure **two WiFi networks:**

**IoT network for instruments — Network → Wireless → Add:**
- SSID: `RUT_SOUL_2G` (or your own name)
- Hidden SSID: **on** (not visible to users)
- Security: WPA2
- Zone: **lan**

**Guest/skipper network — Network → Wireless → Add:**
- SSID: `RUT_SOUL_5G` (or your own name)
- Hidden SSID: **off** (visible)
- Security: WPA2, shared with skipper
- Zone: **lan**

### 2.3 WireGuard VPN

**VPN → WireGuard → Add instance:**
- Name: `wg_bt`
- Listen port: `51820`
- IP: `192.168.31.X/24` (unique per boat)
- Generate keys → copy **Public key** → send to server admin (step 1.1)

**Peer (server):**
- Public key: VPN server key
- Endpoint: `<server public IP>:51820`
- Allowed IPs: server LAN ranges + VPN tunnel range
- Keepalive: **25 s**

### 2.4 Zeus3S WiFi client (if using B&G chartplotter NMEA 0183 feed)

**Network → Wireless → Add (client):**
- SSID of chartplotter (e.g., `Zeus3S xxxx`)
- **Advanced Settings → Use default gateway: off, metric: 200**
- Firewall zone: **wan**
- **Network → Failover:** WiFi client = **off** (must not be in failover)

### 2.5 SIM switch — guest sharing

**Network → Mobile → SIM Switch:**

| Priority | SIM slot | Condition | Enabled |
|---------|----------|-----------|---------|
| 1 | SIM1 | On SIM not inserted | on |
| 2 | SIM2 | Switch to next SIM after delay | on |

> Edit button is hidden off-screen — scroll the table to the right to find it.

### 2.6 Failover MWAN3

**Network → Failover:**

| Metric | Interface | Enabled |
|--------|-----------|---------|
| 1 | `mob1s1a1` (SIM1) | on |
| 2 | `mob1s2a1` (SIM2) | on |
| others | wan, wifi0 | off |

Tracking: Ping, IPs 1.1.1.1 + 8.8.8.8, Up=3, Down=3, Interval=3s.

### 2.7 GTK rekey — CRITICAL for ESP32 devices

**Network → Wireless → SSID (boat WiFi) → Advanced Settings:**
- **Time interval for rekeying GTK = 65535**

> **Without this setting, ESP32 devices disconnect every ~10 minutes (AUTH_EXPIRE reason 2).**  
> Value must be 1–65535. Value 0 or >65535 breaks the WiFi AP.

### 2.8 Security

- **System → Administration → User Settings:** add user `skiper` / group `user` (read-only)
- **System → Administration → Access Control:** Remote access = **off** for SSH/HTTP/HTTPS

---

## Part 3 — Yacht4CAN

### 3.1 Firmware

```bash
cd src/yacht4can

# Edit config before flashing:
nano data/config.json

# Build + flash firmware
~/.platformio/penv/bin/pio run --target upload

# Flash LittleFS (config.json)
~/.platformio/penv/bin/pio run --target uploadfs
```

### 3.2 config.json

```json
{
  "ssid":        "RUT_SOUL_2G",
  "password":    "WIFI_PASSWORD",
  "ip":          "192.168.100.2",
  "gateway":     "192.168.100.1",
  "subnet":      "255.255.255.0",
  "sd_log":      true,
  "engines":     2,
  "engine0_can": 2,
  "engine1_can": 3
}
```

For single-engine boat: `"engines": 1, "engine0_can": 2`

On first boot with an SD card, config is automatically copied to the card → subsequent changes go to the SD card only (no reflashing needed).

### 3.3 SD card preparation (macOS)

```bash
diskutil list
diskutil eraseDisk FAT32 YACHT4CAN MBRFormat /dev/diskX
```

### 3.4 Physical installation — wiring

```
Yanmar engine ECU (Deutsch 6-pin DT)    Yacht4CAN terminals
  Pin 2 = CANH (yellow) ──────────────── CANH CAN2/CAN3
  Pin 3 = CANL (green)  ──────────────── CANL           ] 120 Ω ← add here
  Pin 6 = GND  (black)  ──────────────── GND

CAN4 output → NMEA2000 Micro-C:
  CANH ──── white wire
  CANL ──── blue wire
  ] 120 Ω add on CAN4 terminals
```

> **CRITICAL:** Each CAN channel requires a 120 Ω termination resistor at the board end.  
> Without it: MCP2515 receives 0 frames even if loopback test passes.

### 3.5 Verification — serial console (115200 baud)

Expected output:
```
[CFG] src=SD  engines=2  CAN=[2,3]  sd_log=on
[E0/CAN2] rx=NNN RPM=800 cool=70C oil=200kPa batt=12.5V
[E1/CAN3] rx=NNN RPM=800 cool=70C oil=200kPa batt=12.5V
[N2K] rapid=NNN dyn=NNN addr=22 TEC=0 txOK=NNN txF=0
```

`txF=0` = no NMEA2000 output errors.

---

## Part 4 — Yacht4Mon

### 4.1 Firmware

```bash
cd src/Yacht4Mon

# Edit config:
nano data/config.json

# Build + flash
~/.platformio/penv/bin/pio run --target upload
~/.platformio/penv/bin/pio run --target uploadfs
```

### 4.2 config.json

```json
{
  "ssid":     "RUT_SOUL_2G",
  "password": "WIFI_PASSWORD",
  "ip":       "192.168.100.3",
  "gateway":  "192.168.100.1",
  "subnet":   "255.255.255.0"
}
```

### 4.3 Physical installation — SH-ESP32 rev2

```
NMEA2000 backbone ─── Micro-C connector ─── SH-ESP32
  12V  ─────────────── red
  GND  ─────────────── black
  CANH ─────────────── yellow (→ SN65HVD230 → GPIO34)
  CANL ─────────────── blue   (→ SN65HVD230 → GPIO32)
```

### 4.4 Verification

Expected serial output:
```
[BOOT] Reset reason: 1 (POWERON)
[WEB] WiFi connected (IP: 192.168.100.3)
[STATUS] t=30s  rx=1500
  nav:  pos=OK  sog/cog=OK  hdg=OK  stw=OK
  wind: app=OK  true=OK
  depth: OK  env=OK  att=OK
```

Web dashboard: `http://192.168.100.3/`  
Prometheus metrics: `http://192.168.100.3/metrics`

---

## Part 5 — Network addresses

Standard assignment (same for every boat — VPN tunnels provide isolation):

| Device | IP |
|--------|----|
| RUTX11 (boat gateway) | `192.168.100.1` |
| Yacht4CAN | `192.168.100.2` |
| Yacht4Mon | `192.168.100.3` |
| RUTX11 WireGuard | `192.168.31.X` (unique per boat) |

---

## Complete installation checklist

### Preparation at home (before going to the boat)

- [ ] Yacht4CAN: firmware flashed
- [ ] Yacht4CAN: LittleFS config flashed (correct SSID/password/IP)
- [ ] Yacht4CAN: SD card formatted FAT32 MBR, inserted
- [ ] Yacht4Mon: firmware flashed
- [ ] Yacht4Mon: LittleFS config flashed
- [ ] RUTX11: basic setup (LAN IP, WiFi, SIM, passwords)
- [ ] RUTX11: WireGuard configured, public key sent to server admin
- [ ] RUTX11: GTK rekey = 65535
- [ ] Server: WireGuard peer added, Prometheus scrape job added
- [ ] Home test: J1939 simulator → Yacht4CAN serial console shows RPM

### On-boat installation

- [ ] Yacht4CAN mounted in dry accessible enclosure
- [ ] CAN2 connected to STBD engine + 120 Ω on terminals
- [ ] CAN3 connected to PORT engine + 120 Ω on terminals
- [ ] CAN4 connected to NMEA2000 backbone + 120 Ω on terminals
- [ ] Yacht4CAN power: 12V + GND, fuse max 1A
- [ ] Yacht4Mon connected to NMEA2000 backbone
- [ ] Yacht4Mon power: 12V via DeviceNet or separate supply + GND
- [ ] RUTX11 mounted, SIM2 inserted, antenna connected, 12V power

### Verification on boat

- [ ] Yacht4CAN serial: `[CFG] src=SD` + both engines have `rx>0`
- [ ] Yacht4CAN web: `http://192.168.100.2` — engine data showing
- [ ] Zeus3S displays engine data
- [ ] Yacht4Mon: `http://192.168.100.3` — nav/wind/depth OK
- [ ] WireGuard VPN: ping to boat IP from remote server succeeds
- [ ] Prometheus: target `UP` in UI
- [ ] Grafana: all panels show data

### During first voyage

- [ ] Both engine RPMs in Grafana match actual engine speed
- [ ] SOG/COG matches GPS on chartplotter
- [ ] WireGuard tunnel holds during LTE handovers
- [ ] SD log on Yacht4CAN growing (`/api/logs`)

---

## Customization for a different boat

| Parameter | Where | Notes |
|-----------|-------|-------|
| WiFi SSID/password | `config.json` on both devices | Match the RUTX11 WiFi network |
| Engine count | Yacht4CAN `config.json`: `engines: 1` or `2` | |
| CAN channel assignment | Yacht4CAN `config.json`: `engine0_can`, `engine1_can` | |
| WireGuard IP | RUTX11 + VPN server peer | Unique per boat |
| Prometheus instance label | `prometheus.yml` | To distinguish boats in Grafana |
| Chartplotter IP/port | Yacht4Mon `nmea_tcp.cpp` constant | Default: `192.168.76.1:10110` (B&G Zeus3S) |

---

## Troubleshooting

### CAN receives 0 frames (rx=0)

1. Check 120 Ω termination at BOTH ends of CAN bus (ECU + board)
2. Check CANH/CANL polarity (swap if rx=0)
3. Measure voltage: CANH ~2.5–3.5V, CANL ~1.5–2.5V on active bus
4. MCP2515 loopback test always passes — not proof of correct external bus wiring

### ESP32 disconnects from WiFi every 10 minutes

Cause: GTK rekey interval on RUTX11  
Fix: RUTX11 → WiFi SSID → Advanced → GTK interval = **65535**

### WireGuard tunnel not working

```bash
# On RUTX11 (SSH):
wg show
# Must show: latest handshake < 2 minutes
```

Most common causes:
- Wrong public key (a new one was generated when configuring the peer)
- Peer missing from WireGuard server instance
- Boat subnet missing from Allowed IPs on server

### Prometheus target DOWN

- Verify VPN ping: `ping 192.168.100.3`
- Verify endpoint: `curl http://192.168.100.3/metrics`
- Check RUTX11 LTE connection

### Grafana "No data" on stat panels

- Do not use `instant: true` in Prometheus queries — bug in Grafana 12
- Use range queries + `lastNotNull` reduce function

---

*[← Back to README](../README.md)*
