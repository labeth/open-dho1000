# 06 — Vertical and Analog Front End

This document specifies the CPU-side control of the analog path of the DHO1000 (S2 board, one ADC):
the per-channel RT1642 AFE chips, the auxiliary dual DAC, the per-channel offset-gear lines, the
LMK04821 clock generator, the ADC (RT8847) register interface, the 50 Ω overload inputs and the
hardware-code straps, plus how a vertical setting (V/div, offset, coupling, bandwidth, invert, probe)
is turned into register values. Everything here except the ADC is driven from RK3399 GPIOs by
bit-banging kernel drivers; the ADC is programmed through an FPGA SPI bridge. Calibration data that
feeds the values is in `10-calibration.md`; FPGA registers named here are in `02-register-map.md`.

GPIO numbers are Linux global GPIO numbers (RK3399: `bank*32 + group*8 + pin`, group A=0..D=3).
Voltages are in nV and V/div in nV/div unless stated otherwise.

**Nothing in this spec has been measured on a unit; every value, sequence and timing is UNVERIFIED**
until it is. Items marked **UNVERIFIED** inline are open even in the description itself.

## 1. Control map

| Control | Device node | Chip / function |
|---|---|---|
| Per-channel AFE registers (path, gains, coupling, filters, offset DAC, trims, e-fuse) | `/dev/spi_simulate`, slaves 0..3 | RT1642 AFE, one per channel (§2) |
| Auxiliary dual 10-bit DAC (reference-oscillator trim) | `/dev/spi_simulate`, slave 4 | TLV5617A-compatible (§3) |
| Aux-DAC source switch | `/dev/spi_simulate` ioctl `0x7802` | GPIO `dac_insw1_control` (§3) |
| Per-channel offset gear (3 lines / channel) + enable | `/dev/dac_gear_change` | GPIO bank (§5) |
| Clock tree | `/dev/spi_3wires` | TI LMK04821 (§4) |
| ADC registers | FPGA BAR `0x1034` / `0x1038` | RT8847 (§8) |
| 50 Ω overload status (4 inputs) | `/dev/afe_rms` | GPIO inputs (§6) |
| Board/FPGA variant straps | `/dev/hdcode_gpio` | GPIO inputs (§7) |
| EXT trigger level DAC | FPGA BAR `0x1404` | FPGA-driven DAC (§9.7) |

The channel **offset DAC is inside the AFE** (AFE register `0x28`, §2.5). FPGA registers
`0x1418 + 4*ch` are written once with `0x8000` at start-up and are not used for the vertical offset.

All nodes are misc devices with dynamic minors, mode 0777 after boot.

## 2. `/dev/spi_simulate` and the RT1642 AFE

### 2.1 Pins

| Signal | GPIO | Ball | Direction / idle |
|---|---|---|---|
| CS slave 0 (AFE CH1) | 89 | GPIO2_D1 | out, idle high |
| CS slave 1 (AFE CH2) | 88 | GPIO2_D0 | out, idle high |
| CS slave 2 (AFE CH3) | 83 | GPIO2_C3 | out, idle high |
| CS slave 3 (AFE CH4) | 87 | GPIO2_C7 | out, idle high |
| CS slave 4 (aux DAC SYNC) | 90 | GPIO2_D2 | out, idle high |
| SCLK | 86 | GPIO2_C6 | out |
| MOSI | 85 | GPIO2_C5 | out |
| MISO | 84 | GPIO2_C4 | in |
| `dac_insw1_control` | 132 | GPIO4_A4 | out, initialised high |

Channel `n` (0 = CH1) is addressed as slave `n`. Whether CS 89 is wired to the CH1
connector is **UNVERIFIED** (measure: touch CH1 with a signal while writing a distinctive gain to
slave 0 only).

### 2.2 Driver interface

| Call | Effect |
|---|---|
| `ioctl(fd, 0x7800, n)` | select slave `n` (0..4) for the following transfers. Slave 4 also sets SPI mode 1; slaves 0..3 set mode 0. |
| `ioctl(fd, 0x7801, m)` | force SPI mode `m` (0..3) |
| `ioctl(fd, 0x7802, v)` | drive `dac_insw1_control` high if `v == 1`, else low |
| `write(fd, buf, n)` | one frame: `n` bytes from `buf`, in order, MSB first. Returns `n`. |
| `read(fd, buf, n)` | one frame: send `buf[0]`, then clock in `n` bytes into `buf[0..n-1]`. Returns `n`. Fails (−1) on slave 4. |

The selection persists and the driver keeps no per-file state; a process performs select + transfer
without interleaving.

### 2.3 Wire timing

Half-period 5 µs (≈100 kHz), MSB first, bytes back-to-back inside one CS assertion.

| Mode | Idle SCLK | MOSI change | Slave samples |
|---|---|---|---|
| 0 (AFE) | low | while SCLK low | rising edge |
| 1 (aux DAC) | low | before the rising edge | falling edge |

Read (mode 0): per bit SCLK low, 5 µs, SCLK high, sample MISO, 5 µs. Frame for slaves 0..3: CS low,
5 µs, bytes, 5 µs, CS high, 5 µs, then one extra SCLK pulse with CS high (high then low, 5 µs each).
Frame for slave 4: one SCLK pulse with SYNC high, SYNC low, 5 µs, bytes, 5 µs, SYNC high. An
implementation reproduces the trailing pulse; whether the AFE needs it is **UNVERIFIED**.

### 2.4 AFE register protocol

- Registers are 16 bit. Write: `{addr, data >> 8, data & 0xFF}` (24 clocks, `addr` bit 7 = 0).
- Read: send `0x80 | addr`, then 24 clocks; the value is the first 16 bits received (MSB first). With
  the driver: `buf[0] = 0x80 | addr`, `read(fd, buf, 3)`, value = `buf[0] << 8 | buf[1]`.
- Only address `0x00` (the status word, §2.5) is ever read. Configuration registers are never read
  back: the host keeps a shadow of every register per channel and always writes whole registers.

### 2.5 Register map

Register `0x00` is read-only status; `0x06`, `0x07` and `0x20..0x37` are written. "Init" is the value
written at power-up (§2.7). Field names are descriptive.

**`0x00` STATUS (read)**

| Bits | Meaning |
|---|---|
| 0 | e-fuse data bit (valid during a fuse read, §2.8) |
| 1 | comparator of the attenuated 1 MΩ first stage; with `0x26[15:11] = 01011` it compares the VGA output instead (VGA zero on every path) |
| 2 | comparator of the direct 1 MΩ first stage |
| 3 | second ("HP") comparator; input selected by `0x2A[6:5]`, offset trim `0x07[6:0]` |
| 4 | overload trip: 1 = the 50 Ω path has tripped (chip fell back to the 1 MΩ path) |
| 15:13 | chip revision (`(status >> 12) & 0xE`) |

The zero calibration (spec 10 §11.7) treats a comparator bit = 1 as "lower the trim search index"; the
physical sign is **UNVERIFIED**.

**`0x06` MUXZERO** (init `0x1D00`) — `[6:0]` MUX zero trim A, `[14:8]` MUX zero trim B; bits 7, 15
preserved. A and B are one-sided 7-bit trims of opposite direction; after a calibration at most one is
non-zero.

