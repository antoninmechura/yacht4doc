# Průvodce nasazení — instalace na novou loď

Kompletní průvodce instalací monitoring systému Yacht4SRV.  
Vychází z referenční instalace na Lagoon 450 „Soulmate" (2026).

---

## Co systém dělá

```
Motory (J1939)            NMEA2000 síť                 Internet (VPN)
Yanmar ECU ──► Yacht4CAN ──► NMEA2000 backbone ◄── Yacht4Mon ──► Prometheus → Grafana
  CAN2/CAN3     gateway        (chartplotter)      monitor       (scrape)      vzdálený
  J1939 250kbps  PGN 127488/9  Zeus/Garmin/...     SH-ESP32      30s interval  dashboard
```

---

## Požadavky na loď

| Požadavek | Povinné | Poznámka |
|-----------|---------|---------|
| Yanmar motory s J1939 CAN bus | ✅ | Deutsch 6-pin DT konektor na motoru |
| NMEA2000 backbone síť | ✅ | Yacht4Mon čte navigační data |
| 12V napájení v místě instalace | ✅ | Yacht4CAN ~200 mA, Yacht4Mon ~200 mA |
| Chartplotter s NMEA2000 sítí | doporučené | Zeus, Garmin, Simrad |
| LTE signál v typickém kotvišti | doporučené | Pro vzdálený přístup |
| Server s Prometheus + Grafana + VPN hub | pro vzdálený monitoring | Viz Část 1 |

---

## Hardware seznam

### Lodní část

| Součástka | Množství | Poznámka |
|-----------|---------|---------|
| **Yacht4CAN deska** | 1 | Custom PCB: ESP32-S2 + 4× MCP2515 galvanicky izolované |
| **SH-ESP32 rev2** (hatlabs.fi) | 1 | Pro Yacht4Mon; integrovaný SN65HVD230 CAN transceiver |
| **RUTX11** (Teltonika) | 1 | LTE router, dual SIM, WireGuard VPN |
| **SIM karta** (datová) | 1 | Do SIM2 RUTX11 — záložní/majitelova |
| **SD karta 8–32 GB** | 1 | FAT32 MBR formát; pro J1939 logy |
| **Rezistor 120 Ω / 0.25 W** | 3 | CAN terminace — CAN2, CAN3, CAN4 (každý kanál zvlášť) |
| **NMEA2000 Micro-C konektor** (male) | 2 | Yacht4CAN CAN4 + Yacht4Mon vstup |
| **Deutsch DT 6-pin** (female) | 1–2 | Dle počtu motorů; Y-kabel k ECU |
| **CAN kabel 2× kroucený pár** | dle délky | CANH/CANL + GND pro každý motor |
| **12V → 5V USB napájecí adaptér** | 1–2 | Ze switchpanelu nebo 12V rozvodu |

### Serverová část (jednou pro všechny lodě)

| Součástka | Poznámka |
|-----------|---------|
| VPS nebo home server | Prometheus + Grafana + WireGuard VPN hub |
| Veřejná IP adresa | Pro WireGuard endpoint (UDP port 51820) |

---

## Část 1 — Serverová příprava (jednou)

### 1.1 WireGuard hub

Přidat nový peer pro každou novou loď:

1. VPN → WireGuard → Peers → Add
2. **Name:** název lodi (např. `soulmate2`)
3. **Public key:** klíč z RUTX11 (viz krok 2.3)
4. **Allowed IPs:** `192.168.31.X/32, 192.168.100.0/24`
   - `192.168.31.X` = unikátní WireGuard IP pro novou loď
   - `192.168.100.0/24` = lodní LAN (stejná pro každou loď — VPN tunely jsou izolované)

### 1.2 Prometheus — přidat scrape job

```yaml
scrape_configs:
  - job_name: 'yacht'
    scrape_interval: 30s
    scrape_timeout: 10s
    static_configs:
      - targets:
          - '192.168.100.3:80'
        labels:
          instance: 'nazev_lode'
```

