# Petkit Fresh Element Mini → ESPHome

Local control of the **Petkit Fresh Element Mini** automatic cat feeder via [ESPHome](https://esphome.io/) and [Home Assistant](https://www.home-assistant.io/). No cloud, no Petkit app, no subscription.

This replaces the ESP8266's stock firmware while keeping the original ARM Cortex M0 motor controller intact. The two MCUs communicate over an internal UART bus that has been reverse-engineered — this project provides a complete working implementation of that protocol in pure ESPHome YAML (no custom C++ component required).

> **Status:** working in production. Door open/close, food dispensing (with adaptive portioning), beep, status sensors, food-low indicator LED, daily intake tracking, and Home Assistant integration all functional.

## Features

- **Feed Now** — dispense a target weight in grams from Home Assistant
- **Step-by-step calibration sequence** triggered by the chassis button
- **Adaptive portion calculation** based on empirical scoop-weight data (non-linear dispenser behavior compensated)
- **Sensors:** food level, door activity, AC adapter connected, battery/system voltage, last feed amount, total fed today (auto-resets at midnight)
- **Automatic indicator LEDs:** power LED on at boot, food-low LED tracks the sensor automatically
- **Manual door control** (open/close) and **beep test** from HA
- **OTA updates** after the first cable flash

## Hardware

- **Device:** Petkit Fresh Element Mini (board marked `D2_MAIN_V1.1`)
- **MCUs inside:**
  - ESP8266EX (ESP-WROOM-02 module, 2 MB flash) — Wi-Fi and high-level logic. **This is what we re-flash.**
  - Nuvoton ISD91230 (ARM Cortex-M0) — drives motors, reads IR sensors. **We leave this untouched** and talk to it over UART.

The two communicate at 115200 baud on the ESP8266's primary UART (GPIO1/GPIO3). The full protocol is documented in [docs/PROTOCOL.md](docs/PROTOCOL.md).

## Quick start

### 1. Back up your stock firmware (mandatory)

Solder wires to the debug header on the main board (TX0 = TP205, RX0 = TP207, GND = TP406) and connect a 3.3 V USB-UART adapter. Then:

```bash
# With KEY_RST pressed and held while powering on, release after 2 seconds
esptool.py --port /dev/tty.usbserial-XXXX --before no_reset --baud 115200 \
    read_flash 0x0 0x200000 stock-firmware-backup.bin
```

Take the read **twice** and verify the SHA-256 hashes match. **Keep this backup safe** — it is the only way back to a working stock device.

> ⚠️ **Disconnect the USB-UART adapter when not flashing.** It loads the bus and prevents the M0 from responding to ESPHome.

### 2. Configure secrets

```bash
cp secrets.yaml.example secrets.yaml
# Edit secrets.yaml with your Wi-Fi credentials and generated keys
openssl rand -base64 32   # for api_encryption_key
```

### 3. Flash via ESPHome

Either using the ESPHome dashboard in Home Assistant, or via CLI:

```bash
esphome compile petkit-mini.yaml
esphome upload petkit-mini.yaml --device /dev/tty.usbserial-XXXX
```

First-time flash needs the USB-UART cable and the flash-mode procedure above. **Subsequent updates work over Wi-Fi OTA.**

### 4. Calibrate

Each `0x0B` dispense command produces roughly one "scoop" of food, but the size varies non-linearly with how many scoops are run in sequence and how full the tank is. The provided `calibration.xlsx` and built-in calibration sequence in HA help you dial in your specific food/density combination.

See [calibration.xlsx](calibration.xlsx) for the procedure.

## What works, what doesn't

✅ **Works**
- Wi-Fi, Home Assistant API, OTA, captive-portal fallback
- Reading status from M0: food sensor, door sensor, AC voltage, battery voltage
- Door open/close (motor moves visibly)
- Dispense (using the protocol values from [earlynerd/petkit-serial-bus](https://github.com/earlynerd/petkit-serial-bus)'s Python script: `duration=3, distance=1, direction=0, current=16`)
- Beep
- Both indicator LEDs (auto-managed)
- Daily fed counter with midnight reset (via SNTP)

⚠️ **Limitations**
- **Portion accuracy is mechanical, not protocol.** The M0 firmware refuses to dispense when the food sensor sees no food. Manual states: *"When the food level is low, the portion size may shrink."* Adaptive lookup compensates somewhat.
- **Tank-state dependence:** scoop size depends on how full the tank is. Recalibrate occasionally if you want precision.
- **Some protocol fields remain unknown.** Specifically the 0x0C "dispense complete" payload bytes 2–5 — likely encoder counts and sensor state, but exact meaning unconfirmed.

## Repository contents

| File | Purpose |
|---|---|
| `petkit-mini.yaml` | The ESPHome config (paste into HA dashboard or use locally) |
| `secrets.yaml.example` | Template for your Wi-Fi/API credentials |
| `calibration.xlsx` | 10-step calibration tracker with auto-averaging formulas |
| `docs/PROTOCOL.md` | Protocol corrections and findings from this project |
| `LICENSE` | MIT |

## Acknowledgements & sources

This project stands on the shoulders of several earlier efforts. The work that made it possible:

### Primary reverse-engineering work

- **[earlynerd/petkit-serial-bus](https://github.com/earlynerd/petkit-serial-bus)** — the foundation. Full UART protocol reverse-engineering, packet-type tables, boot-sequence captures, flash dumps of both MCUs, logic analyzer captures, and the `petkitMaster.py` Python reference implementation. Without this, none of the rest is possible.
  - The README is a good first read, **but the working values are in `petkitMaster.py`** — particularly the `dispense(3, 1, 0, 16)` call which is the only documented working dispense command.
- **[earlynerd/Petkit-feeder-mini-customFW](https://github.com/earlynerd/Petkit-feeder-mini-customFW)** — a related custom-firmware exploration by the same author.

### Community discussion

- **[Home Assistant Community: DIY Petkit feeder local integration via ESPHome](https://community.home-assistant.io/t/diy-petkit-feeder-local-integration-to-home-assistant-via-esphome/338971)** — long-running thread covering several Petkit feeder models. The Fresh Element Solo (single-MCU) discussion there is well-documented; this project tackles the harder Mini variant (dual-MCU) using the same ideas.

### Vendor documentation

- **Petkit Fresh Element Mini official user manual** (model P530) — confirmed several details we'd otherwise have guessed at:
  - "Each serving is approximately 5g" (matches our calibration)
  - "When the food level is low, the portion size may shrink" (explains the non-linear scoop weights)
  - LED behavior: food adequate = off, food low = constant on, fault = blinking
  - Power: 5× AA alkaline + 6 VDC 1 A AC adapter
  - The manual is copyright Petkit and not redistributed in this repository.

### Tooling

- **[ESPHome](https://esphome.io/)** — the framework that makes all of this trivially scriptable in YAML. No external custom component needed; the entire M0 protocol is implemented in inline lambdas thanks to ESPHome's expressive automation syntax.
- **[esptool.py](https://github.com/espressif/esptool)** — flash dumping/writing for the ESP8266.
- **[Home Assistant](https://www.home-assistant.io/)** — the integration layer.

### Corrections this project contributed

Several details that diverge from or extend earlier documentation, all verified empirically (see [docs/PROTOCOL.md](docs/PROTOCOL.md) for the full reasoning):

- **Status packet bytes 5 and 6 are swapped** in the original README's prose description (verified by observing which byte changes when the physical door is moved).
- **The `0x0D` motor params payload is 12 bytes**, not 11. The original README's boot-sequence log entry had a 1-byte transcription difference vs. the command-table entry; the command-table version (and the Python script) is correct.
- **The `0x11` init command is required** (or at least present in the working stock firmware sequence). The README mentions `0x11` exists but doesn't include it in the prose init description; the Python script sends it both before and after motor params.
- **Empirical dispense values** for a single working scoop: `duration=3, distance=1, direction=0, current=16`.
- **The Petkit dispenser is non-linear**: scoops 1-3 deliver ~10 g each, but subsequent scoops back-to-back deliver less (~4-6 g). A single `grams_per_scoop` calibration multiplier therefore cannot fit the full range — adaptive table-based portion sizing works much better.
- **Driving GPIO13 low from the ESP does NOT simulate a manual-feed-button press to the M0.** The physical button stops a runaway dispense (M0-native behavior), but the ESP-side signal alone does not. This suggests the button is wired in a way we don't fully see from the ESP side.

## Safety

This device's M0 has built-in safety logic that refuses motor commands when sensors report invalid states. Don't try to bypass it. If you reassembled the device and sensors aren't reading correctly, fix the wiring rather than spoofing signals.

Always keep your stock firmware backup. The Petkit-Backup/ folder is git-ignored on purpose — back up your own dump separately.

## License

MIT — see [LICENSE](LICENSE).