**`0x07` INPUT** (init `0x8440`)

| Bits | Field | Values |
|---|---|---|
| 15 | COUPLING | 1 = DC, 0 = AC |
| 14:13 | PATH | `10` 1 MΩ direct (gain 0.98); `00` 1 MΩ through the attenuator (`0x20[2:0]`); `11` 50 Ω, input gain 0.5; `01` 50 Ω, input gain 0.05. Bit 13 = 50 Ω select; bit 14 = high-gain select. With bit 13 = 1 the effective path is `bit14 | (STATUS.4 ? 0 : 1) << 13`, i.e. a tripped overload drops to 1 MΩ by hardware. |
| 12:8 | OV_TRIP | overload-trip threshold code (init 4). The 0..23 level index maps to the field as `i` for `i < 6`, `i + 4` for `6 ≤ i < 20`, `i + 8` for `20 ≤ i < 24` (5 bits) |
| 6:0 | CMP_TRIM | offset trim of the STATUS bit 3 comparator, 7 bit, mid `0x40` (init); searched by the zero calibration (spec 10 §11.7), not stored. A self-test value for the whole register is `0x0060` |

**`0x20..0x37`** (shadow index = address − `0x20`)

| Addr | Init | Fields |
|---|---|---|
| `0x20` | `0x0008` | `[2:0]` 1 MΩ attenuator: `0` gain 0.24, `4` gain 0.05215, `6` gain 0.00518 (only on PATH `00`); `[6:4]` BW trim T6; `[15:13]` set to `101` when the 0.24 attenuator is selected |
| `0x21` | `0x0A91` | `[6:0]` BW trim T8, `[11:7]` BW trim T7, `[15:12]` BW trim T9 |
| `0x22` | `0x5A00` | `[7:0]` BW trim T0, `[15:8]` BW trim T1 |
| `0x23` | `0x1000` | `[13:11]` 1 MΩ front-end select: `101` direct, `010` attenuated, `000` off (50 Ω in use); `[9:0]` + `[10]` zero trim "first stage" of the direct path (magnitude, sign; monotonic order: `[10] = 1` magnitude `0x3FF` … 0, then `[10] = 0` magnitude 0 … `0x3FF`) |
| `0x24` | `0x0000` | `[9:0]` BW trim T10 (direct path); bits 10, 14 = 1: direct first-stage zero-cal self-check (STATUS bit 2) |
| `0x25` | `0x0000` | `[9:0]` BW trim T11; `[10]` power mode |
| `0x26` | `0x3200` | `[9:0]` + `[10]` zero trim "first stage" of the attenuated path (sign/magnitude order as `0x23`); `[15:11]` zero-cal mode: `10011` attenuated first-stage self-check (STATUS bit 1), `01011` VGA zero (STATUS bit 1 compares the VGA output); operating value `00110` (from the init `0x3200`; every trim write preserves it). The AFE zero calibration (spec 10 §11.7) leaves `00010` (effect **UNVERIFIED**) |
| `0x27` | `0x0044` | `[6:0]` SGA zero trim B, `[14:8]` SGA zero trim A (1 MΩ; one-sided trims of opposite direction, at most one non-zero after a calibration); bits 15 and 7 = offset-DAC enable (init sets both; always written together) |
| `0x28` | `0x0000` | **OFFSET_DAC**, 16-bit offset DAC code, mid-scale `0x8000` (§9.3) |
| `0x29` | `0x0282` | `[2:0]` test-point-1 select, `[3]` test-point route (during VGA zero `[3:0]` = `0000` direct VGA 25, `0001` direct VGA 2.5, `1000`/`1001` 50 Ω input 0.5 VGA 25/2.5, `1010` 50 Ω input 0.05), `[5:4]` temperature-sense current, `[7:6]` ATT (`1` = 0.5, `2` = 1.0), `[9:8]` VGA (direct path: `0` = 25, `1` = 2.5; attenuated path: `2` = 2.5; `3` = path off), `[15:10]` VGA 2.5 zero trim (high bits; low bits `0x35[11:9]`) |
| `0x2A` | `0x3C40` | `[2:0]` e-fuse bit address (word 5); `[6:5]` test-point-2 / STATUS bit 3 comparator input select (`10` SGA 1 MΩ, `00` SGA 50 Ω, `01` MUX, `11` with `[8]` = 1 comparator self-check); `[7]` test-point-2 enable; `[8]` comparator self-check (left at 1 after the zero calibration); `[9]` e-fuse strobe; `[11:10]` BW trim T3; `[13:12]` SGA gain (direct: `0` = 2.5, `1` = 0.9; attenuated: `2` = 2.5, `3` = 0.9); `[15:14]` SGAR (`0` = 1, `1` = 0.714, `2` = 0.524, `3` = path off) |
| `0x2B` | `0x4C00` | `[3:0]` e-fuse bit address, `[6:4]` e-fuse word select, `[9:8]` written `11` during the e-fuse read and after the attenuated first-stage zero calibration (meaning **UNVERIFIED**: a register-type select rather than a pure e-fuse access enable), `[10]` output buffer A enable, `[11]` output buffer B enable, `[12]` e-fuse program strobe, `[13]` test-point-1 to output, `[14]` buffer mode. The output buffer stage has gain 3 when bit 10 or 11 is set. |
| `0x2C` | `0x000C` | 50 Ω path: `[1:0]` SGA (`0`/`2` = 5.0, `1`/`3` = 1.9, see §2.6), `[3:2]` always 1, `[9:4]` VGA zero trim, `[11:10]` VGA (`0` = 25, `1`/`2` = 2.5), `[13:12]` temperature-sense current, `[15:14]` ATT (`1` = 0.5, `2` = 1.0; `11` = off) |
| `0x2D` | `0x0000` | 50 Ω path: `[6:0]` SGA zero trim B, `[13:7]` SGA zero trim A (as `0x27`), `[15:14]` SGAR (as `0x2A[15:14]`; `11` = off) |
| `0x2E` | `0x1FF9` | `[5:0]` filter F, `[7:6]` filter A, `[8]` overload-protection enable, `[10:9]` filter B, `[13:11]` MUX gain (1 MΩ: `2` = 0.7, `3` = 1.05; 50 Ω: `0` = 0.7, `1` = 1.05), `[15:14]` MUXR (`0` = 1, `1` = 0.9, `2` = 0.85) |
| `0x2F` | e-fuse word 0 \| `0x000C` | `[0]` 50 Ω enable, `[3:2]` always 1, `[6:5]` BW trim T2, `[15:7]` VGA zero trim (1 MΩ) |
| `0x30` | `0x0000`, then e-fuse word 1 | frequency-flatness trim |
| `0x31` | `0x0000`, then e-fuse word 2 | frequency-flatness trim |
| `0x32` | `0xC000`, then (e-fuse word 3 & `0x3FFF`) \| `0x8000` | `[13:8]` VGA zero trim (alt.), `[15:14]` flatness select |
| `0x33` | `0x0088`, then e-fuse word 4 | `[3:0]`, `[7:4]` buffer zero trims (e-fuse value only; never calibrated) |
| `0x34` | `0x0009`, then e-fuse word 5 (5 bits) | flatness trim |
| `0x35` | `0x0000` | `[8:0]` 1 MΩ VGA 25 zero trim; `[11:9]` 1 MΩ VGA 2.5 zero trim (low bits) |
| `0x36` | `0x0000` | 50 Ω VGA zero trim (low bits) |
| `0x37` | `0xC000` | `[15:12]` BW trim T5 |