```bash
systemctl restart prometheus
```

### 1.3 Grafana dashboard

Importovat `grafana_dashboard.json` z tohoto repozitáře:  
Dashboards → Import → Upload JSON → nastavit Prometheus datasource.

---

## Část 2 — RUTX11 router

### 2.1 Základní nastavení

1. Přihlásit se na `192.168.1.1` (výchozí IP po resetu)
2. **System → Administration** → změnit heslo `admin`
3. **Network → LAN** → IP: `192.168.100.1`, maska `255.255.255.0`
4. **Network → Mobile** → SIM2 karta vložena, APN nastaveno

### 2.2 WiFi — síť pro lodní IoT zařízení

**Network → Wireless → Add:**
- SSID: `RUT_SOUL_2G` (nebo vlastní název)
- Skrytý SSID: **on** (nezobrazuje se při prohledávání)
- Zabezpečení: WPA2, silné heslo
- Zóna: **lan**

### 2.3 WireGuard VPN

**VPN → WireGuard → Add instance:**
- Name: `wg_bt`
- Listen port: `51820`
- IP: `192.168.31.X/24` (unikátní pro každou loď)
- Vygenerovat klíče → zkopírovat **Public key** → předat správci serveru (krok 1.1)

**Peer (server):**
- Public key: klíč VPN serveru
- Endpoint: `<veřejná IP serveru>:51820`
- Allowed IPs: rozsahy serverové LAN + VPN tunel
- Keepalive: **25 s**

### 2.4 Připojení k WiFi chartplotteru (pokud B&G Zeus3S)

**Network → Wireless → Add (klient):**
- SSID chartplotteru (např. `Zeus3S xxxx`)
- **Advanced Settings → Use default gateway: off, metric: 200**
- Firewall zóna: **wan**
- **Network → Failover:** WiFi klient = **off** (nesmí být ve failoveru)

### 2.5 SIM switch — sdílení s hosty

**Network → Mobile → SIM Switch:**

| Priorita | SIM slot | Podmínka | Enabled |
|---------|----------|----------|---------|
| 1 | SIM1 | On SIM not inserted | on |
| 2 | SIM2 | Switch to next SIM after delay | on |

> Tlačítko Edit je skryté — nutno scrollovat tabulku doprava.

### 2.6 Failover MWAN3

**Network → Failover:**

| Metrika | Interface | Enabled |
|---------|-----------|---------|
| 1 | `mob1s1a1` (SIM1) | on |
| 2 | `mob1s2a1` (SIM2) | on |
| ostatní | wan, wifi0 | off |

Tracking: Ping, IP 1.1.1.1 + 8.8.8.8, Up=3, Down=3, Interval=3s.

### 2.7 GTK rekey — KRITICKÉ pro ESP32

**Network → Wireless → SSID (lodní WiFi) → Advanced Settings:**
- **Time interval for rekeying GTK = 65535**

> **Bez tohoto nastavení se ESP32 zařízení odpojují každých ~10 minut (AUTH_EXPIRE).**  
> Hodnota musí být 1–65535. Hodnota 0 nebo >65535 rozbije WiFi AP.

### 2.8 Zabezpečení

- **System → Administration → User Settings:** přidat uživatele `skiper` / skupina `user` (read-only)
- **System → Administration → Access Control:** Remote access = **off** pro SSH/HTTP/HTTPS

---

## Část 3 — Yacht4CAN

### 3.1 Firmware

```bash
cd src/yacht4can

# Upravit config před flashováním:
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
  "password":    "WIFI_HESLO",
  "ip":          "192.168.100.2",
  "gateway":     "192.168.100.1",
  "subnet":      "255.255.255.0",
  "sd_log":      true,
  "engines":     2,
  "engine0_can": 2,
  "engine1_can": 3
}
```

Pro jednomotorovou loď: `"engines": 1, "engine0_can": 2`

Po prvním startu s SD kartou se config automaticky zkopíruje na kartu → další změny pouze na SD kartě bez reflashování.

