# 10 — Calibration

Calibration data lives in small binary files under `/rigol/data/` (factory/self-cal results), with
optional fallbacks under `/rigol/data/default/`. All files share one checked container. This spec
gives the container, every file's payload layout, and how each payload is applied to the hardware.
Register names: spec 02. AFE/ADC/DAC wire protocols and code meanings: spec 06. The formats, values
and procedures here are **UNVERIFIED** on a unit.

---

## 1. Container (all `cal_*.hex` files)

Little-endian, 28-byte header followed by the payload.

| Offset | Size | Field | Value |
|---|---|---|---|
| `0x00` | u32 | `hdr_crc` | CRC-32 (IEEE 802.3, zlib) of bytes `0x08 … 0x1B` |
| `0x04` | u32 | `hdr_len` | `0x14` (20 = length of the header block that follows) |
| `0x08` | u32 | `version` | 0 (other values **UNVERIFIED**) |
| `0x0C` | 8 bytes | `timestamp` | `u8 sec, u8 min, u8 hour, u8 mday, u8 mon (1–12), u16 year (four digits), u8 0` (local time of the save) |
| `0x14` | u32 | `payload_len` | must equal the expected size of the file type (§2) |
| `0x18` | u32 | `payload_crc` | CRC-32 of the payload |
| `0x1C` | `payload_len` | payload | file-specific (§3 onward) |

Load rules (the implementation MUST apply them in this order):

1. Open `/rigol/data/<name>`; it must be a regular file of at least 28 + expected payload size bytes.
2. Read `hdr_crc`, `hdr_len`; `hdr_len` must be `0x14`; read the 20-byte block; its CRC must equal
   `hdr_crc`. Otherwise → header error.
3. `payload_len` must equal the expected size; read the payload into a scratch buffer; its CRC must
   equal `payload_crc`. Copy it into the live table only after both CRCs match: a failed file MUST
   NOT change the table.
4. On any failure, retry with `/rigol/data/default/<name>`. If that also fails, keep the
   compiled-in defaults for that table (§10).

Save: write header (fresh timestamp, version 0) and payload with both CRCs computed as above.

---

## 2. File list

| File | Payload size | Content | Applied to |
|---|---|---|---|
| `cal_afe.hex` | `0x2D80C0` (current) or `0x700C0` (legacy/default) | per-V/div front-end setup and digital gain | AFE gear codes, offset-DAC zero, `PB_GAIN` (§3) |
| `cal_lsb.hex` | `0x300` | offset-DAC volts-per-code per channel and gear | offset DAC programming (§4) |
| `cal_afe_zero.hex` | `0x230` | AFE internal zero (VGA/SGA/MUX trims) | AFE trim registers (§5) |
| `cal_afe_bandwidth.hex` | `0x140` | AFE bandwidth-shaping trims | AFE BW registers (§5) |
| `cal_afe_filter.hex` | `0x180` | AFE filter trims | AFE filter registers (§5) |
| `cal_dac.hex` | `0x80` | offset-DAC polynomial fit | offset DAC (§4) |
| `cal_adc.hex` | `0x4D0` | ADC core gain / offset / phase per channel mode | ADC chip core registers (§6) |
| `cal_adc_SpuGain.hex` | `0x2C0` | SP fine gain per core | `LANE_TRIM` (§6) |
| `cal_ext.hex` | `8` | external-trigger level DAC gain/offset | `EXT_THRESH` (§7) |
| `cal_clock.hex` | `8` | reference-clock trim DAC codes | clock trim DAC (§8) |
| `cal_ddr.hex` | `0x30` | DDR3 PHY tap overrides | `DDR_TAPSET` (§9) |

On a unit, `default/` holds `cal_afe.hex` (legacy size), `cal_lsb.hex` and `adc_vert.cal`; no
calibration in this spec uses `adc_vert.cal`.

---

## 3. `cal_afe.hex` — vertical setup table

### 3.1 V/div ladder

Vertical scales are indexed on a fixed fine ladder of **801** values (index 0–800), in nV/div:
start at 100 000 (100 µV/div) and add a step; the step changes after a fixed number of entries:

| Segment | Step (nV) | Entries |
|---|---|---|
| 0 | 2 000 | 50 |
| 1 | 5 000 | 60 |
| 2 | 10 000 | 50 |
| 3 | 20 000 | 50 |
| 4 | 50 000 | 60 |
| 5 | 100 000 | 50 |
| 6 | 200 000 | 50 |
| 7 | 500 000 | 60 |
| 8 | 1 000 000 | 50 |
| 9 | 2 000 000 | 50 |
| 10 | 5 000 000 | 60 |
| 11 | 10 000 000 | 50 |
| 12 | 20 000 000 | 50 |
| 13 | 50 000 000 | 60 |
| 14 | 100 000 000 | 50 |

`scale(i) = 100 000 + Σ steps of the first i entries`. Checks: index 110 = 500 µV, 160 = 1 mV,
320 = 10 mV, 480 = 100 mV, 640 = 1 V, 800 = 10 V. `index(scale)` = first index whose scale ≥ the
requested value. Requests ≤ 100 µV/div give index 0; requests above 10 V/div MUST be clamped to
index 800 (index 801 is an empty record: scale 0, gain 0).

### 3.2 Record (56 bytes)

| Offset | Type | Field |
|---|---|---|
| `0x00` | i64 | `scale` — the V/div this record serves (nV/div) = `ladder[index]`; 0 = unused |
| `0x08` | 7 × i32 | `setup[0..6]` = AFE setup codes s0..s6 (INPUT, VGA, ATT, SGA, SGAR, MUX, MUXR; spec 06 §2.6) |
| `0x24` | u32 | padding (0) |
| `0x28` | f64 | `gain` — residual digital gain `G = native_scale / scale` (≥ 1) |
| `0x30` | i32 | `zero` — offset-DAC code (AFE register `0x28`) that puts 0 V at ADC mid-scale (≈ 32 750) |
| `0x34` | u32 | padding (0) |

`native_scale = scale · gain` is the V/div of the gear without digital gain; for every gear
`native_scale × nominal AFE gain ≈ 0.19 V/div` (spec 06 §2.6), which is a usable sanity check.

### 3.3 File layout

```
current (0x2D80C0):  13 slots × 4 channels × 1024 records × 56 B   (= 13 × 0x38000)
                     + 0xC0-byte tail (§3.4)
legacy  (0x700C0):   slot H (1 MΩ) 4 × 1024 × 56 B, slot L (50 Ω) 4 × 1024 × 56 B, tail
```

