# GE Café Range Burner Indicator

**A local, cloud-free "you left the stove on" light for a GE Café CGY366P2TS1 gas range, built by tapping the range's own GEA3 bus through its RJ45 service port.**

The CGY366 has no burner-on indicator light. The only place burner status exists is the SmartHQ cloud app — the range knows exactly which burners are lit, but GE's public API doesn't expose cooktop status, and a cloud round-trip is the wrong foundation for a safety light anyway.

This project reads the cooktop status register (ERD `0x5522`) directly off the range's internal GEA3 serial bus. An ESP32 subscribes to that register and lights a NeoPixel strip red while any burner is on. No Wi-Fi, no cloud, no account, nothing phones home — and the whole thing is powered by the service port's own 5 V rail, so it runs standalone with no computer attached.

Status: **working** on the author's CGY366P2TS1 (November 2026). Byte 0 of `0x5522` is `0x00` with all burners off and goes non-zero when a burner lights. Whether that byte is a per-burner bitmap, a count, or an any-on flag is not fully decoded yet — "non-zero means something is on" is proven, and that's all an indicator needs.

## How it works

1. The range's RJ45 service port (bottom front of the range, behind no panels) carries the GEA3 bus: 230400 baud, 8N1, 5 V logic.
2. An ESP32 running [ESPHome](https://esphome.io) with the [esphome-gea](https://github.com/mguaylam/esphome-gea) component connects through a 74HCT245 buffer (the range's receive line has a stiff 683 Ω pull-up that a bare 3.3 V GPIO cannot pull low).
3. On boot the hub sends a subscribe-all and the range dumps every ERD it supports — 33 of them. `0x5522` (`COOKTOP_STATUS_EXT`, 32 bytes) is the cooktop.
4. A binary sensor watches byte 0 of `0x5522`; a 5-second interval re-asserts the light state so it can never drift from reality.
5. The strip lights red (first 12 LEDs only, 60% brightness, ~150 mA). The other 83 LEDs of the author's 95-LED strip are never driven — all 95 at full white would draw ~5.7 A, which neither the rail nor Dupont wires should carry.

One protocol fact that shaped the design: **the range never announces state changes on its own.** Its transmit line carries only keepalive frames (four heartbeats on a ~1 s cadence, plus a minutes counter every ~60 s). Lighting a burner changes nothing on the wire until you ask. GEA is request/response — you have to subscribe or read.

## Parts

| Part | Notes |
|---|---|
| ESP32 dev board | Classic ESP32 (D0WD) is what was used; no Wi-Fi is used at runtime |
| SN74HCT245N (DIP-20) | Must be **HCT** (TTL-level inputs) — an HC part's input threshold is marginal at 3.3 V. Only one of its 8 channels is used |
| RJ45 breakout board | See the mirroring warning below before trusting its silkscreen |
| 1 kΩ resistor | RX path only (range TX → ESP32). Do **not** put one in the TX path — see docs |
| WS2812B NeoPixel strip | Any length; firmware drives the first 12 |
| Breadboard + jumpers | |

## The service port pinout

Only 4 of the 8 contacts are populated. **These numbers are the labels printed on the author's RJ45 breakout board — and that board's labels turned out to be mirrored relative to the jack.** This mirror cost two evenings of debugging (see `docs/reverse-engineering.md`). If your breakout differs, verify with the meter checks below instead of trusting any silkscreen, including this table's numbering: what matters is which *function* sits on which *physical contact*, and the functions are unambiguous.

| Breakout label | Function | Powered, vs pin 8 | Breaker off, vs pin 8 |
|---|---|---|---|
| 3 | **+5 V power rail** | 5.07 V, rock steady | 2.21 kΩ |
| 4 | Range **TX** (data out) | ~5.05 V, visibly fluctuating (traffic) | unstable MΩ readings* |
| 5 | Range **RX** (data in), 683 Ω pull-up to the rail | 5.07 V, steady | 2.89 kΩ |
| 8 | **Ground** | — | — |
| 1, 2, 6, 7 | *no contact present* | — | — |

\* A CMOS output on an unpowered board has no stable resistance reading; the meter's own test current phantom-powers the board through the pin's protection diodes. Wildly wandering MΩ values are the expected signature of the TX pin, not a fault.

Identifying the pins on your own range:

- The **rail** and the **RX** input both sit at ~5.07 V powered on. Tell them apart with the breaker **off**: resistance between the two is **~683 Ω** (the RX pull-up). Or powered: load each with 5 kΩ to ground — the rail doesn't sag, the RX line drops to ~4.4 V through its pull-up.
- **TX** is the one whose voltage visibly jitters on a meter and which streams structured frames at 230400 baud.
- Board ground **floats relative to the chassis** — there is no continuity between chassis metal and any pin. Use pin 8 as your reference, not the range body.

Cross-checks against GE's own design: GE publishes the schematic of its official adapter ([geappliances/home-assistant-adapter](https://github.com/geappliances/home-assistant-adapter)). It confirms two contacts are alternate power feeds (4.3–15 V), the bus is buffered through GT125-style level shifters, and GE uses 51 Ω series resistors on the data lines.

## Wiring

74HCT245 straddling the breadboard groove, notch up:

- Chip pin 20 (VCC) → 5 V node; pin 10 (GND) → ground; pin 1 (DIR) → 5 V (A→B); pin 19 (OE) → ground.
- ESP32 **GPIO22** → chip pin 3 (A2). Chip pin 17 (B2) → **plain jumper, no resistor** → breakout **pin 5** (range RX). A 1 kΩ series resistor here forms a divider against the 683 Ω pull-up and the low level only reaches ~3.1 V — an invalid low. GE's own design uses 51 Ω.
- Breakout **pin 4** (range TX) → **1 kΩ resistor** → ESP32 **GPIO23**. This resistor stays: it keeps the range's 5 V TX signal from overstressing the 3.3 V input, and this exact path decoded 871 consecutive frames without a framing error during development.
- Breakout **pin 8** → ESP32 GND.
- Breakout **pin 3** (+5 V rail) → ESP32 **5V** pin. This powers everything: ESP32, buffer, and strip share that 5 V node.
- ESP32 **GPIO25** → strip DIN (data enters at the end the strip's arrows point away from). Strip 5V and GND → the same 5 V node and ground.

**Never connect USB power and the rail wire at the same time.** Two 5 V sources will fight through the board. To reflash or read logs: pull the rail wire first, then plug in USB.

Total draw is roughly 300 mA worst case (ESP32 ~150 mA, buffer negligible, 12 red LEDs at 60% ~150 mA). The rail's exact current limit is unmeasured; it powers GE's own Wi-Fi board and technician adapter by design, and 300 mA has been unremarkable in practice.

## Firmware

The complete, working ESPHome configuration is in [`firmware/stove-indicator.yaml`](firmware/stove-indicator.yaml).

```bash
git clone https://github.com/mguaylam/esphome-gea
# adjust the external_components path in the YAML to your clone
esphome run firmware/stove-indicator.yaml
```

The config contains no Wi-Fi or API sections — after flashing, the ESP32 needs nothing but the range. The serial logger stays enabled (921600 baud) for bench debugging; it's silent in normal standalone use because nothing is attached.

## What was found on the bus

Subscribe-all returns 33 ERDs. Identified so far:

| ERD | Size | Notes |
|---|---|---|
| `0x5522` | 32 B | `COOKTOP_STATUS_EXT`. All-off: `0001FF0101FF0101FF0101FF0101FF0101FF0100FF0000FF0000000000000000`. Front right burner lit: byte 0 → `01`, all other bytes unchanged |
| `0x551C` | 2 B | `0x0105`; did not change with a burner |
| `0x5520` | 13 B | All zeros at idle |
| `0x5670`, `0x5674`, `0x5675` | — | Present, undecoded |

ERD names are per the gehomesdk project's register research; the cooktop identification here was confirmed empirically by toggling burners, not by trusting the name — via the cloud SDK this same register is known to read stuck-off on a sibling Café model while the app shows "On", which is why this project reads it on the local bus and verified it by diffing.

## Safety

- This is a **technician service interface**, not a user feature. Using it is at your own risk and may affect your warranty.
- You're working next to a **gas appliance**. Nothing in this project touches gas lines, valves, or ignition — it only plugs into a low-voltage data port — but if any step makes you unsure, stop.
- **Kill the breaker for resistance/continuity measurements.** A meter in Ω mode injects its own current; readings on a live circuit are meaningless and can damage the meter. Voltage measurements are done powered, resistance measurements unpowered — never mix them up.
- Pin 3 is a **power rail**, not a signal. Never connect it to a GPIO.
- The ESP32's **3V3 pin must never see the 5 V rail** — land rail power only on the pin marked 5V/VIN, which feeds the onboard regulator.
- Not affiliated with or endorsed by GE Appliances. "GE", "Café", and "SmartHQ" are their trademarks.

## Credits

- [esphome-gea](https://github.com/mguaylam/esphome-gea) by mguaylam — the GEA hub component that makes the ESP32 side almost trivial.
- GE FirstBuild's public [home-assistant-adapter](https://github.com/geappliances/home-assistant-adapter) schematic — ground truth for the port's power arrangement and buffering.
- The [gehomesdk](https://github.com/simbaja/gehome) project — ERD numbering and names.

## License

MIT — see [LICENSE](LICENSE). The long version of how the pinout and the register were identified (including the mistakes) is in [`docs/reverse-engineering.md`](docs/reverse-engineering.md).