### 3.3 SD karta — příprava (Mac)

```bash
diskutil list
diskutil eraseDisk FAT32 YACHT4CAN MBRFormat /dev/diskX
```

### 3.4 Fyzická instalace — zapojení

```
Motor ECU Yanmar (Deutsch 6-pin DT)    Yacht4CAN svorky
  Pin 2 = CANH (žlutý) ─────────────── CANH CAN2/CAN3
  Pin 3 = CANL (zelený) ────────────── CANL           ] 120 Ω ← přidat na svorky
  Pin 6 = GND (černý) ──────────────── GND

CAN4 výstup → NMEA2000 Micro-C:
  CANH ──── bílý vodič
  CANL ──── modrý vodič
  ] 120 Ω přidat na svorky CAN4
```

> **KRITICKÉ:** Každý CAN kanál potřebuje 120 Ω terminační rezistor na straně desky.  
> Bez rezistoru: MCP2515 přijme 0 rámců i když loopback test projde.

### 3.5 Ověření — sériová konzole (115200 baud)

Očekávaný výstup:
```
[CFG] src=SD  engines=2  CAN=[2,3]  sd_log=on
[E0/CAN2] rx=NNN RPM=800 cool=70C oil=200kPa batt=12.5V
[E1/CAN3] rx=NNN RPM=800 cool=70C oil=200kPa batt=12.5V
[N2K] rapid=NNN dyn=NNN addr=22 TEC=0 txOK=NNN txF=0
```

`txF=0` = žádné chyby výstupu NMEA2000.

---

## Část 4 — Yacht4Mon

### 4.1 Firmware

```bash
cd src/Yacht4Mon

# Upravit config:
nano data/config.json

# Build + flash
~/.platformio/penv/bin/pio run --target upload
~/.platformio/penv/bin/pio run --target uploadfs
```

### 4.2 config.json

```json
{
  "ssid":     "RUT_SOUL_2G",
  "password": "WIFI_HESLO",
  "ip":       "192.168.100.3",
  "gateway":  "192.168.100.1",
  "subnet":   "255.255.255.0"
}
```

### 4.3 Fyzická instalace — SH-ESP32 rev2

```
NMEA2000 backbone ─── Micro-C konektor ─── SH-ESP32
  12V ──────────────── červený
  GND ──────────────── černý
  CANH ─────────────── žlutý (→ SN65HVD230 → GPIO34)
  CANL ─────────────── modrý  (→ SN65HVD230 → GPIO32)
```

### 4.4 Ověření

```
[BOOT] Reset reason: 1 (POWERON)
[WEB] WiFi connected (IP: 192.168.100.3)
[STATUS] t=30s  rx=1500
  nav:  pos=OK  sog/cog=OK  hdg=OK  stw=OK
  wind: app=OK  true=OK
  depth: OK  env=OK  att=OK
```

Web dashboard: `http://192.168.100.3/`  
Prometheus metriky: `http://192.168.100.3/metrics`

---

## Část 5 — Síťové adresy

Standardní přiřazení (stejné pro každou loď — VPN tunely je izolují):

| Zařízení | IP |
|----------|----|
| RUTX11 (lodní GW) | `192.168.100.1` |
| Yacht4CAN | `192.168.100.2` |
| Yacht4Mon | `192.168.100.3` |
| RUTX11 WireGuard | `192.168.31.X` (unikátní pro každou loď) |

---

## Kompletní checklist

### Příprava doma (před odjezdem na loď)

- [ ] Yacht4CAN: firmware naflashován
- [ ] Yacht4CAN: LittleFS config naflashován (správné SSID/heslo/IP)
- [ ] Yacht4CAN: SD karta zformátovaná FAT32 MBR, vložena
- [ ] Yacht4Mon: firmware naflashován
- [ ] Yacht4Mon: LittleFS config naflashován
- [ ] RUTX11: základní nastavení (LAN IP, WiFi, SIM, hesla)
- [ ] RUTX11: WireGuard nakonfigurován, public key předán správci serveru
- [ ] RUTX11: GTK rekey = 65535
- [ ] Server: WireGuard peer přidán, Prometheus scrape job přidán
- [ ] Domácí test: simulátor J1939 → sériová konzole Yacht4CAN ukazuje RPM

