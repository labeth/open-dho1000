# 08 — Front Panel

The DHO1000 front panel (keys, rotary encoders, key LEDs, power switch) is driven by a small panel
MCU on a serial link. The CPU talks to it over **one UART, `/dev/ttyS0`, 115200 8N1**, with a 5-byte
framed protocol. Nothing goes through the Linux input subsystem or the FPGA. The capacitive
touchscreen is separate: an ordinary Linux input device (§8).

One process of the firmware should own `/dev/ttyS0` for both directions: key frames in, LED and
power frames out. The byte protocol in §2–§5 is complete without any Android-specific parts. Every
fact in this spec is **UNVERIFIED** until checked against a ttyS0 capture on a unit (§9).

---

## 1. Devices

| Purpose | Node | Access |
|---|---|---|
| Panel MCU link | `/dev/ttyS0` (RK3399 UART0) | `O_RDWR \| O_NONBLOCK`, raw termios |
| 50 Ω overload status | `/dev/afe_rms` (`afe_rms_gpio.ko`) | `ioctl(fd, 0x7800, &u32)`: the driver writes 4 bytes; the low byte is `ch1 \| ch2<<2 \| ch3<<4 \| ch4<<6` (one GPIO level each, spec 06) |
| Beeper | `/dev/pwm-1` (`beeper_gpio.ko`) | §7 |
| Touchscreen | `/dev/input/eventN` (`focaltech_ts.ko` / `ilitek_ts.ko`) | standard evdev multitouch |

### 1.1 UART configuration

- 115200 baud, 8 data bits, no parity, 1 stop bit, `CREAD | CLOCAL`, no flow control.
- Raw mode: `ICANON`, `ECHO`, `ECHOE` and `ISIG` cleared; no input/output processing.
- `VMIN = 1`, `VTIME = 10`, or non-blocking reads.
- Flush the input queue (`TCIFLUSH`) once after configuring.

A failed write should be logged and the firmware should continue.

---

## 2. Frame format (both directions)

Every message in either direction is exactly 5 bytes:

| Byte | Value |
|---|---|
| 0 | `0x5A` (start) |
| 1 | `cmd` |
| 2 | `data` |
| 3 | `crc7(cmd, data)` |
| 4 | `0xA5` (end) |

`crc7` is **CRC-7/MMC** over the two bytes `cmd, data`: polynomial 0x09, initial value 0, MSB first,
no reflection, no final XOR. The result is 7 bits (0x00–0x7F). Check value for ASCII `"123456789"`
is 0x75.

| cmd, data | crc |  | cmd, data | crc |
|---|---|---|---|---|
| 01, 00 | 0x0B | | 02, 18 | 0x47 |
| 02, 00 | 0x16 | | 02, 98 | 0x06 |
| 03, 00 | 0x1D | | 03, 01 | 0x14 |
| 01, FF | 0x72 | | 03, 05 | 0x30 |
| 02, FF | 0x6F | | 01, 40 | 0x6F |
| 03, FF | 0x64 | | 01, 80 | 0x4A |
| 06, BB | 0x03 | | 05, 03 | 0x3C |

**Receiver rules.** Gather bytes into frames. Accept a frame only if byte 0 is 0x5A, byte 4 is 0xA5
and byte 3 matches the CRC. Data and CRC bytes can themselves be 0x5A (code 90 is 0x5A), so on a
malformed frame drop byte 0 only and search for the next 0x5A from byte 1 of the rejected frame.
Treat a frame lost this way as a release (code 0).

---

## 3. CPU → MCU commands

| cmd | data | Meaning |
|---|---|---|
| `0x01` | LED bitmask, group 1 | Set the state of all 8 group-1 LEDs |
| `0x02` | LED bitmask, group 2 | Set the state of all 8 group-2 LEDs |
| `0x03` | LED bitmask, group 3 (bits 0–2) | Set the state of the group-3 LEDs |
| `0x04` | `trunc(t × 10)`, 0 ≤ t ≤ 25.5 s | Power-LED flash period (effect **UNVERIFIED**) |
| `0x05` | `0x01` | Shut the system down (power off) |
| `0x05` | `0x02` | Restart the system |
| `0x05` | `0x03` | Application loaded: send once when the firmware has finished starting |
| `0x05` | `0x04` | Power-off cancelled (the user dismissed the shutdown request) |
| `0x06` | `0xBB` | Request the panel firmware version |

There is no way to address a single LED. Keep a shadow copy of all three masks and send all three
frames (01, 02, 03, in that order) on every LED change. A set bit means the LED is on.

