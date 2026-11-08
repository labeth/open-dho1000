# 05 — Triggering

The trigger is **fully digital and in the FPGA** (block TRG, `0x2000`–`0x226F`). Per analog channel
the TRG compares the sample stream against two programmable Schmitt-trigger threshold pairs, turns the
comparator outputs into edge and state signals, qualifies them in three event engines (A, B, R) by
time, logic pattern or a protocol decoder, applies holdoff and hands one trigger strobe to the SEQ
record FSM (spec 03), which places it in the record by the pre/post-trigger counts. The CPU only
programs registers; it never searches the record for a trigger. The external trigger input is the
only analog comparator (threshold DAC `EXT_THRESH`, `0x1404`).

Units in this spec: voltages in nV, times in fs (spec 02 §1.4). All register names refer to
spec 02 §5; this document is the authoritative definition of the TRG fields.

**Nothing in this spec has been measured on a unit; every value and behaviour is UNVERIFIED** until
it is. Items marked **UNVERIFIED** inline are open even in the description itself.

Two levels of statement are used below. **Encodings** (what value goes into which field for which
trigger setting) are what a driver writes to the FPGA image the unit ships with (the factory image).
**Engine behaviour** (what the FPGA does with the fields) describes the factory image (spec 21,
cited where used). An FPGA implementation MUST at least produce the behaviour of §9 for the field
values of §8. Where an encoding does not give the behaviour the trigger setting implies, §8 states
what the hardware does and adds a **Recommendation** for an open driver or TRG.

---

## 1. Signal flow

```
 CHn samples (record domain, pre PB_GAIN) ─┬─▶ [coupling filter, only CH = TRG_COUPLING[3:2]] ─▶ CMP A(n): Schmitt THR_A(n)
                                           │                                            └▶ CMP B(n): Schmitt THR_B(n)
 D0..D15 (MSO), EXT comparator, AC line ───┴──────────────────────────────────────────────────▶ (1-bit inputs)
                comparator outputs ──▶ event engines A, B, R (EV_CFG/EV_TMIN/EV_TMAX/EV_THI/EV_IN*)
                                     ──▶ mode logic (TRG_SEL.TRG_KIND: single A, A|B, ABR sequence,
                                         protocol decoders 0x2100–0x21F0, video)
                                     ──▶ holdoff (HOLDOFF_*) ──▶ trigger strobe ──▶ SEQ (+ AUX OUT)
 force (TRG_SEL.5) and SEQ auto-trigger timeout inject a trigger without a qualified event.
```

The TRG has **4 sample slots per channel per 4 ns clock**, 12-bit (record-code bits `[15:4]`), and
evaluates the comparators per slot (spec 21 §2). Event times are counted in slots, and every
width, timeout and delay is programmed in 1 ns ticks, so one slot = 1 ns in every channel mode. The slots are filled from the channel's samples (spec 21 §2.1):

| Active channels | Samples per channel per clock | Slots 0–3 hold | Consequence |
|---|---|---|---|
| 1 (2 GSa/s) | 8 | per 1 ns: one of the two 0.5 ns samples, chosen by the peak rule of spec 21 §2.1 | the maximum of every 2 ns half is kept in its 1 ns slot; steps land within ±0.5 ns; a 0.5 ns dip can be lost |
| 2 (1 GSa/s) | 4 | the samples at 0, 1, 2, 3 ns (every sample) | 1 ns exact |
| 3–4 (500 MSa/s) | 2 | the two samples (2 ns apart), each in two slots | times quantised to 2 ns |

An open TRG MAY use every sample (8 per clock in 1-channel mode) as long as it keeps 1 ns time
units. Behaviour of the factory image behind these registers (storage, comparator, filter,
output and force logic) is specified in spec 21; the sections below give both what to write and,
where it matters, what the factory image does with it.

Every trigger change is followed by a re-run of the acquisition configuration (stop, reprogram,
re-arm, spec 03 §3). TRG registers are written only while the record FSM is stopped; writes while it
runs are not known to work. Exception: `FORCE` is pulsed while running.

---

## 2. Register summary (TRG block)

| Offset | Name | Fields (details in the section given) |
|---|---|---|
| `0x2004`, `0x200C`, `0x20B8`, `0x20C0` | THR_A CH1–4 | `[15:0]` upper threshold, `[31:16]` lower threshold (§3); the factory image stores `[15:4]`, `[31:20]` |
| `0x2008`, `0x2010`, `0x20BC`, `0x20C4` | THR_B CH1–4 | same (§3) |
| `0x2014` / `0x202C` / `0x2044` | EV_IN1 A/B/R | 2 bits per input 0–15 (§5.4) |
| `0x2018` / `0x2030` / `0x2048` | EV_IN2 A/B/R | `[9:0]` inputs 16–20, `[11:10]` combine op (§5.4; not stored, always AND) |
| `0x201C` / `0x2034` / `0x204C` | EV_CFG A/B/R | comparator/edge selects, output selects, TQ_MODE (§5.1) |
| `0x2020` / `0x2038` / `0x2050` | EV_TMIN A/B/R | lower time bound, ns, bits 31:0 (§5.3) |
| `0x2024` / `0x203C` / `0x2054` | EV_TMAX A/B/R | upper time bound, ns, bits 31:0 |
| `0x2028` / `0x2040` / `0x2058` | EV_THI A/B/R | `[15:0]` lower bound bits 47:32, `[31:16]` upper bound bits 47:32 |
| `0x205C` | TRG_SEL | `[4:0]` TRG_KIND, `5` FORCE, `[10:6]` ABR enables, `11` ABR_CLR, `12` TRG_PATH (§6); stored `[13:0]` |
| `0x2060` / `0x2064` | AB_DLY_LO / AB_DLY_HI | 4 ns ticks — not needed by any type of §8 |
| `0x2068` / `0x206C` | R_IDLE_LO / R_IDLE_HI | 4 ns ticks (§8.12) |
| `0x2070` / `0x2074` | A_COUNT / B_COUNT | event counts (§8.12); B not needed |
| `0x2080` | HO_COUNT | event-count holdoff — not needed |
| `0x2084` / `0x2088` | HO_TIME_LO / HO_TIME_HI | 4 ns ticks; `0x2088[18:16]` holdoff type, written 4, not stored (§7) |
| `0x208C` | TRG_COUPLING | `[1:0]` coupling filter, `[3:2]` channel, `[17:4]` coefficient (read back only, §4) |
| `0x2100`–`0x2104` | UART | §8.14 |
| `0x2108`–`0x2114` | I2S | §8.14 |
| `0x2118`–`0x2138` | LIN | §8.14 |
| `0x213C`–`0x2164` | CAN | §8.14 |
| `0x2168`–`0x2174` | FlexRay | §8.14 |
| `0x2178`–`0x2188` | SPI | §8.14 |
| `0x218C`–`0x21A0` | I2C | §8.14 |
| `0x21A4` / `0x21A8` / `0x21AC` | VID_SEL / VID_SYNC / VID_LINES | §8.4 |
| `0x21B0`–`0x21BC` | MIL-STD-1553B | §8.14 |
| `0x21D0`–`0x21F0` | SENT | §8.14 |
| `0x221C` (write) | UART_BIT_HI | UART clock divider bits 30:21 in `[9:0]` (shares the offset with the counter read-back FC_T0_L) |
| `0x2264` | EXT_STATE | R `[3:0]` EXT comparator state; W bit 16 (§3.4) |
| `0x1404` (SP block) | EXT_THRESH | EXT comparator DAC code `[15:0]` (§3.4) |
| `0x1430` (SP block) | AUXOUT_SEL | `[2:0]` rear AUX OUT source: write 0 for trigger out, 3 for pass/fail; full decode §10 |

Treat all TRG configuration registers as write-only and write them as whole words from a software
shadow (spec 02 §1.3). For TRG_SEL and HO_TIME_HI keep the bits a write does not address (16-bit
read-modify-write of `[15:0]` on the shadow). Initial value of every shadow is 0. The factory image
decodes only address bits A2–A9 in the TRG (`0x2000`–`0x23FF` aliases at `0x2400`, `0x2800`,
`0x2C00`) and its configuration registers read back the last written value (spec 21 §1). Engine B
and R `EV_IN1/2`, engine R's time bounds and engine R's `TQ_MODE` need not be written by any
type of §8; they keep their configuration-time values (spec 02 §5.2).

---

## 3. Sources, comparators and thresholds

### 3.1 Comparator index

Event engines select an input by a 6-bit **comparator index**:

| Source | Index of comparator A | Index of comparator B |
|---|---|---|
| CH1…CH4 | `2·(ch−1)` = 0, 2, 4, 6 (uses THR_A(ch)) | `2·ch−1` = 1, 3, 5, 7 (uses THR_B(ch)) |
| D0…D15 | `32 + n` = 32…47 | same |
| AC line | `0x30` | `0x30` |
| EXT | `0x31` | `0x31` |
| none / invalid | `0x33` | `0x33` |

AC line and EXT are 1-bit inputs with no threshold in the FPGA (spec 21 §5.5). **AC line** is a
mains-synchronous digital input (ball A18, spec 12): synchronised, then edge-qualified, i.e. an input
edge is passed only if the input held the previous level for ≥ 4096 clocks (16.384 µs); rejected
edges are ignored; event latency ≈ 24 clocks from the pad. **EXT** is §3.4. For both, the slope is
taken from `OINV` (§9 detector (2,2)).

An implementation MUST accept 32–47 for D0–D15.

When an analog channel is a trigger source it is forced on in the acquisition channel mask (spec 04
§3) even if not displayed. For an Edge trigger on EXT or AC line with no analog channel on, CH1 is
forced on.

### 3.2 Threshold registers (Schmitt pairs)

Each threshold register (THR_A, THR_B) holds one comparator's two switching thresholds in ADC record codes:

