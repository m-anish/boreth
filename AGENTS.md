# AGENTS.md — orientation for AI assistants

## What this is

boreth is a **synchronised clock system for campuses**. Two devices:

- **dials** — wall clocks, six seven-segment digits (`HH:MM:SS`) at 2.3" or 4"
  digit height, mains powered, each with its own temperature-compensated RTC on
  a coin cell. ESP-NOW always; LoRa footprint always laid out, populated only
  where the site needs it.
- **the keeper** — one per building or campus. Takes real time from GNSS or NTP,
  holds it, broadcasts a signed beacon to the dials over **both** radios, and
  hosts the admin interface.

**Battery backup is optional on every unit**, dial and keeper alike, and is not
the same thing as the coin cell: the coin cell is standard and keeps the *clock*
running; battery backup is an extra that keeps the *device* running — display
lit, radio alive. Never conflate the two in any document.

Named for the Klingon monastery world where monks guard the time crystals. The
register is **monastic**: institutional patience, keeps the hour, does not
explain itself, would rather not be interrupted. Status line: `Dial 000 ·
Keeping vigil`. Not pompous (that is Sirious), not nervous (jigawatt), not
quietly foreknowing (forsyth).

Protocol vocabulary that should stay consistent everywhere: a dial is `synced`,
in `vigil` (running on its own corrected RTC), or `lost`. The time broadcast is a
**beacon**.

## Read the design studies before designing anything

`docs/` encodes decisions that are already made, with sources. Do not
re-litigate them from general knowledge — several have a plausible wrong answer
that the documents exist to rule out:

- [`docs/architecture.md`](docs/architecture.md) — the accuracy budget (±25 ms
  between dials, ±250 ms to IST) is the root decision; everything else follows
  from it. Also every failure mode, including the Indian power cut, and why
  beacons must be authenticated.
- [`docs/display-driving.md`](docs/display-driving.md) — **MAX7219, TM1637 and
  HT16K33 cannot drive these displays.** Segments are multiple dies in series at
  7.4–13 V. The answer is cascaded 8-bit shift-register drivers, statically
  driven, one per digit, four MCU pins total. **The specific driver IC is
  provisional** — see the guardrail below. §8 covers how one digit board
  connects to the next.
- [`docs/radio.md`](docs/radio.md) — **HC-12 and 433 MHz LoRa are not legal in
  India at useful settings** (10 mW ERP, 10 kHz channel). 865–867 MHz under
  G.S.R. 853(E) is the band. ESP-NOW is peer-to-peer and connectionless; the
  beacon is **broadcast**, which has no peer limit (unicast caps at 20 peers)
  and no link-layer encryption — which is why the application-layer AES-CMAC is
  mandatory rather than belt-and-braces. Keeper fits both radios; dials fit
  LoRa only where the survey says so.
- [`docs/timekeeping.md`](docs/timekeeping.md) — **DS3231 over RV-3028** despite
  the worse headline ppm, because the RV-3028 has no temperature compensation.
  NTP from NPL India (`time.nplindia.org`), median of ≥3 servers.
- [`hardware/BOM.md`](hardware/BOM.md) — costed, with ⟨est⟩ markers on every
  unquoted line. Keep those markers honest.

## Guardrails

- **Nothing has been built.** No board, no firmware, no bench test. The status
  line and the README say so; keep it that way until something is actually
  measured. Placeholder directories and an honest `planning` badge beat invented
  specs.
- **The segment driver is provisional, and that is deliberate — not an
  oversight to tidy away.** Two unknowns gate it and neither can be resolved
  from a catalogue: (1) forward voltage, where sources span 7.4 V to 13 V per
  segment for nominally similar displays, and (2) **common anode vs common
  cathode**, which changes the driver family outright — every cheap cascadable
  LED driver is a *sink*, so common-anode keeps TPIC6B595 while common-cathode
  forces a MIC5891-class *source* driver. Once real display models arrive,
  measure both and commit properly. Until then, do not pick a rail voltage, a
  resistor value or a power supply in any document without flagging the
  dependency — and do not quietly promote the leading candidate to a decision.
- **Mark estimates.** Prices and dimensions either carry a source link or an
  ⟨est⟩ marker. The BOM's credibility is the point of the BOM.
- **The 10 × 10 cm PCB rule is a cost heuristic, not a law** — and the 4" display
  breaks it unavoidably (its pin rows are 107 mm apart). Do not invent a
  contortion to honour it.
- **Regulatory claims need the instrument cited.** This is a product intended for
  sale to institutions in India; a hand-wave about "the ISM band" is not good
  enough. Cite the G.S.R.
- **Bell scheduling is out of scope for v1** but the beacon reserves a message
  type for it. Do not build it; do not design it out either.

## Family context

A spoke of [starstucklab](https://github.com/m-anish/starstucklab), which lives
at `../starstucklab` on disk alongside the siblings (`../jigawatt`, `../lokki`,
`../forsyth`, `../sirious`, `../kuhu`, `../gliim`, `../clear-skies`).

- **Voice**: canonical in `../starstucklab/kit/VOICE.md` and
  `../starstucklab/src/data/persona_preamble.txt`. Load it; never improvise a
  "similar" tone. Dry, short sentences, metaphors from wood, paper, solder and
  starlight, humour soft and resigned. No cheerleading, no superlatives, no
  drama, no first person.
- **Site** (`site/`): zero-build static — plain HTML and CSS, no framework, no
  build step. **Dark only, red, and slanted** — the page is themed *as* the
  product: a seven-segment display is red light leaning at about 8° on dark
  glass, so the palette and the digits follow it rather than sitting beside it.
  (gliim's light-default-with-toggle is the odd one out here; jigawatt and lokki
  are dark-only too.) Deliberately **low on text**: the system is shown as a
  live graphic — one keeper, three dials, all painted from a single timer so
  they genuinely tick together — rather than described in paragraphs. It vendors
  the lab kit in `site/assets/kit/` (synced by
  `../starstucklab/kit/sync-kit.sh`); boreth's own `styles.css` overrides the
  `--sl-*` tokens (LED red `#ff2d16`, brass `#c8a44a`) and keeps the canonical
  family-band and footer markup from `kit/partials/`.
- **Hub card**: in `../starstucklab/src/data/machines.json`; blurb and image
  tooling is `../starstucklab/tools/gen_machine_cards.py`, which harvests this
  site's `og:` tags.
- **Don't edit sibling repos** from here. The hub card is the one exception, and
  it is done from `../starstucklab`.
- Cloudflare Pages and DNS are set up by the owner, never by assistants.