### Instalace na lodi

- [ ] Yacht4CAN namontován v suché přístupné skříňce
- [ ] CAN2 zapojen na STBD motor + 120 Ω na svorky
- [ ] CAN3 zapojen na PORT motor + 120 Ω na svorky
- [ ] CAN4 zapojen do NMEA2000 backbone + 120 Ω na svorky
- [ ] Napájení Yacht4CAN: 12V + GND, pojistka max 1A
- [ ] Yacht4Mon zapojen do NMEA2000 backbone
- [ ] Napájení Yacht4Mon: 12V přes DeviceNet nebo samostatný přívod
- [ ] RUTX11 namontován, SIM2 vložena, anténa připojena, napájení 12V

### Ověření na lodi

- [ ] Yacht4CAN sériová konzole: `[CFG] src=SD` + oba motory `rx>0`
- [ ] Yacht4CAN web dashboard: `http://192.168.100.2` — data motorů
- [ ] Zeus3S zobrazuje data motorů
- [ ] Yacht4Mon: `http://192.168.100.3` — nav/vítr/hloubka OK
- [ ] WireGuard VPN: ping na lodní IP z Bartechu prochází
- [ ] Prometheus: target `UP` v UI
- [ ] Grafana: všechny panely zobrazují data

### Za provozu (první plavba)

- [ ] RPM obou motorů v Grafaně odpovídají reálným otáčkám
- [ ] SOG/COG odpovídá GPS na chartplotteru
- [ ] WireGuard tunel drží při LTE přepínání
- [ ] SD log na Yacht4CAN roste (`/api/logs`)

---

## Co přizpůsobit pro jinou loď

| Parametr | Kde | Poznámka |
|----------|-----|---------|
| WiFi SSID/heslo | `config.json` obou zařízení | Dle RUTX11 WiFi |
| Počet motorů | Yacht4CAN `config.json`: `engines: 1` nebo `2` | |
| CAN kanály motorů | Yacht4CAN `config.json`: `engine0_can`, `engine1_can` | |
| WireGuard IP | RUTX11 + VPN server peer | Unikátní pro každou loď |
| Prometheus instance label | `prometheus.yml` | Pro rozlišení lodí v Grafaně |
| Chartplotter IP/port | Yacht4Mon `nmea_tcp.cpp` | Výchozí: `192.168.76.1:10110` (B&G Zeus3S) |

---

## Řešení problémů

### CAN přijímá 0 rámců (rx=0)

1. Zkontrolovat 120 Ω terminaci na OBOU koncích CAN bus (ECU + deska)
2. Zkontrolovat polaritu CANH/CANL (přehodit pokud rx=0)
3. Měřit napětí: CANH ~2.5–3.5V, CANL ~1.5–2.5V při aktivním bus
4. MCP2515 loopback test projde vždy — není důkaz správného zapojení externího bus

### ESP32 se odpojuje od WiFi každých 10 minut

Příčina: GTK rekey interval na RUTX11  
Řešení: RUTX11 → WiFi SSID → Advanced → GTK interval = **65535**

### WireGuard tunel nefunguje

```bash
# Na RUTX11 (SSH):
wg show
# Musí být: latest handshake < 2 minuty
```

Nejčastější příčiny:
- Špatný public key (vygeneroval se nový při konfiguraci peera)
- Chybí peer v server instanci WireGuard
- Chybí Allowed IPs pro lodní subnet na serveru

### Prometheus target DOWN

- Ověřit VPN ping: `ping 192.168.100.3`
- Ověřit endpoint: `curl http://192.168.100.3/metrics`
- Zkontrolovat LTE připojení RUTX11

---

*[← Zpět na README](../README.md)*