Record address: `slot · 0x38000 + ch · 0xE000 + index · 56` (`ch` 0–3, `index` 0–800; records
801–1023 zero). 1 MΩ slots use indices 0–800 (10 V/div), 50 Ω slots 0–640 (1 V/div).

Slot selection by input impedance and the bandwidth in effect:

| Impedance | Bandwidth → slot |
|---|---|
| 1 MΩ | 20 MHz → 0, 70 MHz → 1, 100 MHz → 2, 200 MHz → 3, 250 MHz → 4 if the slot-4 record has s0 = 0, else 5, 400 MHz → 6, any other → 7 |
| 50 Ω | 20 MHz → 8, 200 MHz → 9, 250 MHz → 10, 400 MHz → 11, any other → 12 |

The bandwidth in effect is the model bandwidth when the limit is OFF (DHO1074: 70 MHz). Loading a
legacy file: slot H into slots 0–7, slot L into slots 8–12; the legacy tail is not used. Load order:
`/rigol/data/cal_afe.hex` as legacy size, else as current size, else `default/cal_afe.hex` as legacy
size. DHO1000 accepts only 1 MΩ (spec 06 §6); keep the 50 Ω slots anyway, the format is shared.

The `default/cal_afe.hex` (legacy) found on units contains, per channel, 801 1 MΩ and 641 50 Ω records; all
scales equal the ladder. Its 1 MΩ gears use s0 = 0 up to 60 mV/div (CH1, CH3) or 59 mV/div (CH2,
CH4), s0 = 2 up to 1.18 V/div (CH1, CH3) or 1.14 V/div (CH2, CH4), and s0 = 3 above; the 0.24
attenuator (s0 = 1) is not used. 50 Ω: s0 = 0 up to 120 mV/div (CH1, CH3) or 112 mV/div (CH2, CH4),
s0 = 1 above. The default file does not follow the gear table of §11.8: it contains
setups that are not in that table (e.g. (3,2,2,2,0,3,0)) and none with s0 = 1. An
implementation MUST accept any s0..s6 combination in a record.

### 3.4 Tail (0xC0 bytes)

24 f64: per channel 4 values for the 1 MΩ input gears s0 = 0..3, then per channel 2 values for the
50 Ω input gears (defaults: 6.10e-5, −2.54e-4, −1.12e-3, −1.14e-2 and 1.65e-4, 1.34e-3, identical for
all channels). They have the dimension of offset-DAC volts per code (compare §4) and are presumably
nominal design values. No run-time use: an implementation may ignore them (**UNVERIFIED** purpose).

### 3.5 Applying a record

For channel `ch`, impedance, bandwidth code and requested scale `S` (probe ratio applied):

1. `r = record(slot, ch, index(S))`.
2. Program the AFE gear from `r.setup` (spec 06 §2.7), the offset gear lines (spec 06 §5) and the
   AFE zero/bandwidth trims for that gear (§5).
3. Offset DAC (AFE `0x28`) = `trunc(r.zero − (position + sign·offset)/1e9 / LSB)` with
   `LSB = cal_lsb[ch][s0 + 1]` (1 MΩ) or `cal_lsb[4 + ch][g]` (50 Ω, `g` = 0 below 500 µV/div else
   `s0 + 1`); if `LSB = 0` write `r.zero` (spec 06 §9.3).
4. Digital gain `G = r.gain` (for scales on the ladder; otherwise × `r.scale / S`); if `G ≥ 2`: ADC
   coarse gain ×2 and `G /= 2`. `g = min(G, 64)` (float32); `PB_GAIN = trunc(g · 1024)`,
   `PB_OFS = trunc(32768 − g · 32768)`, truncation toward zero (spec 03 §2). GND coupling: gain 0.
5. The trigger threshold for the channel uses this `G` before the `G ≥ 2` halving of step 4, also
   while stopped (spec 05 §3.3, spec 03 §2).

### 3.6 How the records are produced (self-calibration)

For every gear `k` of the gear table (1 MΩ §11.8; 50 Ω analogously, §11.9) the calibration finds two
offset-DAC codes: `D1` that puts the channel average at `A1 ≈ 55 268` (+3 div) and `D2` for
`A2 ≈ 10 268` (−3 div) (16-bit domain, 7 500 codes per division, search §11.2); 55 268 and 10 268 are
ADC targets, not DAC codes. Then

```
Sk   = | LSB[ch][s0 + 1] · (D1 − D2) · 7500 / (A1 − A2) |     (V/div; stored ×1e9 as nV/div)
zero = trunc( D1 − (A1 − 32752) · (D1 − D2) / (A1 − A2) )      (offset-DAC code for mid-scale)
```

For every ladder index `i` the record is taken from the gear with the smallest `Sk > scale(i)`
(strictly greater; the first such gear on a tie): setup = the gear's s0..s6, `gain = Sk / scale(i)`,
`zero` = the gear's zero. Indices for which no gear qualifies keep their previous record. The calibration runs per bandwidth slot (§11.8).

---

## 4. `cal_lsb.hex` — offset-DAC volts per code; `cal_dac.hex`

`cal_lsb` payload = 12 rows × 8 f64 (row stride 64 bytes): `LSB[row][gear]` in **volts at the input
per offset-DAC code**:

| Rows | Content |
|---|---|
| 0–3 | CH1–CH4, 1 MΩ; gear 0 = zero-calibration gear, gear `s0 + 1` = input gear s0 (1–4) |
| 4–7 | CH1–CH4, 50 Ω; gear 0 (< 500 µV/div), gears 1–2 |
| 8–11 | unused (0) |

Defaults (CH1): 1 MΩ 1.713e-5, 6.379e-5, −7.67e-4, −1.173e-3, −4.611e-3; 50 Ω 1.661e-5, 4.131e-5,
1.82e-4 (negative = the injection is inverted relative to the signal in that gear). Measured by the
host-driven LSB calibration with an external reference (§11.10).

`cal_dac` (0x80): per channel 4 f64 (stride 32 bytes) = coefficients of a cubic least-squares fit of
the offset-DAC transfer, produced by the DAC-linearity calibration (§11.10). Nothing at run time
needs it; an open firmware may ignore it.

