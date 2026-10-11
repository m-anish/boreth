# Driving the display

> Status: **design study**, not built. The forward-voltage figures below come
> from manufacturer datasheets and Indian distributor listings that **disagree
> with each other**. That disagreement is the single biggest open risk in the
> project and is treated as such at the end.

## 1. The question, answered first

> *"I wonder if the displays can be daisy chained, or the control board would
> need to connect to them individually."*

**Neither, exactly. A seven-segment display is a passive array of LEDs with no
logic inside it — it cannot be chained, addressed, or spoken to.** It has an
anode pin per segment and a common cathode (or the reverse) and that is all it
is.

What *can* be chained is a **driver** sitting on each digit's board. Put one
8-channel high-voltage shift register behind each digit, cascade the registers
(`SER OUT` → `SER IN`), and the result behaves exactly as the brief hoped:

```
control board ──6 wires──> digit 1 ──6 wires──> digit 2 ──6 wires──> digit 3 ──> …
```

Six conductors — `V+`, `GND`, `SER`, `SRCK`, `RCK`, `OE` — carry the entire
display, and that count **does not grow with the number of digits**. A clock is
a control board plus however many digit boards you ordered, in a line.

The three candidate topologies, compared honestly:

| Topology | Wires from control board | Display GPIO on the MCU | Brightness | Verdict |
|---|---|---|---|---|
| **Static, cascaded drivers** (one 8-ch register per digit) | 6 total, chained | **4** | Full — every segment lit 100 % of the time | **Chosen** |
| Individual static drive | 8 per digit + commons ≈ 49 | 48 | Full | Absurd wiring, no MCU has the pins |
| Multiplexed (segment lines + digit enables) | 8 + 6 = 14, all routed back to the control board | 14 | 1/6 duty — needs 6× peak current for equal apparent brightness | Rejected, see §4 |

## 2. Why every popular 7-segment driver IC is disqualified

The displays in this brief are physically large, which means each segment is not
one LED but **several LED dies in series**. That raises the segment's forward
voltage far above logic levels, and it is where the obvious parts die:

| Display | Segment V<sub>f</sub> | Source |
|---|---|---|
| 2.3" red, common anode | **7.40 V** typ @ 20 mA | [Futurlec 7SR23011BS](https://www.futurlec.com/LED/7SR23011BS.shtml) |
| 2.3" green | **8.8 V** typ @ 20 mA | [Futurlec 7SG23011AG](https://www.futurlec.com/LED/7SG23011AG.shtml) |
| 2.3" white | **12.8 V** typ @ 20 mA | [Futurlec 7SG23011AW](https://www.futurlec.com/LED/7SG23011AW.shtml) |
| 2.3" (AND-2307 series) | **8.4 V** typ, 2 dies in series | [Purdy AND-2307](https://www.scribd.com/document/479314518/AND2307) |
| 4" red, common anode | **7.40 V** typ @ 20 mA | [Kingbright SA40-19SRWA](https://www.digikey.com/en/products/detail/kingbright/SA40-19SRWA/3084505) |
| 4" red (Indian stock) | **13 V**, 15–20 mA | [ElectronicsComp 4"](https://www.electronicscomp.com/7-segment-display-common-anode-4-inch) |

So the design must tolerate **7.4 V to 13 V per segment**. Against that:

| Part | Why it fails |
|---|---|
| **MAX7219 / MAX7221** | 5 V part; its segment drive tops out around 4 V. Analog Devices' own application guidance is that series-connected LED displays are out of reach. Confirmed by [Hackaday's survey of high-voltage 7-segment driving](https://hackaday.com/2013/12/15/controlling-high-voltage-7-segment-displays/). This is the part everybody reaches for first and it cannot light a single segment of these displays. |
| **TM1637 / TM1638** | Same 5 V ceiling, plus a fixed scan architecture. |
| **HT16K33** | 5 V, 16×8 multiplex. Same ceiling. |
| **74HC595** | Voltage-capable on the logic side but only ~6 mA per output and no high-voltage drain. |

This is worth stating plainly because the entire internet's "how to build a
7-segment clock" answer is *cascade some MAX7219s*, and at 2.3" and above that
answer is simply wrong.

## 3. The drivers that do work

| Part | Rating | Interface | Price | Notes |
|---|---|---|---|---|
| **TPIC6B595** | **50 V**, 150 mA/ch, 8 ch, open-drain DMOS | Shift register, `SER IN`→`SER OUT` cascade | **$0.595** (SOIC-20, [LCSC](https://www.lcsc.com/product-detail/Registers_Texas-Instruments_C7094.html)) ≈ ₹53 | **Primary choice.** Sinks current → needs common-anode displays. Per-segment resistor sets current. [TI datasheet](https://www.ti.com/lit/ds/symlink/tpic6b595.pdf) |
| **TLC5917 / TLC5916** | 17 V, 3–120 mA constant current, 8 ch | Same cascade | similar class | No per-segment resistors; one R<sub>EXT</sub> sets all channels; uniform brightness across a V<sub>f</sub> spread. **But see the thermal note below.** |
| HV5812 | 80 V, 20 ch | Cascade | dearer | Overkill unless the displays turn out to be 20 V+ |

### The thermal argument — and why resistors win for the big digits

This decides between the two realistic parts, and it is not obvious:

- With **TPIC6B595 + resistors**, the driver is a *switch*. It drops almost
  nothing, and the surplus between the rail and V<sub>f</sub> is burned in the
  resistors — distributed across 48 cheap components in open air.
- With **TLC5917**, the driver is a *current sink*. The surplus is burned
  **inside the package**: `P = (V_rail − V_f) × I × 8 channels`.

A worked case: a 15 V rail, a 7.4 V display, 20 mA per segment, all 8 channels
on → `(15 − 7.4) × 0.02 × 8 = 1.2 W` in one SOIC-20. That is past comfortable.
At the 40 mA a 4" digit may want for corridor brightness, it is hopeless.

Two consequences:

1. **TPIC6B595 + per-segment resistors is the primary architecture.** Cheap,
   50 V of headroom for whatever the displays turn out to need, thermally
   relaxed, and 150 mA/ch leaves room to push brightness.
2. **Choose the rail tight to V<sub>f</sub>** regardless of driver: `V_rail ≈
   V_f(max) + 2 V`. A 12 V rail for 7.4 V parts, 15 V for 13 V parts. A single
   "one rail fits all" 24 V choice would waste more power in heat than the
   display consumes in light.

TLC5917 stays the right answer *if* the chosen display turns out to have a wide
V<sub>f</sub> spread between units and the rail can sit within ~2 V of it — then
it buys resistor-free uniform brightness.

## 4. Static, not multiplexed

With one driver per digit there is no reason to multiplex, and four reasons not
to:

1. **Brightness.** A 1/6-duty multiplex needs 6× the peak current for the same
   apparent brightness: 120 mA per segment. That is at the TPIC6B595's 150 mA
   ceiling and far past TLC5917's.
2. **Cameras.** A multiplexed clock photographs as a partly-lit mess, and the
   clock on an exam-hall wall gets photographed. Static display, clean photo.
3. **No high-side switching.** Multiplexing needs a digit-enable switch carrying
   8 × 120 mA ≈ 1 A at 12–15 V — six P-channel MOSFETs and their gate drive,
   which costs more than the five extra shift registers it saves.
4. **Failure isolation.** One driver per digit means one dead digit, not a dead
   display.

Cost of static over multiplexed: six driver ICs (≈ ₹320) instead of one plus six
FETs (≈ ₹150). Buy the ₹170 of simplicity.

## 5. The MCU pin question, answered

> *"whatever MCU we use should have enough GPIOs"*

With cascaded drivers, the entire display needs **four GPIO pins**:

| Pin | Function |
|---|---|
| `SER` | Serial data (MOSI) |
| `SRCK` | Shift clock (SCK) |
| `RCK` | Latch — all digits update on one edge, so the display never tears |
| `OE` (`G`) | Output enable — **PWM this for global brightness** |

(`SRCLR` ties to V+.) Three of those four are a hardware SPI peripheral, so the
MCU shifts 6 bytes by DMA and raises one latch line. **GPIO count stops being a
selection criterion for the dial MCU** — which frees the choice to be made on
radio, cost and tooling instead. See [`radio.md`](radio.md).

Dimming is not a luxury: a clock at full brightness in a dark corridor at 2 a.m.
is unpleasant, and a clock at night brightness is unreadable at noon. An ambient
light sensor (LDR, or a BH1750 on the same I²C bus as the RTC) driving the `OE`
PWM duty handles it for a few rupees.

## 6. Power

Segment counts for the digits 0–9 average 4.9 segments lit; worst case
(`88:88:88`) is 42 segments plus 2 colon LEDs.

| Case | Segments lit | @20 mA | @40 mA (4", bright) |
|---|---|---|---|
| Typical | ~31 | 0.62 A | 1.24 A |
| Worst (`88:88:88`) | 44 | 0.88 A | 1.76 A |

At a 12 V rail that is **~11 W** worst case for a 2.3" dial, and a 4" dial driven
hard wants a 15 V / 2 A supply. A sealed open-frame AC-DC (MeanWell IRM-20-12 or
IRM-30-15 class) is the sane choice; it also means the whole dial has one input
and no external adapter to go missing.

Note how much of that is heat, not light: at a 12 V rail into a 7.4 V display,
38 % of display power is burned in the current-limiting resistors. The
`V_rail ≈ V_f + 2 V` rule above is what keeps this reasonable, and it is the
reason the rail is a per-display-part decision rather than a product-wide one.

## 7. PCB partitioning under 10 × 10 cm

The brief's cost rule — keep every board inside 100 × 100 mm — survives for 2.3"
and **breaks for 4"**, for a reason that is not the display's size but its
**pin-row span**.

| Display | Body | Digit height | Pin rows | Source |
|---|---|---|---|---|
| 2.3" single digit | ≈ 69.7 × 47.8 × 12 mm | 57 mm | top/bottom | [listing](https://www.indiamart.com/proddetail/2-digit-7-segment-led-display-3323220091.html) — verify per part |
| 4" single digit | **122 × 90 × 15 mm** | 101.2 mm | **10-DIP, rows 107 mm apart** | [Kingbright SA40-19SRWA](https://www.digikey.com/en/products/detail/kingbright/SA40-19SRWA/3084505) |

**2.3" — one board per digit pair.** Two bodies side by side ≈ 96 mm wide ×
70 mm tall. That fits, but only just: add an inter-digit gap for legibility and
the board lands at 100–106 mm, right on the limit. Keep the gap tight to stay
inside 100 mm, or move to one digit per board (≈ 50 × 70 mm) and accept two more
connectors. A 2.3" dial is then **3 pair-boards + 1 control board**.

**4" — the pin span forbids it.** The two pin rows are 107 mm apart, so any board
carrying both rows of a single digit must be ≈ 112–115 mm in that axis. There is
no orientation that fixes this and no way to carry only one row. Options:

| Option | Board | Verdict |
|---|---|---|
| **(a) One digit per board, ≈ 115 × 95 mm** | 6 digit boards + control | **Recommended.** Breaks the 100 mm rule by 15 mm on one axis only, moving to the next fab tier — a few hundred rupees across a 5–10 piece order. Verify at quote time. |
| (b) Two strip boards per digit (one per pin row, ≈ 95 × 25 mm) joined by ribbon | 12 strips + control | Honours the rule, doubles the connectors and the assembly labour. Not worth it. |
| (c) Board narrower than the display (≈ 115 × 60 mm), display overhangs sideways | 6 + control | Fine if there is a frame; same tier as (a) with less support. |

The 10 × 10 cm rule is a cost heuristic, not a law. Spending engineering effort
and 12 extra connectors per clock to avoid a one-tier fab bump is the wrong
trade — option (a), with the cost delta confirmed before ordering.

**Colon LEDs** need no driver of their own: a digit's 8th channel is its decimal
point, which a clock never uses, so the colon hangs off the two spare decimal
point channels of the neighbouring digits.

**Chain mechanics:** put the 6-pin chain connectors at the same relative position
on both edges of every digit board, so boards tile edge-to-edge in either digit
size, and the control board can sit at either end.

## 8. The alternative that was considered and rejected

Large clocks are often built from **addressable LED strip** (WS2812B/SK6812),
2–4 LEDs per segment. It is tempting: one data wire, any digit height, any
colour, the driver problem disappears entirely, and chaining is native.

Rejected for this product because: 5 V at high current (≈ 3 W per digit), far
more solder joints and failure points, softer edges than a moulded segment,
poor readability in daylight, and an invitation to colour effects that a clock
on an institutional wall should not have. Worth re-opening only if sourcing
moulded displays in both sizes proves unreliable.

## 9. Open questions — the bench tasks before any PCB

1. **Measure V<sub>f</sub>.** Buy three 2.3" and three 4" displays from the
   intended Indian supplier and measure segment forward voltage at 20 mA and
   40 mA. The sources above span 7.4 V to 13 V for nominally equivalent parts,
   and some Indian listings state "3.3–5 V", which cannot be true for a 57 mm
   segment. **No rail voltage, resistor value or PSU can be chosen until this
   number is measured.** This is the project's first task.
2. **Confirm common-anode availability in both sizes** from one supplier.
   TPIC6B595 sinks, so common-cathode parts would force a different driver.
3. **Confirm the 2.3" body dimensions** and whether a pair plus a legible gap
   actually lands inside 100 mm.
4. **Get a real fab quote** for 115 × 95 mm versus 100 × 100 mm at 5, 10 and 50
   pieces, to price option (a) honestly.
5. **Judge brightness by eye** at 20 mA in a lit corridor before assuming 40 mA
   is needed — it doubles the power budget and the supply cost.