Start-up order: all three masks 0x00, then `05 03`, then the state masks. All three as 0xFF is a
lamp test. The MCU's reaction to the `0x05` commands (power sequencing, whether it acknowledges)
is **UNVERIFIED**.

### 3.1 LED map

`idx` is the row number of [`appendix/leds.tsv`](appendix/leds.tsv).

| idx | LED | Group | Bit |
|---|---|---|---|
| 0 | CH1 (yellow) | 2 | 3 |
| 1 | CH2 (blue) | 2 | 4 |
| 2 | CH3 (pink) | 2 | 5 |
| 3 | CH4 (dark blue) | 2 | 6 |
| 4 | RUN (green) | 2 | 7 |
| 5 | STOP (red) | 3 | 0 |
| 6 | SINGLE red | 3 | 1 |
| 7 | SINGLE orange | 3 | 2 |
| 8 | not fitted on DHO1000 | 1 | 0 |
| 9 | not fitted on DHO1000 | 1 | 1 |
| 10 | not fitted on DHO1000 | 1 | 2 |
| 11 | Touch lock | 1 | 3 |
| 12 | Knob 1 ▲▼ | 1 | 4 |
| 13 | Knob 2 ◀▶ | 1 | 5 |
| 14 | Slope 1 (rising) | 1 | 6 |
| 15 | Slope 2 (falling) | 1 | 7 |
| 16 | Knob 1 ◀▶ | 2 | 0 |
| 17 | Knob 2 ▲▼ | 2 | 1 |
| 18 | Intensity | 2 | 2 |

There is no blink state in the protocol; a blinking LED is the host toggling its bit.

Suggested run-state patterns (group 2 bit 7, group 3 bits 0–2):

| State | RUN | STOP | SINGLE red | SINGLE orange |
|---|---|---|---|---|
| Running | 1 | 0 | 0 | 0 |
| Stopped | 0 | 1 | 0 | 0 |
| Single armed (waiting) | 0 | 0 | 1 | 1 |

The two SINGLE LEDs light the one key, so drive them together. Slope LEDs: rising = slope 1 only,
falling = slope 2 only, either = both. The knob direction LEDs say which kind of quantity a
multipurpose knob adjusts (◀▶ horizontal, ▲▼ vertical); the Intensity LED marks knob 1 adjusting
waveform intensity. CHn LEDs show the channel's display state.

Example with CH1 and CH2 on, running, rising edge: `5A 01 40 6F A5`, `5A 02 98 06 A5`,
`5A 03 00 1D A5`.

### 3.2 Version query

Send `{5A 06 BB 03 A5}`. The reply is a frame with `cmd = 0x06` and `(data & 0x70) != 0`; the panel
firmware major version is `data >> 4`. Poll for up to 100 × 2 ms; no reply means version 0.

### 3.3 Power-on mode (persistent, not on the UART)

The "power on automatically when mains is applied" setting is not sent over the UART. It is held in
Rockchip vendor storage item 0x11 (4 bytes): `ioctl(/dev/vendor_storage, 0x40047602 /*VENDOR_WRITE*/,
req)` with `req = {u32 tag 0x56524551, u16 id 0x11, u16 len 4, bytes 5A 22 55 A5}` for "on" and the
same with `5A 22 54 A5` for "off". The 4 data bytes have the shape of a panel frame (`cmd 0x22`,
data `0x55`/`0x54`, no CRC). Which component replays them to the MCU at boot is **UNVERIFIED**
(boot loader or MCU-side read).

---

## 4. MCU → CPU key frames