The 1 MΩ vertical calibration (§11.8) needs the unit's own `/rigol/data/cal_lsb.hex` (the `default/`
copy holds nominal values, not this unit's); without it that step must fail before touching the
hardware.

---

## 5. AFE trims (`cal_afe_zero`, `cal_afe_bandwidth`, `cal_afe_filter`)

Register fields named here are in spec 06 §2.5.

### 5.1 `cal_afe_zero` (0x230 = 4 × 140 B, i32)

Per channel 35 words, applied at every gear change for the active gear class, in table order (one
frame per register write; `v` = the word):

| Words | Gear class | Content → register |
|---|---|---|
| 0–4 | 50 Ω, s0 = 0, s1 = 0 | 0 VGA: `0x32[13:8] = v >> 3`, then `0x36[14:12] = v & 7`; 1, 2 SGA A, B: `0x2D[13:7]`, `[6:0]`; 3, 4 MUX A, B: `0x06[6:0]`, `[14:8]` |
| 5–9 | 50 Ω, s0 = 0, s1 = 1 | 5 VGA: `0x2C[5:4] = (v >> 7) & 3`, then `0x36[11:5] = v & 0x7F`; 6–9 as 1–4 |
| 10–14 | 50 Ω, s0 = 1, s1 = 2 | 10 VGA: `0x2C[9:6] = (v >> 5) & 0xF`, then `0x36[4:0] = v & 0x1F`; 11–14 as 1–4 |
| 15–21 | 1 MΩ direct (s0 = 0), s1 = 0 | 15 sign, 16 magnitude: `0x23[10]`, `0x23[9:0]` (one write); 17 VGA: `0x35[8:0] = v`; 18, 19 SGA A, B: `0x27[14:8]`, `[6:0]`; 20, 21 MUX A, B: `0x06[6:0]`, `[14:8]` |
| 22–28 | 1 MΩ direct (s0 = 0), s1 = 1 | 22, 23 as 15, 16; 24 VGA: `0x29[15:10] = v >> 3`, then `0x35[11:9] = v & 7`; 25–28 as 18–21 |
| 29–34 | 1 MΩ attenuated (s0 = 1..3), s1 = 2 | 29 sign, 30 magnitude: `0x26[10]`, `0x26[9:0]`; 31, 32 SGA A, B: `0x27`; 33, 34 MUX A, B: `0x06` |

Other bits of these registers are preserved (`0x27` and `0x06` bits 15, 7; `0x2D[15:14]`;
`0x23`/`0x26[15:11]`). Combinations not listed (for example 1 MΩ attenuated with s1 ≠ 2) apply no zero
trims. With no file all words are 0 and are still written, so the power-up values of `0x06` and
`0x27[14:8]`, `[6:0]` become 0 at the first gear change.

Produced by the AFE zero calibration (§11.7).

### 5.2 `cal_afe_bandwidth` (0x140 = 4 × 80 B, i32)

Per channel 20 words. For 1 MΩ attenuated gears s0 = 1, 2, 3 the words `[6·s0 − 5 .. 6·s0 − 1]` are
the trims T1, T6, T7, T8, T9; for the direct gear (s0 = 0) words 18, 19 are T10, T11. Words
`6·s0 − 6` (0, 6, 12) hold T0 (`0x22[7:0]`) of the attenuated gears s0 = 1, 2, 3; they are written by the
factory bandwidth calibration (§11.10) and never applied at gear changes. Field targets: T1 →
`0x22[15:8]`, T6 → `0x20[6:4]`, T7 → `0x21[11:7]`, T8 → `0x21[6:0]`, T9 → `0x21[15:12]`, T10 →
`0x24[9:0]`, T11 → `0x25[9:0]` (other trims T2, T3, T5 are only used by the calibration itself).
Defaults when no file: T1 = 0, T6 = 3, T7 = 15, T8 = 115, T9 = 0, T10 = T11 = 0. Applied on 1 MΩ gear
changes only.

### 5.3 `cal_afe_filter` (0x180 = 4 × 96 B, i32)

Per channel 8 triples `(A, B, F)` → `0x2E[7:6] = A`, `0x2E[10:9] = B`, `0x2E[5:0] = F`:

| Triple | Used at run time for | Written by the filter calibration (§11.10) for bandwidth | Default (no file) |
|---|---|---|---|
| T0 | 20 MHz limit | 20 MHz | (0, 0, 5) |
| T1 | — | 70 MHz | (0, 0, 20) |
| T2 | — | 100 MHz | (0, 3, 27) |
| T3 | full bandwidth, DHO1000/2000 (70/100/200 MHz settings) | 200 MHz | (3, 1, 27) |
| T4 | 250 MHz, s0 = 0 | 250 MHz, together with T5 | (3, 0, 27) |
| T5 | 250 MHz, s0 ≠ 0 | 250 MHz, together with T4 | (2, 1, 27) |
| T6 | — | 400 MHz | (3, 2, 20) |
| T7 | DHO4000 full bandwidth | DHO4000 full bandwidth | (3, 3, 27) |

On a DHO1074 only T0 and T3 are used; T1, T2, T6, T7 can be calibrated but have no effect.

---

## 6. ADC (`cal_adc`, `cal_adc_SpuGain`)

- `cal_adc` (0x4D0 = 11 blocks × 0x70 bytes, i16). `m` = channel mode as for `cal_adc_SpuGain`
  (0–3 single channel, 4–9 the pairs (0,1), (0,2), (0,3), (1,2), (1,3), (2,3), 10 four channels);
  `b` = input channel 0–3; `d = m` for m = 1, 2, 3, else 0. ADC register fields: spec 06 §8.2.

  | Payload offset | Content | ADC register |
  |---|---|---|
  | `0x70·m + 0x10·adc + 2·core` | core offset, −1024…1023 | `0x14 + 8·core [10:0]` |
  | `0x70·b + 0x20 + 0x10·adc + 2·core` | core gain, −64…63; each core takes the block of the input it samples in mode `m` | `0x15 + 8·core [6:0]` |
  | `0x70·d + 0x40 + 8·adc + 2·k`, k = 0..3 | sampling-clock delay, 0…255 (mid 0x80) | k = 0 `0x04[7:0]`, 1 `0x04[15:8]`, 2 `0x05[7:0]`, 3 `0x05[15:8]` |
  | `0x50`, `0x52` | reference trim ADC 0 / ADC 1 | `0x09[5:0]` |
  | `0x70·(4 + d) + 0x40 + 4·adc` / `+ 0x42 + 4·adc` | clock delay fine / coarse | `0x01[9:0]` / `0x01[12:10]` |

  Applied for channel mode `m` (at start-up `m = 0`; then at every channel-mode selection, spec 06 §10
  step 13) in this order: reference trims, `LANE_TRIM`,
  core gains, core offsets, sampling-clock delays, clock delay ADC 0 then ADC 1 (each `0x01` write
  is followed by a word alignment, spec 20 §5.4). Core delays (`0x15 + 8k [15:8]`) are not stored in
  `cal_adc`: writing a gain also writes them 0, and no calibration sets them. Without a file every value
  is 0 and is still applied. Values outside the register range (offset ±1024, gain −64…63) SHOULD be
  clamped to it.
  The reference trims are written as 0x20 by every DC calibration (§11.4).
