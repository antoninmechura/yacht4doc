# Technický manuál — Lodní monitoring systém

**Určeno:** Technikům firmy starající se o loď  
**Verze:** 1.0 — červenec 2026

> *Fotky doplní majitel.*

---

## Co je nainstalováno

Na lodi jsou tři přidaná elektronická zařízení, která pracují spolu s existujícím B&G Zeus3S chartplotterem a Yanmar ECU:

- **Yacht4CAN** — převodník J1939 CAN bus motorů do NMEA2000, aby chartplotter zobrazoval data motorů
- **Yacht4Mon** — monitor NMEA2000 sítě; publikuje lodní data na web a vzdáleně přes LTE
- **RUTX11** — LTE router s WiFi pro lodní zařízení a VPN tunel pro vzdálený přístup majitele

---

## Yacht4CAN — Engine gateway

**Funkce:** Čte Yanmar J1939 data (RPM, teplota, olej, palivo, napětí, motohodiny) a převádí je do NMEA2000 pro zobrazení na Zeus3S.

**Umístění:** *(foto placeholder)*

**Připojení:**
- **CAN2** → pravý motor (STBD) — Deutsch 6-pin DT konektor (žlutý=CANH, zelený=CANL)
- **CAN3** → levý motor (PORT) — Deutsch 6-pin DT konektor (žlutý=CANH, zelený=CANL)
- **CAN4** → NMEA2000 backbone — bílý Micro-C konektor do Simnet
- **Napájení:** 12V + GND, max 0,35A
- **USB-C** → pouze pro update firmware (v normálním provozu nepotřeba)

**Indikace:** LED na desce bliká při normálním provozu.

**Logování dat:** každý chod motorů se ukládá na SD kartu jako `/j1939_NNN.csv` (rotace po 50 MB, max 100 souborů ≈ ~250 hodin dat motorů).

> ⚠️ **Neodpojujte CAN kabely při běžících motorech.**

*(Foto: Yacht4CAN deska, CAN konektory, terminační rezistory)*

---

## Yacht4Mon — NMEA2000 monitor

**Funkce:** Pasivně poslouchá NMEA2000 síť a publikuje veškerá lodní data (navigace, vítr, hloubka, motory, náklon) na lokální web stránce a vzdáleně přes LTE do Grafana dashboardu.

**Umístění:** *(foto placeholder)*

**Připojení:**
- **NMEA2000** → backbone přes Micro-C konektor (pouze příjem — nevysílá nic do sítě)
- **Napájení:** 12V přes DeviceNet konektor (červený/černý) nebo samostatný 12V přívod

**Web dashboard (lokálně):** Otevřít v prohlížeči na libovolném zařízení připojeném k lodní WiFi:
```
http://192.168.100.3
```
Zobrazuje: polohu, SOG, COG, kurz, STW, log, vítr (zdánlivý + pravý), hloubku, náklon, trim, teplotu moře, data motorů, baterie. Obnovuje se automaticky každé 2 sekundy.

*(Foto: Yacht4Mon deska, NMEA2000 konektor)*

---

## RUTX11 — LTE router

**Funkce:** Lodní WiFi síť, LTE mobilní internet, VPN tunel k serveru majitele pro vzdálený monitoring.

**Umístění:** *(foto placeholder)*

**SIM karty:**
- **SIM 2** (spodní slot) — datová SIM majitele, vždy přítomna, záložní
- **SIM 1** (horní slot) — SIM skipera/hosta; při vložení se automaticky veškerý provoz přesměruje přes ni; po vytažení systém přepne zpět na SIM 2 do ~90 sekund

**WiFi síť pro lodní zařízení:**
- SSID: `RUT_SOUL_2G` (skrytá — není viditelná při prohledávání WiFi sítí, pouze pro přístroje)
- Heslo: *(dodá majitel)*

**Správa routeru:**
- URL: `http://192.168.100.1` (z lodní LAN)
- Přihlášení admin: *(dodá majitel)*
- Read-only účet pro techniky: uživatel `skiper` *(heslo dodá majitel)*

*(Foto: RUTX11 router, SIM sloty, anténa)*

---

## Přehled sítě

```
NMEA2000 backbone (250 kbps)
  ├── B&G Zeus3S chartplotter  (zobrazuje data motorů z Yacht4CAN)
  ├── Yacht4CAN  (výstup CAN4 → backbone)      IP: 192.168.100.2
  ├── Yacht4Mon  (pouze příjem)                IP: 192.168.100.3
  └── ostatní nástroje (hloubka, vítr, GPS, náklon...)

Lodní WiFi LAN  192.168.100.0/24
  ├── RUTX11 router / gateway                  IP: 192.168.100.1
  ├── Yacht4CAN                                IP: 192.168.100.2
  └── Yacht4Mon                                IP: 192.168.100.3

LTE → WireGuard VPN → Server majitele (vzdálený Grafana dashboard)
```

---

## Přístup k datům

### Z lodi (mobil/tablet připojený k lodní WiFi)

