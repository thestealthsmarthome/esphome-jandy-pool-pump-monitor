# Jandy VS FloPro Pump Monitor (read-only, RS485 → Home Assistant)

An ESP32 that passively listens on the RS485 bus between a **Jandy iQPUMP01** controller and a **Jandy VS FloPro** variable-speed pump, and publishes RPM, estimated watts, running state, and energy to **Home Assistant** via ESPHome.

**Read-only by design.** The device never transmits. Your iAqualink app keeps full control of the pump. Nothing about the pump's operation changes.

> Provided as-is, not actively maintained. Issues may sit; PRs welcome. If it helps you, great.

## Why this exists

The iQPUMP01 uses Jandy's "i2d" RS485 protocol, which the common Home Assistant path (`flz/iaqualink-py`, and the HA iAquaLink integration built on it) does not support ([open issue #36](https://github.com/flz/iaqualink-py/issues/36)). There's no native way to see pump RPM/power in HA. This fills that gap without touching the pump's control path, so the app still works and there's no risk of fighting the controller for the bus.

## What you get

| Entity | Notes |
|---|---|
| Pool Pump RPM | Commanded RPM, exact |
| Pool Pump Power (est) | **Modeled from RPM**, see caveat |
| Pool Pump Running | RPM > 100 |
| Pool Pump Energy (est) | Lifetime kWh; add to the HA Energy Dashboard |

### The watts caveat (read this)

The pump does **not** report watts on the bus. The iAqualink app calculates watts from RPM using the pump's affinity curve, and this project does the same by interpolating measured (RPM, watts) points. Because watts is modeled rather than metered, **it cannot independently detect mechanical drag** (a failing bearing draws more watts at the same RPM, which a model can't see). Treat power/energy as a good estimate, not a revenue-grade meter.

## Hardware

- **M5Stack ATOM Lite** (ESP32)
- **M5Stack ATOMIC RS485 base** (SP3485EE transceiver + 6–24 V → 5 V regulator; powers the ATOM off the bus 12 V, no separate supply). In stock at Mouser (part A131), RobotShop, M5Stack store.
- Wire to tap the bus + wire nuts/Wago connectors. Outdoor **Cat6/Ethernet cable works great** here (its twisted pairs suit RS485). You land the individual conductors on the screw terminal, there's no RJ45 plug involved.

## Wiring

The Jandy VS FloPro comm port is a **4-conductor cable on a screw terminal bar** (Jandy "6609"), 9600 baud 8N1:

| Jandy wire | Signal | ATOMIC RS485 base |
|---|---|---|
| Red | +12 V | DC-in (+) — powers the ATOM |
| Black | Data+ / A | A |
| Yellow | Data- / B | B |
| Green | COM / GND | GND |

Land your four wires under the **same terminal-bar screws** the iQPUMP01 already uses (you're joining the bus in parallel, right where the stock Jandy comm cable connects). Ferrule the tap wires so the clamp grips both conductors.

### Using Cat6 / Ethernet cable (recommended)

Cat6 is ideal because its twisted pairs improve RS485 noise rejection over an outdoor run. You do **not** use an RJ45 plug; strip the jacket and land the individual conductors. The one rule that matters: keep **Data+ and Data- on the same twisted pair**.

| Cat6 conductor | Signal (Jandy wire) |
|---|---|
| Blue | Data+ / A (Black) |
| White-Blue | Data- / B (Yellow) |
| Brown | +12 V (Red) |
| White-Brown | GND (Green) |

If the cable is shielded (STP), ground the drain wire at **one end only** (the ATOM end) to avoid a ground loop. Plain UTP: nothing to do. At 9600 baud RS485 runs happily over hundreds of feet, so length is a non-issue.

- **Kill the pump breaker before opening the comm compartment.**
- **Do not add a 120 Ω termination resistor.** The bus is already terminated at both ends; a third load corrupts signaling.
- The RS485 base pins are **RX = GPIO22, TX = GPIO19** on the ATOM Lite.

## Flashing

Uses [ESPHome](https://esphome.io/). Easiest path is the **ESPHome Device Builder** add-on in Home Assistant:

1. Copy `jandy-pump-monitor.yaml` into a new device; fill in `secrets.yaml` from `secrets.yaml.example`.
2. First flash over USB-C in **Chrome or Edge** (Safari has no Web Serial). OTA after that.
3. If the serial port doesn't appear, install the CP210x/CH9102 driver for your ATOM and use a data cable, not charge-only.

## The protocol (for the curious / for adapting to other pumps)

Frames: `10 02 <addr> <cmd> <data...> <cksum> 10 03`. Master (iQPUMP01) = `0x78`, pump replies as `0x00`. Checksum = sum of every byte from the `0x10` start through the last data byte, `& 0xFF`.

- **RPM** comes from the master's demand frame `10 02 78 44 00 <lo> <hi> <cksum> 10 03`. Value = `(lo | hi<<8) / 4`. It's the *commanded* speed (what the schedule asked for), not a measured tach.
- **Watts** appears nowhere on the bus (verified across five speeds). The app derives it.
- `46 xx` frames carry model/serial/firmware ASCII strings. `45 01 0C`/`0E` are runtime/energy accumulators.

If your controller or pump model differs, capture your own bus first: flash a bring-up config with a UART `debug:` block that logs hex, watch the frames, and confirm the `0x78`/`0x44` demand frame before trusting the parser.

## Calibrating watts for your pump

Set the pump to several known speeds in iAqualink; note the RPM and watts the app shows at each. Then edit the arrays in the lambda:

```cpp
const float RX[6] = {0, 750, 1499, 2275, 3000, 3450};  // your RPMs
const float WX[6] = {0,  44,  176,  481, 1064, 1589};  // your watts
```

Cover your full operating range so nothing is extrapolated. The example values are for a Jandy VS FloPro 1.85 HP.

## Home Assistant Energy Dashboard

Add **Pool Pump Energy (est)** as an Individual Device. This config uses ESPHome's `integration` platform (millis-based) rather than `total_daily_energy`, which boot-spikes when the device's clock first syncs. The lifetime counter feeds the Energy Dashboard, which derives daily/monthly itself.

## Troubleshooting

- **All sensors show `unknown` while the device is online:** rare silent parser stall. In testing, a soft reboot *and* a full power-cycle did **not** recover it, but a **clean reflash did**. If it happens, reflash. (Optionally add an HA automation that alerts when the device is up but the sensors are stale for ~10 minutes.)
- **No data at all:** confirm you see hex frames containing `78` on the bus (bring-up `debug:` config), check A/B aren't swapped, and confirm 9600 8N1.
- **Watts looks off at a speed you didn't calibrate:** add that (RPM, watts) point to the arrays.

## Credits

Protocol groundwork from the [AqualinkD](https://github.com/aqualinkd/AqualinkD) project, the [flz/iaqualink-py](https://github.com/flz/iaqualink-py) issue tracker, and the Jandy pump protocol threads on Trouble Free Pool.

## License

MIT. See `LICENSE`.