- `cal_adc_SpuGain` (0x2C0 = 11 rows × 64 B, i32): `LANE_TRIM` (`0x1058`, spec 02) takes 16 values:
  for channel mode `m` 0–9 with row pair `(a, b)` = (0,0), (1,1), (2,2), (3,3), (0,1), (0,2), (0,3),
  (1,2), (1,3), (2,3): values 0–7 = row `a` words 0–7, values 8–15 = row `b` words 8–15. For the
  4-channel mode: values `4j..4j+3` = row `j` words `4j..4j+3`. Rows 0–3 are produced by the DC
  calibration (§11.4); rows 4–10 are neither written nor read.

---

## 7. `cal_ext.hex`

Payload `{i32 k, i32 c}`; `EXT_THRESH (0x1404) = min(0xFFFF, trunc_u(L[nV] / 1e9 · k + c))` (double
arithmetic; `trunc_u` truncates toward zero and gives 0 for negative results; spec 05 §3.4). Built-in defaults when no `cal_ext.hex` loads: `k = 2074` (`0x81A`) codes/V,
`c = 30012` (`0x753C`); 0 V → `0x753C`, +1 V → `0x7D56`, −1 V → `0x6D22`, +5 V → `0x9DBE`.

EXT offset calibration (factory, input at 0 V): binary search for `c` over `[0, 0xFFFF]` in 15 steps
starting at `0x8000`; per step write `EXT_THRESH = c`, wait 100 ms, read `EXT_STATE` (`0x2264`) `[3:0]`;
nonzero (input above the threshold) → raise the lower bound to the current code, else lower the upper
bound. The stored `c` is the code after the 15th halving (it is not itself measured). `k` is not
calibrated by this procedure (it keeps its file or default value).

## 8. `cal_clock.hex`

Payload `{i32 a, i32 b}`: the two codes of the reference-oscillator trim DAC (spec 06 §3), written as
the B frame (`a`) then the A frame (`b`). Defaults `a = 300`, `b = 360`. The reference-source switch is
set independently (internal = 1).

## 9. `cal_ddr.hex`

Payload 0x30 bytes (Xilinx MIG 7-series skip-calibration values):

| Offset | Type | Field | From (DDR calibration, `DDR_TAPSTAT`) | To (apply, `DDR_TAPSET`) |
|---|---|---|---|---|
| `0x00` | u32 | PO_COARSE_TAP, 8 × 3 bit (field `j` = bits `3j+2:3j`) | `0x14A0[23:0]` | `0x1470[23:0]` |
| `0x04` | u32 | 0 | | |
| `0x08` | u64 | PO_STG3_TAPS, 8 × 6 bit | `PO_STG2_TAPS − 0x410410410410`, one 64-bit subtraction (each field − 16; a field below 16 borrows from the next); `0x1498`/`0x149C` are read but not used | `0x1474[17:0]` = bits 47:30, `0x1478[29:0]` = bits 29:0 |
| `0x10` | u64 | PO_STG2_TAPS, 8 × 6 bit | `0x14A4[17:0] << 30 \| 0x14A8[29:0]` | `0x147C`, `0x1480` (same split) |
| `0x18` | u64 | PI_STG2_TAPS, 8 × 6 bit | `0x14AC[17:0] << 30 \| 0x14B0[29:0]` | `0x1484`, `0x1488` |
| `0x20` | u64 | IDELAY_TAPS, 8 × 5 bit | `0x14B4[9:0] << 30 \| 0x14B8[29:0]` | `0x148C[9:0]`, `0x1490[29:0]` |
| `0x28` | u32 | CAL_DEVICE_TEMP (always written 0) | — | `0x1494[11:0]` |
| `0x2C` | u32 | 0 | | |

Apply at start-up, only when `cal_ddr.hex` or `default/cal_ddr.hex` loads: `SP_RESET.RM_RST_B` (bit 3)
= 1 with `RM_RST_A` (bit 2) = 0, write `0x1470 … 0x1494` in address order, `RM_RST_B` = 0. Independently of the
file, every start-up writes the compiled-in values of spec 06 §10 step 3 (without the `RM_RST_B`
pulse); those are fixed constants and do not follow the PO_STG3 rule above. Whether the MIG uses `DDR_TAPSET` values written without the pulse is **UNVERIFIED**; with no
file and if it does not, the MIG's own calibration is used. Procedure: §11.11.

## 10. Missing files

On a unit, `/rigol/data/` holds the factory files. An open firmware MUST read them in place and
never overwrite them except through an explicit, user-confirmed calibration. If a file is absent:
`cal_afe` from `default/`; `cal_lsb` from `default/`; AFE zero trims 0, written at every gear change
(§5.1); bandwidth and filter trims the defaults of §5.2/§5.3; ADC trims 0, written at start-up (§6:
core offsets, gains, `0x01`, `0x04`, `0x05` and `0x09[5:0]` all become 0, replacing the start-up table
values of spec 06 §8.2); `LANE_TRIM` all 0; `cal_ext` and `cal_clock` the defaults of §7/§8. Back
up `/rigol/data/*` before any experiment.

## 11. Calibration procedures

### 11.1 Items, order, result

