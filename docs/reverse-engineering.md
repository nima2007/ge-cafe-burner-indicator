# Reverse-engineering notes

How the pinout and the cooktop register were identified on a GE Café
CGY366P2TS1, including the mistakes — because the mistakes are the parts
most likely to save someone else an evening.

## Starting point

The SmartHQ app shows "COOKTOP: On" for this range family, so the
appliance itself knows burner state. But:

- GE's public v2 API and the official Home Assistant integration do not
  expose cooktop status.
- The open-source gehomesdk (which talks to GE's cloud the way the app
  does) names a candidate register, `COOKTOP_STATUS_EXT = 0x5522` — but
  on a sibling Café model that register reads stuck-off through the cloud
  while the app shows On. Cloud data could not be trusted either way.

So the question became: is cooktop state reachable on the range's local
GEA3 bus, at the RJ45 service port? Nobody had published a range
configuration for esphome-gea. The answer turned out to be yes.

## Phase 1: listening (the bus never announces state)

With an ESP32 UART (230400 8N1) on the port's TX contact, the traffic is
clean and highly structured — 871 frames, zero framing errors. But the
idle vocabulary is only 6 unique frames:

- Four 8-byte heartbeats on a ~1 s cadence:
  `E2 FE 08 9A 42 A1 41 E3`, `E2 FE 08 C0 42 40 35 E3`,
  `E2 FE 08 98 42 C7 23 E3`, `E2 FE 08 9B 42 92 70 E3`
- One 11-byte frame, `E2 FF 0B C0 A0 [xx] 60 0D [crc] E3`, every ~60 s,
  with `[xx]` incrementing — a minutes counter.

Diffing against actions changed nothing: lighting a burner for 15 s,
running the oven at 350°, opening the door — all null. The TX line is a
pure keepalive channel. GEA state has to be *requested*.

## Phase 2: talking (two failures, then success)

An ESPHome hub running esphome-gea auto-detected the appliance address
(`0xC0`) from the heartbeats and fired subscribe-all requests — and got
total silence. Twice, weeks apart, for what turned out to be two
different layers of the same root cause.

### Failure 1: drive strength and the series resistor

The range's RX input is pulled up to the 5 V rail through **683 Ω**
(measured with the breaker off; the diode test reads 0.202 V
symmetrically in both directions, i.e. a pure resistance, consistent
with 683 Ω at the meter's ~0.3 mA test current).

An ESP32 GPIO idles at 3.3 V and, through the 1 kΩ series resistor that
had been fitted "for safety," forms a divider against that pull-up: a
driven low at the range's pin only reaches ~3.1 V. Not a valid low, at
any baud rate. The series resistor in the TX path has to go (GE's own
adapter uses 51 Ω). And the bare 3.3 V push-pull output is marginal into
this load anyway — hence the 74HCT245: an HCT buffer powered from 5 V
accepts the 3.3 V input (HCT counts high from 2.0 V) and sinks the
~7.5 mA needed to drag the pulled-up line to ~0.1 V.

### Failure 2 (the real one): the breakout board's labels are mirrored

Every functional test said one pin behaved like a power rail: it held
5.07 V under a 5 kΩ load with no sag, it won a tug-of-war against the
buffer on two different channels, and an oscilloscope capture showed its
ripple tracking the other power contact's ripple (correlation +0.83 on
the aligned window — which in hindsight was rail ripple showing *through
the pull-up*, not proof of a shared net). Meanwhile a resistance
measurement proved the two pins were linked by 683 Ω — but resistance
alone can't say which side of a pull-up is the input.

The resolution: the RJ45 breakout board's printed pin numbers are
**mirrored** relative to the jack. The pin we had been driving as "RX"
was the rail; the pin we had been avoiding as "the rail" was the RX
input. Moving one jumper to the other contact made the input swing
0.1 ↔ 5.1 V under load, and the very next subscribe-all was answered.

Lesson, generalized: **identify connector pins by measurement, not by
any board's silkscreen** — including the breakout's, and including the
table in this repo's README (which is labeled by that same breakout).
The discriminators that work:

- Rail vs pulled-up input: load test (rail doesn't sag under 5 kΩ;
  pulled-up input drops to ~4.4 V) or breaker-off resistance between
  the pair (~683 Ω = a pull-up links them; the rail is the one that
  also matches the known power contact's behavior under load).
- TX: voltage jitters with traffic; breaker-off resistance is an
  unstable wandering MΩ (phantom-powering artifact of a CMOS output).
- Ground floats vs chassis on this appliance: chassis is *not* a
  reference. The bus ground contact is.

### Success

With the buffer driving the correct contact, subscribe-all returned
**33 ERDs with zero timeouts**, including `0x5522` at 32 bytes. A
reboot re-dump with no burner lit reproduced the baseline byte-for-byte:

```
0x5522 all off:      0001FF0101FF0101FF0101FF0101FF0101FF0100FF0000FF0000000000000000
0x5522 front right:  0101FF0101FF0101FF0101FF0101FF0101FF0100FF0000FF0000000000000000
```

Exactly one byte changes — byte 0, `0x00` → `0x01`. `0x551C` did not
change. The register is live, local, and truthful.

## Open questions

- **Byte 0 semantics.** Bitmap, count, or any-on flag? Distinguishing
  test: dump with a different single burner lit, and with two lit.
  (Front-left alone: `0x01` ⇒ count or any-on; `0x02` ⇒ bitmap.
  Two lit: `0x02` ⇒ count; `0x03` ⇒ bitmap, if the second burner is
  bit 1.) Not needed for an any-burner indicator, worth knowing for
  per-burner displays.
- **The rest of the payload.** The repeating `01 FF 01` pattern suggests
  per-burner blocks (exists / state / power?), but no block byte changed
  when the front right burner lit — per-burner detail may live in a
  different ERD, or in bits of byte 0 only.
- **The other 32 ERDs.** `0x5670`, `0x5674`, `0x5675` and the rest are
  undecoded. Oven state is presumably in there somewhere.
- **Push vs poll.** The firmware re-asserts the light every 5 s from the
  sensor state, so the indicator is correct either way; whether the
  appliance pushes `0x5522` changes to subscribers or the hub's reads
  are doing the work hasn't been isolated.

## Bench notes

- ESP32 GPIOs are **not** 5 V tolerant (3.6 V absolute max). The
  range-TX → ESP32 path runs through 1 kΩ and has been clean for
  thousands of frames, but a divider (10k/20k) is the fully in-spec
  permanent answer for that direction.
- The buffer must be **HCT**, not HC: at a 5 V supply an HC input needs
  ~3.5 V to read high, and the ESP32's 3.3 V is marginal. HCT's TTL
  thresholds (high from 2.0 V) are the whole point.
- AliExpress "SN74HCT245N" clones were suspected mid-debug of having
  dead channels and were fully exonerated — both "failing" channels
  were driving the power rail. Test your parts against a known load
  before blaming them.
- The range emits a faint high-pitched power-supply whine; its pitch
  shifts slightly with the ESP32/buffer connected. Audible evidence of
  coupling, no sign of harm.