VGA zero trims are a segmented 9-bit code, `[8:6]` coarse and `[5:0]` fine, one coarse step ≈ 23 fine
steps (**UNVERIFIED**; inferred from the calibration's search mapping, which above coarse 0 uses only
fine 41…63, so that the search index is linear in the trim; spec 10 §11.7).

The chip has no public datasheet; the field names are our own descriptions and their meanings are
**UNVERIFIED**. Bits not listed are written as preserved from the shadow.

### 2.6 Gain stages

The signal chain is input → (1 MΩ: direct buffer or attenuator / 50 Ω: input stage) → ATT → VGA →
SGA → SGAR → MUX → MUXR → output buffer (×3). The nominal gain of a setting is the product of the
stage gains in §2.5. Seven **setup codes** `s0..s6` describe a complete vertical gear and are what the
calibration records store (spec 10 §3.2):

| Code | 1 MΩ meaning | 50 Ω meaning |
|---|---|---|
| s0 INPUT | `0` direct (0.98), `1` att. 0.24, `2` att. 0.05215, `3` att. 0.00518 | `0` input 0.5, `1` input 0.05 |
| s1 VGA | `0` = 25, `1` = 2.5 (direct); `2` = 2.5 (attenuated) | `0` = 25, `1` = 2.5 (input 0.5); `2` = 2.5 (input 0.05) |
| s2 ATT | `1` = 0.5, `2` = 1.0 | same |
| s3 SGA | `0` = 2.5, `1` = 0.9 (direct); `2` = 2.5, `3` = 0.9 (attenuated) | `2` = 5.0, `3` = 1.9 (input 0.5); `0` = 5.0, `1` = 1.9 (input 0.05) |
| s4 SGAR | `0` = 1, `1` = 0.714, `2` = 0.524 | same |
| s5 MUX | `2` = 0.7, `3` = 1.05 | `0` = 0.7, `1` = 1.05 |
| s6 MUXR | `0` = 1, `1` = 0.9, `2` = 0.85 | same |

Check value: s = (0,0,2,0,0,3,1) gives 0.98·25·1·2.5·1·1.05·0.9·3 = 173.6, the maximum 1 MΩ gain.
For every record of the shipped default file, `native_scale × nominal_gain` lies within
0.174…0.205 V/div (1 MΩ 0.1747–0.2015, 50 Ω 0.1789–0.2047), i.e. one division is ≈0.19 V at the AFE
output and the ADC full scale (65 536/7 500 = 8.74 div) is ≈1.66 V.

### 2.7 Programming sequences

**Power-up, per channel** (in this order):

1. Write the init values of `0x20..0x37` in address order (table §2.5, `0x28 = 0x0000`, `0x2F = 0x5980`,
   `0x30 = 0x0000`, `0x31 = 0x0000`, `0x32 = 0xC000`, `0x33 = 0x0088`, `0x34 = 0x0009`).
2. `0x06 = 0x1D00`, `0x07 = 0x8440`, `0x27 = 0x80C4` (init | `0x8080`), `0x28 = 0x8000`.
3. `0x2B |= 0x0300` (e-fuse access), read e-fuse words 0..5 (§2.8) into the shadows of
   `0x2F, 0x30, 0x31, 0x32, 0x33, 0x34`, then `0x2B &= ~0x0300`.
4. `0x32 = (word3 & 0x3FFF) | 0x8000`; write `0x2F = word0 | 0x000C`, `0x30..0x34`.

After all channels the start-up continues as in §10. Its AFE-related steps, in order: FPGA offset
registers `0x1418 + 4*ch = 0x8000`, CH1 set to 100 mV/div (gear change below), and after the
acquisition start `1 MΩ enable` (§2.9) on all four channels (effect on CH1: §10 step 12).

**Vertical gear change (1 MΩ)**, for setup `s0..s6` (the record selected per §9.1):

1. ADC input switches parked (§8.4: both ADCs `0x54 = 0x4210`, `0x55 = 0x0008`).
2. 50 Ω disable (§2.9), then 1 MΩ enable in the variant of the path currently in effect: `0x07[14]`,
   plus, when `0x07[13] = 1`, a STATUS read giving bit 13 = NOT STATUS.4 (§2.5). Effective `10` gives the
   direct variant, `00` the attenuated one; on an untripped 50 Ω path nothing is written. Steps 3–7
   then set every field, so the variant only affects intermediate frames.
3. INPUT s0: `0x07[14:13]` = `10` for s0 = 0 else `00`; `0x23[13:11]` = `101` for s0 = 0 else `010`;
   for s0 ≥ 1 `0x20[2:0]` = 0, 4, 6 (s0 = 1, 2, 3) and for s0 = 1 also `0x20[15:13] = 101`. Write
   `0x07`, then `0x20` (s0 ≥ 1 only), then `0x23`. `0x20[15:13]` is never cleared again (it stays
   `101` after a later s0 = 2/3).
4. VGA s1 → `0x29[9:8]` (s1 < 2 also forces `0x07[14:13] = 10`, s1 = 2 forces `00`); write `0x07`, `0x29`.
5. ATT s2 → `0x29[7:6]`, write `0x29`.
6. SGA s3 → `0x2A[13:12]` (s3 < 2 forces PATH `10`, else `00`); write `0x07`, `0x2A`.
7. SGAR s4 → `0x2A[15:14]`; MUX s5 → `0x2E[13:11]`; MUXR s6 → `0x2E[15:14]`; write each.
8. Output buffers: `0x2B[10] = 0`, then `0x2B[11] = 1` (written twice, one bit per write).
9. Offset gear (§5): code for gear `s0 + 1`.
10. Zero trims for the active gear class (spec 10 §5.1), in this order: first stage `0x23`/`0x26`,
    VGA `0x35` (or `0x29` then `0x35`), SGA `0x27`, MUX `0x06`. They are written even without a
    calibration file (all 0, replacing the power-up values of `0x06` and `0x27[14:8]`, `[6:0]`).
11. Bandwidth trims for s0 (spec 10 §5.2), one write each in this order: s0 ≥ 1 → `0x22[15:8]` (T1),
    `0x20[6:4]` (T6), `0x21[11:7]` (T7), `0x21[6:0]` (T8), `0x21[15:12]` (T9); s0 = 0 → `0x24[9:0]`
    (T10), `0x25[9:0]` (T11).
12. ADC input switches restored to the current channel mode (§8.4; before the first channel-mode
    selection the restore value is `0x2108`/`0x0004`).
13. Offset DAC re-written (§9.3).

