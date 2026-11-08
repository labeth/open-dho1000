# 09 — Control Plane

The control plane (the instrument's settings, run control, persistence of settings, and how panel,
touch, web and SCPI requests reach them) is open-dso-core's. This spec gives only what is specific to
the DHO1000: where its inputs come from, what per-unit storage exists and must be preserved, and the
model and bandwidth identity. The order in which settings reach the hardware (staging, register
write sequences) is owned by specs 02–06.
Nothing in this spec has been checked on a unit: every value and behaviour is **UNVERIFIED** until it is.


```
 panel MCU ──ttyS0──► key frames (spec 08) ──►┐
 touch ──evdev─────────────────────────────►│  control plane (open-dso-core)
 web remote ───────────────────────────────►│        │
 SCPI (VXI-11 / 5555 / USB-TMC / GPIB) ────►┘        ├──► FPGA / AFE / DAC / clocks (specs 02–06)
                                                     ├──► I²C FRAM (settings, §4.2)
                                                     └──► ttyS0 (LEDs, power frames, spec 08)
```

---

## 1. Inputs

Every input source is equal: panel keys and encoders (spec 08), touch (spec 08 §8), the web remote
and SCPI (spec 11) change the same settings. Settings depend on each other (scale limits, memory
depth, bandwidth), so a front end shows the value the control plane reports back, not the value it
asked for.

Quantities the hardware specs use: times in seconds (the FPGA works in 1 ns or 4 ns ticks, spec 04),
voltages in volts (the AFE in DAC codes, spec 06). The waveform grid is 10 horizontal × 8 vertical
divisions: the time of grid line i is `(i − 5) × s/div + delay`, the voltage of grid line j (0 = top)
is `(4 − j) × V/div − offset`.

---

## 2. Interface operations

Owned by open-dso-core. Nothing unit-specific.

---

## 3. Life cycle

1. The hardware bring-up of spec 01 §4 (FPGA, clock chip, PCIe) completes.
2. The firmware loads its stored settings (§4) and applies them to the hardware.
3. It sends the panel frame `05 03` ("application loaded", spec 08 §3), then the LED state.
4. Power off: the POWER key (code 170, spec 08) asks for confirmation; confirming sends `05 01`
   (shutdown) to the panel MCU, cancelling sends `05 04`. Restart sends `05 02`.

---

## 4. State and persistence

### 4.1 Where per-unit data lives

| What | Where | Rule |
|---|---|---|
| Settings | the I²C FRAM (§4.2), or a file on `/data` | the firmware's own format |
| User files | `/data/UserData/`; USB sticks under `/mnt/media_rw/<id>/` | — |
| Calibration | `/rigol/data/cal_*.hex`, fall-back `/rigol/data/default/cal_*.hex` (spec 10) | read only; never overwrite |
| Probe data | `/rigol/data/probe/` | preserve |
| Device identity | `/rigol/data/vendor.bin`; `/dev/vendor_storage` items (serial number, model, MAC) | read only |
| Power-on-with-mains setting | vendor storage item 0x11 (spec 08 §3.3) | — |

Back up the FRAM, `/rigol/data/` and the vendor storage items before the first write to any of them
(spec 01 §2).

### 4.2 FRAM

A non-volatile I²C FRAM holds settings that must survive power-off:

| Property | Value |
|---|---|
| Bus / address | `/dev/i2c-4`, 7-bit slave address **0x50** |
| Size | 8 KiB (0x2000) |
| Access | `I2C_RDWR` (ioctl 0x0707); a write message is `{addr_hi, addr_lo, data…}` (16-bit big-endian byte address); a read is a 2-byte address write followed by a read message |

The FRAM of a unit holds the vendor firmware's settings. An open firmware may use the FRAM with its
own format, protected by a checksum, and must treat a block whose checksum does not match as "no
stored state" (defaults). It should keep its data in a range it owns and write the FRAM only after
the original content has been backed up, so the unit can go back to the vendor firmware with its
settings intact. **UNVERIFIED** on a unit.

FRAM writes are byte-addressed and do not wear out in practice; writing the settings about 1 s after
the last change is enough.

### 4.3 Save and restore rules

The firmware's choice (open-dso-core). Restoring the last settings at power-on, with a choice of
defaults instead, matches the public `:SYSTem:PON` command (LATest / DEFault).

### 4.4 Default setup

The firmware's choice. The DHO1000 user guide lists the instrument's default setup (Default key);
following it keeps the open firmware familiar to users.

---

## 5. Front-end bindings

### 5.1 Panel keys

Key and encoder codes are in spec 08 §4. What each key does is the firmware's choice; the key labels
(user guide §4.4.1) say what users expect.

### 5.2 Touch

Touch reports screen pixels (spec 08 §8); what gestures do is the firmware's choice.

### 5.3 LEDs

The LED masks (spec 08 §3.1) should be derived from the state model in one place, and all three
masks resent on every change.

---

## 6. Options and bandwidth

### 6.1 Option state

An open firmware decides its own feature set. It does not read or verify vendor licences.

### 6.2 Bandwidth

| Model | Channels | Analog bandwidth |
|---|---|---|
| DHO1072 | 2 | 70 MHz |
| DHO1074 | 4 | 70 MHz |
| DHO1102 | 2 | 100 MHz |
| DHO1104 | 4 | 100 MHz |
| DHO1202 | 2 | 200 MHz |
| DHO1204 | 4 | 200 MHz |

(Public data sheet.) The public programming guide lists OFF, 20M and 250M for
`:CHANnel<n>:BWLimit` (shared with the DHO4000); below 250 MHz of analog bandwidth only the 20 MHz
limit changes anything.

How bandwidth becomes hardware (AFE band-limit filter selection, calibration file
`cal_afe_bandwidth.hex`, any digital filter) is owned by specs 06 and 10. **UNVERIFIED:** whether
the 70 MHz limit is purely an AFE filter setting; confirm with a swept sine.

### 6.3 Model identity

The model is the unit's identity: read it, with the serial number, from `/dev/vendor_storage`
(spec 01 §2) and report it as is in `*IDN?`, USB descriptors and the UI. If the item is missing,
the board HW code gives the channel count (spec 13 §2). **UNVERIFIED** on a unit.

---

## 7. Remaining open points

| Item | Measurement |
|---|---|
| FRAM content of a retail unit, and which range is free | Dump `/dev/i2c-4` @0x50 (8 KiB) with `i2ctransfer` as root |
| Model string and serial number items in vendor storage | Read every `/dev/vendor_storage` item on a unit |

## 8. Machine-readable catalogue

None in this repository. The front-panel tables are [`appendix/keys.tsv`](appendix/keys.tsv) and
[`appendix/leds.tsv`](appendix/leds.tsv) (spec 08).
