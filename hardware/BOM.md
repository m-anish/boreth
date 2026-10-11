# BOM and cost analysis

> **Status: first-pass, parts only.** Prices are Indian retail, single-unit,
> October 2026. Lines marked **⟨est⟩** are engineering estimates with no quote
> behind them yet — chiefly PCBs, power supplies and enclosures. Nothing here
> includes assembly labour, installation, testing or margin.
>
> Volume pricing is not modelled. At 40+ units the display, driver and module
> lines all move materially, and the fab lines move a lot.

## Per dial — 2.3" digits

| Item | Qty | Unit (low) | Unit (high) | Low | High | Source |
|---|---|---|---|---|---|---|
| 2.3" seven-segment, common anode | 6 | ₹60 | ₹100 | ₹360 | ₹600 | [ElectronicsComp ₹85 + GST](https://www.electronicscomp.com/7-segment-display-common-anode-2.3-inch-size); wholesale listings lower |
| TPIC6B595 (SOIC-20) | 6 | ₹45 | ₹53 | ₹270 | ₹318 | [LCSC $0.595](https://www.lcsc.com/product-detail/Registers_Texas-Instruments_C7094.html) |
| Segment resistors + passives | ~60 | — | — | ₹50 | ₹100 | ⟨est⟩ |
| ESP32-C3 module | 1 | ₹150 | ₹289 | ₹150 | ₹289 | [robu C3 board ₹279](https://robu.in/product/esp32-c3-development-board-with-soldering/); bare module lower |
| DS3231SN + CR2032 + holder | 1 | ₹120 | ₹350 | ₹120 | ₹350 | [module ₹235–350](https://www.indiamart.com/superdebug/time-clock.html); bare IC lower |
| AC-DC 12 V / 1.5 A, open frame | 1 | ₹350 | ₹650 | ₹350 | ₹650 | ⟨est⟩ IRM-20-12 class |
| PCBs: 3 digit-pair + 1 control, ≤100×100 mm | 4 | — | — | ₹150 | ₹400 | ⟨est⟩ at qty 5–10 |
| Chain connectors, wire | — | — | — | ₹60 | ₹120 | ⟨est⟩ |
| Ambient light sensor (LDR or BH1750) | 1 | ₹20 | ₹90 | ₹20 | ₹90 | ⟨est⟩ |
| **Electronics subtotal** | | | | **₹1,530** | **₹2,917** | |
| Enclosure, face, frame | 1 | — | — | ₹300 | ₹900 | ⟨est⟩ |
| **Dial total** | | | | **₹1,830** | **₹3,817** | |
| *Optional* SX1262 LoRa module | 1 | ₹320 | ₹850 | +₹320 | +₹850 | [see radio.md](../docs/radio.md) |

## Per dial — 4" digits

Same architecture; three lines change. The display costs more, the supply has to
be bigger, and the PCBs cross the 100 mm fab tier because the 4" package's pin
rows are 107 mm apart ([why](../docs/display-driving.md) §7).

| Item | Qty | Low | High | Source |
|---|---|---|---|---|
| 4" seven-segment, common anode | 6 | ₹630 | ₹1,272 | [IndiaMart ₹105](https://www.indiamart.com/proddetail/4-inch-7-segment-display-20957765497.html) · [Diode House ₹126](https://www.diodehouse.net/segment-dispaly.html) · [ElectronicsComp ₹180+GST](https://www.electronicscomp.com/7-segment-display-common-anode-4-inch) |
| TPIC6B595 | 6 | ₹270 | ₹318 | as above |
| AC-DC 15 V / 2 A, open frame | 1 | ₹500 | ₹900 | ⟨est⟩ IRM-30-15 class |
| PCBs: 6 digit boards @ ~115×95 mm + control | 7 | ₹400 | ₹900 | ⟨est⟩ next fab tier — **get a real quote** |
| Everything else (MCU, RTC, passives, connectors, sensor) | — | ₹400 | ₹949 | as above |
| **Electronics subtotal** | | **₹2,200** | **₹4,339** | |
| Enclosure, face, frame | 1 | ₹400 | ₹1,100 | ⟨est⟩ larger |
| **Dial total** | | **₹2,600** | **₹5,439** | |

A 4" dial is roughly **1.4×** the parts cost of a 2.3" one. The displays are
only part of it; the bigger supply and the larger boards matter just as much.

## Per keeper

| Item | Low | High | Note |
|---|---|---|---|
| ESP32-S3 module | ₹400 | ₹700 | ⟨est⟩ |
| DS3231SN + CR2032 | ₹120 | ₹350 | as above |
| AC-DC 5 V / 1 A | ₹150 | ₹350 | ⟨est⟩ |
| PCB + enclosure | ₹300 | ₹700 | ⟨est⟩ |
| **Base keeper** | **₹970** | **₹2,100** | NTP only, Wi-Fi |
| *+* NEO-6M GNSS + antenna | +₹409 | +₹790 | [NEO-6M ₹259–390](https://robu.in/product/neo-6m-gps-module-with-eprom-normal-quality/) + antenna ⟨est⟩ |
| *+* W5500 Ethernet | +₹250 | +₹500 | ⟨est⟩ — worth it for campus IT |
| *+* SX1262 LoRa | +₹320 | +₹850 | for out-of-building dials |
| **Fully loaded keeper** | **₹1,949** | **₹4,240** | |

The keeper being this cheap is what makes **one keeper per building** the
preferred topology rather than one per campus — it is cheaper than the LoRa
modules it would save, and it removes the long radio hop entirely
([radio.md](../docs/radio.md) §3).

## A worked campus

40 clocks across three buildings: 30 at 2.3", 10 at 4", one keeper per building,
GNSS on one of them, no LoRa (all dials in ESP-NOW range of their building's
keeper). Mid-range unit costs.

| Line | Qty | Each | Total |
|---|---|---|---|
| 2.3" dials | 30 | ₹2,800 | ₹84,000 |
| 4" dials | 10 | ₹4,000 | ₹40,000 |
| Keepers, base | 2 | ₹1,500 | ₹3,000 |
| Keeper with GNSS | 1 | ₹2,400 | ₹2,400 |
| **Parts total** | | | **₹1,29,400** |
| | | **per clock** | **≈ ₹3,235** |

Notes on what that number is not:

- **Parts only.** No assembly, no enclosure finishing, no cabling, no install
  labour, no spares, no margin. A sellable price is a different exercise.
- **Retail, qty 1.** The display and driver lines are the two that fall most
  with volume, and they are ~45 % of a dial.
- **No competitive comparison.** Commercial GPS-synchronised campus clock
  systems are sold per clock, and this project has no verified pricing for them.
  A market survey belongs in `docs/` before any claim about being cheaper —
  see [forsyth's competitive landscape](https://github.com/m-anish/forsyth) for
  the shape that should take.

## Cost levers, in order of size

1. **Display choice.** 2.3" versus 4" is the single biggest fork: ~₹1,200 per
   clock, and it also drags the supply and the fab tier with it.
2. **Radio population.** ₹320–850 per dial, avoidable on every dial inside a
   building with a keeper. Do the site survey before buying radios.
3. **Fab tier.** Keeping the 2.3" pair-boards inside 100 × 100 mm is worth real
   money across 30 clocks; the 4" boards cannot be, so quote them properly.
4. **DS3231 sourcing.** ₹120 as a bare authorised-distributor IC versus ₹350 as a
   marketplace module of uncertain provenance — cheaper *and* more trustworthy.
5. **Enclosure.** Entirely unquoted and plausibly the largest single line once
   somebody actually designs it.
