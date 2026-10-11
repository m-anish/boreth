# boreth — system architecture

> Status: **design study.** Nothing in this document has been built. Figures are
> from datasheets and distributor listings, cited where they matter. Where
> sources disagree, that is called out rather than averaged.

## 1. The two devices

Boreth is two products that only make sense together.

**The dial** — a clock. Six seven-segment digits (`HH:MM:SS`) at 2.3" or 4"
digit height, a colon between the pairs, mains powered, wall mounted. Contains
its own temperature-compensated RTC on a coin cell. Shows the right time whether
or not anything else in the system is alive.

**The keeper** — one per building or per campus. Acquires real time from GNSS or
the network, holds it, and broadcasts it to the dials. Hosts the admin interface.
Nothing in the system depends on more than one keeper existing, and nothing
breaks if one dies.

The vocabulary matters for the protocol later: a dial is `synced` when it has
heard a fresh beacon, in `vigil` when it is running on its own corrected RTC,
and `lost` when it has had no beacon for long enough that it is no longer willing
to vouch for itself.

```
         GNSS (PPS)        NPL India NTP            campus NTP
              \                 |                      /
               \                |                     /
                +------------ keeper -----------------+
                           (DS3231 TCXO,
                            disciplined)
                                |
                     signed time beacon, ~60 s
                     ESP-NOW (indoor) / LoRa 865–867 (site)
                                |
            +-------------+-----+-----+-------------+
            |             |           |             |
          dial          dial        dial          dial
        (own RTC)     (own RTC)   (own RTC)     (own RTC)
```

This is deliberately the NTP stratum model: one disciplined source, many clients
that each hold their own oscillator and correct it, nobody trusting the network
to be up.

## 2. The accuracy budget — the most important decision in the project

Everything downstream (which radio, which RTC, how often to transmit, whether
latency needs to be deterministic) falls out of one question: **how wrong is
wrong?**

A campus clock has two requirements, and they are far apart:

**(a) Dials must not visibly disagree with each other.** This is the hard one,
and it is a perceptual limit, not a metrological one. Two clocks in the same
corridor, both ticking seconds, are compared by eye. Our working target is
**≤ 25 ms of skew between any two dials** — below the point where two visual
events read as separate. That threshold is an assumption to validate by eye
during the first install, not a sourced number; if it turns out 100 ms is
invisible, the budget only gets easier.

**(b) Dials must not visibly disagree with the country.** Target **≤ 250 ms from
IST**. Any network time source beats this by an order of magnitude; GNSS beats it
by six. This requirement is nearly free.

What that buys us, which is the whole point:

| Consequence | Because |
|---|---|
| The radio does **not** need deterministic latency | 25 ms of budget against a link whose delay is computable to ~1 ms |
| No GNSS in the dials | A ₹260 GPS per clock buys accuracy 10⁵× tighter than needed |
| No PTP, no hardware timestamping, no sub-µs anything | Off by four orders of magnitude from the requirement |
| A missed beacon is a non-event | See the drift arithmetic below |

### Drift arithmetic