| Item | Kind | Writes |
|---|---|---|
| ADC line: sync delay (spec 06 §8.3) + lane search (spec 20 §5.2) | user | nothing (rerun at every start-up) |
| ADC core gain/offset, DC (§11.4) | factory | `cal_adc`, `cal_adc_SpuGain` |
| ADC gain/offset by sine (§11.5) | factory | — (not possible on 1-ADC boards) |
| ADC sampling-clock delay ("ADC clock", §11.6) | factory | `cal_adc` |
| 50 Ω vertical (§11.9) | not on DHO1000/2000 | `cal_afe` 50 Ω slots |
| 1 MΩ vertical (§11.8), per channel | user | `cal_afe` |
| EXT trigger offset (§11.10) | factory | `cal_ext` |
| AFE zero ("offset", §11.7), per channel | factory | `cal_afe_zero` |
| DDR (§11.11) | factory | `cal_ddr`, then power-off/reboot |

The user self-calibration of a DHO1074 is: ADC line, then 1 MΩ vertical on CH1–CH4, with all inputs
disconnected. Channels selected for the AFE or vertical items are switched on; with none selected, all
channels are used.

Order (each item only if selected; the first failing item stops the run): ADC line → DC → by-sine →
sampling-clock delay, modes 0…nch−1 (save after the last) → AFE zero per selected channel (save after
every channel) → 1 MΩ vertical per selected channel (save per channel) → 50 Ω vertical → EXT (save) →
DDR. All waits below are minimums.

### 11.2 Measurement primitives

- **Core average** `avg[c]`, c = 0…8·N−1 (N = number of ADCs): `STAT_CTRL` (`0x1040`) averaging period field
  `[4:0]` = 14 (2^14 × 4 ns = 65.5 µs), START 0 → 1; poll DONE (`[30]`) every 10 ms, at most ~50 ms
  (then read anyway); for each c: `0x1040[11:8] = c`, read `0x1044` (if it reads `0xEEEEEEEE`, read
  once more). Values are in the 16-bit record domain (12-bit code × 16; mid-scale 32 768).
- **Channel average** of input `ch` in a single-channel mode: mean of `avg[c]` for c = f(ch), f(ch) + 4, …
  < 8·N, with f = {CH1 0, CH2 2, CH3 1, CH4 3}.
- **Core map** used where a calibration addresses ADC core k (k = 0…7, + 8 for ADC 1): measurement index
  `map[k]` = 0, 2, 1, 3, 4, 6, 5, 7 (+ 8 for ADC 1).
- **Peak** `max[c]`, `min[c]`: `0x1040` MM_EN = 1, MM_CLR pulse, wait `t` ms, MM_LATCH pulse, per c
  `0x1040[11:8] = c`, read `0x1048` (`[31:16]` max, `[15:0]` min, each × 16), MM_EN = 0.
- **DAC search** `S(ch, target, dir, lo, hi, t)`: offset DAC (AFE `0x28`) code `x` = 0x8000; repeat at
  most 18 times: write `x`, wait `t`, channel average `a`; done if `|a − target| ≤ 16`; otherwise if the
  ADC must move up (`a < target − 16`): `dir = falling` → `hi = x`, `x = (x + lo) >> 1`; `dir = rising` →
  `lo = x`, `x = (x + hi) >> 1`; and symmetrically for `a > target + 16`. Result: the last `x` and `a`
  (after 18 steps without convergence the search reports no match; the callers below use the last
  point anyway). `falling` (ADC
  code decreases as the DAC code increases) holds for the 1 MΩ direct gears (s0 = 0) and the 50 Ω gears;
  `rising` for the attenuated gears (s0 ≥ 1) and the attenuated 0.24 gear used by §11.10.

### 11.3 ADC line

Precondition: none (inputs may be connected). Spec 06 §8.3 (sync delay) then spec 20 §5.2 (lane search,
taps loaded only on success; the start-up path loads them in any case). Nothing is stored.

### 11.4 ADC core gain and offset (DC)

Precondition: inputs open. Per channel: AFE gear s0..s6 = (0, 1, 1, 1, 0, 2, 1), output buffers as at a
gear change, offset DAC 0x8000; `PB_GAIN` = 0x400 and `PB_OFS` = 0 on all channels; `SP_MODE`
ORDER_CAL (`0x1004` bit 0) = 1; clear the channel inversions and the per-channel sample delays,
program the decimation registers `0x10D0`/`0x10D4`/`0x10D8` for the maximum rate and reconfigure
BWF; record length 0, 5 µs/div, run, wait 100 ms. Then:

1. (Diagnostic only) all core offsets 0; for each core k set offset 0x3FF, read all core averages and log
   which index moved by more than 500 LSB12; restore 0. Not applied.
2. Reference trim (mode 0): `0x09[5:0]` = 0x20 on both ADCs (stored at payload `0x50`, `0x52`), all core
   offsets and gains 0, the two DAC searches of step 3b on CH1; the 2-ADC matching search that would use
   them is skipped on 1-ADC boards.
3. Gain, for mode m = input ch = 0…nch−1 (single-channel mode, spec 06 §8.4):
   a. All core offsets and gains 0, `LANE_TRIM` = 0.
   b. `D1 = S(ch, 55 268, falling, 0, 0xFFFF, 20 ms)`, `D2 = S(ch, 10 268, falling, 0, 0xFFFF, 20 ms)`.
   c. Core swing `g[c]`: DAC = D1, wait 100 ms, mean of 10 core averages; DAC = D2, wait 100 ms, mean of
      10; `g[c]` = difference. `mid` = (max g + min g)/2.
   d. Fine gain: `r = max(g)/g[c] − 1` → code 3 if r ≥ 0.012, 2 if r ≥ 0.004, 1 if r ≥ 0.002, else 0;
      write `LANE_TRIM` (16 codes), store as `cal_adc_SpuGain` row m, words 0–15; wait 200 ms.
   e. Up to 12 rounds: write all core gains, measure `g`, `mid` (step c); for each core k with
      `e = (g[map[k]] − mid)/16`, if `|e| > 2`: `gain[k] = trunc(gain[k] − 2048·e/(mid/16))`, clamped to
      −64…63. Done when no core changed; otherwise the item fails.
   f. Mid-scale DAC `Z[ch] = S(ch, 32 768, falling, 0, 0xFFFF, 20 ms)` (kept in RAM for step 4).
4. Offset, for mode m = 0…10: select mode m; `LANE_GAIN` fields 0 and 2 = code 2, other fields unchanged
   (the CH1 field of the 4-channel layout, in every mode; whether another field is meant per mode is
   **UNVERIFIED**); core offsets 0,
   offset DAC of every channel = `Z[ch]`; up to 20 rounds: write offsets, core averages, for each core k
   `e = (avg[map[k]] − 32 768)/16`; if `|e| > 1`: `offset[k] = trunc(offset[k] − e)`. Done when no core
   changed, else fail. Stored per mode (§6).
