# Dial firmware

Placeholder. Nothing written yet.

Target: ESP32-C3 (or C6), with ESP-NOW always and an SX1262 populated only on
dials the survey puts out of range. Responsibilities, in rough order of
importance:

- Drive the cascaded TPIC6B595 chain over SPI + one latch line; PWM `OE` for
  brightness from the ambient sensor ([display-driving.md](../../docs/display-driving.md))
- Keep time on the DS3231; survive mains loss on the coin cell and display the
  correct time immediately on restore
- Receive and authenticate time beacons; maintain `offset` and `rate` estimates;
  **slew, never step**; write the DS3231 aging register
  ([timekeeping.md](../../docs/timekeeping.md))
- Report state `synced` / `vigil` / `lost` ([architecture.md](../../docs/architecture.md))