- `[15:0]` **upper** threshold `U`: the comparator output goes 1 when a sample is **strictly above**
  `U` (`sample > U`);
- `[31:16]` **lower** threshold `Lo`: the output goes 0 when a sample is **at or below** `Lo`
  (`sample ≤ Lo`); otherwise it holds.

The comparator state is evaluated per 1 ns sample slot, in time order within each 4 ns clock
(`state = (sample > U) | (state_prev & (sample > Lo))`, spec 21 §2.2); state 1 = "above". The
software writes 16-bit codes; the factory image compares the 12-bit values `sample[15:4]` against
`U[15:4]` and `Lo[15:4]` (bits 3:0 of each threshold are ignored). An open image MAY compare at full
16-bit resolution.

With hysteresis `h` the exact trigger level `L` goes on the threshold that the selected edge
uses: a rising-edge comparator gets `(U, Lo) = (L, L−h)`, a falling-edge comparator
`(L+h, L)`. Default assignment (all types not listed below):

| Register | `[15:0]` | `[31:16]` | used for |
|---|---|---|---|
| THR_A(ch) | `code(L)` | `code(L − h)` | rising edges at exactly `L` |
| THR_B(ch) | `code(L + h)` | `code(L)` | falling edges at exactly `L` |

Types with two levels (Slope 2, Runt 7, Window 8, 1553B 22) use `L_A` = upper level for THR_A and
`L_B` = lower level for THR_B, with per-type direction:

| Type, setting | THR_A | THR_B |
|---|---|---|
| Slope, rising (slope 0) | `(L_A, L_A − h)` | `(L_B, L_B − h)` |
| Slope, falling (slope 1) | `(L_A + h, L_A)` | `(L_B + h, L_B)` |
| Window, event "exit" (1) | `(L_A + h, L_A)` | `(L_B, L_B − h)` |
| Window, event "enter" (0) / "time" (2) | `(L_A, L_A − h)` | `(L_B + h, L_B)` |
| Runt, 1553B | `(L_A, L_A − h)` | `(L_B + h, L_B)` |

The direction flags are global: when they change, all four channels' THR_A/THR_B are rewritten.
For single-level types `L_A = L_B = L` (the trigger level of the source channel).

### 3.3 Level → code

For channel `ch` with vertical scale `S` (nV/div), vertical offset `O` (nV), residual digital gain `G`
(the calibration-record gain of the current scale, spec 10 §3.5 step 4, **not** halved when `G ≥ 2`
and not the stopped-rescale `PB_GAIN`; spec 03 §2; the TRG sees samples before the play-back gain,
i.e. `7500/G` codes per division; see spec 03 §2 for the **UNVERIFIED** `G ≥ 2` case):

```
code(V) = trunc( (V + O) · 7500 / S / G + 32768 )      # toward zero; clamp: ≤ 0 → 1, ≥ 0xFFFE → 0xFFFE
h       = trunc( NR · S )                              # hysteresis in nV, NR from §3.5
```

The example level words below take `NR · S` in single-precision arithmetic, which at 50 mV/div gives
`h` = 15 000 001 nV instead of the exact 15 000 000; the difference can move a threshold by one code.
Either value works.

`code(L ± h)` is computed as `code(V)` of the shifted voltage (offset added after the shift). The
offset term `O` is omitted for Edge type with coupling AC (level relative to the AC-coupled mean). If
`S = 0` the level is not written.

Useful level range (clamp before conversion): analog `[−4.5·S − O, +4.5·S − O]` (±4.5 div around
screen centre; not clamped for Edge with coupling AC or LF reject), digital channels ±20 V, EXT
±8 V. For two-level types keep `L_A ≥ L_B + S/10`.

Example (shipped default calibration, spec 10 §3.3: CH1 `S` = 50 mV/div, `G` = 1.08798392, `O` = 0, NR
off, `h` = 15 000 001 nV): `L` = 0 → THR_A `0x77EB8000`, THR_B `0x80008814`; `L` = 100 mV →
THR_A `0xADC6B5DA`, THR_B `0xB5DABDEF`; `L` = −100 mV → `0x42104A25` / `0x4A255239`; `L` =
−225 mV (clamp) → THR_A `0x000106D3`. NR on (`h` = 35 000 000 nV), `L` = 0 → `0x6D268000` /
`0x800092D9`. `L` = 100 mV, `O` = 100 mV → THR_A `0xE3A1EBB5`.

### 3.4 External trigger input

The level goes to the EXT comparator DAC, not to a threshold register:

```
EXT_THRESH (0x1404) = min( trunc_u(L / 1e9 · k + c), 0xFFFF )    # L in nV, double; trunc_u: toward zero, negative → 0
k, c = cal_ext.hex {i32 k, i32 c} (spec 10 §7); built-in default k = 2074 codes/V, c = 30012
```

(0 V → 30012, +1 V → 32086, −1 V → 27938, +5 V → 40382 with the defaults.) The comparator output is
read at `EXT_STATE[3:0]` (nonzero = input above the DAC threshold) and enters the event engines as
index `0x31`. It reaches the FPGA as the extra LVDS lane of the ADC interface (balls N3/N2, spec 20
§1), deserialised like an ADC lane: 4 bits per clock, one per 1 ns slot, used directly as the
comparator state (spec 21 §5.5). EXT offset calibration (input at 0 V): 15-step binary search on `c` over
`[0, 0xFFFF]` starting at `0x8000`, wait 100 ms per step, read `EXT_STATE[3:0]`, nonzero → raise the
lower bound to the current code. Clear `EXT_STATE` bit 16 whenever the noise-reject setting is
applied (both on and off); its function (comparator hysteresis) is **UNVERIFIED**: in the factory
image the bit has no known effect (spec 21 §1.2).

With EXT as the trigger source the pre/post-trigger split includes a latency compensation of 119 ns
(1-channel mode), 400 ns (2 ch) or 401 ns (3–4 ch) (spec 02 §6.6); these are working values. In the
factory image the EXT lane reaches the event 20 clocks after its deserialiser, an analog edge 44 /
141 / 111 clocks after the ADC deserialiser (1 / 2 / 3–4 channels; the multi-channel paths include a
FIFO whose start-up latency is **UNVERIFIED**), so an analog edge arrives 96 / 484 / 364 ns later
than an EXT edge. The
1-channel value matches the 119 ns compensation (96 ns + ADC pipeline); the 2- and 3–4-channel
values do not match 400 / 401 ns, and the hardware multi-channel latency is **UNVERIFIED** (measure:
one edge into EXT and an analog channel, compare the trigger positions in 2- and 4-channel mode).
An open image MUST make its own EXT-vs-analog latency difference known to the firmware, which
compensates it.

### 3.5 Noise reject

The noise-reject setting selects the hysteresis fraction `NR` (in divisions): **0.3 div when
off, 0.7 div when on**. There is always hysteresis. Changing it rewrites THR_A/THR_B of all four
channels and clears `EXT_STATE` bit 16.

---

## 4. Coupling (trigger filter)

| Coupling | `TRG_COUPLING[1:0]` | `TRG_PATH` |
|---|---|---|
| DC | 0 | 0 (1 if the source is not CH1–4) |
| AC | 3 | 1 |
| LF reject | 2 | 1 |
| HF reject | 1 | 1 |

```
TRG_COUPLING = (shadow & 0xFFFC0000) | code[1:0] | (ch − 1) << 2 | 0x800 << 4
```

`[3:2]` = channel index 0–3 of the source (`ch` = 1–4). Write the coefficient `[17:4]` as `0x800`.
Coupling applies only to Edge with an analog source; every other trigger type writes `TRG_COUPLING` with
code 0 (DC) for each channel it uses (Pattern and Duration: all four channels). A driver MUST write
`TRG_COUPLING` (code, source) every time the Edge configuration is applied, so that the filter, `TRG_PATH`
(§6) and the level code (`O` omitted for AC, §3.3) always agree.

Filter behaviour (factory image, spec 21 §2.4 and §3):

- There is **one** coupling filter. It acts only on the channel selected by `TRG_COUPLING[3:2]` and only
  when `TRG_COUPLING[1:0]` ≠ 0; every other channel's comparators see raw samples. The last `TRG_COUPLING`
  write wins. The coefficient field has no effect (the filter constants are fixed in the logic).
  `TRG_SEL.TRG_PATH` does not take part in this selection (§6).
- Input: one sample of the selected channel per 4 ns clock (250 MSa/s), 12-bit.
- Code 1, **HF reject**: first-order low-pass, **−3 dB at 91.3 kHz**
  (`y[n] = 158·x[n] + 32618·⌊y[n−2]/2^15⌋`, output `y[26:15]`, DC gain 1.053).
- Code 2, **LF reject**: `x − LPF` with the same 91.3 kHz corner (first-order high-pass).
- Code 3, **AC**: `x − mean`, where `mean` is the average of the LPF output over consecutive blocks
  of 2^30 clocks (4.295 s), updated once per block (DC tracking only; meaningless during the first
  block after configuration).

An open image MUST pass code 0 unfiltered and MAY implement codes 1–3 as standard first-order
filters with ≈ 91 kHz (HF/LF reject) and ≤ 1 Hz (AC) corners. The re-centring and saturation of the
code 2/3 outputs in the factory image are **UNVERIFIED** (spec 21 §3.2). The factory low-pass feeds
back only `⌊y/2^15⌋`, so it has a dead band: its settled output depends on the signal history
within a band of ≈ 218 12-bit codes (≈ 0.5 div; one division is 7500/G/16 ≈ 430 12-bit codes), i.e.
up to ≈ ±0.25 div of level uncertainty for HF reject, LF reject and AC (spec 21 §3.1). An open image SHOULD NOT copy this.

---

## 5. Event engines A, B, R