**Vertical gear change (50 Ω)**: as above with 1 MΩ disable / 50 Ω enable, INPUT s0 → `0x07[14:13]`
(`11` for s0 = 0, `01` for s0 = 1) and `0x2F[0] = 1`; VGA s1 → `0x2C[11:10]`; ATT s2 → `0x2C[15:14]`;
SGA s3 → `0x2C[1:0]`; SGAR s4 → `0x2D[15:14]`; MUX/MUXR as 1 MΩ; buffer mode `0x2B |= 0x4C00`;
offset gear (§5) code for gear 0 below 500 µV/div else `s0 + 1`; no bandwidth trims.

### 2.8 E-fuse read

Words `w` = 0..5, each read as follows (every step is a separate frame, even when the value does not
change):

1. `0x2B |= 0x0100`; `0x2B |= 0x0200`; `0x2A &= ~0x0200`; `0x2B[6:4] = w`.
2. Per bit `k` (16 bits for words 0..4, 5 bits for word 5, LSB first): write the bit address
   (`0x2B[3:0] = k`, or `0x2A[2:0] = k` for word 5); `0x2B &= ~0x1000`; `0x2A &= ~0x0200`; read
   STATUS and take bit 0 as fuse bit `k`.
3. `0x2B &= ~0x0100`; `0x2B &= ~0x0200`.

The read leaves the last addresses in the shadows: `0x2B[6:4] = 5`, `0x2B[3:0] = 0xF` and
`0x2A[2:0] = 4`. Leave them there, so every later `0x2A`/`0x2B` frame carries them (the effect of
clearing them is **UNVERIFIED**).

`0x2B[12]` and `0x2A[9]` are the fuse programming strobes; a firmware must never use them to program
fuses.

### 2.9 Path enable

| Operation | Writes |
|---|---|
| 1 MΩ enable, direct (PATH `10`) | `0x23[13:11] = 101`, `0x29[9:8] = 00`, `0x2A[15:12] &= 0x0FFF` |
| 1 MΩ enable, attenuated (PATH `00`) | `0x23[13:11] = 010`, `0x29[9:8] = 10`, `0x2A[15:12] = 0010` |
| 1 MΩ disable | `0x23[13:11] = 000`, `0x29[9:8] = 11`, `0x2A[15:14] = 11` |
| 50 Ω enable | `0x2C = (0x2C & 0x33FC) \| 0x0402` (input 0.5) or `\| 0x0800` (input 0.05); `0x2D[15:14] = 00`; `0x2F[0] = 1` |
| 50 Ω disable | `0x2F[0] = 0`, `0x2C \|= 0xCC00`, `0x2D \|= 0xC000` |

## 3. Auxiliary dual 10-bit DAC (slave 4): reference-oscillator trim

The frame is 16 bits, mode 1, framed by SYNC (CS 90):

| Bits | 15 | 14 | 13 | 12 | 11..2 | 1..0 |
|---|---|---|---|---|---|---|
| Field | R1 (0 = write B and buffer, 1 = write A, update B from buffer) | SPD = 0 | PWR = 0 | R0 = 0 | value[9:0] | 0 |

This is the TI TLV5617A / TLC5617 command format; the part number is **UNVERIFIED** (read the package
marking). Writing both outputs: `{v1 >> 6 & 0x0F, v1 << 2}` (B), then `{0x80 | v2 >> 6 & 0x0F, v2 << 2}`
(A, both outputs update), then `ioctl(0x7802, 1)`.

The two codes are the two values of `cal_clock` (spec 10 §8; defaults 300 and 360) and are written
once at start-up. Both outputs trim the 10 MHz reference oscillator that feeds the LMK04821 OSCin (§4.4)
(coarse/fine assignment **UNVERIFIED**). `dac_insw1_control` selects the reference source: 1 = internal
oscillator (default), 0 = external reference; the analog function of the switch is **UNVERIFIED**
(measure OSCin frequency and the trim-DAC pins with and without a 10 MHz signal on the reference input).

## 4. LMK04821 clock generator: `/dev/spi_3wires`

### 4.1 Pins

| Signal | GPIO | Ball | Direction |
|---|---|---|---|
| CS (uWire LE) | 76 | GPIO2_B4 | out, idle high |
| SCLK | 75 | GPIO2_B3 | out |
| SDIO (bidirectional) | 74 | GPIO2_B2 | out; input during read data |
| RESET | 73 | GPIO2_B1 | out (LMK RESET pin, configured as input with pull-down) |
| lock status | 125 | GPIO3_D5 | in (Status_LD pin, push-pull; **UNVERIFIED** whether LD1 or LD2) |
| SYNC | 66 | GPIO2_A2 | out (LMK SYNC/SYSREF_REQ pin) |

### 4.2 Interface

| Call | Effect |
|---|---|
| `write(fd, buf, n)` | CS low, 1 µs, `n` bytes MSB first, 1 µs, CS high, 1 µs; returns 0 |
| `read(fd, buf, n)` | `*pos < 256`, `n` clamped to `256 − *pos`; sends `buf[0], buf[1]`, turns SDIO to input, clocks in `n` bytes, restores SDIO, CS high; advances `*pos`; **returns 0** |
| `ioctl(fd, 0x7800, v)` | RESET := `v` |
| `ioctl(fd, 0x7801, int *p)` | `*p` := status pin level |
| `ioctl(fd, 0x7802, v)` | SYNC := `v` |

Per bit: set SDIO, SCLK low, 1 µs, SCLK high, 1 µs (500 kHz, latched on the rising edge). Word format
(3-wire): 24 bits `[R/W][W1 W0 = 00][A12..A0][D7..D0]`; write `{addr >> 8, addr & 0xFF, data}`, read
`{0x80 | addr >> 8, addr & 0xFF}` then one data byte.

### 4.3 Programming sequence

1. Identify: registers `0x004, 0x005, 0x006, 0x00C, 0x00D` must read `0xD0, 0x5B, 0x24, 0x51, 0x04`.
2. RESET := 1, wait 1 ms, RESET := 0.
3. Write the configuration list in order, 10 µs apart; for address `0x000` clear data bit 4.
4. SYNC := 1. SYNC is low (its state after the module loads) until here and stays high, also across retries.
5. Lock check, no earlier than 1 ms after step 4: locked = status pin nonzero, or register `0x183`
   bit 1 (`RB_PLL2_LD`) = 1 over SPI (independent of which Status_LD pin GPIO 125 is; an
   implementation SHOULD check both). PLL2 lock detect needs `PLL2_DLD_CNT` = 8192 phase-detector
   cycles (`0x16A`/`0x16B` = `0x20`/`0x00`) = 0.82 ms at fPD2 = 10 MHz after PLL2 has settled, so poll
   for up to 10 ms. If not locked, repeat 2–5, up to 10 times.

The list is the unit's `/rigol/resource/conf/init_all_clock.txt` (fallback
`/rigol/data/default/init_all_clock.txt` if present); format TICS-Pro export, `R<n>\t0xAAAADD`: take
the first `0x` token V, `addr = V >> 8`, `data = V & 0xFF`; 129 entries, keep file order. Run the full
initialisation once per power-up, before the FPGA's PCIe link is brought up (spec 01 §4).

### 4.4 Derived clock plan (shipped `init_all_clock.txt`)

