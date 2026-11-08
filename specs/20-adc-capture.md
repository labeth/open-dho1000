# 20 — ADC Capture

How the FPGA receives the single ADC (RT8847IQ, 8 cores, 2 GSa/s) on the DHO1000 single-ADC board
("S2" family), turns the 28 LVDS data lanes into 8 samples per 4 ns fabric clock, aligns them, and the
register-level training interface. Ball/bank assignment and I/O standards are
in `12-fpga-board-interface.md` §4; the register map is `02-register-map.md` §2.5–2.6; what happens to the
samples afterwards (gain/offset, decimation, trigger, record) is `03-acquisition-engine.md`.
`SPU_H12S2B` (XC7A75T) has the same configuration frames as `SPU_H12S2`; this spec applies to it unchanged
(other images: spec 12 §1.2).

Nothing in this spec has been checked on a unit: every value and behaviour is **UNVERIFIED** until it is.

Terms: **lane** = one LVDS data pair; **group** = 14 lanes forming one 14-bit parallel bus (2 groups);
**word** = what one lane delivers per fabric clock (4 bits); **acquisition clock** = 250 MHz fabric clock.

---

## 1. Lanes

28 data lanes in two groups. Lane index `n` (0–13) is the bit position inside the 14-bit ADC code and is
also the lane index used by the training registers (§5). Balls are P / N.

| n | group 0 (bits 28 / `[3:0]` / `[7:0]` fields) | group 1 (bits 29 / `[11:8]` / `[23:16]` fields) |
|---|---|---|
| 0 (LSB) | M7 / L7 (bank 34) | H6 / G6 |
| 1 | H7 / G7 | G4 / F4 |
| 2 | E6 / D6 | D4 / C4 |
| 3 | E5 / D5 | B5 / A5 |
| 4 | D3 / C3 | F8 / F7 |
| 5 | C2 / B2 | H9 / G9 |
| 6 | C1 / B1 | B4 / A4 |
| 7 | A3 / A2 | J4 / H4 |
| 8 | F3 / E3 | J6 / J5 |
| 9 | H8 / G8 | E1 / D1 |
| 10 | L8 / K8 | F2 / E2 |
| 11 | K7 / K6 | G2 / G1 |
| 12 | E21 / D21 (bank 16) | B22 / A22 (bank 16) |
| 13 (MSB) | B20 / A20 (bank 16) | C21 / B21 (bank 16) |

All lanes not marked otherwise are in bank 35. The ADC data clock (DCO) is G5 / F5. Which ADC output
bus is group 0 and which is group 1 is defined only through the registers above (the training fields
for group 0 act on the left column). Which of the two groups carries the earlier sample of an
interleaved pair is **UNVERIFIED** (§3.2).

Ball N3 / N2 (bank 34) is deserialised exactly like a data lane (no input delay) but is **not** part of
the sample words: it is the **EXT trigger comparator** input. Its 4 bits per CLK250 cycle (one per 1 ns
slot) go to the trigger unit as the EXT comparator state, source index `0x31` (spec 05 §3.4, spec 21
§5.5).

## 2. Clocking and deserialisation

| item | value |
|---|---|
| DCO | 500 MHz differential, into the bank-35 clock-capable pair G5/F5 |
| Lane rate | 1 Gb/s per lane, DDR on the DCO |
| I/O clock (CLK/CLKB) | DCO through a regional I/O buffer (BUFIO), 500 MHz; CLKB = inverted CLK |
| Divided clock (CLKDIV) | DCO / 2 through a regional clock divider (BUFR), 250 MHz |
| Acquisition clock | DCO / 2 (BUFR) through a global buffer, 250 MHz; all sample logic |
| Bank 34 (lane M7, EXT trigger lane N3) | own BUFIO/BUFR pair driven from the same DCO |
| Bank 16 (lanes n = 12, 13 of both groups) | 500 MHz and 250 MHz regenerated from the acquisition clock by an MMCM (M = 4, ÷2, ÷4) on global buffers |

Per lane the image must instantiate:

1. `IBUFDS`, LVDS_25, `DIFF_TERM = TRUE` (on-die 100 Ω), `IBUF_LOW_PWR = FALSE`; the same for the
   DCO pair G5/F5 and for N3/N2.