The MCU sends one frame per key event. **Only `data` (byte 2) is the key code.** A frame with
`cmd = 0x06` and `(data & 0x70) != 0` is always the version reply (§3.2) and never a key; for every
other frame `cmd` (byte 1) is ignored. Most key codes have `data & 0x70 != 0`, so key frames must not
use `cmd 0x06` (the MCU's actual key-frame `cmd` value is **UNVERIFIED**, below). `data = 0xFF` means
"no key" and must be discarded.

### 4.1 Press / release

- A key press produces a frame with the key code (§4.2).
- A key release produces a frame with **data = 0x00**. Treat code 0 as "release" and never act on
  it.
- Encoder detents produce one frame per detent with the CW/CCW code. Fast turning produces runs of
  identical codes (§6).
- Do not act on a held key; there is no need for host-side auto-repeat.

**UNVERIFIED (needs a ttyS0 capture):** the `cmd` byte value in key frames, whether encoder pushes
also send a release frame, whether a release frame is sent between consecutive detents, and MCU-side
auto-repeat while a key is held. A host that follows the rules above works for all variants.

### 4.2 Key codes

Codes are scan-matrix coordinates: `code = row | (col << 4)`. Buttons use rows 0–9 of columns 0–2;
encoders use rows 10 (CW), 11 (CCW), 12 (push) of columns 1–7. The DHO1074 panel (user guide
§4.4.1) has exactly the keys below; codes 5, 25, 36 and 41 are unpopulated matrix positions.
DHO1072/1102/1202 (2-channel) panels lack CH3/CH4.

| Code | Key | Code | Key |
|---|---|---|---|
| 1 | Measure | 20 | Auto |
| 2 | Touch Lock | 21 | CH3 |
| 3 | Trigger (menu) | 22 | Clear |
| 4 | CH1 | 23 | CH4 |
| 6 | Cursor | 24 | Single |
| 7 | Slope | 32 | Zoom |
| 8 | CH2 | 33 | RUN/STOP |
| 9 | Math | 34 | Quick |
| 16 | Default | 35 | Acquire |
| 17 | Analyse | 37 | Navigate |
| 18 | Force | 38 | Navigation ▶ (right key) |
| 19 | Ref | 39 | Navigation ■ (middle key) |
| 170 (0xAA) | Power switch | 40 | Navigation ◀ (left key) |

The three navigation keys sit in a row below the Navigate key in the order ◀ ■ ▶ (left to right);
code 38 is the right-hand key, 40 the left-hand key.

### 4.3 Encoders

Each encoder uses three consecutive codes: **base = clockwise detent, base+1 = counter-clockwise
detent, base+2 = push**.

| Base | Encoder |
|---|---|
| 26 | Multipurpose knob 1 |
| 42 | Multipurpose knob 2 |
| 58 | Trigger LEVEL |
| 74 | Horizontal POSITION |
| 90 | Vertical POSITION (active channel) |
| 106 | Vertical SCALE (active channel) |
| 122 | Horizontal SCALE |

Encoder acceleration is the host's job (§6).

### 4.4 50 Ω overload

The overload sense is not a panel key. A channel's 2-bit field in the `/dev/afe_rms` status (§1)
changing from 1 to 0 between two reads means that channel's 50 Ω input is overloaded; the firmware
must then switch the channel to 1 MΩ (spec 06). Poll at 10 ms or faster, in its own thread, so a
slow read never delays key handling. Pass a 4-byte buffer to the ioctl.

---

## 5. Unused protocol surface

None known on the DHO1000 panel beyond §3 and §4.

---

## 6. Host key processing

A host that works with this panel:

1. Reads all pending UART bytes every ≤ 50 ms and extracts valid frames (§2).
2. Merges each run of identical consecutive encoder codes into one event with `count` = the run
   length, and uses `count` as the step multiplier, so a fast spin gives a larger step.
3. Acts on a button once per press (on the press frame; the release frame re-arms the key).

---

## 7. Beeper and panel MCU reset

`/dev/pwm-1` drives the beeper: `ioctl` requests 0x40084101–0x40084104 set the period, set the duty,
start and stop the beeper PWM; the driver returns 0 for any other request and does nothing. There is
no known CPU-controlled reset of the panel MCU; the firmware must not depend on one. **UNVERIFIED:**
whether the board has a panel-reset line at all.

---

## 8. Touchscreen

- Standard Linux multitouch input from `focaltech_ts.ko`; `ilitek_ts.ko` must be inserted before it
  (spec 01 §4 step 10). Panel resolution 1280 × 800; touch coordinates are screen pixels.
- Touch lock (key 2, LED 11) is a firmware function: ignore finger touches while it is on.
- The waveform grid is 10 horizontal × 8 vertical divisions (spec 07 §2.1).

---

## 9. Open (hardware only)

| Item | Measurement that settles it |
|---|---|
| `cmd` byte of key frames; release frame after encoder pushes/detents; auto-repeat when held | Stop every process holding the port (`am force-stop com.rigol.launcher`), `stty -F /dev/ttyS0 115200 raw; cat /dev/ttyS0 \| od -An -tx1 -w5`, then press/hold each key and turn each knob slowly and fast |
| Panel reaction to `05 01..04` and `04 xx` | Same capture while sending them; watch the power LED |
| Who replays the vendor-storage power-on bytes | Toggle the item, power-cycle on mains, capture ttyS0 from boot |
| MCU behaviour while the CPU is off (power key, standby current) | Logic analyser on the UART and the power-enable line |