Three identical engines. In the single-engine modes A produces the trigger; B is the second
alternative (TRG_KIND 1) or the qualifier of a sequence; R is the reset/idle engine of the ABR
sequence and the data input of the protocol decoders.

### 5.1 EV_CFG

| Bits | Field | Meaning |
|---|---|---|
| 5:0 | `CMP1` | comparator index of input 1 (usually the **end** event; role per detector, §9) |
| 7:6 | `EDGE1` | edge of `CMP1`: 0 = rising (output 0→1), 1 = falling (1→0), 2 = either edge, 3 = never |
| 13:8 | `CMP2` | comparator index of input 2 (usually the **start** event) |
| 15:14 | `EDGE2` | edge of `CMP2`, same encoding |
| 18:16 | `OSEL1` | output select 1; decoded together with `OSEL2` as a detector pair (§9) |
| 19 | `OINV1` | polarity 1 (§9) |
| 22:20 | `OSEL2` | output select 2 |
| 23 | `OINV2` | polarity 2 |
| 26:24 | `TQ_MODE` | time qualifier (§5.2) |

The factory image stores all of `[26:0]` in every engine. Some detectors ignore the `EDGE` fields
or `CMP2` (§9).

Write order for one engine: (`TQ_MODE`, then `EV_TMIN`, `EV_TMAX`, `EV_THI`) as one
group, then `[15:0]` (comparators/edges, upper half kept), then `[23:16]` (outputs, other bits
kept). Each group is one full-word write of the shadow.

Edge encoding: a **rising** signal edge at level `L` is
(`CMP1/2` = comparator A, `EDGE` = 0); a **falling** edge at `L` is (comparator B, `EDGE` = 1)
(comparator A rises exactly at `L`, comparator B falls exactly at `L`, §3.2).

### 5.2 TQ_MODE (time qualifier)

| `TQ_MODE` | Event at the detector's end instant when (`Δt` = slots from start to end, §9; `T_MIN`, `T_MAX` = §5.3) | Condition |
|---|---|---|
| 0 | `Δt ≥ T_MIN` | greater than |
| 1 | `Δt < T_MAX` | less than |
| 2 | `T_MIN ≤ Δt < T_MAX` | inside |
| 3 | `Δt < T_MIN` or `Δt ≥ T_MAX` | outside |
| 4 | timeout, no end instant needed: see below | Timeout/Window |
| 5 | **never** (factory image: no event in any detector) | — |
| 6, 7 | never | — |

`Δt` is counted in 1 ns sample slots from the start instant to the end instant, independent of the
slot phase within the 4 ns clock. Lower bounds are inclusive, upper bounds exclusive.
Edge-like types write `TQ_MODE = 0` with `T_MIN = T_MAX = 0`.

A time condition of "none" (any width) is encoded as `TQ_MODE` 0 with `T_MIN = T_MAX = 0`
(fires at every end) for every type; `TQ_MODE` 5 never fires on the factory image. An open TRG MAY
fire on `TQ_MODE` 5 at every end ("any width").

**TQ_MODE 4 (timeout)**: with the start in slot `S = 4c + s` (clock `c`, slot `s`), the event is
issued in the clock that contains slot `S + T_MIN` if the detector's end has not occurred by the
last slot of that clock, i.e. the qualifying level must last ≥ `4·⌊(S + T_MIN)/4⌋ + 4 − S` slots
(start slots 0–3: `T_MIN` = 1000 → 1004, 1003, 1002, 1001; `T_MIN` = 1002 → 1004, 1003, 1006, 1005;
`T_MIN` = 1001, start slot 3 → 1005, in the next clock). An open TRG SHOULD fire at exactly
`S + T_MIN`.

### 5.3 Time fields

```
lo = t_low_fs  / 1 000 000      # ns, signed 64-bit, truncation toward zero
hi = t_high_fs / 1 000 000
EV_TMIN  = lo & 0xFFFFFFFF
EV_TMAX = hi & 0xFFFFFFFF
EV_THI  = ((lo >> 32) & 0xFFFF) | ((hi >> 32) & 0xFFFF) << 16
```

`T_MIN` = `lo` and `T_MAX` = `hi` are the 48-bit lower and upper bounds used in §5.2 and §9.

The event time base is 1 ns slots at every sample rate (48-bit range, ≈ 78 h); the slot content per
channel mode and the resulting 2 ns quantisation (3–4 channels) and ±0.5 ns step placement (1 channel) are
in §1. Limit widths to 10 s (an upper bound of 10 s is `EV_TMAX = 0x540BE400`,
`EV_THI = 0x00020000`) and, for the two-bound conditions (TQ_MODE 2 and 3), keep the upper bound
above the lower one. With `step(x)` = 1 % of the decade of `x` (`10^(⌊log10 x⌋ − 2)` fs, at least
1 fs) and the type's range `[min, max]` (pulse width: 10⁵ fs … 10¹⁶ fs), one workable clamp rule is:

```
set upper U:  U = max(min(U, max), min + step(min)) ;  L = min(L, U − step(U))
set lower L:  L = min(max(L, min), max − step(max)) ;  U = max(U, L + step(L))
```

With the other conditions only the bound that is set is clamped to `[min, max]`. Any rule that keeps
`T_MIN < T_MAX` for TQ_MODE 2 and 3 will do; with `T_MIN ≥ T_MAX`, TQ_MODE 2 never fires and
TQ_MODE 3 always fires (§5.2).

### 5.4 Logic (pattern) registers

Input index `i`: 0–3 = CH1–CH4 (comparator A output), 4–19 = D0–D15, 20 = EXT. Two bits per input:

| code | condition |
|---|---|
| `00` | don't care (X; also used for the edge input) |
| `01` | never true (not needed) |
| `10` | input **high** (comparator state 1, above the threshold) |
| `11` | input **low** (comparator state 0) |