| Item | Setting | Result |
|---|---|---|
| OSCin | 10 MHz reference oscillator (`OSCin_FREQ` = 0..63 MHz) | **10 MHz** (inferred: the only value that puts VCO0 in its 1930–2075 MHz range; **UNVERIFIED**, measure OSCout, §11) |
| PLL1 | powered down (`0x140` bit 7) | reference oscillator free-running, trimmed by §3 |
| PLL2 reference | doubler on, R = 2 | fPD2 = **10 MHz** |
| PLL2 feedback | `PLL2_NCLK_MUX` = feedback mux = SYSREF divider (÷8), N = 25 (calibration path: prescaler 8 × N_CAL 25) | |
| VCO | VCO0 (`VCO_MUX` = 0) | **2000 MHz** = distribution clock |
| SYSREF divider | ÷8, SYSREF powered, pulser on | 250 MHz, used only as the PLL2 feedback (`0x13F = 0x15`) |
| SYSREF distribution | `SYSREF_MUX` = 0, Normal SYNC (`0x139 = 0x00`) | SDCLKout7/11 carry the SYNC path, not the divider |
| OSCout | buffered OSCin, LVPECL 2000 mV | 10 MHz |

| Output | Divider / source | Format | Frequency | Consumer |
|---|---|---|---|---|
| CLKout0 / 1 | ÷25, group powered down | LVDS | (80 MHz) off | "AFE channel clock" A, unused (§4.5) |
| CLKout2 | ÷16 | LVDS | **125 MHz** | FPGA reference clock, balls D19/C19 (spec 12 §3.1) |
| CLKout3 | — | off | — | |
| CLKout4 | ÷10 | LVDS | **200 MHz** | **UNVERIFIED** consumer (the FPGA derives its own 200 MHz from 125 MHz) |
| CLKout5 | — | off | — | |
| CLKout6 | ÷1 (DCC) | LVPECL 2000 mV | **2000 MHz** | ADC 0 sample clock |
| CLKout7 | SYSREF output (`0x11C = 0x20`), source Normal SYNC | LVDS | follows the SYNC path, static while SYNC is held (**UNVERIFIED**, measure) | ADC 0 synchronisation input (**UNVERIFIED**) |
| CLKout8 / 9 | group powered down | off | — | |
| CLKout10 | ÷1 (DCC) | LVPECL 2000 mV | 2000 MHz | ADC 1 sample clock (only populated on 2-ADC boards; enabled regardless) |
| CLKout11 | SYSREF output (`0x12C = 0x20`), source Normal SYNC | LVDS | as CLKout7 | ADC 1 synchronisation input (**UNVERIFIED**) |
| CLKout12 / 13 | ÷25, group powered down | LVDS | (80 MHz) off | "AFE channel clock" B, unused |

The ADC synchronisation input is driven either by LMK04821 SDCLKout7 (SDCLKout11 for ADC 1) or by the
FPGA LVDS pair G17/F17 (spec 12 §7.5); which one, and what the other drives, is **UNVERIFIED** (trace on
the board).

The ADC returns a 500 MHz data clock (DCO = 2000 MHz / 4) to the FPGA (spec 12/20). Status pins:
`0x15F = 0x0B` (Status_LD1 = PLL1 DLD, push-pull), `0x16E = 0x13` (Status_LD2 = PLL2 DLD, push-pull);
with PLL1 off only LD2 can indicate lock, so GPIO 125 is presumably Status_LD2 (**UNVERIFIED**).

Constants for an implementation: `F_REF = 10 MHz`, `F_VCO = 2000 MHz`, `F_ADC_CLK = 2000 MHz`,
`F_FPGA_REF = 125 MHz`, `F_CLKOUT4 = 200 MHz`, `F_SYSREF_DIV = 250 MHz` (internal, PLL2 feedback only),
`F_ADC_DCO = 500 MHz`.
Measure on a unit: frequency at OSCout and on CLKout4/7 (and whether CLKout7 toggles continuously).

### 4.5 Run-time writes

After initialisation the only writes needed are `0x130` and `0x100` (CLKout12 and CLKout0 divider
bytes, value `0x00` = ÷32) and `0x136 = 0xF8`, `0x106 = 0xF8` (both groups powered down). Other
run-time controls exist in the chip (CLKout4 divider `0x110 = 2000 MHz / f`, SDCLK delay registers
`0x11D`/`0x12D`) but are not needed. A firmware leaves the clock plan of §4.4 untouched after
initialisation.

### 4.6 Alternative plan (reference)

A plan with the same PLL2 result (VCO 2000 MHz, CLKout2 125 MHz, CLKout4 200 MHz, CLKout6/10 2 GHz)
can also enable PLL1 (CLKin1 manual, R = N = 120, i.e. the 10 MHz oscillator locked to a 10 MHz
CLKin1), with PLL2 on the prescaler feedback without doubler (R = 1). Whether CLKin1 is wired on this
board is **UNVERIFIED**; use the plan of §4.4.

## 5. Offset gear lines: `/dev/dac_gear_change`

Each channel has three GPIO outputs (a, b, c) plus one shared enable:

| | a | b | c |
|---|---|---|---|
| CH1 | 158 GPIO4_D6 | 157 GPIO4_D5 | 156 GPIO4_D4 |
| CH2 | 154 GPIO4_D2 | 153 GPIO4_D1 | 152 GPIO4_D0 |
| CH3 | 150 GPIO4_C6 | 149 GPIO4_C5 | 146 GPIO4_C2 |
| CH4 | 133 GPIO4_A5 | 131 GPIO4_A3 | 128 GPIO4_A0 |
| enable | 33 GPIO1_A1 (initialised high) | | |

`ioctl(fd, 0x7801, u8 *p)`: `*p = ch | code << 4`, sets a := code bit 0, b := bit 1, c := bit 2 (ch > 3
fails). `ioctl(fd, 0x7802, v)`: enable := `v`.

The code is chosen per gear (gear = s0 + 1 on 1 MΩ, §2.7) and scales the offset-DAC injection (the
offset-DAC volts per code of each gear, spec 10 §4, are the result):

| Path | Gear 0 (zero calibration) | Gear 1 (s0 = 0) | Gear 2 (s0 = 1) | Gear 3 (s0 = 2) | Gear 4 (s0 = 3) |
|---|---|---|---|---|---|
| 1 MΩ | 2 (b) | 0 | 4 (c) | 0 | 1 (a) |
| 50 Ω | 1 (a) | 0 | 1 (a) | 0 | — |

What each line switches on the board is **UNVERIFIED** (trace the three lines to the offset network).

## 6. 50 Ω overload detectors: `/dev/afe_rms`

Four GPIO inputs: CH1 56 (GPIO1_D0), CH2 54 (GPIO1_C6), CH3 45 (GPIO1_B5), CH4 42 (GPIO1_B2).
`ioctl(fd, 0x7800, u32 *p)` stores `ch1 | ch2 << 2 | ch3 << 4 | ch4 << 6` (raw levels; other bits 0).
A 1→0 change means overload; the channel is then switched to 1 MΩ. The AFE also drops a tripped
50 Ω path by itself (§2.5 `0x07`, STATUS bit 4).

The DHO1000 has a 1 MΩ input only (data sheet), so on a DHO1074 these inputs have no function; their
polarity and whether they are populated is **UNVERIFIED**.