5. ORDER_CAL = 0, save.

Fine-gain step size (spec 21 §2.1): fine-gain codes 1/2/3 add 1/256, 2/256, 4/256 to
the lane's `LANE_GAIN` factor (2.5 at code 1, 5 at codes 2/3), i.e. +0.16/+0.31/+0.63 % at code 1 and
+0.08/+0.16/+0.31 % at code 2; no effect at code 0. The thresholds of step 3d (r ≥ 0.002/0.004/0.012) are
a choice and do not equal these steps. The model behind the other steps (**UNVERIFIED** on hardware):
1 offset code ≈ 1 LSB12; 1 gain code ≈ 1/2048 relative gain; `STAT_CTRL` sees the samples after
`LANE_TRIM` and `LANE_GAIN` (the measurement tap point is **UNVERIFIED**).

### 11.5 ADC gain/offset by sine

On 1-ADC boards (DHO1074) this item cannot run (its reference matching needs two ADCs). An
implementation need not provide it.

### 11.6 ADC sampling-clock delay ("ADC clock")

Precondition: a 10 MHz sine on the channel being calibrated (one per mode m = 0…nch−1, i.e. CH1…CH4 in
single-channel mode), amplitude so that some scale between 2.0 and 0.3 V/div gives 4–8 div peak-peak (e.g.
1.2–16 Vpp). Steps per mode m:

1. Single-channel mode m, `PB_GAIN` ×1, all other channels off, trigger source EXT.
2. Scale search: DC coupling, offset 0; for S = 2.0, 1.9, …, 0.3 V/div: wait 500 ms, peak over 30 ms;
   accept when `30 001 ≤ max − min ≤ 59 999`, `min > 5 000`, `max < 60 000` (over all cores). None → fail.
3. ORDER_CAL = 1; clock delay `0x01` fine = coarse = 0 (each write followed by a word alignment,
   spec 20 §5.4); all four sampling delays = 0x80; wait 1 s; sync delay + lane search + tap load + word
   alignment; wait 1 s; record length 0, 5 µs/div, run, wait 1 s.
4. Core skew `t[c]`: 16 acquisitions; per core c the DFT of its own samples (every 8th sample) at
   1/25 cycle per core sample (10 MHz); phase φ[c] (unwrap +2π when φ[0] < 0 < φ[last]); `t[c] =
   (φ[c] − φ[0])/π · 50 ns`, averaged over the 16 acquisitions.
5. Coarse (≤ 10 rounds): `E = mean_{k=1..3}(t[k] + k·500 ps)`; stop when `|E| < 2 ps`; else
   `code0 = clamp(code0 − E/0.1 ps, 0, 255)` → sampling delay field 0.
6. Fine (≤ 10 rounds, 100 ms apart): for k = 0…3, `e_k = t[k] + k·500 ps`; for `|e_k| ≥ 2 ps`:
   `code_k = clamp(code_k + e_k/0.1 ps, 0, 255)`; write `code_0` → field 0, `code_1` → field 2, `code_2` →
   field 1, `code_3` → field 3. Success when all four are within ±2 ps.
7. ORDER_CAL = 0. On success store `code_0..3` at payload `0x70·m + 0x40 + 2k` and the clock delays
   (0 on 1-ADC boards) at block `4 + m`; on failure store nothing.

The restore of §6 writes field k ← `code_k`, i.e. with fields 1 and 2 swapped relative to step 6. Which mapping is physically right is **UNVERIFIED** (the search converging on a unit would
confirm step 6's); an implementation MUST use one mapping for both search and restore. (2-ADC boards
additionally search ADC 1's clock delay `0x01`, 66 fs per fine code, fine 0…1024 then coarse +1, coarse
≤ 6.)

### 11.7 AFE zero ("offset")

Precondition: input open. All searches below are bisections over a linear index `c` with comparator
reading `q` (one STATUS read after the write and a 10 ms wait): `q = 1` → `hi = c`, `c = lo + (c − lo)/2`;
`q = 0` → `lo = c`, `c = c + (hi − c)/2` (integer division); the last measured `c` is written once more and
kept.

| Stage | Index range, start | Measurements | Code written | Comparator |
|---|---|---|---|---|
| First stage (direct / attenuated) | 0…0x7FF, 0x3FF | 15 | `0x23[10:0]` / `0x26[10:0]` = sign `c < 0x400`, magnitude `c < 0x400 ? 0x3FF − c : c − 0x400` | STATUS bit 2 (direct) / bit 1 (attenuated) |
| VGA | 0…0xE0, 0x70 | 9 | `v = c < 64 ? c : c + 41·((c − 64)/23 + 1)` (9-bit), form per §5.1 | STATUS bit 1 |
| Comparator offset | 0…0x7F, 0x40 | 8 | `0x07[6:0] = c` | STATUS bit 3 |
| SGA | 0…0xFF, 0x7F | 9 | A = max(0, 0x7F − c), B = max(0, c − 0x80) → `0x27 = B[6:0] \| A << 8` (1 MΩ), `0x2D = B[6:0] \| A << 7` (50 Ω) | STATUS bit 3 |
| MUX | 0…0xFF, 0x7F | 9 (first at (0,0)) | B = max(0, 0x7F − c), A = max(0, c − 0x80) → `0x06 = A[6:0] \| B << 8` | STATUS bit 3 |

Per channel ch:

1. Single-channel mode ch; offset-DAC enable off (`0x27` bits 15, 7 = 0); offset gear line code 2
   (gear 0, spec 06 §5).
2. For each zero-cal gear (DHO1000: Z3, Z4, Z5): program s0..s6 with the gear-change writes of spec 06
   §2.7 steps 3–8 (no trims, no ADC parking); wait 1 s; read the effective input path (spec 06 §2.5) and
   run the stage list for it:

| Gear | s0..s6 | Path | Stages (in order) → `cal_afe_zero` words |
|---|---|---|---|
| Z3 | 0,0,2,0,0,3,1 (VGA 25) | direct | First(direct) → 15, 16; VGA → 17; Comparator; SGA → 18, 19; MUX → 20, 21 |
| Z4 | 0,1,2,1,0,2,1 (VGA 2.5) | direct | First(direct) → 22, 23; VGA → 24; Comparator; SGA → 25, 26; MUX → 27, 28 |
| Z5 | 1,2,2,3,0,3,0 | attenuated | First(attenuated) → 29, 30; `0x2B[9:8] = 11`; Comparator; SGA → 31, 32; MUX → 33, 34 |
| Z0 | 0,0,1,2,0,1,1 (50 Ω, input 0.5, VGA 25) | 50 Ω 0.5 | VGA → 0; Comparator; SGA → 1, 2; MUX → 3, 4 (DHO4000 only) |
| Z1 | 0,1,1,3,0,1,1 (VGA 2.5) | 50 Ω 0.5 | VGA → 5; …; SGA → 6, 7; MUX → 8, 9 |
| Z2 | 1,2,1,0,0,1,1 | 50 Ω 0.05 | VGA → 10; …; SGA → 11, 12; MUX → 13, 14 |

   Any other path → the item fails. Wait 1 s after the stages.
3. Offset-DAC enable on (`0x27` bits 15, 7 = 1). Results stay in the RAM table (applied at the next gear
   change) and are saved per §11.1.

Stage set-up and tear-down writes (one frame each, in this order):

- First (direct): `0x23 |= 0x2000`; `0x24 = (0x24 & 0x87FF) | 0x4400`; search; `0x24 &= 0xBBFF`.
- First (attenuated): `0x26 = (0x26 & 0x07FF) | 0x9800`; search; `0x26 &= 0x37FF`.
- VGA (1 MΩ direct): First(attenuated) is run first (its words 29, 30 are overwritten later by Z5); then
  `0x26 = (0x26 & 0x07FF) | 0x5800`; `0x29[3:0] = 0000` (VGA 25) or `0001` (VGA 2.5); search. (50 Ω:
  `0x29[3:0]` = `1000`/`1001` for input 0.5, `1010` for input 0.05.)
- Comparator: `0x2A |= 0x0160`; `0x07[6:0]` search. Not stored; it stays set for the following stages.
- SGA: `0x2A[6:5] = 10` (1 MΩ) or `00` (50 Ω). MUX: `0x2A[6:5] = 01`. Bit 8 of `0x2A` stays 1.

None of these writes is undone by the procedure; the next ordinary gear change rewrites the gain fields
and the trims, but `0x2A[8]`, `0x2B[9:8]`, `0x07[6:0]` and `0x26[13]` (cleared: `0x26[15:11]` ends at
`00010` instead of the power-up `00110`) keep the calibration values until power-up (**UNVERIFIED**
whether that matters; an implementation SHOULD restore `0x2A[8:5]`, `0x2B[9:8]` and `0x26[15:11]` to
their pre-calibration values and keep the searched `0x07[6:0]`).

There is no acceptance test: a stage always returns its last code. Duration ≈ 8 s per channel.

### 11.8 1 MΩ vertical gain and zero (user self-calibration)

