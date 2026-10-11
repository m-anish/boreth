# Knowing what time it is

> Status: **design study.** Covers the keeper: which MCU, where real time comes
> from, which RTC holds it, and how a phone administers it.

## 1. Where the time comes from

Four sources, in descending order of trust. The keeper uses the best one
available and **says which one it used** in every beacon, so a dial always knows
how much to believe it.

### GNSS — the best answer, and it needs no internet at all

A GPS module's **PPS output** is a pulse on the UTC second, accurate to tens of
nanoseconds. It is six orders of magnitude better than this project needs, which
means it is not about accuracy — it is about **independence**. A keeper with a
GPS antenna on the roof needs no network, no campus IT, no credentials, and
nothing to renew. For a campus with unreliable connectivity, this is the right
primary source.

NEO-6M modules are **₹259–390** in India
([robu](https://robu.in/product/neo-6m-gps-module-with-eprom-normal-quality/))
and expose PPS. Cost is the antenna run to sky view, which is an installation
problem rather than a BOM problem — which is why GNSS is an **option** rather
than standard on every keeper.

### NTP — and specifically NPL India

India's legal time is kept by **CSIR-NPL**, from caesium clocks and a hydrogen
maser, with a systematic uncertainty of **±2.8 ns** against UTC. IST is
`UTC(NPLI) + 5:30`. NPL disseminates it publicly on two **stratum-1** servers:

```
time.nplindia.org
time.nplindia.in
```

These are synchronised to the national atomic time scale by 1 PPS input
([CSIR-NPL Time & Frequency Division](https://www.nplindia.org/sites/default/files/pdf/webpage_time&frequency.pdf)).

There is a second, non-technical reason to use them. **CERT-In's cybersecurity
directions require Indian entities to synchronise system clocks to the NTP
servers of NIC or NPL, or to servers traceable to them**
([reporting](https://www.medianama.com/2022/12/223-specs-ntp-nic-npl-time-servers-clocks-rti/)).
A campus clock estate traceable to NPL is therefore not just accurate, it is
*defensible* — which matters more to an institution than milliseconds do.

Configured by default, in order: `time.nplindia.org`, `time.nplindia.in`,
`in.pool.ntp.org`, plus any campus NTP server the site adds.

**Never trust one server.** Query at least three, take the median, and reject
any answer that disagrees with the median by more than a threshold or with the
local RTC by more than a configured maximum step. One misconfigured or hostile
NTP server should not be able to move a campus; the arithmetic for this is
trivial and the failure it prevents is not.

### The RTC — holdover

Between syncs, and through every power cut, the keeper and every dial run on a
real-time clock. See §2 — it is the one component choice where the obvious spec
sheet comparison gives the wrong answer.

### Manual

A keeper with no GNSS, no network and a dead RTC must still be settable from the
admin interface, and must mark its beacons `source=manual` forever after, so the
degradation is visible rather than silent.

## 2. The RTC: why DS3231 beats a "more accurate" part

The tempting choice is the **RV-3028-C7**: **±1 ppm** against the DS3231's
±2 ppm, and 45 nA against ~840 nA. Better on both headline numbers.

It is the wrong part here, because of what the headline omits:

| | DS3231 | RV-3028-C7 |
|---|---|---|
| Accuracy | **±2 ppm, 0 °C to +40 °C** | ±1 ppm **at 25 °C** |
| Temperature compensation | **Integrated TCXO** | **None** |
| At 40 °C | still ±2 ppm | ≈ 8 ppm |
| At 0 °C | still ±2 ppm | ≈ 22 ppm |
| Backup current | ~0.84 µA | 45 nA |

Sources: [DS3231 datasheet](https://www.analog.com/media/en/technical-documentation/data-sheets/ds3231.pdf),
[RV-3028-C7 datasheet](https://www.microcrystal.com/fileadmin/Media/Products/RTC/Datasheet/RV-3028-C7.pdf),
and [Hackaday's RTC comparison](https://hackaday.com/2020/09/30/choosing-the-right-rtc-for-your-project/)
for the temperature behaviour.

A clock on a corridor wall in India sees 10 °C in a Himalayan January and 40 °C
in a Delhi May. The RV-3028's ±1 ppm is a laboratory number; at 40 °C it is
**four times worse** than the DS3231, which is specified across the range.
Temperature compensation beats a better room-temperature figure, and the 45 nA
advantage is irrelevant to a mains-powered clock with a coin cell.

**DS3231SN**, then — with two cautions:

1. Specify the **SN** suffix. The **DS3231M** is a MEMS-oscillator part at
   ±5 ppm, and the cheap "ZS-042" modules on Indian marketplaces are
   inconsistent about which they carry, or carry counterfeits. Module prices run
   **₹235–350** ([listings](https://www.indiamart.com/superdebug/time-clock.html));
   the bare IC from an authorised distributor is cheaper and known-good, which
   matters for a product.
2. Use the **aging offset register** — roughly 0.1 ppm per LSB — to trim each
   unit against the keeper's measured rate. This is what turns the holdover
   numbers in [`architecture.md`](architecture.md) §2 from datasheet-nominal into
   per-unit-measured.

Backup: a CR2032 at ~0.84 µA runs for years, and keeps time through every power
cut. On a dial this is the most valuable component in the build: it is why a
clock shows the right time the instant the power returns, rather than waiting
for a beacon.

**The coin cell is standard; battery backup is a separate, optional tier.** The
coin cell keeps the *clock* alive for pennies — nothing else. Keeping the
*device* alive through an outage — display lit, radio listening, keeper still
broadcasting — is an extra, available on both the dial and the keeper and
specified per site. The keeper's version is the cheap and useful one: ~0.5–0.8 W
means a single 18650-class cell carries it most of a day ⟨est⟩, so it is still
disciplined and still transmitting when the dials come back. The dial's version
has to carry the display, which is amps rather than microamps. Both are costed
in [`architecture.md`](architecture.md) §3a and
[`../hardware/BOM.md`](../hardware/BOM.md).

## 3. The keeper's MCU

| Candidate | For | Against |
|---|---|---|
| **ESP32-S3** | Wi-Fi + BLE + ESP-NOW, plenty of flash and RAM for a web UI and TLS, USB for provisioning, the pattern already proven in [forsyth](https://github.com/m-anish/forsyth)'s coordinator | 2.4 GHz only — so it is paired with an SX1262 for reach ([`radio.md`](radio.md) §2b) |
| **ESP32-C6** | Everything above plus **802.15.4** (Thread/Zigbee), and the [module is ₹199](https://aws.robu.in/product/espressif-esp32-c6-mini-1-n4-module/) | Newer toolchain; less headroom than the S3 for a web app |
| ESP32-C3 | Cheapest ([₹279 dev board](https://robu.in/product/esp32-c3-development-board-with-soldering/)) | Tighter RAM for TLS plus a served PWA |
| RPi Zero 2 W / Linux | chrony, real NTP, trivial web stack | Needs an SD card, a shutdown story and a filesystem that survives India's power cuts — the wrong reliability trade for a device whose job is to be up |

**Recommended: ESP32-S3 for the keeper, ESP32-C3 or C6 for the dials.** The
keeper carries the web UI, the TLS stack and the time logic; the dials need SPI,
I²C and a radio.

Note what the keeper's MCU choice does *not* settle: it is dual-radio
regardless. ESP-NOW comes free in the ESP32, and an SX1262 is always populated
alongside it, because one keeper per building makes the module cost irrelevant
against being able to reach a far dial at all. The dials are the asymmetric
side — ESP-NOW always, LoRa only where the survey says so
([`radio.md`](radio.md) §2b).

**Add optional Ethernet (W5500 or LAN8720).** A campus IT department will far
rather give a static IP and a wall port to a clock master than put it on Wi-Fi
with a shared PSK — and a wired keeper is one fewer thing that breaks when the
WLAN is re-keyed. This is a real purchasing consideration for "medium to large
campuses", not a technical nicety.

## 4. Administration from a phone

Two paths, deliberately. The local one must work with the internet unplugged.

**Local — the one that must always work.** First boot brings up a **SoftAP with a
captive portal**: pick the Wi-Fi network or the static IP, set the timezone
offset, set the campus key, name the keeper. After that the keeper joins the
network and serves a small **PWA over mDNS** at `boreth.local`. No app store, no
account, no internet. A phone, a browser, and a clock mounted four metres up a
wall that nobody wants to get a ladder to twice.

**Remote — optional, and never load-bearing.** No campus IT department will port
forward to a clock, so the keeper should not ask them to. Instead it opens an
**outbound WebSocket (or MQTT) to a Cloudflare Worker**, and the administrator
uses a web app — the same stack as [kuhu](https://github.com/m-anish/kuhu)
(Worker + D1), including its **invite-code → bearer token** auth model rather
than a second invented one. Free tier, no inbound firewall rule, no dynamic DNS.

Settings worth exposing, and nothing more:

- Time source order and the NTP server list
- Timezone offset (IST has no DST; keep the field anyway)
- Brightness schedule, or automatic from the ambient sensor
- Dial inventory: name, location, last beacon heard, current state
  (`synced` / `vigil` / `lost`), measured rate error
- Firmware OTA for keeper and dials
- Campus key rotation

That dial inventory is the feature an institution will actually care about:
**which clock on this campus is lying, and since when.** A clock estate's real
failure mode is not inaccuracy, it is nobody noticing.

## 5. Open questions

1. **Does the campus have usable network at the keeper's location?** Decides
   GNSS-primary versus NTP-primary, and whether Ethernet is worth populating.
2. **Measure actual NTP offset and jitter** to `time.nplindia.org` from a
   Himachal connection before assuming it beats the 250 ms absolute budget. It
   should, comfortably, but it is one `ntpdate -q` away from being known rather
   than assumed.
3. **Confirm DS3231SN sourcing** that is not a marketplace module of uncertain
   provenance.
4. **Decide whether dials accept OTA over the time bus** or only over Wi-Fi.
   Firmware over LoRa is slow and worth avoiding; Wi-Fi on a dial is a commission
   -time convenience that may be worth keeping enabled.