## 7. Hardware code straps: `/dev/hdcode_gpio`

Four inputs: bit0 = GPIO 7 (GPIO0_A7), bit1 = GPIO 8 (GPIO0_B0), bit2 = GPIO 11 (GPIO0_B3), bit3 =
GPIO 12 (GPIO0_B4). `read(fd, buf, 1)` returns `b0 | b1<<1 | b2<<2 | b3<<3`. The code selects the
bitstream (spec 13): `0x1, 0x3` → A100T 1-ADC, `0x2, 0x9` → A75T 1-ADC, `0x0` → A100T 2-ADC, `0x4` →
A75T 2-ADC.

## 8. ADC (RT8847) register interface

### 8.1 SPI bridge

The ADC registers (7-bit address, 16-bit data) are reached through two FPGA registers:

| Operation | Sequence |
|---|---|
| write | `0x1034 = sel \| reg << 16 \| (value ^ MASK[reg])`, wait ≥ 200 µs |
| read | `0x1034 = sel \| 0x00800000 \| reg << 16`, wait ≥ 100 µs, `value = (0x1038 & 0xFFFF) ^ MASK[reg]` |

`sel` = `0x01000000` for ADC 0, `0x02000000` for ADC 1 (S2 boards have only ADC 0; writes may go to
both). MASK is the per-register power-on default; all values below are given **after** XOR, i.e. the
logical value (wire = logical ^ MASK). MASK (non-zero entries): `0x00 = 0x00E1`, `0x04 = 0x8080`,
`0x05 = 0x8080`, `0x09 = 0x2720`, `0x0A = 0x8080`, `0x0D = 0x3000`, `0x0E = 0x0080`, `0x0F = 0x7800`,
`0x10+8k = 0x0707`, `0x11+8k = 0x0C08`, `0x12+8k = 0x2920`, `0x13+8k = 0x1B9B` (k = 0..7),
`0x50..0x53 = 0x0820`, `0x54 = 0x1041`, `0x55 = 0x0004`, `0x58 = 0x3954`, `0x59 = 0x781C`,
`0x60 = 0x0805`, `0x63 = 0x0C00`; all others 0. The table covers `0x00..0x67`; the address field is 7
bits, and the only register written above `0x67` is `0x6E`, written once with wire value `0x0000`
(treat its MASK as 0).

### 8.2 Register map (as used)

| Reg | Fields / use |
|---|---|
| `0x00` | global configuration; init `0x00E1`, then `0x00E4` |
| `0x01` | clock delay: `[9:0]` fine (nominal step 66 fs/code, **UNVERIFIED**), `[12:10]` coarse, `[15:13]` preserved (calibrated on 2-ADC boards, spec 10 §6, §11.6); init `0x0000`. Every write MUST be followed by a word alignment (spec 20 §5.4, with the `CAP_RESYNC`/`SP_RST` pulses) |
| `0x02`, `0x03`, `0x0B` | init `0x0000` |
| `0x04`, `0x05` | four 8-bit sampling-clock delays (`0x04[7:0]`, `[15:8]`, `0x05` likewise), mid `0x80`, nominal step 0.1 ps/code (**UNVERIFIED**); calibrated (spec 10 §6, §11.6); init `0x8080` |
| `0x06` | `[10:8]` sync delay (sync-delay calibration, §8.3); init `0x06C3` |
| `0x07` | bit 1 = sync-phase flag (read by the sync-delay calibration); written `0x0000` by the init table |
| `0x08` | init `0x000B`, then `[3:0]` cleared (`0x0000`) at the end of init |
| `0x09` | `[5:0]` reference trim (calibrated); `[14:12]` set to `101` (ADC 0) / `100` (ADC 1) at the end of init; init `0x6721` |
| `0x0A` | init `0x8078` |
| `0x0C` | init `0x3FFF` |
| `0x0D` | `[11:8]` output data mode: `0x2` normal (operating value `0x620F`), `0x5` lane-training pattern (`0x650F`), `0x1` written before the init pass (`0x610F`; the pass itself writes `0x620F`); other modes `0x0`, `0x9`, `0xD`. Data modes of spec 20 §5.2: 1 → `0x1`, 2 → `0x5` (lane-training pattern), 3 → `0xD`, 4 → `0x2` (operating mode), 5 → `0x9` |
| `0x0E` | init `0x0078` |
| `0x0F` | output driver: `[3:2]` power mode, `[9]` output level, `[11:10]` LVDS drive; init `0x7000` then `0x7800` |
| `0x10+8k` | core k (0..7) config 1, init `0x0503` |
| `0x11+8k` | core k config 2, init `0x080B` |
| `0x12+8k` | core k config 3, init `0x2920` |
| `0x13+8k` | core k QI trim, init `0x1B9B`; bit 14 = quantiser-calibration start (§8.3) |
| `0x14+8k` | core k offset `[10:0]` (signed, −1024..1023) |
| `0x15+8k` | core k gain `[6:0]` (signed, −64..63), core k delay `[15:8]` |
| `0x16+8k` | core k output-code/test register, init `0x0004`; bit 15 = output code format, MUST be 0 (binary, spec 20 §3.3); read: bit 6 = quantiser calibration done |
| `0x17+8k` | core k, init `0x0000` |
| `0x50..0x53` | init `0x0830` |
| `0x54`, `0x55` | input switch matrix (§8.4); init `0x1084` / `0x0002` (1 channel, input 3) |
| `0x56` = `0x0007`, `0x57` = `0x0080`, `0x58` = `0x3954`, `0x59` = `0x781C`, `0x5B` = `0x07DE`, `0x5E` = 0, `0x5F` = `0x0800`, `0x60` = `0x080D` (first `0x040D`), `0x61`..`0x67` = 0 (`0x63` = `0x0C00`), `0x6E` = 0 | static configuration |
| `0x5F..0x67` | OTP / fuse access (`0x60`, `0x63` select, data in `0x64..0x67`) |

The init values of the per-core offset/gain registers written by the start-up table are
(k = 0..7): offset `0x14+8k` = `0x61, 0xAA, 0x94, 0xAA, 0x62, 0xAC, 0xBD, 0xCA`; whole word `0x15+8k` =
`0x0000, 0x0058, 0x0069, 0x006E, 0x007E, 0x004D, 0x005C, 0x004B` (delay `[15:8]` = 0). The calibrated
values (spec 10 §6) then replace offset, gain, `0x01`, `0x04`, `0x05` and `0x09[5:0]`; without a
`cal_adc.hex` they are replaced by 0.

### 8.3 Initialisation order

**Register table** (both ADCs, each register written to ADC 0 then ADC 1, 200 µs apart):

1. `0x63 = 0x0C00`, `0x60 = 0x040D`, `0x5F = 0x0800`, `0x00 = 0x00E1`, `0x0A = 0x8078`,
   `0x0E = 0x0078`, `0x0F = 0x7000`, `0x0D = 0x610F`.
2. `0x00 = 0x00E4`, then `0x01..0x59`, `0x5B`, `0x5E..0x67`, `0x6E` in address order with the init
   values of §8.2 (including `0x0D = 0x620F`, `0x0F = 0x7800`). Registers `0x5A`, `0x5C`, `0x5D` and
   `0x68..0x6D` are never written.