`EV_IN1[2i+1:2i]` for `i` ≤ 15; `EV_IN2[2(i−16)+1:2(i−16)]` for 16 ≤ `i` ≤ 20.
Per input the pattern condition is H, L, X (don't care), rising or falling; H → `10`, L → `11`,
X/rising/falling → `00` (the edge input is carried by EV_CFG, §8.5). Example: CH1 = L, CH2 = H,
rest X → `EV_IN1 = 0x0000000B`. The factory image decodes the terms the same way (spec 21 §2.3:
`10` true while the state is 1, `11` true while it is 0). EXT (input 20) is encoded like the other
inputs.

`EV_IN2[11:10]` = combine operator field: 0 = AND, 1 = OR, 2 = NOR; write 0. The factory image
does not store these bits: the input terms are **always ANDed**.

The pattern `P` (AND of the terms) is used only by the pattern detectors of §9 ((0,0), (2,0),
(0,2)). Every other detector ignores `EV_IN`, so an engine-A pattern left by a Pattern or
Duration trigger has no effect on later types.

---

## 6. TRG_SEL (`0x205C`)

| Bits | Field | Meaning |
|---|---|---|
| 4:0 | `TRG_KIND` | 0 = engine A; 1 = engine A **or** B; 3 = A/B/R sequence (Nth edge); 4 UART; 5 I2S; 6 LIN; 7 CAN; 8 FlexRay; 0xA MIL-1553B; 0xB SENT; 0xC SPI; 0xD I2C; 0xE video. 2, 9, 0xF+ unused |
| 5 | `FORCE` | write 1 then 0: inject one trigger now (the factory image acts on the 0→1 edge) |
| 10:6 | `ABR_EN[4:0]` | sequence enables; Nth edge writes `1,1,1,0,1` (bits 6,7,8,10) |
| 11 | `ABR_CLR` | pulse 1→0 resets the sequence (not needed by §8) |
| 12 | `TRG_PATH` | trigger-timing path flag (below) |

The FORCE bit is synchronised into the 4 ns domain; its rising edge sets a pending force that the
SEQ consumes as one trigger. Leaving the bit at 1 forces only once.

`ABR_EN` is only programmed by the Nth-edge trigger and need not be cleared when another type is
selected; the bits MUST be ignored unless `TRG_KIND` = 3. The working value is `0x5C0`
(bits 6, 7, 8, 10), whose behaviour is §8.12. Other values in the factory image: bit 8 off → the count never restarts on idle; bit 11 set → never fires; bit 10 off → the
other edge of engine A's output is counted; bit 6 alone → modulo-N count without idle restart. The
individual meaning of each bit is **UNVERIFIED**; an open TRG need implement only `0x5C0`.

`TRG_PATH` (bit 12) working values: 1 for digital/EXT/AC-line sources, for Edge with coupling ≠
DC, and for Pattern and all protocol triggers; 0 for Duration, Runt, Window and analog-source Edge
(DC), Pulse, Slope, Timeout, Nth, Delay, Setup/Hold. Video and SENT need no particular value.

Factory image (spec 21 §5.3): the bit does **not** select the comparator input and does not gate the
coupling filter (§4). At arm time the SEQ registers a flag = `TRG_PATH` OR (engine A `CMP1` =
`0x31`, EXT) OR (protocol `TRG_KIND`) OR (engine A `OSEL1` = `OSEL2` = 0); the flag switches the SEQ
between two trigger-timing inputs (which is which and their latency difference: **UNVERIFIED**). An
implementation MUST give identical comparator results for both values and SHOULD apply its own
trigger-latency compensation.

---

## 7. Holdoff

```
n = t_fs / 4 000 000                                  # 4 ns ticks, truncation
HO_TIME_HI[18:16] = 4                               # time holdoff (the only type needed; not stored in the factory image)
HO_TIME_LO        = n & 0xFFFFFFFF
HO_TIME_HI[15:0]  = (n >> 32) & 0xFFFF
```

Write order: type (`0x2088`), `0x2084`, `0x2088`; then re-run the acquisition configuration.
Range 8 ns … 10 s (8 ns → 2, 100 ns → 25, 1 s → `0x0EE6B280`, 10 s → `0x9502F900`). A working
choice is holdoff 8 ns for Video, Timeout, Nth edge and every protocol trigger except SENT, and the
user value for the other types. `HO_COUNT` (event-count holdoff) is not needed.

Factory image (spec 21 §4): time holdoff only, between the event engines and the SEQ, independent of
the SEQ state. An engine event passes when at least `n + 2` CLK250 cycles have elapsed since the
previous passing event, whether or not the SEQ accepted that event; events in between are dropped
and do not restart the timer. So the 8 ns minimum (`n` = 2) gives 16 ns event spacing. An
open image SHOULD use `≥ n` cycles and MAY keep the `+2`.

---

## 8. Programming per trigger type

`cA`, `cB` = comparator A/B index of the type's source (§3.1); `src(..)` = per-input sources of
multi-input types. `Src(e, c1,e1, c2,e2)` = EV_CFG `[15:0]` of engine `e`; `Out(e, s1,i1, s2,i2)` =
`[23:16]`; `T(e, type, lo, hi)` = §5.2/5.3. `when'` = the `TQ_MODE` of the type's time condition
(§5.2). Only the engines listed are written; the others keep earlier values (harmless in the given
mode).

The TRG serves the types of §8.1–§8.14. A zone trigger is not a TRG function (it belongs to the SP
mask engine).

### 8.1 Edge

`T(A,0,0,0)`, `T(B,0,0,0)`, then per slope:

| slope | engine A | engine B | TRG_KIND |
|---|---|---|---|
| rising | `Src(cA,0, cB,1)`, `Out(2,1,2,1)` | — | 0 |
| falling | `Src(cB,1, cA,0)`, `Out(2,0,2,0)` | — | 0 |
| either | `Src(cB,1, cA,0)`, `Out(2,0,2,0)` | `Src(cA,0, cB,1)`, `Out(2,1,2,1)` | 1 |

EXT (26) / AC line (25): `Src(A: x,0, x,0)` and `Src(B: x,0, x,0)` with `x` = `0x31` / `0x30`; the
`Out` values and TRG_KIND follow the slope as in the table (the edge fields stay 0, the slope is
carried only by `OINV1/OINV2`). Coupling §4, level §3. A working start-up setting: CH1 rising DC,
`EV_CFG_A = 0x00AA4100`, `TRG_SEL = 0`. Behaviour (detector (2,2), §9): rising fires when comparator A goes 1 (sample >
`L`), falling when comparator B goes 0 (sample ≤ `L`); EXT/AC line fire on the input's rising
(`OINV` 1) or falling (`OINV` 0) change.

With channel invert (`SP_MODE.INV` of the source channel) on, program the **opposite**
slope's row, so the trigger stays on the displayed slope: CH1 rising with invert on writes
`EV_CFG_A = 0x00220041`. An invert toggle also negates the trigger level of that source
(two-level types: `L_A' = −L_B`, `L_B' = −L_A`); the level codes are then computed by §3.3 from the
negated level with the channel offset `O` unchanged. Example (CH1 as in §3.3, `L` = 100 mV): invert
on → level −100 mV, THR_A `0x42104A25`; then `O` = 100 mV → `0x77EB8000`; invert off → level
100 mV, `0xE3A1EBB5`.

### 8.2 Pulse width

| polarity | engine A | 
|---|---|
| positive | `Src(cB,1, cB,0)`, `Out(2,0,1,0)` (width from comparator B rising, sample > `L + h`, to comparator B falling, sample ≤ `L`) |
| negative | `Src(cA,0, cA,1)`, `Out(2,1,1,0)` (width from comparator A falling, sample ≤ `L − h`, to comparator A rising, sample > `L`) |
| any | not supported by this encoding |

`T(A, when', lower, upper)`, TRG_KIND 0, coupling code 0. Example (positive, inside, lower 1 µs,
upper 2 µs): `EV_CFG_A = 0x02120141`, `EV_TMIN = 1000`, `EV_TMAX = 2000`, `EV_THI = 0`. Behaviour (detector (2,1)): the event is at the pulse's end edge; each
pulse is measured on its own (the end edge clears the start whether or not the width qualifies).

### 8.3 Slope / rise-fall time

Levels §3.2 (upper `L_A`, lower `L_B`).

| slope | engine A |
|---|---|
| rising | `Src(cA,0, cB,0)`, `Out(2,1,1,0)` (from crossing `L_B` up to crossing `L_A` up) |
| falling | `Src(cB,1, cA,1)`, `Out(2,0,1,0)` (from crossing `L_A` down to crossing `L_B` down) |

`T(A, when', lower, upper)`, TRG_KIND 0.

Behaviour (detector (2,1), first start wins): rising starts when comparator B goes 1 (> `L_B`) and
ends, with the event, when comparator A goes 1 (> `L_A`); falling starts when comparator A goes 0
(≤ `L_A`) and ends when comparator B goes 0 (≤ `L_B`). A return across the start level before the
end level is reached neither cancels nor restarts the measurement: `Δt` is measured from the
**first** crossing of the start level after the previous end. A signal that crosses `L_B`, falls
back, and later rises through both levels is therefore measured from the first crossing (too long:
spurious "greater than" triggers, missed "less than" triggers).

Recommendation: an open TRG SHOULD restart the measurement on every start-level crossing in the
transition direction (Δt = the last crossing to the end crossing). With the factory image this is
available through detector (2,3) ("a later start replaces the pending one", §9) with the same `Src`,
so an open driver SHOULD use it: rising `Out(2,1,3,0)`, falling `Out(2,0,3,1)`. Example: `L_B`
crossed upward at 0, back below at 400 ns, crossed again at 1000 ns, `L_A` reached at 1300 ns: the
(2,1) encoding of the table measures 1300 ns, the (2,3) encoding 300 ns (the return below `L_B`
cancels the start). The (2,3) end does not consume the start, so a signal that crosses `L_A` again
without first returning across `L_B` fires again with `Δt` from the same start (falling: crossing
`L_B` again without returning above `L_A`).

### 8.4 Video

Engine R: `Src(cA,0, cA,0)`, `Out(0,0,2,0)` (`EV_CFG_R = 0x00200000`). Engine A: only
`Out(2,p,2,p)` with `p` = sync polarity bit; `[15:0]` and TQ_MODE of A need not be written. TRG_KIND `0x0E`. Video registers:

```
VID_SEL (0x21A4) = mode[1:0] | line[10:0] << 2
    mode bit0 = (sync ≠ 0); bit1 = 1 for progressive standards (480p … 1080p)
    line = the line number, clamped to 1 … lines of the standard (table below)
VID_SYNC (0x21A8) = (t1 / 4 ns) | (t2 / 4 ns) << 16
VID_LINES (0x21AC) = a[10:0] | lines[10:0] << 11 | trilevel << 22 | sd_interlaced << 23
```

Sync: any line, line N, odd field, even field. Odd/even field are line triggers on a fixed start
line, interlaced standards only: NTSC odd 4, even 267; PAL/SECAM odd 1, even 313; 1080i/60 and
1080i/50 odd 1, even 563.

| Standard | t1 | t2 | a | lines | trilevel | sd_interlaced | VID_SEL bit 1 |
|---|---|---|---|---|---|---|---|
| NTSC | 4.700 µs | 58.790 µs | 5 | 525 | 0 | 1 | 0 |
| PAL/SECAM | 4.700 µs | 59.300 µs | 2 | 625 | 0 | 1 | 0 |
| 480p/60 | 4.700 µs | 27.077 µs | 8 | 525 | 0 | 0 | 1 |
| 576p/50 | 2.350 µs | 29.650 µs | 2 | 625 | 0 | 0 | 1 |
| 720p/60 | 3.500 µs | 21.680 µs | 2 | 750 | 1 | 0 | 1 |
| 720p/50 | 3.500 µs | 26.130 µs | 2 | 750 | 1 | 0 | 1 |
| 720p/30 | 3.500 µs | 43.906 µs | 2 | 750 | 1 | 0 | 1 |
| 720p/25 | 3.500 µs | 52.795 µs | 2 | 750 | 1 | 0 | 1 |
| 720p/24 | 3.500 µs | 55.017 µs | 2 | 750 | 1 | 0 | 1 |
| 1080p/60 | 1.778 µs | 14.519 µs | 2 | 1125 | 1 | 0 | 1 |
| 1080p/50 | 1.778 µs | 17.482 µs | 2 | 1125 | 1 | 0 | 1 |
| 1080p/30 | 1.778 µs | 29.037 µs | 2 | 1125 | 1 | 0 | 1 |
| 1080p/25 | 1.778 µs | 34.963 µs | 2 | 1125 | 1 | 0 | 1 |
| 1080p/24 | 1.778 µs | 36.444 µs | 2 | 1125 | 1 | 0 | 1 |
| 1080i/60 | 1.778 µs | 29.037 µs | 2 | 1125 | 1 | 0 | 0 |
| 1080i/50 | 1.778 µs | 34.963 µs | 2 | 1125 | 1 | 0 | 0 |

Ticks truncate (`4.7 µs` → 1175, `58.790 µs` → 14697). Exact fs constants for t2: NTSC 58 790 000 000,
PAL 59 300 000 000, 480p 27 077 000 000, 576p 29 650 000 000, 720p 21 680 000 000 / 26 130 000 000 /
43 906 000 000 / 52 795 000 000 / 55 017 000 000, 1080 14 519 000 000 / 17 482 000 000 /
29 037 000 000 / 34 963 000 000 / 36 444 000 000 (rounded to µs·10⁻³ here; the 4 ns tick values above
are exact).

Sync separator behaviour (factory image, progressive standards, negative sync): each frame is
referenced to its **first broad (vertical-interval) sync pulse**; a sync pulse counts as broad when
it lasts longer than about one line period minus 1.5·`t1`. With line sync (`VID_SEL` bit 0 = 1) and
line number `N` the trigger is the leading edge of the sync pulse of the line whose index, counted
from 0 at the first broad pulse, is `(N + 1 − a) mod lines`; i.e. `a − 1` is the standard's number
of the line that carries the first broad pulse (PAL/576p/720p/1080: line 1, NTSC: 4, 480p: 7), and
`N` is the standard's line number. "Any line" (`VID_SEL` bit 0 = 0) triggers on every sync pulse,
broad ones included. The first frame after configuration is skipped until a vertical interval has
been seen (line 1 of that frame does not trigger). Progressive standards (`VID_SEL` bit 1 = 1) count every
sync pulse, half-line ones included, and every vertical interval resets the count.

Interlaced standards (`VID_SEL` bit 1 = 0): sync pulses that arrive within the half-line rejection window
after the previous accepted pulse (equalising and broad pulses at half-line positions) are ignored, so
exactly one pulse per full line is counted. The line counter is referenced to the field-1 vertical
interval (the first broad pulse at a full-line position); the field-2 vertical interval does not reset
it. Line `N` (1 … `lines`) triggers on the leading edge of the sync pulse that starts the standard's
line `N` of the frame, counted continuously through both fields; the counter wraps after `lines`. Odd
and even field are line triggers on the fixed lines above (PAL 1/313, NTSC 4/267): PAL 313 is the
equalising pulse at the start of line 313, NTSC 267 the first full-line broad pulse of field 2. `t2`
sets the rejection window: with the table's values half-line pulses are rejected and full lines
accepted; a window of about `t2 − 2·t1` after the end of the previous accepted pulse fits (exact form
**UNVERIFIED**). The interlaced behaviour of this paragraph is **UNVERIFIED**, for full-length frames
in particular, as is the exact condition that marks the field-1 vertical interval.

**UNVERIFIED**: the purpose of `VID_LINES` bit 23 (no effect on the event), tri-level sync (`VID_LINES` bit 22),
the role of `t1` in interlaced mode, and the polarity bit (engine A `OINV`; no known effect with
negative sync). An open TRG MUST produce this line numbering for the `CFG` values above.

### 8.5 Pattern

Coupling code 0 for CH1–4; `EV_IN1/2` of A per §5.4 for all 21 inputs (combine op AND). If an input
has an edge condition (the first of the 21 inputs with rising/falling), engine A:
`Src(c, e, c, 2)`, `Out(2, r, 0, 0)` with `c` = the §3.1 comparator-A index of that input, `e` =
0/`r` = 1 for rising, `e` = 1/`r` = 0 for falling; without an edge input `Out(0,0,0,0)`.
`T(A,0,0,0)`, TRG_KIND 0, TRG_PATH 1.
(`EV_CFG_A` with a rising edge on CH3: `0x000A8404`; falling: `0x00028444`.)

Behaviour: with an edge input (detector (2,0)) the event is the selected edge while the pattern is
true, the pattern being taken from the slot before the edge (an edge in the slot in which the pattern
becomes true does not fire; one in the slot in which it becomes false does). Without an edge input (detector (0,0) with `OINV` = 0) the event fires when the pattern
becomes **false** (end of the true interval), not when it becomes true.

Recommendation: to trigger when the pattern becomes true, an open driver uses
`Out(0,1,0,1)` (detector (0,0) with `OINV1` = `OINV2` = 1: event when `P` becomes true, `Δt` = time it
was false) with `T(A,0,0,0)`; an open TRG SHOULD fire on the start of the true interval.

### 8.6 Duration

Coupling code 0 for CH1–4, `EV_IN1/2` of A as Pattern, `T(A, when', lower, upper)`, TRG_KIND 0,
TRG_PATH 0. Engine A:

- without an edge input: `Out(0,0,0,0)` (`[15:0]` unchanged);
- with an edge input (first input with rising/falling): `Src(c, falling, c, 2)`,
  `Out(0,0,2, rising)` with `c` = the §3.1 comparator-A index of that input (`falling`, `rising` =
  1 if the edge is so, else 0).

Behaviour: without an edge input (detector (0,0)) the event fires when the pattern becomes false,
qualified by `Δt` = time it was true — the duration trigger. With an edge input (detector
(0,2)) the event is the end of the pattern while comparator `c` is 0 (rising-edge input) or 1
(falling-edge input), qualified by `Δt` from the **latest** change (either direction) of `c` to the end
of the pattern, not by the time the pattern was true; every further end of the pattern fires again
without a new change of `c`.

Recommendation: an open driver SHOULD treat an edge condition in Duration as don't care
(no-edge encoding).

### 8.7 Timeout

`Src(A: cA,0, cB,0)`, `Src(B: cB,0, cA,0)`, `T(A,4,t,0)`, `T(B,4,t,0)` (`t` = timeout):

| slope | engine A | engine B | TRG_KIND |
|---|---|---|---|
| positive | `Out(2,0,2,0)` | (unused) | 0 |
| negative | `Src(cB,0, cA,0)`, `Out(2,1,2,1)` | — | 0 |
| either | `Out(2,0,2,0)` | `Out(2,1,2,1)` | 1 |

Behaviour (detector (2,2) with TQ_MODE 4): positive fires `t` after the signal rose above `L`
(comparator A goes 1) if it has stayed above (comparator A not back to 0, sample > `L − h`);
negative fires `t` after it fell to ≤ `L` (comparator B goes 0) if it has stayed there (comparator
B not back to 1, sample ≤ `L + h`). Exact
timing per §5.2.

### 8.8 Runt

Levels: upper `L_A` (THR_A), lower `L_B` (THR_B). Time: `T(A, when', lower, upper)`, but
`T(A,0,0,0)` when when = none.

| polarity | engine A | engine B | TRG_KIND |
|---|---|---|---|
| positive | `Src(cA,0, cB,0)`, `Out(3,0,6,0)` | — | 0 |
| negative | `Src(cB,1, cA,1)`, `Out(3,1,6,0)` | — | 0 |
| either | `Src(cA,0, cB,0)`, `Out(3,0,1,0)` | `Src(cB,1, cA,1)`, `Out(3,1,6,0)`, same `T` | 1 |

TRG_PATH 0.

Behaviour (detector (3,6) / (3,1), equivalent): positive: window from comparator B going 1 (>
`L_B + h`) to comparator B going 0 (≤ `L_B`), aborted if comparator A went 1 (> `L_A`) in between
(including the start slot); negative: from comparator A going 0 (≤ `L_A − h`) to comparator A going
1 (> `L_A`), aborted if comparator B went 0 (≤ `L_B`) in between. The event is at the window end,
`Δt` = window length, qualified by `T`. Either: engine A detects positive runts, engine B
negative runts, each as above. The engine-B row MUST be the negative row of the table: a row
`Src(B: cA,1, cB,1)`, `Out(B: 3,1,1,0)` does **not** detect negative runts (it fires when the signal
returns above `L_B + h` after a fall below `L_B`, on nearly every full-swing negative pulse). An open
TRG MUST implement the negative-runt detector for that row.

### 8.9 Window / Over

Levels: window top `L_A`, bottom `L_B`, direction per event (§3.2). Base: `Src(A: cA,0, cB,0)`,
`Src(B: cB,0, cA,0)`; slope "falling side" (1): `Src(A: cB,0, cA,0)`. `t` = window time.

| slope \ event | enter (0) | exit (1) | time (2) |
|---|---|---|---|
| 0 | `T(A,0,0,0)`, `Out(A: 2,1,2,1)`, mode 0 | `T(A,0,0,0)`, `Out(A: 2,0,2,0)`, mode 0 | `T(A,4,t,t)`, `Out(A: 2,0,2,0)`, mode 0 |
| 1 | `T(A,0,0,0)`, `Out(A: 2,0,2,0)`, mode 0 | `T(A,0,0,0)`, `Out(A: 2,1,2,1)`, mode 0 | `T(A,4,t,t)`, `Out(A: 2,1,2,1)`, mode 0 |
| 3 (either) | `T(A,0,0,0)`, `T(B,0,0,0)`, `Out(A: 2,1,2,1)`, `Out(B: 2,0,2,0)`, mode 1 | `T(A,0,0,0)`, `T(B,0,0,0)`, `Out(A: 2,0,2,0)`, `Out(B: 2,1,2,1)`, mode 1 | `T(A,4,t,t)`, `T(B,4,t,t)`, `Out(A: 2,0,2,0)`, `Out(B: 2,1,2,1)`, mode 1 |

TRG_PATH 0.

Behaviour (detector (2,2), one level per engine): slope 0 uses only `L_A`: enter = crossing `L_A`
upward, exit = crossing `L_A` downward, time = above `L_A` for `t` (TQ_MODE 4); slope 1 uses only
`L_B`: enter = crossing `L_B` downward, exit = upward, time = below `L_B` for `t`. Either = slope 0 in
engine A OR slope 1 in engine B. So enter/exit refer to the region beyond a level, not to the band
between the levels.

Recommendation: with slope 0 or 1 the other level has no effect, and no setting tests the band
`L_B` < signal ≤ `L_A` (for example "inside the window for `t`"); the factory image's engines have no
band detector, so a driver for the factory image cannot add one. An open TRG that is
to offer a true window trigger needs a band state (comparator B = 1 and comparator A = 0) as an
additional detector input.

### 8.10 Delay

One engine measures from the source-A edge to the source-B edge:

```
CMP1/EDGE1 = edge of source B   (rising: cA(srcB),0 ; falling: cB(srcB),1)
CMP2/EDGE2 = edge of source A   (rising: cA(srcA),0 ; falling: cB(srcA),1)
Out(A: 1,0,1,0);  T(A, when', lower, upper);  TRG_KIND 0
```

Coupling code 0 for both sources; TRG_PATH 1 if either source is digital. (`AB_DLY` `0x2060` is not
needed.) Example A falling on CH1, B rising on CH2: `EV_CFG_A[23:0] = 0x114102`; both rising:
`0x110002`; both falling: `0x114143`.

Behaviour (detector (1,1), first start wins): `Δt` runs from the **first** selected source-A edge
after the previous selected source-B edge to the next selected source-B edge; further A edges in
between are ignored, and a B edge without a pending A edge does nothing. Each B edge ends the
measurement whether or not it qualifies. An A edge in the same slot as a B edge is ignored (the B edge
ends an earlier pending measurement, if there is one).

### 8.11 Setup / hold

Clock source `clk` with clock slope `s` (0 rising, 1 falling), data source `dat` with the data
type `d` (0 or 1). `C` = clock comparator/edge = (`cA(clk)`,0) if `s` = 0 else (`cB(clk)`,1).

| when | engine A | engine B | mode |
|---|---|---|---|
| setup | `CMP1/EDGE1` = `C`; `CMP2/EDGE2` = (`cA(dat)`,0) if `d`=0 else (`cB(dat)`,1); `Out(2, s==0, 3, d==1)`; `T(A,1,0,t_setup)` | — | 0 |
| hold | `CMP1/EDGE1` = (`cA(dat)`,0) if `d`=1 else (`cB(dat)`,1); `CMP2/EDGE2` = `C`; `Out(2, d==1, 6, 0)`; `T(A,1,0,t_hold)` | — | 0 |
| both | setup as above | hold as above in engine B | 1 |

Setup = data edge to clock edge shorter than `t_setup` (violation); hold = clock edge to data edge
shorter than `t_hold`. Behaviour (detectors (2,3) setup, (2,6) hold): setup measures from the
selected data edge (`d` = 0 rising, 1 falling) to the selected clock edge; the opposite data edge
cancels the measurement (the data must keep its new level up to the clock edge), and the clock edge does
not end it, so every selected clock edge while it is pending is tested (each with `Δt` from the same
data edge); a data edge and a clock edge in the same slot: that clock edge is ignored and the data edge
starts a measurement. Hold measures from the **last** selected clock edge (a later one replaces the
pending one) to the selected data edge (`d` = 1 rising, 0 falling), which ends it. The event is at the
clock edge (setup) or data edge (hold) when `Δt <` the bound. In the hold detector, after a start was
replaced, `Δt` reads 1 ns short (factory image; an open TRG SHOULD NOT copy this).

### 8.12 Nth edge

```
rising:  Src(A: cA,0, cB,0), Out(A: 2,1,2,1)
falling: Src(A: cB,1, cA,1), Out(A: 2,0,2,0)
T(A,0,0,0); T(B,0,0,0)
ABR_EN = 1,1,1,0,1  (bits 6,7,8 = 1, bit 9 = 0, bit 10 = 1)
A_COUNT = N                         # trigger on the N-th edge
R_IDLE = idle_fs / 4 000 000          # R_IDLE_LO, R_IDLE_HI; the edge count restarts after an idle time without edges
TRG_KIND = 3
```

(`TRG_SEL = 0x05C3` after programming.)

Behaviour: the TRG counts engine-A events (the selected edges) and triggers on the
`A_COUNT`-th (1 = first; 0 = never); after that, no further trigger until the count restarts. The
count restarts when no engine-A event occurred for more than `R` = `R_IDLE` clocks (4 ns
resolution), by a restart window that repeats while the input is idle. With `t` = clocks since the last
counted edge and `t = k·(R + 7) + j`, `0 ≤ j ≤ R + 6`: an edge with `k = 0`, `j ≤ R` continues the count;
`j = R + 1 … R + 5` is discarded (and the count is cleared); `j = R + 6` is counted as edge 1 and the
count is cleared after it (it does not become the reference for `t`, and the next edge is edge 1 again);
any other edge (`k ≥ 1`, `j ≤ R`) is edge 1 of a new count. So an edge after a long idle time is
discarded when it falls into a later window (with `R` = 20: an edge 23 or 50 clocks after the last
counted edge is discarded, one 37 clocks after it is edge 1). With two or more edges inside one discard
window the result is **UNVERIFIED**, and the phase of the window at the first edge after programming is
not defined. Engines B and R take no part with `ABR_EN = 0x5C0`. An open TRG SHOULD make an edge after
more than `R_IDLE` idle clocks edge 1 (no discard window).

### 8.13 Protocol triggers: common wiring

Each protocol type writes coupling code 0 for its sources, TRG_PATH 1 (SENT leaves it unchanged), and connects
its lines through the engines (comparator A of each source, `Src(e, cA,0, cA,0)`):

| Type | TRG_KIND | R | A | B |
|---|---|---|---|---|
| RS232/UART | 4 | data, `Out(0,0,2,inv)` (`inv` = polarity) | — | — |
| I2C | `0x0D` | — | SCL, `Out(2,0,2,0)` | SDA, `Out(2,0,2,0)` |
| SPI | `0x0C` | MOSI/MISO data, `Out(2,0,2,0)` | SCLK, `Out(2,e,2,e)` (`e` = clock slope) | CS, `Out(2,p,2,p)` (`p` = CS polarity) |
| CAN | 7 | bus, `Out(0,0,2,0)` | — | — |
| FlexRay | 8 | bus, `Out(2,0,2,0)` | — | — |
| LIN | 6 | bus, `Out(2,0,2,0)` | — | — |
| I2S | 5 | SDA, `Out(0,0,2,0)` | SCLK (slope), `Out(0,0,2,0)` | WS, `Out(0,0,2,0)` |
| MIL-1553B | `0x0A` | `Src(R: cA,0, cB,0)` (both thresholds), `Out(2,p,3,p)` (`p` = polarity) | — | — |
| SENT | `0x0B` | bus, `Out(0,0,2,0)` | — | — |

Bit-time dividers are in 4 ns ticks: `div = 1e15 / (baud · 4e6)` = `250e6 / baud` (integer),
sample position `pos = (int)(div · sample_point_percent · 0.01)`.

Line polarity seen by the factory image's decoders with these encodings (comparator state 1 = above the
level): UART with `inv` = 0 takes state 1 as idle/mark; I2C SCL/SDA are not inverted; SPI samples
data on the SCLK rising edge when engine A `OINV` = 0 (falling when 1) and takes CS as active low
when engine B `OINV` = 0 (active high when 1). CAN with
`0x213C[19:18]` = 0 (CAN_H) takes state 1 as dominant. The decoder event uses the same output
pipeline as the analog event engines.

