# Keeper firmware

Placeholder. Nothing written yet.

Target: ESP32-S3. Responsibilities, in rough order of importance:

- Acquire time: GNSS PPS → NTP (NPL India, median of ≥3 servers, outlier
  rejection) → disciplined RTC → manual ([timekeeping.md](../../docs/timekeeping.md))
- Discipline and hold the DS3231; declare its own trust level honestly
- Transmit signed beacons every 60 s over ESP-NOW and/or LoRa
  ([radio.md](../../docs/radio.md))
- Serve the admin PWA: SoftAP captive portal for commissioning, then mDNS on the
  campus network; optional outbound WebSocket to a Cloudflare Worker for remote
- Track every dial: last beacon heard, state, measured rate error
