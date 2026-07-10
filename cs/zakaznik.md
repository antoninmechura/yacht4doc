# Yacht4SRV — Pro majitele lodi

## Vaše Yanmar motory konečně na chartplotteru

Máte Yanmar motory a B&G, Garmin nebo Simrad chartplotter, ale data motorů se tam nezobrazují?  
Yacht4SRV to vyřeší — a přidá vzdálený monitoring přes mobilní internet.

---

## Co to dělá

### Motory na chartplotteru
Yacht4SRV čte data z Yanmaru (J1939 CAN bus) a posílá je do NMEA2000 sítě. Váš chartplotter okamžitě zobrazí:

- **RPM** obou motorů
- **Teplotu** chladicí kapaliny
- **Tlak oleje**
- **Průtok paliva** (L/h)
- **Napětí** palubní sítě
- **Motohodiny**

Žádná instalace proprietary softwaru. Žádné předplatné. Funguje s vaším stávajícím chartplotterem.

### Web dashboard na mobilu
Připojte se k palubní WiFi a otevřete prohlížeč. Na `http://192.168.100.3` uvidíte všechna lodní data v reálném čase — motory, navigaci, vítr, hloubku, náklon, baterie. Funguje offline, bez internetu.

### Vzdálený přístup (volitelné, s LTE variantou)
Vidíte loď odkudkoliv na světě. Otevřete Grafana dashboard a sledujte:
- Jestli je loď v pořádku když sedíte doma
- Historické grafy — teploty motorů za celou sezónu
- Polohu lodi na mapě

Bez měsíčního předplatného za software — platíte jen za SIM kartu.

---

## Je to pro moji loď?

### Kompatibilní motory

| Motor | Kompatibilní |
|-------|-------------|
| Yanmar 4JH (series) | ✅ Ověřeno |
| Yanmar 3JH (series) | ✅ |
| Yanmar 4LHA, 4LH (series) | ✅ |
| Yanmar s Deutsch 6-pin DT konektorem na ECU | ✅ |
| Yanmar starší generace (před ~2005, bez CAN bus) | ❌ |
| Volvo Penta, Mercury, Suzuki | ❌ (zatím) |

### Kompatibilní chartplottery (NMEA2000)

| Chartplotter | Kompatibilní |
|-------------|-------------|
| B&G Zeus3, Zeus3S, Vulcan, Triton | ✅ Ověřeno |
| Garmin GPSMAP 7x2, 9x2, echoMAP | ✅ |
| Simrad NSS, NSO, GO | ✅ |
| Raymarine Axiom, Element | ✅ |
| Jakýkoliv chartplotter s NMEA2000 vstupem | ✅ |

### Požadavky na loď

- Yanmar motor s J1939 CAN bus (Deutsch 6-pin DT konektor)
- NMEA2000 síť na lodi (Simnet/Micro-C backbone)
- 12V napájení v místě instalace
- Pro vzdálený přístup: LTE signál v přístavišti

---

## Varianty

| Model | Co obsahuje | Cena |
|-------|------------|------|
| **Yacht4SRV Base** | Gateway box, SD karta, instalační kabeláž | 7 900 Kč (~315 €) |
| **Yacht4SRV LTE+** | LTE + nutná datová SIM karta na 1 rok | 13 900 Kč (~555 €) |
| **Instalace** | Montáž + uvedení do provozu (8 hodin) | dle lokality |

> **Žádné roční poplatky za software.** Vzdálený monitoring vyžaduje pouze datovou SIM kartu (~200–400 Kč/měsíc).

---

## Referenční instalace — Lagoon 450 „Soulmate"

Yacht4SRV běží v produkci od června 2026 na katamaránu Lagoon 450 se dvěma motory Yanmar 4JH a B&G Zeus3S chartplotterem.

**Co funguje každý den:**
- RPM, teplota, olej a palivo obou motorů na Zeus3S ✅
- Web dashboard dostupný z mobilu na palubě ✅
- Vzdálený monitoring přes Grafana z domova ✅
- Automatické logování každé jízdy na SD kartu ✅

*(Fotodokumentace instalace — viz [img/](../img/))*

---

## Nejčastější otázky

**Potřebuji programátorské znalosti k instalaci?**  
Ne. Instalaci provádíme my nebo váš lodní technik. Konfigurace se mění editací jednoduchého JSON souboru na SD kartě — bez počítače, bez softwaru.

**Co se stane když přijdu o LTE signál na moři?**  
Nic. Všechna data fungují offline — chartplotter zobrazuje motory, web dashboard funguje na palubní WiFi. LTE je pouze pro vzdálený přístup z domova.

**Mohu mít dva motory?**  
Ano. Systém podporuje 1 nebo 2 Yanmar motory, každý na samostatném galvanicky izolovaném CAN kanálu.

**Musím platit předplatné?**  
Ne za software. Vzdálený monitoring (Grafana dashboard) vyžaduje vlastní server nebo VPS (~100–300 Kč/měsíc) a datovou SIM kartu.  

**Mám Lagoon /4xx/5xx — funguje to?**  
Ano, přesně pro tento typ lodí byl Yacht4SRV primárně navržen a ověřen.

**Je to certifikované zařízení?**  
Aktuálně se jedná o pilotní sérii. Zařízení je testováno a ověřeno na reálné lodi. Formální NMEA2000 certifikace je plánována pro komerční sérii.

---

## Zájem o beta program?

Hledáme první majitele lodí pro pilotní instalace.  
**Beta zákazníci získávají: osobní asistenci při instalaci, přímou komunikaci s vývojářem, zvýhodněnou cenu.**

📧 Kontakt: *(email majitele)*  
💬 Telegram: `@mbt674`

---

*Yacht4SRV — vyvinuto na základě reálné potřeby majitele Lagoon 450.*

*[← Zpět na README](../README.md)*