### 8.14 Protocol registers

UART:

| Reg | Fields |
|---|---|
| `0x2100` | `[19:0]` = div[19:0] (bit time, ticks), bit 20 written 0; `[22:21]` event class: 0 start of frame, 1 data, 2 error; `[25:23]` write 0 (other values **UNVERIFIED**); `[27:26]` error kind: 0 parity (check) error, 1 stop-bit (framing) error, 2 any error; `[29:28]` stop bits: 0 = 1, 1 = 2, 2 = 1.5; `[31:30]` parity: 0 none, 1 odd, 2 even |
| `0x221C` | `[9:0]` = div[30:21], bit 10 written 0, `[31:11]` of the shadow kept. div bit 20 is not transmitted (affects baud ≤ 238 Bd, div ≥ 2^20) |
| `0x2104` | `[2:0]` data bits − 1 (5–8 bits); `[10:3]` data min; `[18:11]` data max (write both = the data value) |

Conditions: Start → `[22:21]=0,[27:26]=0`; Error → 2, 2; Check error → 2, 0; Data → 1, 0.
Example 115 200 Bd, 8 N 1, data 0x41: `0x2100 = 0x0000087A` (Data: `0x0020087A`),
`0x2104 = 0x00020A0F`.

UART decoder behaviour (factory image): bit `k` of a frame (start bit = 0) is read at its centre,
start edge + (`k` + ½)·div clocks. Start fires at the start-bit centre. Data fires at the centre of
the last data bit when the byte equals `0x2104[10:3]`; `[18:11]` has no effect (equality only;
"min = max = value" is the only usable form). Error (`[22:21]` = 2) fires at the centre of
the stop or parity bit: `[27:26]` = 2 on a frame error (stop bit read 0) **or** a parity error, 0 on a
parity error only, 1 on a frame error only. Parity `[31:30]`: 1 odd, 2 even.