2. `IDELAYE2`: `IDELAY_TYPE = VAR_LOAD`, `DELAY_SRC = IDATAIN` (IDATAIN = the IBUFDS output),
   `HIGH_PERFORMANCE_MODE = TRUE`, `IDELAY_VALUE = 0`, `REFCLK_FREQUENCY = 200`,
   **`IS_IDATAIN_INVERTED = 0`**. `DATAIN` is unused (tie 0; `IS_DATAIN_INVERTED` is don't care).
   `C` = the lane's CLKDIV clock, `CE = INC = 0`, `LDPIPEEN = 0`, `REGRST = 0`, `CINVCTRL = 0`.
   `CNTVALUEIN[4:0]` and `LD` from §5.1.
3. `ISERDESE2`: `INTERFACE_TYPE = NETWORKING`, `DATA_RATE = DDR`, `DATA_WIDTH = 4`, MASTER,
   `IOBDELAY = BOTH` (both the deserialiser and `O` take DDLY; `D` is unused, `IS_D_INVERTED` is
   don't care), `NUM_CE = 2` with both CE tied high, **`BITSLIP` tied 0** (no hardware bit-slip is
   used), `DYNCLKSEL = DYNCLKDIVSEL = 0`.
4. One register stage in the acquisition-clock domain directly behind Q1..Q4.

**Polarity: no lane is inverted.** Neither the I/O primitives nor the fabric invert any lane between
the pad and the sample word: code bit `n` is the IBUFDS output of lane `n` (1 = P ball above N ball).
An image built this way receives the codes the unit's stored calibration (spec 10) was made with. Whether that level is the ADC's output bit or its complement
(pair orientation on the board) is **UNVERIFIED**; it matters only to logic that recognises ADC
patterns by value (§5.4).

The four IDELAYCTRLs (banks 13, 16, 34, 35) use the 200 MHz reference from the reference PLL
(spec 12 §3.2). One tap = 1/(32 × 2 × 200 MHz) = 78.125 ps; 32 taps span 2.42 ns, more than two bit
periods (1 ns).

Lane N3 (EXT trigger, §1) uses the same ISERDES settings but has no IDELAYE2: `IOBDELAY = NONE` (D direct),
`IS_D_INVERTED = 0`.

## 3. Bit and sample mapping

### 3.1 Words to samples

Per acquisition clock every lane delivers Q1..Q4. With ISERDESE2 networking mode **Q4 is the
earliest** bit and Q1 the latest. A sample is formed from the **same Q position of all 14 lanes of one
group**: code bit `n` = lane `n`. One clock therefore carries 2 groups × 4 positions = **8 samples ×
14 bits**:

```
group g, position p in (Q4, Q3, Q2, Q1):   code[13:0] = { lane13.Qp, lane12.Qp, ..., lane0.Qp }
```

Bits are not re-ordered between lanes: lane `n` is weight `2^n`.

### 3.2 Order of the 8 samples

Within a group, sample time order is Q4 → Q3 → Q2 → Q1 after word alignment (§5.3). The two groups are
the ADC's two 1 GSa/s output buses; at 2 GSa/s they interleave, giving the order
`g0.Q4, g1.Q4, g0.Q3, g1.Q3, …`. On the FPGA side (record and TRG, word alignment off) the image uses
exactly this order: `t` = 0…7 = g0.Q4, g1.Q4, g0.Q3, …, g1.Q1; in 2-channel mode group 0 carries the lower
channel; in 3–4-channel mode CH1 = `t` 0/4, CH2 = 2/6, CH3 = 1/5, CH4 = 3/7 (spec 21 §8.2, spec 04 §3.1).
How the 8 ADC cores map onto (group, position), and the order after word alignment, is
**UNVERIFIED** (set by the ADC's output mode, spec 06, and by `SP_MODE.CORE_ORDER`, spec 02 §2.2).

### 3.3 Code format

* The FPGA performs **no Gray decoding**: `CAP_CTRL.GRAY` (`0x1014` bit 3) is stored and read back
  but has no effect on the data path. The ADC must therefore be programmed for a binary output code:
  ADC registers `0x16 + 8·core` bit 15 = 0 on every core (spec 06 §8.2), with `GRAY = 0`. An open
  image may implement a Gray decoder behind bit 3.
* After alignment each sample is **trimmed to 12 bits**: code bits `[13:2]` are used. At `LANE_GAIN` code
  0 lanes `n = 0, 1` of both groups do not enter the sample path (they are then used only by the
  per-lane training checks); at codes 1–3 the gain stage below uses the full 14-bit word. The 12-bit
  value (at `LANE_GAIN` code 0; at codes 1–3 the stage output, ×2.5 or ×5 about mid-scale) is the record
  code `a` of
  `03-acquisition-engine.md` §2 (record domain `a << 4`) and is **offset binary**, `0x800` = mid-scale
  (**UNVERIFIED**; consistent with the ADC calibrations of spec 10 §11.4, §11.8, which centre every
  core at 32 768 and place zero at 32 752 = `0x7FF · 16`, and so need codes that rise monotonically
  through mid-scale). Also **UNVERIFIED**: whether the `STAT_CTRL` path (`0x1044`) applies an MSB
  inversion of its own.
* The per-lane coarse/fine digital gain (`LANE_GAIN` `0x105C`, `LANE_TRIM` `0x1058`) follows the
  alignment. It is a shift-add multiplier about mid-scale on the full 14-bit lane word, ×2.5 (code 1)
  or ×5 (codes 2 and 3) with saturation, and code 0 bypasses it (bits `[13:2]`). The exact function is
  in spec 21 §2.1. Where the `STAT_CTRL` path (`0x1044`/`0x1048`) taps the stream, before or after
  this stage, is **UNVERIFIED**.

### 3.4 Interface boundary

The capture block hands the rest of the SP **8 × 12-bit samples per 250 MHz clock** (plus a constant
latency), in the acquisition-clock domain, continuously. These are the outputs of the `LANE_GAIN` stage
(§3.3). Everything downstream (channel gain/offset, filter, decimation, trigger taps, DDR3 record
path) is outside this spec.

## 4. Latency and timing

| path | value |
|---|---|
| pad → ISERDES Q | ISERDES 1:4 internal pipeline |
| Q → first fabric register | 1 acquisition clock (same 250 MHz frequency; bank-35 CLKDIV and the acquisition clock are separate BUFRs of the same DCO, bank 16 uses the MMCM copies) |
| word alignment | 1–2 clocks (previous + current word are kept to rotate across the word boundary) |
| tap load | `LD` is a one-clock pulse in the lane's CLKDIV domain; the new tap is effective on the next CLKDIV cycle |

The DCO-derived clock is monitored by the FPGA (`CLK_METER` `0x1210`, healthy 62 500 ± 6, spec 02).

## 5. Training interface

All registers are in the SP block of the bypass BAR; writes go to the 125 MHz register file and are
transferred into the 250 MHz domain as described. Bit numbers are of the 32-bit register value.

### 5.1 `0x1018` CAP_TAP — per-lane tap load

| bits | meaning |
|---|---|
| `[4:0]` | tap value (0–31) driven onto `CNTVALUEIN[4:0]` of every lane (bits `[8:5]` are written by software, ignored) |
| `[25:12]` | lane select, bit `12 + n` = lane `n` |
| 28, 29 | group select (bit 28 = group 0, bit 29 = group 1; **UNVERIFIED** which physical column is which, see §1) |

A lane `(g, n)` loads `[4:0]` when the combination `bit(12+n) AND bit(28+g) AND CAP_CTRL[12]`
(`CAP_SEL` bit 0 = ADC 0, §5.5) goes from 0 to 1: the condition is synchronised (two flip-flops) into the
lane's CLKDIV domain and its rising edge is the one-clock `LD` pulse. Hence the required write sequence:
`tap`, `tap | select`, `tap` (the tap bits must be stable while select is high). The tap field is not
synchronised; it must be written first and held. Numbering the lanes `L = 0…27`: `L < 14` is
group 0 lane `n = L`, else group 1 lane `n = L − 14`; select = `1 << (12 + n) | 1 << (28 + g)`, or
`0x3FFFF000` for all lanes at once.

The `CAP_SEL` term is required: a driver written for both board types also loads the 28 lanes of
ADC 1 (with `CAP_SEL` = ADC 1). An image for the 1-ADC board MUST ignore loads while
`CAP_CTRL[12] = 0`, or it loses the trained taps.

### 5.2 `0x1014` bit 2 FLAG_ARM and `0x1024` / `0x1028` CAP_FLAGS — per-lane error flags

Each lane has a sticky error flag in the acquisition domain. A three-write pulse on `0x1014` bit 2
(0→1→0) clears all flags and arms them; afterwards a flag is set in any clock in which the lane's **Q4
bit differs from its Q4 bit one clock (4 ns) earlier**. The check is meant for an ADC test pattern that
is constant per sample position (ADC data mode 2, spec 06), so any flag set means "this tap samples in
the data eye transition".

Read-back (only the low halves are needed):

| register | `[13:0]` | `[29:16]` |
|---|---|---|
| `0x1024` | flags of group 0, bit `n` = lane `n` | **UNVERIFIED** second flag set (not needed) |
| `0x1028` | flags of group 1 | **UNVERIFIED** |

A compatible image must implement `[13:0]` of both; `[29:16]` may mirror `[13:0]`.

The ADC data mode is ADC register `0x0D` bits `[11:8]`; the register is written as
`0x600F | code << 8`, bit 4 cleared, before the XOR mask of spec 02 §2.6. Two codes are needed:
`0x2` (`0x620F`) is the operating mode and `0x5` (`0x650F`) the lane-training pattern (spec 06
§8.2). Codes `0x0`, `0x1`, `0x9` and `0xD` select further ADC patterns. The patterns are defined by
the ADC (**UNVERIFIED**). Below, "data mode 4" means code `0x2` and "data mode 2" code `0x5`.

Lane search (at every start-up and in the self-calibration; taps are not stored). Flag word after
tap `t`: `f[t] = (0x1028[15:0] << 14) + 0x1024[15:0]`, so bit `L` = lane `L` of §5.1 (an image MUST
read bits 15:14 of both registers as 0). The procedure below is one that fits the hardware; its
thresholds are **UNVERIFIED** on a unit.

1. ADC data mode 2 on all ADCs, `0x16 + 8·core` bit 15 = 0, `GRAY = 0`, pulse `CAP_RESYNC`.
2. `CAP_SEL` = ADC 0; for `t = 0…31`: load `t` into all lanes, pulse FLAG_ARM, read `0x1024` and
   `0x1028`.
3. For each lane, find the runs of consecutive taps with the flag clear (the eye). A usable eye is at
   least 9 taps (≈ 0.7 ns) wide. Because 32 taps span more than two bit periods (§2), a lane can show
   two eyes; all lanes of a group must use the eye of the same bit period, so take the reference lane
   (`L = 2`) first and, for every other lane, the eye that contains or lies nearest to the reference
   eye's centre.
4. Each lane's tap is the centre of its chosen eye (taps below 0 become 0).
5. ADC data mode 4 on all ADCs.
6. Load the taps one lane at a time (§5.1) with `CAP_SEL` = ADC 0.

If a lane has no usable eye, repeat the search; if it still fails, load a fixed mid-range tap (14)
on every lane and report the failure.

### 5.3 `0x1020` CAP_ROTATE — word alignment

The 4-bit word boundary of the ISERDES is arbitrary with respect to the ADC's 4-sample frame. Each group
has a 2-bit **rotation** `r` that selects, for all 14 lanes of the group together, which 4 consecutive
bits of {previous word, current word} form the aligned word (4:1 multiplexer per lane and position).

| bits | meaning |
|---|---|
| `[1:0]` | rotation `r` for group 0 (`[3:0]` is written; `[2]` reaches the 250 MHz domain and is 0 on a 1-ADC board) |
| 4 | group 0 alignment enable |
| `[9:8]` | rotation for group 1 (`[11:10]` as above) |
| 12 | group 1 alignment enable |

The register is written in the 125 MHz domain and copied into the acquisition domain only while
`CAP_CTRL[12]` (`CAP_SEL` ADC 0) is 1; set `CAP_SEL` before writing `0x1020`.
The exact mapping of `r` to the rotation direction is internal: the driver only writes back the value the
FPGA reports in `0x1030` (§5.4), so an image may define it freely as long as §5.4 and §5.3 agree.

### 5.4 `0x1030` CAP_ROT_ST and `0x1014[9:8]`

With the ADC in data mode 4 (the operating mode, register `0x0D = 0x620F`) the ADC marks its 4-sample
output frame and the FPGA reports where the mark was found. The samples are real conversions, so the mark is not a sample value: it is presumably carried by
the code bits that do not enter the sample path (lanes `n = 0, 1`, §3.3; lane N3 is the EXT trigger
input, §1), **UNVERIFIED**.

| bits of `0x1030` | meaning |
|---|---|
| `[7:0]` | group 0 alignment result; `[1:0]` is the rotation to write into `0x1020[1:0]` |
| `[23:16]` | group 1 alignment result; `[17:16]` → `0x1020[9:8]` |

Both groups should report the same value (`[7:0] == [23:16]`, the frame at the same position); a
difference points to a misaligned group. `0x1014[9:8]` selects one of four sub-results; only the one
with `[9:8] = 0` is needed (the others' meaning is **UNVERIFIED**). The detection logic itself is **UNVERIFIED**; an image must report the
rotation that aligns the mark, so that writing it back to `0x1020` puts the mark in a fixed position.

Procedure (run at start-up and after every write of ADC register `0x01`, spec 06 §8.2):

1. `0x1014[31:24] = 0x14`; ADC data mode 4 on all ADCs.
2. `CAP_RESYNC` pulse (`0x1000` bit 1: 0, 1, 0), then `SP_RST` pulse (bit 0: 1, 0), on every run of
   the procedure.
3. Per ADC: `CAP_SEL` = that ADC, `0x1020 &= ~0x1010` (both enables off).
4. Per ADC: `CAP_SEL` = that ADC; `0x1014[9:8] = 0`, read `0x1030` (= `R`).
5. Per ADC: `CAP_SEL` = that ADC; `0x1020 = (0x1020 & 0xF0F0) | R[17:16] << 8 | R[1:0]`, then the
   same value with bits 4 and 12 set.

### 5.5 `0x1014` CAP_CTRL fields used by capture

| bits | meaning |
|---|---|
| 0 | capture enable (always 1) |
| 2 | FLAG_ARM pulse (§5.2) |
| 3 | GRAY (stored only, §3.3) |
| `[7:4]` | sub-index for `0x102C` (lane pair 0–6 group 0, 8–14 group 1) |
| `[9:8]` | sub-index for `0x1030` |
| `[19:12]` | CAP_SEL, one-hot ADC: bit 12 = ADC 0 gates tap loads (§5.1) and `0x1020` transfer (§5.3) |
| `[23:20]` | LANE_EN per channel lane (spec 04 §3) |
| `[31:24]` | set to `0x14` during word alignment (§5.4; meaning **UNVERIFIED**) |

### 5.6 `0x101C` CAP_SLIP and `0x102C` CAP_SLIP_ST

The capture needs no per-lane bit-slip (ISERDES `BITSLIP` is tied low and there is no per-lane
rotation; alignment is per group, §5.3). A driver does not need these registers, and an image MAY omit
both; an image that keeps them SHOULD accept writes to `0x101C` and read `0x102C` as 0. Layout, if
kept: `0x102C` is read per lane pair `i` selected by `0x1014[7:4]` (`[31:16]` = lane 2i, `[15:0]` =
lane 2i+1); `0x101C = 0x000FFFF0` addresses all lanes with slip 0. What `0x102C` reads on a unit is
**UNVERIFIED**.

### 5.7 Bring-up order

The capture steps inside the whole start-up order (spec 06 §10):

1. Clocks: DCO present (`0x1210`), IDELAYCTRL ready; `0x1014` bit 0 = 1.
2. ADC register initialisation and quantiser calibration (spec 06 §8.3); calibrated ADC trims,
   including `0x01`, whose every write is followed by a word alignment (§5.4).
3. Sync-delay calibration (spec 06 §8.3), then the lane search and tap loads (§5.2).
4. Word alignment with the `CAP_RESYNC` and `SP_RST` pulses (§5.4); the ADC stays in data mode 4 and
   the samples are valid.

## 6. ADC configuration port (for reference)

`0x1034` / `0x1038` drive a 4-wire serial port (spec 02 §2.6, spec 12 §7.1): F19 bidirectional data,
F18 serial clock (**UNVERIFIED**), **G15 = chip select of ADC 0 (`0x1034` bit 24)**, F15 = chip select of
ADC 1 (bit 25, unused on this board); selects are low while a frame is in progress. A frame is 24 bits,
MSB first: bit 23 (1 = read), address `[22:16]`, data `[15:0]`. The F19 output enable depends on bit 23:
for a read the FPGA releases F19 for the data phase (**UNVERIFIED** turnaround point) and shifts the
returned bits into `0x1038[15:0]`.
