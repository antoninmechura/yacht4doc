# Yacht4SRV — For Boat Owners

## Your Yanmar engines. Finally on the chartplotter.

You have Yanmar engines and a B&G, Garmin or Simrad chartplotter — but engine data isn't showing up there?  
Yacht4SRV fixes that. And adds remote monitoring via mobile internet.

---

## What it does

### Engine data on your chartplotter
Yacht4SRV reads data from your Yanmar engines (J1939 CAN bus) and sends it to the NMEA2000 network. Your chartplotter immediately shows:

- **RPM** for both engines
- **Coolant temperature**
- **Oil pressure**
- **Fuel flow** (L/h)
- **Battery voltage**
- **Engine hours**

No proprietary software to install. No subscription. Works with your existing chartplotter.

### Web dashboard on your phone
Connect to the boat WiFi and open a browser. At `http://192.168.100.3` you'll see all vessel data in real time — engines, navigation, wind, depth, heel, batteries. Works offline, no internet needed.

### Remote access (optional, with LTE variant)
See your boat from anywhere in the world. Open the Grafana dashboard and check:
- Whether everything is OK when you're at home
- Historical charts — engine temperatures over an entire season
- Your boat's position on a map

No monthly software subscription — you only pay for a SIM data plan.

---

## Is it for my boat?

### Compatible engines

| Engine | Compatible |
|--------|-----------|
| Yanmar 4JH (series) | ✅ Verified |
| Yanmar 3JH (series) | ✅ |
| Yanmar 4LHA, 4LH (series) | ✅ |
| Any Yanmar with Deutsch 6-pin DT connector on ECU | ✅ |
| Older Yanmar (~pre-2005, no CAN bus) | ❌ |
| Volvo Penta, Mercury, Suzuki | ❌ (not yet) |

### Compatible chartplotters (NMEA2000)

| Chartplotter | Compatible |
|-------------|-----------|
| B&G Zeus3, Zeus3S, Vulcan, Triton | ✅ Verified |
| Garmin GPSMAP 7x2, 9x2, echoMAP | ✅ |
| Simrad NSS, NSO, GO | ✅ |
| Raymarine Axiom, Element | ✅ |
| Any chartplotter with NMEA2000 input | ✅ |

### Boat requirements

- Yanmar engine with J1939 CAN bus (Deutsch 6-pin DT connector)
- NMEA2000 network on board (Simnet/Micro-C backbone)
- 12V power at the installation location
- For remote access: LTE signal in your home marina

---

## Variants

| Model | What's included | Price |
|-------|----------------|-------|
| **Yacht4SRV Base** | Gateway box, SD card, installation cables | ~€315 |
| **Yacht4SRV LTE+** | LTE + required data SIM card for 1 year | ~€555 |
| **Installation** | On-site fitting and commissioning (8 hours) | by arrangement |

> **No annual software fees.** Remote monitoring only requires a data SIM card (~€8–16/month).

---

## Reference installation — Lagoon 450 "Soulmate"

Yacht4SRV has been running in production since June 2026 on a Lagoon 450 catamaran with two Yanmar 4JH engines and a B&G Zeus3S chartplotter.

**What runs every day:**
- RPM, temperature, oil and fuel for both engines on Zeus3S ✅
- Web dashboard on phone while on board ✅
- Remote monitoring via Grafana from home ✅
- Automatic logging of every engine run to SD card ✅

*(Installation photos — see [img/](../img/))*

---

## Frequently asked questions

**Do I need any technical knowledge to install it?**  
No. Installation is carried out by us or your boat service technician. Configuration is changed by editing a simple JSON file on an SD card — no computer, no software.

**What happens if I lose LTE signal at sea?**  
Nothing changes on board. All data works offline — the chartplotter shows engine data, the web dashboard works on the boat WiFi. LTE is only for remote access from home.

**Can I have two engines?**  
Yes. The system supports 1 or 2 Yanmar engines, each on a separate galvanically isolated CAN channel.

**Do I have to pay a subscription?**  
No software subscription. Remote monitoring (Grafana dashboard) requires a data SIM card (~€8–16/month).

**I have a Lagoon 4xx/5xx — will it work?**  
Yes, Yacht4SRV was primarily designed and verified for exactly this type of vessel.

**Is it a certified device?**  
Currently this is a pilot series. The device has been tested and verified on a real boat in real conditions. Formal NMEA2000 certification is planned for the commercial production series.

**What makes it different from YDEG-04?**  
YDEG-04 (€214–249) does the basic J1939→NMEA2000 conversion but has no web dashboard, no remote access, and no data logging. Yacht4SRV adds all of that — for a modest price premium with much more functionality.

---

## Interested in the beta program?

We are looking for the first boat owners for pilot installations.  
**Beta customers get: personal assistance during installation, direct access to the developer, early-adopter pricing.**

📧 Contact: *(email)*  
💬 Telegram: `@mbt674`

---

*Yacht4SRV — built from real need by a Lagoon 450 owner.*

*[← Back to README](../README.md)*