| Co | Adresa | Poznámka |
|----|--------|---------|
| Všechna lodní data | `http://192.168.100.3` | Yacht4Mon dashboard, obnovuje se 2s |
| Data motorů + stav gateway | `http://192.168.100.2` | Yacht4CAN dashboard |
| Stažení logů motorů | `http://192.168.100.2` → záložka Logy | CSV soubory, jeden za jízdu |
| Správa routeru | `http://192.168.100.1` | RUTX11 (read-only: uživatel `skiper`) |

### Vzdáleně (pouze majitel, přes VPN)

Majitel přistupuje ke všem dashboardům odkudkoliv přes VPN. Vzdálený Grafana dashboard zobrazuje historická data s grafy.

---

## Normální provoz — co očekávat

**Při startu motorů:**
- Yacht4CAN detekuje J1939 rámce do ~5 sekund
- Zeus3S stránka motorů zobrazuje živé RPM, teplotu, tlak oleje
- Web dashboard Yacht4Mon zobrazuje data motorů do ~10 sekund
- Logování na SD kartu se spustí automaticky

**Na kotvišti / motory vypnuty:**
- Yacht4Mon nadále zobrazuje navigační data (GPS, vítr, hloubka), pokud jsou NMEA2000 nástroje napájeny
- Vzdálený monitoring pokračuje, pokud má RUTX11 LTE signál

**Restart / odpojení napájení:**
- Yacht4CAN a Yacht4Mon naběhnou za ~10–15 sekund a automaticky se připojí k WiFi
- Po obnovení napájení není potřeba žádný manuální zásah

---

## Přehled indikátorů

| Zařízení | Indikátor | Normální stav | Problém |
|---------|-----------|--------------|---------|
| Yacht4CAN | LED na desce | Bliká | Nebliká = bez napájení nebo crash firmware |
| Yacht4Mon | LED na desce | Bliká | Nebliká = bez napájení nebo crash firmware |
| RUTX11 | Přední LED | Svítí pruhy signálu, WiFi LED svítí | Žádný signál = problém se SIM nebo anténou |
| Dashboard Yacht4Mon | RPM motorů | Zobrazuje hodnotu nebo `---` | `---` = Yacht4CAN neposílá J1939 data |
| Dashboard Yacht4Mon | nav/vítr/hloubka | Zobrazuje hodnoty | `---` = NMEA2000 přístroj neposílá |

---

## Řešení základních problémů

### Zeus3S nezobrazuje data motorů

1. Zkontrolovat, zda Yacht4CAN má napájení (LED bliká)
2. Zkontrolovat CAN2/CAN3 kabely zapojené k motorům — Deutsch konektory musí být pevně zajištěny
3. Zkontrolovat NMEA2000 kabel z CAN4 výstupu Yacht4CAN
4. Otevřít `http://192.168.100.2` — pokud Motor 0/1 ukazuje `rx=0`, kabely jsou pravděpodobně přehozeny nebo chybí terminační rezistory

### Web dashboard zobrazuje `---` u všech hodnot

1. Zkontrolovat napájení Yacht4Mon
2. Zkontrolovat lodní WiFi — jste připojeni k `RUT_SOUL_2G`?
3. Zkusit `http://192.168.100.3` — pokud se stránka nenačte, Yacht4Mon nemá IP (WiFi problém)
4. NMEA2000 síť musí mít napájení, aby se data přístrojů zobrazovala

### Žádné LTE / žádný vzdálený přístup

1. Zkontrolovat, zda je SIM karta zasunuta do slotu SIM 2 v RUTX11
2. Zkontrolovat připojení LTE antény RUTX11
3. Otevřít `http://192.168.100.1` → Síť → Mobilní → zkontrolovat sílu signálu
4. Pokud je vložena SIM 1 (SIM hosta) a nemá signál, vytáhnout ji — systém se vrátí na SIM 2 do 90 s

### Logování motorů — SD karta plná nebo chybí

- SD karta je uvnitř krytu Yacht4CAN
- Formát: FAT32, MBR partition (ne GUID/GPT)
- Staré logy se automaticky mažou při dosažení 100 souborů
- Stažení logů bez vyjmutí SD: `http://192.168.100.2` → záložka Logy → Stáhnout

---

## Nedotýkat se

| Položka | Důvod |
|---------|-------|
| Nastavení GTK rekey na RUTX11 (musí být 65535) | Nižší hodnota způsobuje odpojení ESP32 zařízení každých 10 minut |
| WireGuard VPN konfigurace na RUTX11 | Změna klíčů přeruší vzdálený monitoring |
| Terminační rezistory 120 Ω na CAN svorkách | Bez nich Yacht4CAN nepřijme žádné rámce i při správném zapojení |
| Firmware Yacht4CAN/Yacht4Mon přes USB | Pouze majitel — vyžaduje specifické vývojové prostředí |
| NMEA2000 kabeláž při zapnutých přístrojích | Standardní bezpečnost lodní elektroniky |

---

## Kontakt

Pro technické dotazy k této instalaci:

**Majitel:** *(jméno)*  
**Telegram:** `@__________________`  
**Email:** *(email)*

---

*Tento dokument popisuje vlastní (DIY) instalaci. Nevztahuje se na ni komerční záruka.*

*[← Zpět na README](../README.md)*
