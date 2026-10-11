# The radio link

> Status: **design study.** The regulatory section is the important part and the
> conclusion is not the obvious one.

## 1. The headline: HC-12 is out, and not for technical reasons

The brief guessed at HC-12 or LoRa. HC-12 is the cheaper, simpler, more tempting
of the two and it has to be rejected — **primarily on Indian spectrum
regulation, before any engineering argument.**

### What India actually permits

| Band | Condition | Instrument |
|---|---|---|
| **865–868 MHz** | Licence-exempt. **Up to 1 W ERP**, 200 kHz carrier bandwidth | [*Use of Low Power Equipment in the Frequency Band 865–868 MHz for Short Range Devices (Exemption from Licence) Rules, 2021* — G.S.R. 853(E), 10 Dec 2021](https://thc.nic.in/Central%20Governmental%20Rules/use%20of%20low%20power%20Equipment%20in%20the%20frequency%20band%20865%20to%20868%20MHz%20for%20Short%20Range%20Devices%20Exemption%20from%20Licence%20Rules,2021.pdf) (supersedes the 2005 rules, which covered only 865–867) |
| **433–434.79 MHz** | Licence-exempt but **10 mW ERP maximum, ≤10 kHz channel bandwidth, built-in antenna, indoor** | G.S.R. 680(E) and G.S.R. 698(E) — [summary](https://pcnindiaglobal.com/2026/07/07/wpc-eta-433-mhz-sub-ghz-short-range-devices-india/) |
| **2.4 GHz ISM** | Licence-exempt (Wi-Fi, BLE, 802.15.4) | — |

Everything sold into India also needs **WPC Equipment Type Approval (ETA)**
before import or sale, which in a de-licensed band is a self-declaration route.
That applies to a product for campuses — it does not apply to a bench
prototype, but it does apply the day one is sold.

### Measured against that, HC-12

| HC-12 fact | Against the rule |
|---|---|
| Operates **433.4–473.0 MHz** | Only 433–434.79 MHz is de-licensed. Most of its range is not de-licensed *at all*. |
| **100 mW** max transmit power | The 433 MHz limit is **10 mW ERP** — 10× over. |
| Channel bandwidth far wider than 10 kHz in its FU3 default | Over the bandwidth condition. |

Specifications from the [HC-12 v2.4 manual](https://statics3.seeedstudio.com/assets/file/bazaar/product/HC-12_english_datasheets.pdf).
A unit throttled to 10 mW in a narrow mode might be made to comply, at which
point it is a slow, short-range radio with no advantage left.

The technical objections are secondary but they all point the same way:

- **Transparent-serial means opaque timing.** HC-12 is a byte pipe; it buffers
  and packetises internally, and users document [undocumented delays](https://forum.allaboutcircuits.com/threads/hc-12-undocumented-limitations.151513/).
  For time distribution, a link whose delay you cannot compute is the one thing
  you do not want — see §4.
- **No transmit-done signal**, so there is nothing to timestamp against.
- **No framing**, so there is no clean place to hang the authentication that
  [`architecture.md`](architecture.md) §3 says is mandatory.

### The same trap catches 433 MHz LoRa

The cheapest LoRa modules in India are 433 MHz SX1278 boards — Ra-02 at **₹349**
([robu](https://robu.in/product/sx1278-lora-module-ra-02-433mhz-wireless-spread-spectrum-transmission/)).
They are half the price of the 868 MHz parts and they are equally unusable here:
LoRa's narrowest practical bandwidth is 125 kHz against a **10 kHz** channel
limit, and these modules transmit at up to 20 dBm against a **10 mW (10 dBm)**
limit. Cheap, available, and non-compliant.

**865–867 MHz is the band.** It is licence-exempt, it allows 1 W ERP, LoRa at
BW 125 kHz fits inside the 200 kHz carrier condition, and it is the same IN865
band the Indian LoRaWAN ecosystem already uses. Note the bandwidth condition
rules out LoRa BW 250 kHz and 500 kHz.

## 2. Options, costed

| Option | Band | India | Indoor range | Latency | Per-dial cost | Verdict |
|---|---|---|---|---|---|---|
| **ESP-NOW** (already inside the ESP32) | 2.4 GHz | exempt | 40–100 m through walls; >900 m open field in LR mode | **<25 ms** even at 900 m | **₹0** | **Default inside a building** |
| **802.15.4 / Thread** (ESP32-C6) | 2.4 GHz | exempt | mesh — every dial repeats | ms | **₹0** (the [C6-MINI-1-N4 module is ₹199](https://aws.robu.in/product/espressif-esp32-c6-mini-1-n4-module/)) | Best long-term for a dense campus; more firmware |
| **LoRa SX1262 @ 865–867** | 865–867 | **exempt, 1 W ERP** | km, through buildings | 57–185 ms, **computable** | ₹320–₹850 | **For outbuildings and no-network sites** |
| LoRa SX1278 @ 433 (Ra-02) | 433 | **non-compliant** at useful settings | — | — | ₹349 | Avoid |
| HC-12 @ 433 | 433.4–473 | **non-compliant** | — | opaque | ₹200–250 | Avoid |
| nRF24L01+ | 2.4 GHz | exempt | 100 m with PA | low | ₹130 | No advantage over ESP-NOW, and costs an SPI bus and an antenna |

LoRa module prices, India, October 2026:

| Module | Chip | Price | Source |
|---|---|---|---|
| DRF1262T | SX1262 | **₹320** | [IndiaMart](https://www.indiamart.com/proddetail/lora-sx1262-module-drf1262t-868-915mhz-2850710774573.html) — verify stock and authenticity |
| E220-900T22D | LLCC68 | **₹775** incl GST | [Hubtronics](https://hubtronics.in/e22-900m22s) |
| E220-900T22D | LLCC68 | ₹849 | [Nxt Kart](https://www.nxtkart.com/p/ebyte-e22-900t22d-v2-sx1262-lora-868mhz-wireless-module-uart-22dbm-5km-long-range-fec-sma-k-antenna-rf-wireless-transmitter) |
| E22-900T22D-V2 | SX1262 | ₹846–1,299 | [ElectroPi](https://www.electropi.in/ebyte-e22-900t22d-v2-sx1262-lora-868mhz-wireless-module-uart-22dbm-5km-long-range-fec-sma-k-antenna-rf-wireless-transmitter) / Hubtronics |
| Ra-01SH | SX1262 | ≈$11 | [Rokland](https://store.rokland.com/products/ra-01sh-p-lora-module-ai-thinker) |

Prefer an **SPI** module (Ra-01SH, E22-900M22S) over a **UART** one
(E220-900T22D) despite similar prices: SPI exposes the `DIO1`/`TxDone` interrupt
that §4 needs, where a UART module hides it behind the same kind of buffering
that disqualified HC-12.

## 3. The recommendation: the time bus is a role, not a radio

Paying ₹700–850 per clock for kilometres of range inside a building where 40 m
would do is how a 40-clock campus acquires ₹30,000 of unnecessary radio. But a
gatehouse, a sports field or an unwired block genuinely needs it.

So: **one dial board design, two populations.**

- Every dial has an ESP32 and therefore ESP-NOW for free.
- Every dial board carries a **footprint and a 6-pin header for an SX1262
  module**, populated only where the site survey demands it.
- A keeper is cheap enough (see [`../hardware/BOM.md`](../hardware/BOM.md)) that
  **one keeper per building**, each independently synced to the same upstream
  time, is usually better than one campus keeper straining to reach everything.
  Two keepers pinned to the same NTP source agree with each other to ~10 ms
  without ever talking, which is well inside the 25 ms budget.
- LoRa then carries time only between buildings, or to the dials no network
  reaches.

This also means the 2.4 GHz congestion question (a campus Wi-Fi estate is a busy
band) is answered by range, not by power: short hops, a keeper per building, and
a retry budget of 59 missed beacons per hour.

## 4. Why computable airtime matters

LoRa's time-on-air is an **exact function** of spreading factor, bandwidth,
coding rate, preamble length and payload length. For a 20-byte beacon,
BW 125 kHz, CR 4/5:

| Setting | Symbol time | Time on air | Duty at one beacon / 60 s |
|---|---|---|---|
| SF7 | 1.02 ms | ≈ 57 ms | 0.10 % |
| SF9 | 4.10 ms | ≈ 185 ms | 0.31 % |

So a dial can take the `RxDone` interrupt, subtract a number it computed itself,
and recover the keeper's transmit instant to about a millisecond — no round
trip, no handshake, no acknowledgement. That is a quarter of the 25 ms budget
spent on the radio, with the rest left for the RTC.

A transparent-serial module cannot do this at all, because the delay between
"the application handed over a byte" and "the byte left the antenna" is
undocumented and variable. This is the real engineering reason to prefer LoRa
here, independent of range, and it is invisible until you try to synchronise
something.

India's 865–868 MHz rules impose no EU-style 1 % duty cycle, but 0.1–0.3 % is
good manners in a shared band and leaves room for many keepers.

## 5. The beacon

```
beacon {
  utc_seconds   : u32     // UTC at the moment the last bit left the keeper
  utc_millis    : u16
  source        : enum    // gnss | ntp | holdover | manual
  stratum       : u8
  keeper_id     : u16
  sequence      : u32     // monotonic, for replay rejection
  tag           : u128    // AES-CMAC over all of the above, per-campus key
}
```

Broadcast, unacknowledged, every 60 s. Dials never reply, so the protocol does
not scale with clock count and a campus of 400 dials costs the same airtime as
one. Reserved message type: `scheduled_event`, so period bells can be added
later without a protocol change ([`architecture.md`](architecture.md) §5).