I2C:

| Reg | Fields |
|---|---|
| `0x218C` | `[2:0]` when (0 start, 1 stop, 2 restart, 3 missing ack, 4 address, 5 data, 6 address & data); `[5:3]` address-type field; `[8:6]` write/read type field; `[10:9]` direction (0 write, 1 read, 2 either); `[11]` = address width bit 0 (0 7-bit, 1 10-bit); `[14:12]` data bytes − 1. In 8-bit address mode `[11]` = 0 and `[10:9]` = bit 0 of the 8-bit address min (the R/W bit) instead of the direction |
| `0x2190` | `[9:0]` address min, `[19:10]` address max, `[29:20]` address mask (default mask `0x3FF`); 8-bit address mode writes the values `>> 1` |
| `0x2194` / `0x2198` / `0x219C` | data min / max / mask, bytes 0–3 (byte 0 in `[7:0]`) |
| `0x21A0` | bit 3 = bit 35 of the data min (byte 4); bit 11 = bit 35 of max/mask (a byte-4 compare is **UNVERIFIED**) |

I2C decoder behaviour (factory image, 7-bit addresses): START, STOP and repeated START are SDA edges
while SCL is high; bits are sampled at SCL rising edges, MSB first. Address (when 4) fires at the SCL
rising edge of the R/W bit (before the ACK, also when the address is NACKed) when `(a & mask) ==
(min & mask)` for the 7-bit address `a` and the direction matches `[10:9]` (0 write, 1 read, 2
either); the address max is not used. Data (when 5) compares the first `[14:12]` + 1 data bytes after
the address byte, **first byte most significant**, as `(d & mask) == (min & mask)` (max not used) and
fires at the SCL rising edge of the last compared data bit; later bytes are ignored. Address & data
(when 6) fires at that data byte when both match. Missing ACK (when 3) fires at the SCL rising edge
of an ACK bit read high. **UNVERIFIED**: 10-bit and 8-bit address modes. A driver MUST put the first
transmitted byte in the most significant compared byte.