A DS3231 is specified at **±2 ppm** from 0 °C to +40 °C
([datasheet](https://www.analog.com/media/en/technical-documentation/data-sheets/ds3231.pdf)).
At 2 ppm:

| Interval with no sync | Worst-case error of one dial | Worst-case skew between two dials |
|---|---|---|
| 1 hour | 7.2 ms | 14.4 ms |
| 1 day | 0.17 s | 0.35 s |
| 1 week | 1.2 s | 2.4 s |
| 1 year | 63 s | 126 s |

Two dials can drift in opposite directions, so the pairwise skew is double the
single-dial error. Read against the 25 ms target, that table makes the design
decision for us:

- **Beacon every 60 s**, which costs almost nothing.
- **Require only one beacon per hour** to stay inside budget — so 59 consecutive
  lost beacons are tolerable, and the radio can be lossy, duty-limited and
  unacknowledged.
- **Daily sync would not be enough.** 0.35 s of pairwise skew is visible when two
  clocks are in one sightline. This is the trap worth naming: a "sync once a
  night" design looks fine on a bench and fails in a corridor.

### Rate correction — why holdover is better than the table above

The table assumes each dial runs at its oscillator's nominal rate. It doesn't
have to. Every beacon is a free measurement of that dial's own error, so the
dial can estimate its individual **rate** (ppm) over days and correct for it —
and the DS3231's aging register trims the oscillator directly, roughly 0.1 ppm
per LSB.

A dial that has been listening for a week knows its own crystal better than the
datasheet does. Residual error after correction is dominated by temperature
swing rather than by the nominal 2 ppm, which in an indoor corridor is small.
This is the difference between "a week of silence costs a second" and "a month
of silence costs a second", and it costs nothing but firmware.

## 3. Failure modes

The brief asked for failsafe. Ranked by how often each will actually happen in
an Indian campus building:

| # | Failure | What happens | Design requirement |
|---|---|---|---|
| 1 | **Mains cut** (routine, daily in much of India) | Display goes dark. RTC keeps running on its coin cell. On power restore the dial shows correct time **immediately**, without waiting for a beacon or a network. | Coin cell on every dial's RTC. This is the single most important failsafe and the reason the per-dial RTC in the brief is right. A DS3231 keeps time at ~0.84 µA from a CR2032 — years. |
| 2 | **Keeper loses internet** | Keeper enters holdover on its own TCXO, keeps broadcasting, marks the beacon `source=holdover` so dials know the chain is degraded. Dials unaffected for days. | Keeper must broadcast its own trust level, not just the time. |
| 3 | **Dial loses radio** | Enters `vigil`: runs on its rate-corrected RTC. Still correct to well under a second for a week. | Dial must never blank or show an error because the radio is quiet. |
| 4 | **Keeper dies completely** | Every dial runs in `vigil`. The campus degrades to "all clocks drift together, slowly" — which is what an unsynchronised clock estate does anyway, except these started from the right time and know their own rate. | No single point of *failure*, only a single point of *correction*. |
| 5 | **Two keepers disagree** | Dials prefer the beacon with the better source (GNSS > NTP > holdover), then the freshest sequence, then the lowest keeper ID, **with hysteresis** so a dial on the boundary doesn't oscillate between two keepers every minute. | Deterministic tie-break, written down, with hysteresis. |
| 6 | **Forged or replayed beacon** | Rejected. Every beacon is authenticated and carries a monotonic sequence number. | See below — this is not optional. |
| 7 | **Display driver or digit fails** | One digit dark; the rest keep working, because each digit is an independent driver on a cascaded bus. A dead digit board can be swapped without touching the others. | Per-digit driver, per-digit board. |

### On authentication

A campus clock system decides when exams end and, sooner or later, when the bell
rings. An unauthenticated time broadcast means anyone within radio range and ₹800
of hardware can move the whole campus's clocks. That is a prank for a student and
a liability for the institution.

So: every beacon carries an **AES-CMAC (or HMAC-SHA256) tag** over
`{utc, source, stratum, keeper_id, sequence}`, keyed per campus, with a monotonic
sequence number and a rejection window for stale packets. Keys are provisioned at
commissioning through the keeper's captive portal, never shipped as a default.

This also quietly rules out transparent-serial radio modules, which give you a
byte pipe with no framing to authenticate — see [`radio.md`](radio.md).

## 4. Time distribution, in one paragraph

The keeper sends a beacon: *"when the last bit of this packet left me, UTC was
X"*, plus its trust level and a signature. The dial timestamps packet-done on its
own clock, adds the link's **computable** airtime (LoRa's time-on-air is an exact
function of spreading factor, bandwidth, coding rate and payload length — this is
why a radio with opaque internal buffering is the wrong choice), and gets the
keeper's time referenced to its own. Then it does not step its clock: it slews,
and it updates its rate estimate. Only a cold start or an error too large to slew
causes a visible jump.

Dials show UTC plus a configured offset, so the system is not India-only and a
timezone or DST policy change is a setting, not a firmware release. IST has no
DST, which makes the common case trivial.

## 5. Deliberate non-goals for v1

- **Bell scheduling.** A system that keeps a campus's time is one step from the
  system that rings its periods, and schools already buy bell systems. It is the
  obvious adjacency and it is explicitly out of scope — but the beacon protocol
  reserves a `scheduled event` message type so adding it later is not a redesign.
- **Analogue dials.** Different mechanism (stepper, hands, homing), different
  product.
- **Cloud dependency.** The admin interface may optionally be reachable remotely;
  nothing about keeping time may depend on it.
- **A competitive market survey.** Not done. Commercial GPS-synchronised campus
  clock systems exist and are priced per clock; this project has no verified
  figures for them yet and will not quote any until it does.
