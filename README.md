# boreth

![status](https://img.shields.io/badge/status-keeping_vigil-ff2d16)
![dials](https://img.shields.io/badge/dials_installed-000-lightgrey)
![display](https://img.shields.io/badge/display-6%C3%97_seven--segment-c8a44a)
![drive](https://img.shields.io/badge/drive-static%2C_cascaded_shift_registers-blue)
![driver](https://img.shields.io/badge/driver_IC-provisional-lightgrey)
![band](https://img.shields.io/badge/radio-865--867_MHz_(IN)-ff2d16)
![holdover](https://img.shields.io/badge/holdover-TCXO_RTC_in_every_dial-forestgreen)
![license](https://img.shields.io/badge/license-MIT-blue)

*Monks have kept the hour for centuries. They would rather not be asked.*

**Dial 000 · Keeping vigil** — [boreth.starstucklab.com](https://boreth.starstucklab.com)

Boreth is a **synchronised clock system for campuses**: a wall of six-digit
seven-segment **dials** that all agree with each other, and one **keeper** that
knows what time it actually is. The dials show `HH:MM:SS` at 2.3" or 4" digit
height. The keeper takes its time from GNSS or from NPL India's stratum-1 NTP
servers, holds it in a temperature-compensated oscillator, and broadcasts it
over a private sub-GHz link so no clock ever depends on campus Wi-Fi, campus IT,
or campus electricity staying up.

Every dial keeps its own TCXO-backed real-time clock on a coin cell. Lose the
keeper, lose the radio, lose the mains — the dial still knows the hour, and says
so when the power comes back. **Battery backup is optional on every unit**,
dial and keeper alike, for sites that want the display to stay lit through the
cut rather than merely wake up correct.

A [starstucklab](https://starstucklab.com) project.

## What it does

- **Dials**: six seven-segment digits plus a colon, driven **statically** — every
  segment lit continuously, no multiplexing, so the display is full-brightness,
  flicker-free, and photographs cleanly. One 8-channel high-voltage shift
  register per digit, cascaded, so the whole display costs the controller
  **four GPIO pins** no matter how many digits it has.
- **Keeper**: one per building or per campus. GNSS PPS → NTP → disciplined RTC →
  manual, in that order of trust, with multiple NTP servers cross-checked against
  each other so one wrong answer can't move a campus.
- **Holdover, not hope**: the keeper learns each dial's crystal error and tells
  the dial about it, so a dial that loses contact drifts on a corrected rate
  rather than a nominal one. A week of silence costs about a second.
- **A time bus, not a radio religion**: ESP-NOW is peer-to-peer and already in
  the MCU, so it is free and has no peer limit when broadcast. The **keeper
  carries both radios**; a **dial carries ESP-NOW always and LoRa optionally**,
  on the same board, populated to suit the site survey. LoRa at 865–867 MHz
  reaches the gatehouse, the field and the unwired block.
- **Admin from a phone**: SoftAP captive portal for commissioning, then a small
  local PWA over mDNS. No app, no cloud dependency, no port forwarding.

## The decisions, and why

Four questions settled before any board gets laid out. Each one has a document
because each one has a wrong answer that looks right.

| Document | Settles |
|---|---|
| [`docs/architecture.md`](docs/architecture.md) | Keeper/dial split, the accuracy budget (and why milliseconds, not nanoseconds), every failure mode including the Indian power cut |
| [`docs/display-driving.md`](docs/display-driving.md) | Why 5 V drivers (MAX7219, TM1637, HT16K33) **cannot** light these displays, what daisy-chaining actually means for a passive LED array, how the PCBs partition under 10×10 cm, and how one digit board connects to the next |
| [`docs/radio.md`](docs/radio.md) | Why HC-12 is out on *regulatory* grounds before technical ones, why 433 MHz is a trap in India, and the band that is actually licence-exempt |
| [`docs/timekeeping.md`](docs/timekeeping.md) | Time sources, why DS3231 beats a "more accurate" RTC, RTC discipline, and the signed beacon protocol |
| [`hardware/BOM.md`](hardware/BOM.md) | Costed BOM per dial and per keeper, both digit sizes, with sourced India pricing |

## Where things live

| Path | What |
|---|---|
| [`docs/`](docs/) | The design studies above — research and decisions, no built hardware yet |
| [`hardware/`](hardware/) | BOM; schematics and layout land here (EasyEDA, by hand) |
| [`firmware/dial/`](firmware/dial/) | Dial firmware — display chain, RTC discipline, beacon receive |
| [`firmware/keeper/`](firmware/keeper/) | Keeper firmware — time acquisition, beacon transmit, admin PWA |
| [`site/`](site/) | Marketing site (Cloudflare Pages → boreth.starstucklab.com) |

## Status

**Dial 000 · Keeping vigil.** There are no dials on any wall. There is no
keeper. There is a researched design with its four hard questions answered,
sourced component pricing, a costed BOM for both digit sizes, and an accuracy
budget that says what "in sync" has to mean before anyone solders anything.

Nothing here has been breadboarded, and **the segment driver is deliberately
not committed yet**. Two things have to be measured first: the forward voltage
(datasheets and distributor listings **disagree with each other** — 7.4 V to
13 V per segment for nominally similar parts) and whether the available
displays are common anode or common cathode, which decides the driver family
outright, since every cheap cascadable LED driver is a current *sink*. So the
first bench task is a multimeter and three sample displays, not a PCB order.
See the open questions at the end of
[`docs/display-driving.md`](docs/display-driving.md).

## Family

Part of [starstucklab](https://starstucklab.com) — building small machines for an
indifferent universe. Sibling machines:
[forsyth](https://github.com/m-anish/forsyth) (knew it would rain),
[jigawatt](https://github.com/m-anish/jigawatt) (disagrees with lightning),
[gliim](https://github.com/m-anish/gliim) (the gnome that tries),
[kuhu](https://github.com/m-anish/kuhu) (a soft call before the power turns),
[Sirious](https://github.com/m-anish/sirious) (opinions about your aim).