SPI:

| Reg | Fields |
|---|---|
| `0x2178` | `[0]` when (0 CS framed, 1 timeout framed); `[3:1]` data-compare mode (0 = equal); `[8:4]` data bits − 1 |
| `0x217C` | timeout in 4 ns ticks (when = timeout), else 0 |
| `0x2180` / `0x2184` / `0x2188` | data min / data max / mask: bit `i` = data bit `i` (1 = '1'; mask bit = "care") |

SPI decoder behaviour (factory image): a word is the first `[8:4]` + 1 bits after CS becomes active
(when 0) or after SCLK has been idle for more than `0x217C` clocks (when 1), shifted in MSB first on
the selected SCLK edge; only the first word of a frame is compared. The event is at the last
sampled SCLK edge. Compare mode `[3:1]` on `d & mask` against min `m` and max `M` (also masked): 0
`d = m`, 1 `d ≠ m`, 2 `d > m`, 3 `d ≤ m`, 4 `m ≤ d ≤ M`, 5 `d < m` or `d > M`, 6 and 7 never.

CAN:

| Reg | Fields |
|---|---|
| `0x213C` | `[1:0]`, `[4:2]`, `[6:5]`, `[9:7]`, `[12:10]`, `[16:14]` six frame-match fields (when / field / frame / error settings); `[13]` ID-mask enable; `[17]` data-mask enable; `[19:18]` signal type (0 CAN_H, 1 CAN_L, 2 TX/RX, 3 differential); `[20]` 1 = extended ID; `[24:21]` data byte count |
| `0x2140` | `[15:0]` div, `[31:16]` sample position (ticks) |
| `0x2144` / `0x2148` / `0x214C` | ID min / max / mask |
| `0x2150`+`0x2154`, `0x2158`+`0x215C` | data min, data max (64-bit, lo/hi, byte 0 LSB) |
| `0x2160` / `0x2164` | data mask lo/hi |

`0x213C` working values per condition: start of frame `0x00200000`, end of frame `0x00202018`,
remote frame `0x00202039`, error `0x00200079`, ID `0x00202019`, ID and data `0x00220019`; 1 Mb/s 50 %: `0x2140 = 0x007D00FA`; 500 kb/s 75 %:
`0x017701F4`.

CAN decoder behaviour (factory image, standard data frames, signal type 0): each bit is sampled
`0x2140[31:16]` clocks after its start, resynchronised on edges; a frame is recognised only after a
bus-idle period. `0x00200000` (SOF) fires at the SOF sample point; `0x00202018` (end of frame) at the
last EOF bit; `0x00202019` (ID) at the IDE bit's sample point when the 11-bit ID equals ID min (under
the ID mask); `0x00220019` (ID and data) at the last bit of the first data byte when ID and data
match. `0x00202039` (remote frame) fires at the IDE sample point (SOF + 13.5 bits) of a remote frame
(RTR recessive) whose ID matches (`[6:5]` = 1 selects remote frames, 0 data frames); data frames with
the same ID do not fire. SOF also fires on remote frames and on frames aborted by an error flag; end of
frame fires on remote frames but not on aborted frames. **UNVERIFIED**: the
condition of `0x00200079` ("error"; it does not fire on stuff, CRC, form or ACK errors), the meaning
of the six match fields beyond these values, extended IDs, multi-byte data compares (same byte-order
question as I2C), and whether signal types 1–3 invert the line in the decoder (if not, a recessive-
high TX/RX line needs engine R `OINV2` = 1).

LIN:

| Reg | Fields |
|---|---|
| `0x2118` | `[2:0]` when (0 sync, 1 ID, 2 data, 3 data & ID, 4 sleep frame…); `[5:3]`, `[8:6]`, `[10:9]` further fields (error type in `[10:9]`); `[12:11]` LIN version (0 1.x, 1 2.x, 2 any); `[18:13]` ID min; `[24:19]` ID max (ID only, the parity bits do not fit); `[28:25]` number of data bytes compared |
| `0x211C` / `0x2120` | div / sample position |
| `0x2124`+`0x2128`, `0x212C`+`0x2130` | data min / max (64-bit) |
| `0x2134` / `0x2138` | data mask |

Sleep frame (when 4): additionally `0x2118` low part `= 3`, ID field `0x3C`, `[28:25]` = 8, data min
`0x00FFFFFFFFFFFFFF`, mask all ones. Defaults: 9600 Bd → div `0x65B9`, pos `0x32DC`.

LIN decoder behaviour (factory image, version 2 = any): Sync (when 0) fires at the centre of the sync
byte's stop bit; ID (when 1) at the centre of the PID stop bit when the 6-bit ID equals `[18:13]`
(`[24:19]` has no effect); Data (when 2) compares the first `[28:25]` data bytes, first byte most
significant, as `(d & mask) == (min & mask)` (max not used) and fires at the stop-bit centre of the
last compared byte, without checking the ID; Data & ID (when 3) requires both. **UNVERIFIED**:
versions 0 (1.x) and 1 (2.x) — no event is known to fire with them (checksum-type dependence not
established), and the data byte order (as I2C).

FlexRay: `0x2174[20:0] = 99 | 49 << 7 | 24 << 14` (bit times − 1 for 2.5/5/10 Mb/s, upper bits
kept); `0x2168` 13 packed match fields (`[1:0]`,`[3:2]`,`[5:4]`,`[7:6]`,`[9:8]`,`[12:10]`,
`[13]`,`[16:14]`,`[17]`,`[19:18]`,`[20]`,`[24:21]`,`[31:25]`); `0x216C` = ID min `[10:0]`, ID max
`[21:11]`; `0x2170[10:0]` ID mask; cycle min `[5:0]`, cycle max `[11:6]` and cycle mask `[17:12]`,
also at `0x2174`. The bit-time fields and the cycle fields overlap in `0x2174`; only one layout can
be in the register at a time, and which one the decoder uses is **UNVERIFIED** (spec 02 §5.4).

I2S: `0x2108 = a[2:0] | b[1:0]<<3 | c[1:0]<<5 | d[5:0]<<7 | e[5:0]<<13` (when, alignment,
channel, used width, word width); `0x210C`/`0x2110`/`0x2114` = data min/max/mask.

MIL-STD-1553B: `0x21B0 = a[2:0] | b[1:0]<<3 | c[2:0]<<5 | d[2:0]<<8 | e[2:0]<<11 | f[2:0]<<14
| g[1:0]<<17 | h[7:0]<<19`; `0x21B4`/`0x21B8`/`0x21BC` = compare words (bits `i` and `i+11` from the
compare bit patterns).

SENT: `0x21D0 = a | b[2:0]<<3 | c[3:0]<<6 | d<<10 | e<<11 | f[19:0]<<12` (f = tick in
ticks); `0x21D4…0x21E0` = sync-pulse window bounds `(1 ± tol)·tick·56` for the fast and slow
tolerance; `0x21E4 = a | b[3:0]<<4 | c[23:0]<<8`; `0x21E8 = a | b<<24 | c<<28`; `0x21EC` and `0x21F0` have
no known fields (write 0; the factory image stores nothing at these offsets, spec 21 §1.2).

Field meanings of I2S, 1553B, FlexRay and SENT beyond the formulas given are **UNVERIFIED**. In the
factory image writes to the FlexRay, MIL-1553B and SENT registers and to the write side of `0x221C`
have no known stored effect (spec 21 §1.2; **UNVERIFIED** whether these registers exist).

---

## 9. Engine behaviour (for an FPGA implementation)

Behaviour of the factory image for the field values of §8 (spec 21 §2, §4).
Field combinations that §8 does not use are not needed; an open TRG SHOULD never fire for them.

1. Each comparator is a Schmitt trigger on the (optionally filtered, §4) channel stream, evaluated
   per 1 ns slot: output 1 when `sample > U`, 0 when `sample ≤ Lo`, else hold (§3.2). EXT and AC
   line are 1-bit inputs used as the comparator state (§3.1, §3.4).
2. **Edges.** For each engine input (`CMP1`/`EDGE1`, `CMP2`/`EDGE2`) an edge is detected per slot
   from the slot's state and the previous slot's: `EDGE` 0 = rising (0→1), 1 = falling (1→0), 2 =
   either, 3 = never.