Precondition: input open; `/rigol/data/cal_lsb.hex` valid (§4); AFE zero trims loaded (applied per gear;
the bandwidth trims of §5.2 are not re-applied by this procedure, the last gear change's values stay). Per selected channel ch:

1. Wait 2 s. Single-channel mode ch, `PB_GAIN` ×1, `PB_OFS` 0, 50 Ω disable / 1 MΩ enable, run.
2. For each bandwidth `bw` in {20, 70, 100, 200 MHz} (DHO1000) → `cal_afe` slot 0, 1, 2, 3: set the AFE filter triple and the FPGA filter for `bw` as at run time (spec 06 §9.6).
3. For each gear k = 0…76 of the table below: program s0..s6, output buffers, offset gear line `s0 + 1`,
   AFE zero trims for (s0, s1) (§5.1); wait 500 ms if s0 differs from the previous gear, else 20 ms. Then
   up to 19 attempts: `(D1, A1) = S(ch, 55 268, dir, 20 000, 40 000, 20 ms)`, `(D2, A2) = S(ch, 10 268, …)`
   with `dir` = falling for s0 = 0, rising for s0 ≥ 1; compute `Sk`, `zero` (§3.6). For s0 ≥ 1 accept
   at once. For s0 = 0 write `zero`, wait 100 ms, accept if the channel average is within 32 768 ± 320,
   else retry. No accepted attempt → the item fails for this channel (nothing saved).
4. After all gears: select the records (§3.6) into slot(bw).
5. After all bandwidths: restore the user bandwidth setting, save `cal_afe.hex` (current size).

Gear table (nominal V/div is informative; only the measured `Sk` is used):

| k | V/div | s0..s6 | k | V/div | s0..s6 | k | V/div | s0..s6 | k | V/div | s0..s6 |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 0 | 0.001 | 0 0 2 0 0 3 1 | 20 | 0.0164 | 0 1 2 0 0 2 2 | 40 | 0.1643 | 1 2 2 3 0 2 0 | 60 | 1.8434 | 2 2 1 3 2 3 0 |
| 1 | 0.0012 | 0 0 2 0 0 3 2 | 21 | 0.0184 | 0 1 1 0 0 3 0 | 41 | 0.1843 | 1 2 2 3 0 2 1 | 61 | 2.0683 | 2 2 1 3 2 3 1 |
| 2 | 0.0013 | 0 0 2 0 0 2 0 | 22 | 0.0207 | 0 1 1 0 0 3 1 | 42 | 0.2068 | 1 2 1 3 0 3 0 | 62 | 2.3207 | 2 2 1 3 1 2 1 |
| 3 | 0.0015 | 0 0 2 0 0 2 1 | 23 | 0.0232 | 0 1 1 0 0 3 2 | 43 | 0.2321 | 1 2 2 3 1 2 0 | 63 | 2.4059 | 2 2 1 3 1 2 2 |
| 4 | 0.0016 | 0 0 2 0 0 2 2 | 24 | 0.026 | 0 1 2 1 0 3 0 | 44 | 0.2604 | 1 2 1 3 0 3 1 | 64 | 2.6038 | 3 2 2 2 0 2 0 |
| 5 | 0.0018 | 0 0 1 0 0 3 0 | 25 | 0.0292 | 0 1 2 1 0 3 1 | 45 | 0.2922 | 2 2 2 2 0 2 1 | 65 | 2.9216 | 3 2 2 2 0 2 1 |
| 6 | 0.0021 | 0 0 1 0 0 3 1 | 26 | 0.0328 | 0 1 1 0 0 2 2 | 46 | 0.3278 | 2 2 1 2 0 3 0 | 66 | 3.278 | 3 2 2 2 0 2 2 |
| 7 | 0.0023 | 0 0 1 0 0 3 2 | 27 | 0.0368 | 0 1 2 1 0 2 0 | 47 | 0.3678 | 2 2 1 2 0 3 1 | 67 | 3.678 | 3 2 1 2 0 3 0 |
| 8 | 0.0026 | 0 0 2 1 0 3 0 | 28 | 0.0413 | 0 1 2 1 0 2 1 | 48 | 0.4127 | 2 2 1 2 0 3 2 | 68 | 4.1268 | 3 2 1 2 0 3 2 |
| 9 | 0.0029 | 0 0 2 1 0 3 1 | 29 | 0.0463 | 0 1 2 1 0 2 2 | 49 | 0.463 | 2 2 2 3 0 3 0 | 69 | 4.6303 | 3 2 2 3 0 3 0 |
| 10 | 0.0033 | 0 0 1 0 0 2 1 | 30 | 0.05 | 0 1 1 1 0 3 0 | 50 | 0.5195 | 2 2 2 3 0 3 1 | 70 | 5.1953 | 3 2 1 2 0 2 0 |
| 11 | 0.0037 | 0 0 2 1 0 2 0 | 31 | 0.0583 | 0 1 2 1 1 2 0 | 51 | 0.5829 | 2 2 2 3 0 3 2 | 71 | 5.8293 | 3 2 2 3 0 3 1 |
| 12 | 0.0041 | 0 0 2 1 0 2 1 | 32 | 0.0654 | 0 1 1 1 0 3 1 | 52 | 0.6541 | 2 2 1 2 0 2 2 | 72 | 6.5405 | 3 2 1 2 0 2 2 |
| 13 | 0.0046 | 0 0 2 1 0 2 2 | 33 | 0.0734 | 1 2 1 2 0 3 0 | 53 | 0.7339 | 2 2 2 3 0 2 0 | 73 | 7.3386 | 3 2 2 3 0 2 0 |
| 14 | 0.0052 | 0 0 1 1 0 3 0 | 34 | 0.0823 | 1 2 1 2 0 3 1 | 54 | 0.8234 | 2 2 2 3 0 2 1 | 74 | 8.234 | 3 2 2 3 0 2 1 |
| 15 | 0.0092 | 0 1 2 0 0 3 0 | 35 | 0.0924 | 1 2 1 2 0 3 2 | 55 | 0.9239 | 2 2 1 3 0 3 0 | 75 | 9.2388 | 3 2 1 3 0 3 0 |
| 16 | 0.0104 | 0 1 2 0 0 3 1 | 36 | 0.1037 | 1 2 2 3 0 3 0 | 56 | 1.0366 | 2 2 2 3 1 2 0 | 76 | 10.3661 | 3 2 1 3 0 3 1 |
| 17 | 0.0116 | 0 1 2 0 0 3 2 | 37 | 0.1163 | 1 2 2 3 0 3 1 | 57 | 1.1631 | 2 2 1 3 0 3 1 | | | |
| 18 | 0.0131 | 0 1 2 0 0 2 0 | 38 | 0.1305 | 1 2 2 3 0 3 2 | 58 | 1.305 | 2 2 1 3 0 3 2 | | | |
| 19 | 0.0146 | 0 1 2 0 0 2 1 | 39 | 0.1464 | 1 2 2 3 1 3 0 | 59 | 1.6429 | 2 2 1 3 1 3 2 | | | |

Slots 4–7 (250/400 MHz and other) are not produced on a DHO1000.

### 11.9 50 Ω vertical

Same as §11.8 with 61 gears, the 50 Ω bandwidths → slots 8–12 (§3.3), `dir` = falling, DAC range
0…0xFFFF, no zero verification. Not selectable on DHO1000/2000 (no 50 Ω input); not specified further.

### 11.10 Host-driven factory procedures

These items need an external source and a host that closes the loop: the host sets the source, asks
the scope to run its side and returns the measured reference value. The command set for this is the
firmware's own (no public SCPI command covers it).

| Item | Scope side | Host side (**UNVERIFIED**) |
|---|---|---|
| LSB | for (ch, gear g = 0…4, offset gear of spec 06 §5 = g): setup L0 (0,1,1,1,0,3,0); L1 (0,0,1,1,0,3,0); L2 (1,2,2,3,0,3,0); L3 (2,2,2,3,0,3,1); L4 (3,2,2,2,0,3,2); AFE zero trims; return `S(ch, 32 768, g ≥ 2 ? rising : falling, 0, 0xFFFF, 100 ms)`; the host's value is stored into `LSB[ch][g]`; save | two (or more) reference DC voltages per gear; `LSB = ΔV / ΔDAC` (sign kept) |
| DAC linearity | gear (1,2,2,3,0,3,0), gear line code for gear 2, attenuated zero words 29–34; `x = S(ch, 32 768, rising, 0, 0xFFFF, 100 ms)`; store point p (0–15) as (x, host value); fit: least squares `y = c0 + c1·x + c2·x² + c3·x³` over all 16 points → `cal_dac[ch]` | 16 reference levels |
| Bandwidth | set trim T(type, s0) into the AFE and `cal_afe_bandwidth`; measure: per-core peak (§11.2) over 262 µs, largest over the cores, scaled by a divisor (whether the host expects the max or the min peak reading is **UNVERIFIED**) | swept/step source |
| Filter | store element (A, B or F) of the triple for a bandwidth code (§5.3) and apply it | |
| Clock | write the two trim-DAC codes (spec 06 §3) and save `cal_clock` | frequency counter on the reference |
| EXT | §7 binary search (15 steps, 100 ms), saving `c` = the value after the 15th halving (not itself measured); the host may also set `c` and `k` directly | EXT input at 0 V |

### 11.11 DDR

Precondition: none. Load the SelfTest bitstream (`SPU_H12S2_SelfTest.bit` for HW codes 1, 2, 3, 9; spec
13); `SP_RESET` bit 3 = 1 then 0 (bit 2 = 0); poll `0x1008` bit 29 every 500 ms (sleep, then read); fail
if it is not set within 51 reads; read the result registers (§9), save `cal_ddr.hex`; wait 8 s; power
off / reboot (the normal bitstream is not restored in place). Failure to load the bitstream, or bit 29
not set in time, fails the item.
