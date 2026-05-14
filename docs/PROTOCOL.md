# Petkit Mini UART Protocol — Corrections & Findings

This document captures protocol details that diverge from or extend the documentation in [earlynerd/petkit-serial-bus](https://github.com/earlynerd/petkit-serial-bus). All findings verified empirically by sending packets and observing both M0 responses and physical device behavior.

## Packet structure

Standard frame:

```
AA AA <len> <type> <seq> <payload...> <crc16_hi> <crc16_lo>
```

- `AA AA` — sync header
- `<len>` — total frame length in bytes (including header and CRC)
- `<type>` — command/response type
- `<seq>` — sequence number (host increments per command type; M0 echoes in responses)
- `<crc16>` — CRC-16/CCITT, seed `0xFFFF`, polynomial `0x1021`, MSB-first, computed over the entire frame except the CRC bytes themselves

## Status packet byte order (correction)

The original README documents the status payload as `food, door, unknown, ac_raw, ac_mv, bat_raw, bat_mv`. **Empirical observation suggests bytes 5 and 6 are swapped.**

When the door is moved manually to a half-open position, byte 5 (originally documented as "food") changes while byte 6 stays constant. Conclusion:

| Byte offset (in status packet `0x02`) | Original docs | Verified meaning |
|---|---|---|
| 5 | Food level | **Door IR sensor** |
| 6 | Door IR sensor | **Food level IR sensor** |

The voltage fields (bytes 8–15) match the original documentation.

## `0x0D` motor params payload

The original README lists two slightly different versions of the payload for command `0x0D`:

| Source | Bytes (hex) | Byte count |
|---|---|---|
| Boot-sequence capture in README | `00 3C 01 90 F0 12 22 20 1F 4F 01` | 11 |
| Command table in README | `00 3C 01 90 0F 01 22 22 01 F4 0F 01` | 12 |

**The 12-byte version is correct.** It matches both the command-table entry and the working `petkitMaster.py` script (`[0,60,1,144,15,1,34,34,1,244,15,1]`). With `<len>` byte = 0x13 = 19, the math is `5 (header overhead) + 12 (payload) + 2 (CRC) = 19`.

The boot-sequence log entry in the README appears to have a transcription error.

## Init sequence (missing command)

The `petkitMaster.py` reference implementation sends `0x11` (described in the README as "requests data of some sort, replies with type 18") both **before and after** the motor parameter sequence. This is not mentioned in the README's prose description and is easy to miss.

Full working init sequence in order:

```
0x11 (no payload)
0x13 (payload: 0x057E)             — motor params
0x03 (payload: 0x00050005)         — unknown
0x05 (payload: 0x0005)             — unknown
0x04 (payload: 0x00FF00FF)         — unknown
0x06 (payload: 0xFFFF)             — unknown
0x0D (payload: 12 bytes above)     — long parameters
0x11 (no payload)
```

In practice **the device works without re-running the init at every ESPHome boot** — the M0 keeps its state across ESP power cycles. Sending init once is enough.

## `0x0B` dispense command — empirical values

The README's parameter ordering is `duration_8b, distance_8b, direction_8b, motorCurrent_8b`. This is correct. However, the description doesn't note that `direction` is clamped to `[0, 1]` (per the Python script's `constrain(direction, 0, 1)`).

The empirically-working dispense call from `petkitMaster.py`:

```python
dispense(ser, 3, 1, 0, 16)
# duration=3, distance=1, direction=0, current=16
```

Resulting packet on wire:

```
AA AA 0B 0B <seq> 03 01 00 10 <crc_hi> <crc_lo>
```

### What we learned about the parameters

- **`current` (byte 3)** — motor drive current limit. Values around 16 work fine for normal scooping. Higher values (≥255) put the motor in a "free spin" mode that does not stop on its own.
- **`direction` (byte 2)** — 0 or 1. Both work; semantically the two rotation directions of the wheel.
- **`distance` and `duration`** — meaning unclear in this single-scoop regime. Values `(3, 1)` produce one clean scoop. Setting both to 0xFF with `current=0xFF` produces an uninterruptible runaway (TEST 2 in our development logs).
- **`0x0F` (sleep MCU) does work to halt** — but the M0 stops responding until reboot. Not practical for routine stops.
- **Pressing the physical manual-feed button stops a runaway dispense.** This is M0-side native behavior. Driving GPIO13 low from the ESP does NOT replicate this — suggesting the button signal is wired to an M0 input pin we don't have direct access to.

### Why a single calibration value can't predict portion size

Each `0x0B` command produces one "scoop", but the amount of food per scoop depends on:

1. **Scoop position in the sequence** — the first scoop after a rest period fills the wheel compartment fully (~10g). Subsequent scoops back-to-back deliver less (~4–6 g each).
2. **Tank fill level** — empty tank produces smaller scoops (as the user manual warns).
3. **Food density and pellet size** — different brands behave differently.

For accuracy, an adaptive lookup table (portion grams → scoop count) gives much better results than a single `grams_per_scoop` multiplier. The provided ESPHome config implements this.

## Status reporting

The M0 sends unsolicited `0x02` status packets approximately every 5–10 seconds (with `seq=0xFF`). It also responds to `0x01` get-status requests with both an ACK (type `0x01`) and a fresh status frame.

Our parser accepts both, distinguishes by sequence number, and updates Home Assistant entities accordingly.

## Hardware pins (ESP8266 side)

Verified from the `D2_MAIN_V1.1` board:

| Function | GPIO | Notes |
|---|---|---|
| UART TX (to M0) | GPIO1 | TP205 on debug header (labeled `TX0`) |
| UART RX (from M0) | GPIO3 | TP207 (`RX0`) |
| Wi-Fi reset button | GPIO0 | Chassis button `KEY_RST` |
| Manual feed button | GPIO13 | Chassis button `KEY_FUN` |
| M0 reset (?) | GPIO15 | Toggling did not help; uncertain function |
| I²C SDA | GPIO5 | PCF8563 RTC on the bus |
| I²C SCL | GPIO14 | |

Debug header pin labels: `RESET CLK DIO GND TX0 RX0` (top to bottom). `DIO` and `CLK` appear to be for SPI-flash programming (not needed for ESP serial flashing — `KEY_RST` held low at boot puts the chip into download mode).

## Operational notes

- **The USB-UART adapter, when connected to the debug header, prevents the M0 from communicating with the ESP** even when "passive". Always disconnect it for normal operation.
- **5 AA alkaline batteries** are documented in the manual (not 3 as some sources suggest). Rated voltage is 6 VDC, 1 A.
- **Power LED = `0x0E` subcmd 1 (upper)**, food-low LED = `0x0E` subcmd 2 (lower). Per the user manual: food adequate = LED off, food low = constant on, fault = blinking.