3. **Detectors.** `OSEL1` and `OSEL2` are decoded as a pair; the pair selects one detector. A
   detector has a *start* instant and an *end* instant; `Δt` = number of 1 ns slots from start to end.
   At the end the event is issued if `TQ_MODE` 0–3 holds (§5.2; `TQ_MODE` 5–7 never fire); with
   `TQ_MODE` 4 the event is issued `T_MIN` after the start if the end has not occurred (rule in §5.2)
   and the end issues nothing. An end consumes the pending start, whether or not it qualifies, except in
   the detectors marked "not consumed"; an end without a pending start does nothing. "First start wins":
   further starts while one is pending are ignored; "latest wins": a further start replaces the pending
   one. `c1`, `c2` = states of `CMP1`, `CMP2`; `P` = pattern (item 4).

   | (OSEL1, OSEL2) | used by | start | end (= event) | further rules |
   |---|---|---|---|---|
   | (2,2) | Edge, Timeout, Window, Nth, protocol lines | `x` goes 0 | `x` goes 1 | `x = (OINV1 ? c1 : ¬c1) AND (OINV2 ? c1 : ¬c1)`; `CMP2` and both `EDGE` fields are ignored; `OINV1 ≠ OINV2` never fires; `Δt` = time `x` was 0 |
   | (2,1) | Pulse, Slope | `CMP2` edge per `EDGE2`, first start wins | `c1` goes to the level `OINV1` (1 = rises, 0 = falls) | the start is cleared by every `CMP1` edge selected by `EDGE1`, whichever its direction (with the §8 values that is the end edge), not by the end as such; a `CMP2` edge in the other direction does not clear it |
   | (1,1) | Delay | `CMP2` edge per `EDGE2`, first start wins | `CMP1` edge per `EDGE1` | `OINV1 = OINV2 = 1` swaps the roles (start `CMP1`/`EDGE1`, end `CMP2`/`EDGE2`) and inverts the pending flag (after configuration a start counts as pending; not used in §8); `OINV1 ≠ OINV2` never fires |
   | (2,3) | Setup | `CMP2` edge per `EDGE2`, latest wins | `CMP1` edge per `EDGE1`, not consumed | the opposite `CMP2` edge (`EDGE2` 0: falling, 1: rising) cancels the start |
   | (2,6) | Hold | `CMP2` edge per `EDGE2`, latest wins | `CMP1` edge per `EDGE1` | after a replacement `Δt` is 1 ns short |
   | (3,6), (3,1) | Runt | `CMP2` edge per `EDGE2` | `CMP2` edge in the opposite direction | a `CMP1` edge per `EDGE1` after the start (including the start slot) and before the end aborts (start dropped, no event); `OSEL2` 1 and 6 are equivalent; `OINV1` as §8.8 writes it (0 positive, 1 negative; other values not characterised) |
   | (0,0) | Pattern and Duration without edge | `y` goes 0 | `y` goes 1 | `y = OINV ? P : ¬P` (both `OINV` equal, as for (2,2)); with `OINV` = 0 the event is `P` becoming false and `Δt` = time `P` was true |
   | (2,0) | Pattern with edge | none needed | `CMP1` edge per `EDGE1` while `P` was true in the **previous** slot, not consumed | `Δt` = time `P` has been true (`P` true since configuration counts); `TQ_MODE` 4 = timeout on `P` true |
   | (0,2) | Duration with edge | every `CMP2` edge per `EDGE2` restarts `Δt` (latest wins); pending while `c1` is at the level an `EDGE1` edge leaves (`EDGE1` 0: `c1` = 0, 1: `c1` = 1) | `P` becomes false while pending, not consumed | evaluated with the state before the slot's own edges; `OINV2` has no effect; `EDGE1` 2/3 not characterised |

   A start and an end in the same slot (only possible when the input crosses two thresholds within one
   slot, or with two sources): (1,1) ignores that start (the end uses an earlier pending start, if any);
   (2,3) ignores that end (the start is kept). Whether (2,1) and (2,6) ignore the start in the same way
   is **UNVERIFIED** (an open TRG SHOULD ignore it). Pair (1,6) is decoded but unused; `OSEL2` = 7 holds
   the engine's second output in reset (spec 21 §4); other pairs are not characterised.
4. **Pattern.** `P` is the AND of all `EV_IN` terms of the engine (§5.4). It acts only in the
   detectors (0,0), (2,0) and (0,2); all other detectors ignore `EV_IN`.
5. **Modes.** `TRG_KIND` 0: trigger = engine A event; 1: engine A event OR engine B event (two
   independent engines); 3: Nth edge, counting engine-A events (§8.12); protocol and video modes:
   the decoder's event (§8.4, §8.14). In protocol and video modes the decoders take their lines from
   the engines' comparators (`CMP1` = `CMP2` in every §8.13 encoding); the `Out` fields act only as
   the polarity and clock-edge bits listed in §8.13, not as the detectors above.
6. A trigger is accepted only when the SEQ is armed and holdoff has elapsed (§7); the event reaches
   the SEQ a fixed pipeline delay after the end slot (spec 21 §4); the trigger time stamp combines the
   CLK250 cycle and the TRG slot of the event (slot encoding **UNVERIFIED**, spec 21 §5.4); the CPU never
   reads a sub-sample trigger position.

An open FPGA driven by open firmware MAY use its own internal encoding (and SHOULD follow the
recommendations of §8), but a TRG that is to work with a driver written to §8 MUST decode these
registers as above.

---

## 10. Sweep, force, trigger out and position

| Sweep | SEQ programming (spec 02 §6.4) |
|---|---|
| Auto | `SEQ_FUNC.AUTO = 1` |
| Normal | `SEQ_FUNC.AUTO = 0` |
| Single | Normal + single-shot run type (spec 03 §3.4); selecting Single needs no reconfiguration by itself |

Auto-trigger timeouts, written on every configuration (4 ns ticks, `tdiv` = main timebase fs/div):

```
m        = min(20 · tdiv, 500 ms)
AUTO_T1 (0x4060) = max(m, 10 ms)
AUTO_T2 (0x4064) = max(m / 2, 100 ms, holdoff)
```

(1 µs/div → `0x002625A0` / `0x017D7840`; 1 ms/div → `0x004C4B40`; 10 ms/div → `0x02FAF080`.)
`AUTO_T1/2` are written by the SEQ configuration step (timebase, sweep, channel changes). A
driver SHOULD also recompute `AUTO_T2` when the holdoff changes, so that it is never shorter than
the holdoff. Semantics of the two timeouts in the factory image (spec 21 §7.1, CLK250 cycles):
`AUTO_T1` is a "no TRG event for a while" timer (granularity 2^15, bits `[14:0]` ignored). It runs
from FPGA configuration (`SEQ_FUNC[0]` configuration value 1) without restart across frames, arms,
auto-triggered frames and stop/run cycles; it is restarted by **every** TRG event, accepted or not
(also one before `RUN`, during the pre-trigger fill or inside the holdoff), held while a FORCE is
pending and from an event or forced accept until that frame ends. Once its flag is set (it stays set
across auto-triggered frames), every frame auto-triggers `AUTO_T2[27:16] · 65536 + R + 1` armed
cycles after its trigger enable (`R` = 16-bit pseudo-random dither, new per frame; `AUTO_T2[15:0]`
and `[31:28]` ignored). So with no signal the first auto trigger comes ≈ `T1 + T2` after the last TRG
event (or after `SEQ_FUNC[0]` became 1) and then one per `T2` of armed time. No timer state can be
read; the restart rule and the dither show only in the trigger timing. Because bits
`[31:28]` are ignored, an `AUTO_T2` ≥ 2^28 ticks (≈ 1.07 s, i.e. a holdoff > 1.07 s) wraps in the
factory image (10 s holdoff `0x9502F900` → ≈ 0.34 s); an open image MUST implement the two-level rule
and MAY use all 32 bits and omit the granularity and the dither. A driver that runs against the
factory image SHOULD write `min(AUTO_T2, 0x0FFFFFFF)`, so that a holdoff above 1.07 s gives the
longest auto period the image supports (≈ 1.07 s plus the dither) instead of a wrapped one; the auto
period is then shorter than such a holdoff, because the auto trigger is accepted outside the
holdoff gate (spec 21 §7).

**Force**: pulse `TRG_SEL.FORCE` (1, then 0) — one trigger now, no reconfiguration (rising-edge
semantics, §6). A forced frame sets `SEQ_STAT` bits 6, 7, 10 and 11 like an event trigger, so it
reads as triggered; an auto trigger sets bits 7 and 10 only (spec 02 §6.5).

**Trigger out**: write `AUXOUT_SEL[2:0]` (`0x1430`) = 0 for trigger out and 3 for mask pass/fail.
Factory image decode (spec 21 §5.1):

| `AUXOUT_SEL[2:0]` | Rear AUX OUT |
|---|---|
| 0, 4, 5, 6, 7 | trigger pulse |
| 1 | SEQ trigger-accept flag |
| 2 | constant low |
| 3 | mask pass/fail |

Trigger pulse: high for 15 CLK250 cycles = **60 ns**; the pin goes high 4 CLK250 cycles (16 ns) after the
SEQ trigger accept, i.e. 5 cycles (20 ns) after the TRG event output and 27 cycles (108 ns) after the
cycle in which the comparator input registers hold the completing sample (spec 21 §4). Generated only
while the acquisition FSM is running (active-high at the FPGA pin; polarity at the BNC after the
board buffer is **UNVERIFIED**).

**Position**: the trigger is placed by `PRE_LEN`/`POST_LEN` (spec 02 §6.6, spec 04 §6). Nominally
the trigger is at window index `PRE_LEN`; in the factory image the first trigger-channel sample at
or after the crossing is at window index `PRE_LEN + 7` (1-, 2- and 3–4-channel modes, word alignment
off; the offset with the ADC configuration of spec 06 is **UNVERIFIED**, spec 02 §6.6). In scan
mode (slow timebase) `SCAN_TPOS` (`0x4058`) `= (n << 31) | ((offset − 5·tdiv) / t_pt)`, with
`t_pt` the scan point interval and `n` = 1 when the pre-trigger sample count is < 1. Sub-sample placement on
screen uses only the configured offset (fractional start sample → `SKEW_FINE`, spec 03 §4 step 5); there is
no per-trigger fine time stamp register.

---

## 11. Write sequence summary

1. Select the type: holdoff (§7), then the type's engines, levels (all four channels), coupling, logic,
   protocol registers, `TRG_KIND` last (except Nth: ABR enables, counts, then mode).
2. Any level, slope, polarity, source, width or noise-reject change re-runs that type's programming
   (the whole table row of §8), not just the changed field.
3. Re-run the acquisition configuration (spec 03) after type, source, coupling, holdoff or sweep
   changes; level-only and time-only changes are written while stopped by the same re-arm cycle.