3. `0x09` (ADC 0 `0x5721`, ADC 1 `0x4721`), `0x08 = 0x0000`.

**Quantiser calibration** (every start-up, after the table): set bit 14 of `0x13 + 8k` on all cores
(logical `0x1B9B | 0x4000`). Then, core by core, read `0x16 + 8k` (100 µs per read) until raw bit 6 =
1 and clear bit 14 of that core. Allow up to 2 s for the whole calibration (both ADCs); on time-out
stop and leave the remaining cores unfinished. On a 1-ADC board an implementation polls
ADC 0 only.

**Sync-delay calibration** (every start-up, after the calibrated trims, and in the self-calibration),
ADC 0: for `d = 0…7`: `0x06[10:8] = d`, wait 20 ms, pulse `CAP_RESYNC` (`0x1000` bit 1: 0, 1, 0), read
`0x07` and keep `p[d]` = bit 1. Select the smallest `d ≤ 6` with `p[d] = 1` and `p[d+1] = 0`, write
`0x06[10:8] = d` and pulse `CAP_RESYNC` again. If there is none the calibration has failed; leave
`d = 7`. (2-ADC boards: ADC 1 uses the same rule, starting at ADC 0's `d`.)

The exact meaning of the static registers is **UNVERIFIED** (no public datasheet); an implementation
replays the values. The order of these steps among the other start-up steps is in §10.

### 8.4 Input switch matrix (`0x54`, `0x55`)

The 8 cores are assigned to the 4 analog inputs by `0x54` (four 4-bit nibbles, at most one bit set
in each) and `0x55` (`[3:0]`, at most one bit set; `0x0008` = parked). Values for a single-ADC board:

| Mode | `0x54` | `0x55` |
|---|---|---|
| 1 channel: input 0 / 1 / 2 / 3 | `0x8421` / `0x0842` / `0x2108` / `0x1084` | `0x0000` / `0x0001` / `0x0004` / `0x0002` |
| 2 channels (0,1) / (0,2) / (0,3) | `0x0821` / `0x2021` / `0x1021` | `0x0001` / `0x0004` / `0x0002` |
| 2 channels (1,2) / (1,3) / (2,3) | `0x2042` / `0x1042` / `0x1108` | `0x0004` / `0x0002` / `0x0002` |
| 4 channels | `0x1041` | `0x0004` |
| parked (during AFE reconfiguration) | `0x4210` | `0x0008` |

Inputs 0..3 correspond to CH1..CH4 (**UNVERIFIED** against the board). Every AFE gear or filter change
parks both ADCs and then restores the current mode (written to both ADCs). The current mode is the
last channel-mode selection (§10 step 13; written to ADC 0 only on a 1-ADC board, to both on a 2-ADC
board; the park and restore writes go to both ADCs on every board); before the first
selection it is `0x2108`/`0x0004` (1 channel, input 2), not the init table's `0x1084`/`0x0002`.

## 9. Vertical control: from settings to registers

### 9.1 Record selection

For channel `ch`, impedance (0 = 1 MΩ), bandwidth code `bw` and scale `S` (probe ratio already
applied, §9.5): `idx = index(S)` on the 801-step ladder, record = `cal_afe[slot(imp, bw)][ch][idx]`
(spec 10 §3). The record gives `s0..s6`, the residual digital gain `G` and the zero code.

### 9.2 Gain

Program the AFE (§2.7), then the digital channel gain `G` (spec 03 / spec 10 §3.5). `G` is the
record's gain for the current scale; the trigger level conversion uses this `G` unhalved, while
`PB_GAIN` carries `G/2` when `G ≥ 2` and a rescaled value after a stopped V/div change (spec 03 §2).

### 9.3 Offset

`OFFSET_DAC (0x28) = trunc(zero + (−(position + sign·offset) / 1e9) / LSB)` with offset and position in
nV, `sign` = ±1 (the channel's offset sign), `LSB = cal_lsb[ch][s0 + 1]` on 1 MΩ and
`cal_lsb[4 + ch][gear]` on 50 Ω (gear 0 below 500 µV/div, else `s0 + 1`). If `LSB = 0` the zero code is
written. Range: 16 bits; a 1 V offset at gear 1 moves the code by ≈15 680. Examples (default
calibration, CH1): 450 µV/div, offset +10 mV → `0x7F5F`; +1 V → `0x42BF`; 100 mV/div, −5 V → `0x6F1D`.

The offset DAC is written only while the channel is on; turning a channel on re-applies the gear
(§2.7) and then the offset. Wait 1 ms after the `0x28` write before acquiring.

### 9.4 Coupling, invert, GND

| Setting | Action |
|---|---|
| DC | `0x07[15] = 1` |
| AC | `0x07[15] = 0` |
| GND | AFE unchanged; the FPGA channel gain is set to 0 (digital) |
| Invert | AFE unchanged; digital (FPGA `INV`, spec 02) and the offset sign |

### 9.5 Probe ratio

The probe ratio does not touch the AFE: the requested display scale is divided by the ratio before the
record lookup, and offsets are scaled likewise.

### 9.6 Bandwidth limit

The AFE filter is set by `0x2E` fields (filter A `[7:6]`, B `[10:9]`, F `[5:0]`) from the
`cal_afe_filter` triple for the setting (spec 10 §5); the limit itself is digital for the model
bandwidth:

| User setting | AFE triple | Values (defaults) | cal_afe slot | FPGA second filter |
|---|---|---|---|---|
| 20 MHz | T0 | A = 0, B = 0, F = 5 | bw 2 | off; channel filter flag set (spec 03) |
| OFF on a 70 MHz model (= 70 MHz), 100 MHz, 200 MHz | T3 | A = 3, B = 1, F = `0x1B` | bw 5 / 6 / 8 | on, FIR set chosen by sample rate (table below) |
| 250 MHz (1 MΩ) | T4 (s0 = 0) / T5 | (3,0,`0x1B`) / (2,1,`0x1B`) | bw 9 | — (4000 series only) |

At 200 µV/div and below, a setting of OFF or 20 MHz always gives 20 MHz. "OFF" selects the model bandwidth (DHO1074:
70 MHz), never more. FPGA second-filter coefficient set (id) per bandwidth and sample rate (normal acquisition;
high-resolution modes use other ids):

| Bandwidth | 250 MSa/s | 500 MSa/s | 1 GSa/s | 2 GSa/s |
|---|---|---|---|---|
| 70 MHz | off | 11 | 10 | 8 |
| 100 MHz | 9 | 5 | 2 | 0 |
| 200 MHz | off | 9 | 5 | 3 (timebase ≤ 2 µs/div) / 2 |

The 14 coefficient sets (ids 0–13) are symmetric 24-tap FIRs with 23-bit two's-complement
coefficients, 22 fractional bits, each summing to `2^22 − 8 … 2^22 − 16` (DC gain 0.999996–0.999998;
treat as unity); their loading and
the per-channel-mode bank/tap layout are in spec 02 §2.10. −3 dB corner of each set as a plain FIR,
in units of the sample rate `fs`, and at the rate where the table above selects it:

| id | −3 dB / fs | id | −3 dB / fs | id | −3 dB / fs |
|---|---|---|---|---|---|
| 0 | 0.0625 | 5 | 0.2449 | 10 | 0.0803 |
| 1 | 0.0857 | 6 | 0.0390 | 11 | 0.1624 |
| 2 | 0.1314 | 7 | 0.3573 | 12 | 0.0298 |
| 3 | 0.1529 | 8 | 0.0401 | 13 | 0.3344 |
| 4 | 0.1824 | 9 | 0.4565 | | |

| Bandwidth | 250 MSa/s | 500 MSa/s | 1 GSa/s | 2 GSa/s |
|---|---|---|---|---|
| 70 MHz | — | id 11: 81 MHz | id 10: 80 MHz | id 8: 80 MHz |
| 100 MHz | id 9: 114 MHz | id 5: 122 MHz | id 2: 131 MHz | id 0: 125 MHz |
| 200 MHz | — | id 9: 228 MHz | id 5: 245 MHz | id 3: 306 MHz / id 2: 263 MHz |

The digital corner sits above the nominal bandwidth; the analog front end (filter triple T3) gives the
remaining roll-off (**UNVERIFIED** combined response; measure with a swept sine). An open firmware
MAY use its own filter designs with the same −3 dB targets; the sets above are not required.

### 9.7 EXT trigger level

`EXT_THRESH (0x1404) = min(0xFFFF, trunc_u(L[V] · k + c))` (double; `trunc_u` truncates toward zero
and gives 0 for negative results; spec 05 §3.4) with `k`, `c` from `cal_ext` (defaults `k = 2074`,
`c = 30012`).

## 10. Start-up order

Before this sequence the FPGA is configured (spec 13) and the LMK04821 programmed (§4.3). Then, in
this order:

1. AFE power-up of channels 0, 1, 2, 3 (§2.7 steps 1–4).
2. ADC register table, both ADCs (§8.3).
3. SP, display and SEQ initialisation (specs 02, 03); version reads; the DDR skip-calibration preset:
   read `DDR_TAPSTAT` (`0x14A0`, `0x14A4`, `0x14A8`, `0x1498`, `0x149C`, `0x14AC`, `0x14B0`, `0x14B4`,
   `0x14B8`; values unused), write `DDR_TAPSET` `0x1470 … 0x1494` in address order = `0x0045228A`,
   `0x00010290`, `0x1031040C`, `0x0001F69E`, `0x1C61869C`, `0x000216A1`, `0x1D65C75B`, `0x000001EF`,
   `0x1F183E30`, `0x00000000` (spec 10 §9 layout: PO_COARSE `0x45228A`, PO_STG3 `0x40A41031040C`,
   PO_STG2 `0x7DA79C61869C`, PI_STG2 `0x85A85D65C75B`, IDELAY `0x7BDF183E30`, temperature 0) without a
   `RM_RST_B` pulse, then `SP_RESET` (`0x1000`) = 0 (`RM_RST_A` = `RM_RST_B` = 0); these constants
   are written at every start-up, with or without `cal_ddr` (whether the MIG applies values written
   without the pulse is **UNVERIFIED**, spec 10 §9). These are fixed constants: their PO_STG3 is not
   PO_STG2 − `0x410410410410` (the rule of spec 10 §9 applies only to values saved by the DDR
   calibration), and an implementation MUST write them as given. Then an optional FPGA self-check:
   `RM_STATUS` (`0x1008`) bit 29 = 1 means the DDR initialised (spec 02 §2.3), and `0x1210` in
   62 494 … 62 506 means the ADC clock is in range (spec 02 §7); report the results, the start-up
   does not depend on them.
4. Calibration: `cal_afe` (spec 10 §3); quantiser calibration (§8.3); `cal_adc` and
   `cal_adc_SpuGain` applied for channel mode 0 (spec 10 §6; each `0x01` write is followed by a word
   alignment); AFE zero, bandwidth and filter tables, `cal_ext`; `cal_clock` written to the aux DAC
   (§3); `cal_ddr` if present (spec 10 §9).
5. Sync-delay calibration (§8.3), then the lane search and tap loads (spec 20 §5.2).
6. ADC data mode 4 and word alignment with the `CAP_RESYNC`/`SP_RST` pulses (spec 20 §5.4); wait 1 ms.
7. FPGA offset registers `0x1418 + 4·ch = 0x8000`.
8. LMK run-time writes (§4.5).
9. ADC bit range (`CAP_CTRL[23:20]`, spec 04 §3).
10. CH1 to 100 mV/div (gear change §2.7, record per spec 10 §3.5).
11. Sinc interpolator, fan, acquisition start (specs 03, 04).
12. `1 MΩ enable` (§2.9) on all four channels, in the variant of each channel's current path. This
    rewrites `0x23[13:11]`, `0x29[9:8]` (VGA s1) and `0x2A[15:12]` (SGAR s4, SGA s3) with the
    variant's fixed values, also on CH1, whose gear step 10 has just set. With the CH1 100 mV/div
    setup from the shipped calibration (s0 = 2, attenuated path, s1 = 2, s3 = 2, s4 = 0) the
    attenuated values equal CH1's own, so nothing changes; with any other CH1 setup the enable
    overwrites s1, s3 and s4. An implementation MUST redo the gear change (§2.7) of every enabled
    channel after this step, before it configures the first acquisition (or skip the step for
    channels whose gear is already set).
13. Channel-mode selection for the enabled channels (spec 10 §6 mode `m`): input switch of §8.4 (on a
    1-ADC board written to ADC 0 only, on a 2-ADC board to both ADCs; the park/restore writes of a
    gear change go to both ADCs on every board), then the `cal_adc`/`cal_adc_SpuGain` values for mode `m` in the
    order of spec 10 §6 (each `0x01` write followed by a word alignment). Repeated whenever the set of
    enabled channels changes.

Until step 13 the restore value of §8.4 is `0x54 = 0x2108`, `0x55 = 0x0004` (single channel,
input 2) while the trims of mode 0 are loaded. So until step 13 every gear change, including CH1's in
step 10, restores the switch to input 2 (CH3, §8.4), and an acquisition made before step 13 samples
input 2, not CH1. An implementation SHOULD do step 13 for the configured channels as soon as they are
known (it may follow step 10 directly), in any case before the first acquisition is configured
(`SP_MODE`, sample rate), and again on every change of the enabled channels.

## 11. Open items needing the hardware

| Item | Measurement |
|---|---|
| AFE CS ↔ connector, ADC input ↔ channel | inject a signal on CH1 only; check which slave's gain changes the trace and which `0x54` input selection shows it |
| Aux DAC part, which output is coarse/fine | package marking; DMM on outputs while writing codes |
| OSCin = 10 MHz, CLKout4 / CLKout7 consumers | frequency counter on OSCout, CLKout4, CLKout7 |
| Status pin = LD1 or LD2 | power PLL1 down/up and watch GPIO 125 |
| Offset-gear lines | trace GPIO 156/157/158 on the board |
| afe_rms pins | DMM on GPIO 56/54/45/42 |
| RT1642 field semantics marked "trim" | only by characterisation (vary a field, measure offset/flatness) |
