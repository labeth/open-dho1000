# 21 — FPGA Processing Logic: Behaviour Behind the Registers

What the FPGA logic does with the registers of spec 02, for the FPGA image the unit loads at boot
(`SPU_H12S2.bit`, and `SPU_H12S2B.bit`, which differs only in the device ID; spec 12 §1.2, spec 13 §2):
register-file decoding, storage and read-back, trigger comparators, coupling filter, trigger output and
force, the bandwidth-limit FIR, the record sequence (holdoff, re-arm, auto trigger, status and
frame-progress registers), the record-memory layout, the play-back read addressing and the C2H output
stage. Driver encodings stay in specs 02–05; this spec states what the logic does with them. It is the
behavioural specification an open FPGA design is written from.

Conformance levels: a **drop-in image** (one that replaces the boot-time image under software written
against specs 02–05) MUST behave as described here; an open image driven by open firmware MAY differ but
SHOULD keep the same numeric conventions. Nothing in this spec has been measured on a unit: all of it is
**UNVERIFIED** until it is. Items marked **UNVERIFIED** inline are open even as description.

Clock names: **CLK125** = register clock (PCIe user clock, 125 MHz); **CLK250** = acquisition clock
(4 ns, from the ADC data clock, spec 20).

---

## 1. Register file

### 1.1 Address decoding

| Block | Selected by | Offset bits decoded |
|---|---|---|
| SP `0x1000`–`0x1FFF` | A12 = 1, A13 = A14 = 0 | full |
| TRG `0x2000`–`0x2FFF` | A13 = 1, A12 = A14 = 0 | A2–A9 only: the block repeats every `0x400` (`0x2400`, `0x2800`, `0x2C00` alias `0x2000`) |
| SEQ `0x4000`–`0x4FFF` | A14 = 1, A12 = A13 = 0 | partial: only the bits needed to tell the implemented registers apart (aliases exist; MUST NOT be relied on) |

Writes are taken in CLK125; every stored bit holds the last value written. The 250 MHz logic uses the
configuration registers without synchronisation, so software MUST change them only while acquisition is
stopped. **TRG configuration registers read back the last written value.**

### 1.2 Stored bits

Bits not listed are not stored (written values are ignored). Compatible images MAY store more.

| Register | Stored bits | Consequence |
|---|---|---|
| THR_A/B (`0x2004`…`0x2010`, `0x20B8`…`0x20C4`) | `[15:4]`, `[31:20]` | thresholds are 12-bit (record code bits 15:4); bits 3:0 / 19:16 ignored |
| EV_IN1 A/B/R | `[31:0]` | |
| EV_IN2 A/B/R | `[9:0]` | the combine operator `[11:10]` has no effect: inputs are always ANDed |
| EV_CFG A, B, R | `[26:0]` | |
| EV_TMIN / EV_TMAX / EV_THI A/B/R | `[31:0]` each | time bounds are 48-bit |
| AB_DLY_LO/HI, R_IDLE_LO/HI | lo `[31:0]`, hi `[15:0]` | 48-bit |
| A_COUNT, B_COUNT, HO_COUNT | `[31:0]` | |
| HO_TIME_LO / HO_TIME_HI | `[31:0]` / `[15:0]` | 48-bit holdoff; the type field `[18:16]` is ignored (time holdoff only) |
| TRG_SEL | `[13:0]` | |
| TRG_COUPLING | `[17:0]` | the coefficient `[17:4]` is read back only; it does not affect the filter (§3) |
| RUNCTL | `0`, `1`, `30`, `31` | bit 2 is a request, not a stored bit: a write with bit 2 = 1 raises a request to the acquisition state machine, which takes it over and clears it (the state-machine reset of a stop also clears it); a write with bit 2 = 0 has no effect, and bit 2 is not read back. What the state machine does with the request (spec 02 REPLAY) is **UNVERIFIED** |
| SEQ_MODE | `[3:0]`, `16`, `17`, `22`, `23`, `25` | logic-analyzer bits 24 and 26 are not implemented in this image |
| SEQ_CMD | `16`, `17`, `18`, `24`, `25`, `27`–`31` | bit 26 (option flag) is not stored |
| FR_A / FR_B / FR_C | `[19:0]` | frame indices are 20-bit |
| SEQ_FUNC | `0`, `1`, `8`–`10`, `12`–`15` | |
| FR_GAP, PRE_LEN, POST_LEN, AUTO_T1, AUTO_T2 | `[31:0]` | |

Registers with storage but no known use: TRG `0x2090` (`[11:0]`, `[23:16]`), `0x2098` (`[4:0]`),
`0x209C` (`[5:0]`, `[13:8]`), `0x20A4` (`[7:0]`). Counter/DVM block (spec 02 §5.7): `0x2210` FC_CFG
stores `[14:0]`, with comparator-state source selects for the frequency counter in `[6:5]` and `[11:10]`;
`0x2214` FC_GATE stores `[31:0]` (gate time in 4 ns ticks); `0x2244` DVM_CTRL stores `[5:0]`, `[9:8]`.
No storage is known for `0x2264` bit 16, `0x221C`, the FlexRay (`0x2168`–`0x2174`), MIL-1553B
(`0x21B0`–`0x21BC`) and SENT (`0x21D0`–`0x21F0`) registers (**UNVERIFIED**: absent from this image, or
stored in a form that does not read back).

Configuration-time values: the registers have no reset; they take their power-on values only at FPGA
configuration (values for registers the software never writes: spec 02 §5.2). Neither the stop sequence
nor `SP_RESET` bit 0 clears the register file (e.g. `FR_C` keeps its value).

---

### 1.3 Read path

A bypass-BAR read writes no register. **Bits that the addressed register does not drive read as 0.** SEQ
reads have no side effects (no clear-on-read). Unimplemented SEQ addresses return an OR of partially
decoded registers and MUST NOT be relied on.

SEQ read-back:

| Register | Read value |
|---|---|
| `PRE_LEN`, `POST_LEN`, `AUTO_T1`, `AUTO_T2`, `FR_GAP`, `SEQ_TIMEOUT`, `0x4058`, `0x4090` | as written (32 bits) |
| `FR_B` (`0x4014`), `FR_C` (`0x4018`) | as written, `[19:0]`; `[31:20]` = 0 |
| `FR_A` (`0x4010`) | the frame-progress register of §7.2 (not the written value once a frame has completed) |
| `RUNCTL` (`0x4004`) | bits 0, 1, 30, 31 as written; the other bits are internal status and MUST be masked |
| `SEQ_STAT` (`0x4034`) | §7.2 |
| `FR_READY` (`0x4080`) | §7.2 |

Writes to `0x404C` (`PRE_ADJ`) are not stored in a register of their own: the partial decode lands them
in the stores of `0x4054` (`POST_LEN`), `0x406C` (`SEQ_TIMEOUT`) and the 32-bit store read back at
`0x4058`. Software MUST therefore write `POST_LEN` and `SEQ_TIMEOUT` after `PRE_ADJ`; only the `0x4058`
store then keeps the value (§8.3).

## 2. Trigger comparators

### 2.1 Sample stream

The TRG sees **4 samples per channel per CLK250 cycle** (slots 0–3, slot 0 earliest). Each sample is the
12-bit unsigned (offset-binary) value of record-code bits `[15:4]`. Which of the 8 aligned ADC samples
`t = 0…7` of the clock (time order of spec 20 / §8.2: g0.Q4, g1.Q4, g0.Q3, …, g1.Q1) fill the slots
depends on `SEQ_MODE[3:0]`:

| Active channels | Slots 0–3 | Path latency (ISERDES Q → comparator sample) |
|---|---|---|
| 1 | max/min of each sample pair, selected per 2 ns half (below) | 22 CLK250 cycles |
| 2 | lower active channel `t` = 0, 2, 4, 6 (group 0); higher `t` = 1, 3, 5, 7 (group 1) | 119 cycles |
| 3–4 | CH1 `t` = 0,0,4,4; CH2 2,2,6,6; CH3 1,1,5,5; CH4 3,3,7,7 (each sample fills two slots) | 89 cycles |

The driver programs all trigger times in 1 ns ticks (spec 02 §1.4), i.e. one slot = 1 ns, and widths
and times are counted in slots in every mode. In 1-channel mode slot `s` carries one of the two samples
`t = 2s, 2s+1` (0.5 ns apart). The selection is made per 2 ns half `h` = 0, 1 from that clock's samples only, with
`a = (x[4h], x[4h+1])` and `b = (x[4h+2], x[4h+3])`:
```
if max(b) > max(a):  slot[2h] = min(a),  slot[2h+1] = max(b)
else:                slot[2h] = max(a),  slot[2h+1] = min(b)
```
The largest sample of each 2 ns half therefore reaches the comparators in the 1 ns slot where it
occurs, and the other slot carries the minimum of its own pair. For a signal that is monotonic over the
half the slots are `t` = 0, 3, 4, 7. A single step lands in its 1 ns slot within ±0.5 ns. A one-sample
positive peak is always kept; a one-sample dip in `a` is lost when `max(b) ≥ max(a)`. In 2-channel mode
every sample of the channel is used, and in 3–4-channel mode each 2 ns sample is used twice.
Inactive channels carry other lanes (don't care). The 2- and 3–4-channel samples pass a FIFO before the
TRG; the 1-channel path does not. `CH_SKEW` (`0x1080`/`0x1084`) does not act on the TRG path. The
multi-channel latencies depend on the FIFO start-up (**UNVERIFIED**). The table is stated in aligned-word
positions (word alignment `0x1020` off); with the alignment rotation enabled the same positions apply to
the aligned words. An open image MAY use all 8 samples (1 channel) as long as it keeps 1 ns slot units,
and SHOULD keep every positive and negative excursion of 0.5 ns.
With valid ADC data the comparators take a new sample set every clock in every channel mode.

Tap point: the comparator samples are taken from the capture stream **after** the coarse digital gain
(`LANE_GAIN` `0x105C` with the per-lane fine gain `0x1058`, applied right after deserialisation, spec 20
§3.3) and **before** the per-output-slot display gain/offset (`PB_GAIN`/`PB_OFS` `0x14C8`…, §9.1, on
the play-back path behind the record memory). The comparator sample and the record code are the same
value for every `LANE_GAIN` code, so the trigger compares the codes that are recorded.
The `LANE_GAIN` stage works per lane. `LANE_GAIN` field `k` and `LANE_TRIM` field `k` act on sample
`t = 2k` (group 0, `k` = 0–3) or `t = 2(k − 4) + 1` (group 1, `k` = 4–7), word alignment off. The input is the
full 14-bit lane word `w` (offset binary, bits [1:0] included). Let `d = w − 8192` and let `F` = 0, 1, 2,
4 for fine code 0–3. The 12-bit output `y` (record code and comparator sample) is:
```
code 0:     y = w >> 2                                                       (bypass, fine code ignored)
code 1:     y = clamp(2048 + floor((2·d + floor(d/2) + floor(F·d/256)) / 4), 0, 4095)
code 2, 3:  y = clamp(2048 + floor((5·d + floor(F·d/256)) / 4), 0, 4095)
```
With `x = w >> 2` this is `y ≈ 2048 + 2.5·(x − 2048)` (code 1) and `2048 + 5·(x − 2048)` (codes 2 and
3). The fine code adds `F/256` to the factor (+0.16 / +0.31 / +0.63 % at code 1), and the output
saturates at 0 and 4095. The latency is the same for every code. `LANE_TRIM` fields 8–15 have no
effect. Code 1 is the unity setting ("×1") and code 2 is "×2": the ratio is exactly 2 at fine code 0,
and the ×2.5 of code 1 relative to the raw ADC code is contained in the calibration gains, which are
measured with code 1 written (spec 10). A drop-in image MUST reproduce codes 1 and 2 as given (factor,
mid-scale centre, floor rounding, saturation).
Consequence: when a factor 2 is moved into `LANE_GAIN` (G ≥ 2, spec 03 §2), the recorded and compared
codes have twice the codes per division of code 1, so a trigger code computed with the full G (spec 05
§3.3) acts at half the intended voltage offset from centre. A drop-in image MUST keep this tap point;
open firmware SHOULD compute trigger codes with the residual gain (G/2 when `LANE_GAIN` = code 2).

### 2.2 Threshold compare and Schmitt state

For every channel and each of its two comparators (A uses THR_A, B uses THR_B) and every slot:

```
above_upper[s] = sample[s] > U        # U  = THR[15:4],  strict >
above_lower[s] = sample[s] > Lo       # Lo = THR[31:20], strict >
state[s]       = above_upper[s] | (state[s-1] & above_lower[s])     # state[-1] = state[3] of the previous cycle
```

So the comparator output becomes 1 when a sample is **strictly above U**, becomes 0 when a sample is **at or
below Lo**, and holds otherwise. The four slot states are evaluated in time order within the cycle (a
transition can occur at any 1 ns slot). The state is 1 = "above".

### 2.3 Pattern (EV_IN) codes

Per input, two bits `c[1:0]`:

| `c` | Input term |
|---|---|
| `00` | 1 (don't care) |
| `01` | 0 (never true) |
| `10` | true while the comparator state is 1 |
| `11` | true while the comparator state is 0 |

The input terms are ANDed (§1.2). With the encoding of spec 05 §5.4 (H → `10`, L → `11`) this makes
H = input above its comparator-A threshold (state 1) and L = below (state 0). A drop-in image MUST
reproduce the table above together with §2.2.

### 2.4 Coupling path into the comparators

Each comparator input slot is either the raw sample or the coupling-filter output (§3): the filter output
is used **only for the channel selected by `TRG_COUPLING[3:2]`** (0 = CH1 … 3 = CH4) and only when
`TRG_COUPLING[1:0]` ≠ 0. All other channels always use raw samples. There is one coupling filter in the
TRG; the last `TRG_COUPLING` write wins. `TRG_SEL.TRG_PATH` does not take part in this selection (§5.3).

---

## 3. Coupling filter

One filter, input = slot 0 of the selected channel every CLK250 cycle (250 MSa/s, 12-bit unsigned).
Coefficients are fixed; `TRG_COUPLING[17:4]` is ignored.

### 3.1 Low-pass core (HF reject, code 1)

```
y[n] = 158·x[n] + 32618·floor(y[n−2] / 2^15)        # two-cycle recursion, 48-bit accumulator
lpf[n] = y[n][26:15]                                 # 12-bit output
```

Pole `a = 32618/32768` per 8 ns ⇒ **−3 dB at 91.3 kHz**, DC gain `158/150 = 1.053`. The two-cycle loop is
two interleaved first-order sections at 125 MSa/s, so the response repeats every 125 MHz (`H(z)` in `z⁻²`)
and the two phases settle independently. The loop feeds back only `⌊y/2^15⌋` truncated to 13 bits
(mod 2^13), so it has a dead band: every output `q` with `1.0533·x − 218.45 < q ≤ 1.0533·x` is stable and
the settled value depends on history (mid-scale `x = 2048`: 1939 … 2157; inputs `x ≤ 207` stay at 0 from a
zero state). For `x ≥ 3889` the ideal output exceeds the 12-bit field `y[26:15]` (wrap or saturation
**UNVERIFIED**). LF reject and AC (§3.2, §3.3) inherit the dead band (DC residual up to ±109 codes at
mid-scale). An open image SHOULD NOT copy these artefacts and SHOULD compute
`y[n] = 158·x[n] + ⌊32618·y[n−2]/2^15⌋`.

### 3.2 LF reject (code 2)

`out = x − lpf` (first-order high-pass with the same 91.3 kHz corner), re-centred to mid-scale and
saturated to 12 bits (**UNVERIFIED** exact offset/saturation; the DC gain 1.053 of `lpf` leaves a residual
of −5.3 % of the DC input).

### 3.3 AC (code 3)

`mean` = average of `lpf` over consecutive blocks of **2^30 CLK250 cycles (4.295 s)**: `lpf` is summed
every cycle (48-bit sum), and at the end of each block `mean` = `sum[41:30]` and the sum restarts from 0.
`out = x − mean`, re-centred as for code 2. The DC estimate is therefore updated every 4.3 s (effective
high-pass corner well below 1 Hz) and is meaningless during the first block after configuration.

An open image MAY implement codes 1–3 as standard first-order filters with ≈ 91 kHz (HF/LF reject) and
≤ 1 Hz (AC) corners; it MUST pass code 0 unfiltered.

---

## 4. Event-engine parameters (storage level)

- Time bounds T_MIN/T_MAX (and AB_DLY, R_IDLE, holdoff) are 48-bit counts; widths are compared in 1 ns
  slots (spec 05 §5.3).
- `Δt` = number of 1 ns slots from the slot of the reference (start) edge to the slot of the end edge
  (a pulse that is high for exactly N samples has `Δt = N`). Qualification, independent of the slot phase:

  | `TQ_MODE` | Event fires at the end edge when |
  |---|---|
  | 0 | `Δt ≥ T_MIN` |
  | 1 | `Δt < T_MAX` |
  | 2 | `T_MIN ≤ Δt < T_MAX` |
  | 3 | `Δt < T_MIN` or `Δt ≥ T_MAX` |
  | 4 | timeout: no end edge. With the start in slot `S = 4c + s` (clock `c`, slot `s`), the event is issued in the clock that contains slot `S + T_MIN` if the qualifying level is still present in the **last slot of that clock**, i.e. the level must last ≥ `4·⌊(S + T_MIN)/4⌋ + 4 − S` slots (`T_MIN` = 1000: start slot 0 → 1004, 1 → 1003, 2 → 1002, 3 → 1001; `T_MIN` = 1002: 1004, 1003, 1006, 1005) |
  | 5, 6, 7 | never (no event in any detector) |

  So the lower bound is inclusive and the upper bound exclusive (as in spec 05 §5.2).
- Edge codes as used by the driver behave as specified in spec 05: `EV_CFG_A = 0x00AA4100` (CH1 rising)
  fires only on the rising crossing, `0x00220041` (CH1 falling) only on the falling crossing.
- Latency, between fixed points: let `c` be the CLK250 cycle in which the **comparator input samples**
  (the 4 × 12-bit slot samples per channel of §2.1) hold the sample that completes the event: the
  crossing slot of an edge, the end-edge slot of a pulse width, or (TQ_MODE 4) the clock defined in the
  table. The threshold compare of §2.2 has its result in `c + 1`, the TRG event output is set
  in `c + 22`, the SEQ trigger accept (the flag that starts the post-trigger count, §7 step 3) in `c + 23`
  (92 ns) and the AUX pin rises in `c + 27`. The latency does not depend on the slot inside the cycle.
  The ISERDES → comparator-input latency per channel mode is in §2.1 (22 cycles in 1-channel
  mode); the record-memory placement of the trigger relative to the accept is in §8.3.
- For `TQ_MODE = 3` the running comparison uses T_MIN as the active bound, otherwise T_MAX.
- **Holdoff** (`H = HO_TIME_HI[15:0]:HO_TIME_LO`, 48-bit, CLK250 cycles = 4 ns): the holdoff sits between
  the event engines and the SEQ and runs independently of the SEQ state. An engine event *passes* when at
  least `H + 2` cycles have elapsed since the previous event that passed; events in between are dropped and
  do not restart the timer. The SEQ accepts the first passing event after its trigger enable (§7). A
  passing event that arrives while the SEQ is not armed is lost but still restarts the timer, so the
  holdoff interval is measured from the last passing event, not from the last accepted trigger or from the
  arm. `H = 0` passes every event. (A drop-in image keeps the `+2`; an open image SHOULD use `≥ H` and
  MAY keep `+2`.)
- **Detectors.** `OSEL1`/`OSEL2` are decoded as a pair; each pair selects one detector with a *start* and
  an *end* instant (`Δt` = slots from start to end; the event is issued at the end when `TQ_MODE` 0–3
  holds). An end consumes the pending start whether or not `TQ_MODE` holds, except where the table says
  "not consumed". `EDGE` codes: 0 rising, 1 falling, 2 either, 3 never. With the driver's values:

  | (OSEL1, OSEL2) | used for | start | end (= event) |
  |---|---|---|---|
  | (2,2) | Edge, Timeout, Window, Nth, protocol lines | `x` falls | `x` rises, `x = (OINV1 ? c1 : ¬c1) & (OINV2 ? c1 : ¬c1)` (CMP1 only; EDGE ignored; OINV1 ≠ OINV2 never fires) |
  | (2,1) | Pulse, Slope | CMP2 edge per EDGE2, first start wins | CMP1 changes to level OINV1; the start is cleared by every CMP1 edge selected by EDGE1, whichever its direction, evaluated after the end, not by the end itself (with the driver's EDGE1 that edge is the end edge) |
  | (1,1) | Delay | CMP2 edge per EDGE2, first start wins | CMP1 edge per EDGE1. OINV1 = OINV2 = 1 swaps the roles and inverts the sense of the pending flag (from configuration a start counts as pending, so the first swapped start is ignored); OINV1 ≠ OINV2 never fires |
  | (2,3) | Setup | CMP2 edge per EDGE2, a later start replaces the pending one; the opposite CMP2 edge (EDGE2 0: falling, 1: rising) cancels it | CMP1 edge per EDGE1, not consumed (each further CMP1 edge fires again, `Δt` from the same start) |
  | (2,6) | Hold | CMP2 edge per EDGE2, a later start replaces the pending one (`Δt` after a restart is 1 slot short) | CMP1 edge per EDGE1 |
  | (3,6), (3,1) | Runt | CMP2 edge per EDGE2 | opposite CMP2 edge; a CMP1 edge per EDGE1 in between aborts |
  | (0,0) | Pattern / Duration without edge | pattern `P` becomes true (OINV = 0; OINV = 11: start and end swapped) | `P` becomes false |
  | (2,0) | Pattern with edge | none needed (`Δt` = time `P` has been true; `P` true since configuration counts; `TQ_MODE` 4: `P` becoming true starts, `P` becoming false cancels) | CMP1 edge per EDGE1 while `P` was true **in the previous slot**; not consumed |
  | (0,2) | Duration with edge input | every CMP2 edge per EDGE2 restarts `Δt` (latest wins); a start is pending while `c1` is at the level that an EDGE1 edge leaves (EDGE1 0: `c1` = 0; 1: `c1` = 1; 2, 3: not characterised) | `P` becomes false while a start is pending; not consumed; evaluated with the state before the slot's own edges; OINV2 has no effect |

  A start and an end in the same slot: (1,1) with OINV 00 drops that start (the end uses an earlier pending
  start; with none, nothing happens); (2,3) ignores that end (the start is kept). Whether (2,1) and (2,6)
  drop the start in the same way is **UNVERIFIED**.

  `P` is the AND of the EV_IN terms (§2.3); **EV_IN acts only in the (0,0), (2,0) and (0,2)
  detectors**. Per-type consequences and the remaining open codes: spec 05.
- N-th edge (`ABR_EN = 0x5C0`): engine-A events are counted with a full 32-bit compare against
  `A_COUNT` (1 = first event, 0 = never); after the trigger no further one until the count restarts.
  Let `t` = clocks since the reference, the last counted event (an event counted at `j = R + 6` below does
  not move it), and `t = k·(R + 7) + j` with `0 ≤ j ≤ R + 6` (`R` = `R_IDLE`): `k = 0, j ≤ R` continues
  the count; `j` = `R + 1` … `R + 5` is discarded (the count is cleared at `j = R + 1` whether or not an
  event arrives); `j = R + 6` is counted, then the count is cleared, so the next event is event 1 again;
  any other `j` (`k ≥ 1`) is event 1 of a new count. So while idle the restart window repeats every
  `R + 7` clocks. Two or more events inside one discard window follow a rule that is **UNVERIFIED**; the
  phase of the sequence at the first event after the `R_IDLE` / `TRG_SEL` writes is not defined.
- `EV_CFG.OSEL2 = 7` holds the engine's second output in reset.

---

## 5. Trigger output, force, input-path flag

### 5.1 AUX OUT

`AUXOUT_SEL[2:0]` (`0x1430`) selects the rear-panel output pin source:

| value | output |
|---|---|
| 0, 4, 5, 6, 7 | trigger pulse |
| 1 | SEQ trigger-accept status (the flag that also clocks the trigger-position capture, §5.4) |
| 2 | constant 0 |
| 3 | mask pass/fail |

Trigger pulse: the pin goes high **4 CLK250 cycles (16 ns) after the SEQ trigger accept** (27 cycles after
the completing sample is at the comparator input, §4) and stays **high for 15 CLK250 cycles = 60 ns**, then
low; pulses are generated only while the acquisition state machine is active (a stop clears the pulse
logic). Polarity at the BNC depends on the board buffer (**UNVERIFIED**, active-high at the FPGA pin).

### 5.2 Force

`TRG_SEL.FORCE` (bit 5) is synchronised into CLK250 (2 stages); its **rising edge** sets a pending-force
flag in the SEQ that stays set until the state machine consumes it as a trigger. Leaving the bit at 1
forces only once; a 1-then-0 pulse is one force.

### 5.3 Input-path flag (TRG_PATH)

At arm time the SEQ registers a flag that is 1 when any of: `TRG_SEL.TRG_PATH` = 1; engine A `CMP1` =
`0x31` (EXT); a protocol `TRG_KIND`; engine A `OSEL1` = `OSEL2` = 0 (pattern without edge). The flag
selects which of two holdoff-elapsed gates qualifies the TRG event at the SEQ accept
(`accept = enabled & ¬accepted & (FORCE | (event & holdoff_ok))`, `accepted` = this frame has accepted
its trigger); it does not select a different event path. The timing difference of the two gates is
**UNVERIFIED**. An open image MUST give identical comparator results for both values (spec 05 §6) and
SHOULD apply its own latency compensation.

### 5.4 Trigger position

At the trigger accept the SEQ captures the trigger time: a free-running CLK250 cycle count combined with
the TRG slot of the event. It becomes the frame header's time stamp (§8.3, 0.5 ns per LSB) and is read
through `TSTAMP` (`0x40B0`/`0x40B4`). The encoding of the slot in the low bits is **UNVERIFIED**;
spec 05 §10 does not use a sub-sample position.

### 5.5 EXT and AC-line inputs

- **EXT** (engine source index `0x31`) is the extra LVDS lane of the ADC interface (balls N3/N2, spec 20
  §1), deserialised like a data lane: 4 bits per CLK250 cycle, Q4 earliest, one bit per 1 ns slot, used
  directly as the comparator state of the EXT source (no threshold in the FPGA). An EXT edge in ISERDES
  cycle `c` gives the TRG event in `c + 20` (2 cycles less than the comparator sample → event
  path of an analog channel, §4). Total ISERDES → event: EXT 20, analog 1 channel 22 + 22 = 44,
  2 channels 141, 3–4 channels 111 cycles; so an analog edge reaches the event 96 / 484 / 364 ns later
  than an EXT edge with the same ISERDES timing (multi-channel values subject to the FIFO start-up of
  §2.1). With the driver's EXT edge encoding (EDGE fields 0, detector (2,2)) the slope comes from OINV
  (1 rising, 0 falling).
- **AC line** (index `0x30`) is the single-ended input of spec 12 §7.3: a 3-stage synchroniser, then an
  edge qualifier that passes an input edge only if the previous level lasted ≥ 4096 cycles (16.384 µs);
  a passed edge reaches the filtered output 4 cycles after the pad change, rejected edges are ignored.
  Event = filtered output + 20 cycles.
- The EXT compensation of spec 02 §6.6 (119 / 400 / 401 ns for 1 / 2 / 3–4 channels) matches the
  1-channel difference (96 ns) plus the ADC pipeline; the 2- and 3–4-channel values do not match the FIFO
  latencies of §2.1 (**UNVERIFIED**).

---

## 6. Bandwidth-limit FIR (`BWF`)

### 6.1 Coefficient port

Each write to `BWF_COEF` (`0x109C`) loads one coefficient register:

```
word[31:30] = bank   (0…3)
word[29:28] = 0
word[27:23] = tap    (0…23)
word[22:0]  = coefficient, 23-bit two's complement (bit 22 = sign)
```

There are 4 banks × 24 taps = 96 coefficient registers, written in any order; other field values are
ignored. Coefficient sets of spec 06 (24-bit two's complement) MUST fit in 23 bits.

### 6.2 Datapath

Each bank is a 24-tap multiply-accumulate column: tap `k` of bank `b` multiplies its coefficient by a
14-bit signed sample selected for that tap; the four banks give the 4 parallel FIR output
phases/channels per cycle. The driver's layout of a 24-tap set `h[0..23]` (spec 02 §2.10): 1 channel:
bank `b` tap `t` = `h[4·(t mod 6) + b]`; 2 channels: `h[2·(t mod 12) + (b mod 2)]`; 4 channels: `h[t]`.
Which input sample (channel, slot, delay) feeds each tap, and how partial sums are combined, is
**UNVERIFIED**.

The FIR is on the **play-back path** (§9): its input is the 12-bit record data read back from memory,
its output feeds the play-back resampler; the record path does not pass through it. With
`BWF_MODE` (`0x1098`) = 1 the FIR is bypassed (output = input). The coefficient registers load only
while `BWF_MODE` = 2; load sequence: write 2, the 96 words, then the final mode. Words written in
mode 0 are ignored, and an unloaded FIR in mode 0 outputs 0 (mid-scale after the transform of §9.1).
The record path carries no gain other than the `LANE_GAIN` stage of §2.1, and `BWF_MODE` does not
change recorded codes.

---

## 7. SEQ record sequence

With the run sequence of spec 03 (stop, configuration including `SEQ_FUNC`, `SEQ_CMD` load of the frame
range with bits 25|29, `RUNCTL.RUN` = 1; no status clear precedes RUN, and the state machine arms
without one):

1. The state machine arms (frame start) ≈ 6 cycles after the RUN write; from then on the record path
   writes into frame `FR_B` (§8); the first memory write follows ≈ 25 cycles later (pipeline).
2. **Pre-trigger fill**: triggers are accepted only after `PRE_LEN` samples have been recorded since the
   arm, counted at the record sample rate (8 samples per CLK250 cycle at 2 GSa/s): enable ≈ arm + 12 +
   `PRE_LEN`/8 cycles (1 channel, 2 GSa/s; that `PRE_LEN` counts record samples at other record rates
   is **UNVERIFIED**). An event already in the 23-cycle trigger pipeline (§4) when the enable rises is accepted.
   While waiting, recording continues and wraps inside the frame's `FRAME_BURSTS` bursts.
3. **Trigger accept**: the first event that passes the holdoff (§4), or a pending FORCE, or the auto
   trigger (§7.1) after the enable is accepted; the trigger position is captured (§5.4) and the
   AUX pulse is generated (§5.1).
4. **Post-trigger**: recording continues for `POST_LEN` samples plus the fixed data-path latency, then
   the frame is closed: one more memory write stores the frame header (§8.3).
5. **Re-arm**: with frames left in `FR_A…FR_C`, the next frame arms 24–34 cycles after the header
   write (recording into the next frame slot); its trigger enable follows after the pre-trigger fill as in
   step 2. Frames are written at
   `FR_B, FR_B+1, …`; recording stops after `FR_C`.

### 7.1 Auto trigger

Auto triggering is enabled by `SEQ_FUNC[0]` = 1. It uses two timers, both in CLK250 cycles (4 ns):

1. **Timer 1** (32-bit up-counter, "no trigger event for a while"). It is reset (held at 0 for that
   cycle) by every TRG event, i.e. every engine event before the holdoff gate, whether or not the SEQ is
   armed and whether or not the event passes the holdoff; it is held at 0 while `SEQ_FUNC[0]` = 0 (its
   configuration-time value is 1), while a FORCE is pending, and from the accept of an event or forced
   trigger until that frame ends (commit or stop). Otherwise it counts **every** cycle: it is not
   restarted by the arm, by a re-arm, by an auto-triggered frame, or by a stop/run cycle that leaves
   `SEQ_FUNC[0]` = 1. Its flag sets when
   `timer1[31:15] > AUTO_T1[31:15]` and is cleared together with the timer. Bits `[14:0]` of
   `AUTO_T1` are ignored, so the flag sets `(floor(AUTO_T1 / 2^15) + 1) · 2^15` cycles after the
   timer starts (granularity 131 µs, ≤ 1.3 % error for values ≥ 2.5 M, as spec 05 uses); `AUTO_T1` = 0
   gives exactly 32 768 cycles. Neither the RUN nor a rewrite of `SEQ_FUNC` = 1 restarts it.
2. **Timer 2** (32-bit up-counter, per frame). It is held at 0 while no frame is active (between frames
   and when stopped) and counts only while the trigger enable is set (armed after the pre-trigger fill,
   §7 step 2) **and** the timer-1 flag is set. At the start of every frame the logic latches an operand
   `OP = {AUTO_T2[27:16], R}`, where `R` is the 16-bit output of a free-running pseudo-random generator
   (re-sampled for every frame; `AUTO_T2` bits `[31:28]` and `[15:0]` are ignored). The auto trigger
   fires when `timer2 > OP`, i.e. after `AUTO_T2[27:16] · 2^16 + R + 1` armed cycles with the flag set;
   `R` dithers the auto period by 0 … 262 µs.
3. The auto trigger is a normal SEQ trigger accept (post-trigger, frame close and re-arm as in §7); it
   sets `SEQ_STAT` bit 7 but not bit 6 (§7.2).

Consequences: with no signal, the first auto trigger comes when timer 2 has run out after the timer-1
flag set: at `max(t1, enable) + OP + 1` (`t1` = time of the flag, `T1` after configuration, after the last
TRG event or after `SEQ_FUNC[0]` became 1; `enable` = the frame's trigger enable); every following frame
auto-triggers `OP + 1` cycles after its own trigger enable (with a new `R` per frame), as long as no TRG
event occurs. Example (`AUTO_T1` = `AUTO_T2` = 0, no events): flag at cycle 32 768 after
configuration, auto accepts at 32 770 + 36 411, then enable + 52 261 and enable + 42 757 for the next
two frames. Any TRG event (also one during the pre-trigger fill, between frames, or suppressed
by the holdoff) clears timer 1 and its flag, so an auto trigger needs `T1` without any event first.
An open image MUST implement the two-level rule (no TRG event for `T1` → then an auto trigger
`T2` after each arm); it MAY omit the 2^15 granularity and the random dither.

### 7.2 Status and frame-progress registers

With the run sequence of spec 03 (e.g. frames `FR_A = FR_B = 5 … FR_C = 8`):

- **Frame progress** (`FR_A` read, 20 bits): loaded by the `FR_A` write, then set to the index of
  each frame when that frame is committed (header written, §7 step 4); not changed by a stop. So
  `R(FR_A) − R(FR_B) + 1` = number of completed frames once at least one frame is complete (it
  also gives 1 before the first commit; `FR_READY` bit 30 tells the cases apart).
- **`FR_READY`** (`0x4080`): bit 30 (frame available) is set at every frame commit and cleared by a write
  with bit 31 = 1 (a pulse `v`, `v | 0x80000000`, `v`); the other written bits do not affect it
  (writing bit 30 = 1 does not set it). Bit 31 reads back as written. Bits `[3:0]` and `[7:4]` count frame
  starts and frame commits (mod 16), bits 27:26 read 1; software need not use them. The bit-31 pulse of
  the full stop also clears the counters.
- **`SEQ_STAT`** (`0x4034`) read bits:

  | Bit | Meaning |
  |---|---|
  | 3:0 | state code: 0 idle, 1 run start, 2 recording a frame (pre-fill, armed, post-trigger), 0xE/0xF frame commit |
  | 4 | WAIT: trigger enable: 1 from the end of the pre-trigger fill until the frame is committed (also during the post-trigger part after the accept); 0 during the pre-trigger fill and between frames. A status decoder that tests bit 6 before bit 4 never shows the post-trigger WAIT |
  | 5 | 4-cycle pulse at the `RUNCTL` bit-31 stop pulse; reads 0 otherwise |
  | 6 | sticky: a frame was triggered by an event or FORCE |
  | 7 | sticky: a frame was triggered (any source, including auto) |
  | 8 | sticky: a record or play sequence ran to `FR_C` |
  | 12 | sticky: a record sequence ran to `FR_C` |
  | 9 | 0 in record and play sequences (other use **UNVERIFIED**) |
  | 10, 11 | the current frame has accepted its trigger (any source / event or FORCE): set at the accept, cleared when the frame is committed (2 cycles after the frame ends), also for the last frame of a sequence |
  | 13 | IDLE: 1 while no record or play sequence is running (before RUN, after the frame at `FR_C`, after the last word of a play, after a stop); 0 from RUN until the sequence ends, **including between frames** |
  | 16–19 | 0 |
  | 14, 15, 20–31 | 0 (not driven) |

  The sticky bits 6, 7, 8 and 12 are cleared by a `SEQ_STAT` write with bit 31 set (clear sequence:
  `0xC0000000 | last`, then `last`; the effect of written bit 30 alone is **UNVERIFIED**). So a status
  decoder that reports T'D for bit 6 and AUTO for bit 7 alone reports an event-triggered frame as T'D and
  an auto-triggered one as AUTO. Bit 13 stays 0 while frames remain to be recorded, so a stop MUST NOT
  wait for it before issuing the stop pulse (spec 03 §3.2).
- **During a play** (and sequence play) command: `[3:0]` ≠ 0 (play states, an end state like a record
  commit at the end), bit 13 = 0 from RUN until the last word of the play has been written to the C2H FIFO,
  then 1; at that point bit 8 is set; bit 12 is set only by a record sequence; bits 4, 10, 11, `FR_READY`
  bit 30 and the frame progress are not changed by a play.

A drop-in image MUST provide bit 30 of `FR_READY`, the frame-progress value of `FR_A`, `FR_B` as
written, bit 4 and bits 6/7 of `SEQ_STAT`; bit 13 SHOULD follow the rule above.

## 8. Record memory (DDR3)

### 8.1 Addressing

The record memory is addressed in **bursts** of 32 bytes (one burst-length-8 access on the 32-bit DDR3
bus; burst `b` = byte address `32·b`, 2^25 bursts = 1 GiB). Only 240 of the 256 bits of a burst are used.

| Content | Burst address |
|---|---|
| frame `f` data | `f · FRAME_BURSTS … f · FRAME_BURSTS + FRAME_BURSTS − 1` (`FRAME_BURSTS` = register `0x10E0`, in bursts) |
| frame `f` header | `0x1F00000 + f` |

Writes within a frame are strictly sequential from the frame start (wrapping inside the frame while
waiting for the trigger, §7). With the frame layout of spec 04 (frame count · `FRAME_BURSTS` = 15 728 640 =
`0xF00000` bursts per ping-pong half) both halves end below `0x1E00000`; the top 32 MiB hold the headers.

### 8.2 Sample packing

Each burst holds **20 consecutive samples of 12 bits** (the ADC code `[13:2]`, spec 20 §3.3), no gaps
between bursts. Bit layout of the 240 used bits (bit 0 = LSB of the 256-bit memory data word):

```
sample j of the burst (j = 0 earliest … 19 latest) occupies bits [12·(19−j) + 11 : 12·(19−j)]
```

i.e. the earliest sample is in bits `[239:228]`, the latest in `[11:0]`. With the word alignment off
(`0x1020` = 0) the 8 samples of a CLK250 cycle (`t` = 0…7 = `g0.Q4, g1.Q4, g0.Q3, g1.Q3, g0.Q2, g1.Q2,
g0.Q1, g1.Q1`, spec 20 groups/positions) are recorded in this order. With the word-alignment setting of
spec 20 (`0x1020 = 0x1010`, both groups enabled, rotation 0) the eight record positions of a cycle hold
`t` = 6, 7, 2, 3, 4, 5, 0, 1 of the incoming ISERDES words (per group Q1, Q3, Q2, Q4); which ISERDES
framing the real ADC delivers at rotation 0, and therefore whether the hardware record is time-ordered,
is **UNVERIFIED**. `CH_SKEW` and `CORE_ORDER` act only together with their update strobes
(**UNVERIFIED** effect).

2- and 3–4-channel modes keep 20 samples of 12 bits per burst with the channels interleaved sample by
sample. With the alignment off the record positions are `t` in time order per cycle for 2 channels
(even positions = group 0 = the lower-numbered active channel, odd = group 1 = the higher, 10 instants per
burst) and `t` = 0, 2, 1, 3, 4, 6, 5, 7 for 3–4 channels, i.e. position `j` holds channel `(j mod 4) + 1`
with CH1 = `t` 0/4, CH2 = 2/6, CH3 = 1/5, CH4 = 3/7 (5 instants per burst) — the same lane-to-channel
assignment as the TRG slots (§2.1). Which ADC core drives which (group, Q) position is set by the ADC
output mode (spec 06).

An open image with its own memory controller MAY use any internal layout as long as play-back (§9) and
the C2H stream (spec 03 §2) are unchanged; this layout is required only for compatibility with memory
contents in this layout (e.g. a replayed import).

### 8.3 Frame header

One burst per frame, written right after the frame's last data burst. Bits `[239:128]` are 0.

| Bits | Content |
|---|---|
| `[31:0]` | `H` = absolute sample index (`burst · 20 + position`, counted from burst 0 of the memory) of the first sample of the frame's `PRE + POST` window |
| `[63:32]` | 0 with the configuration of spec 03; other values with other configurations (199 with `PRE = POST = 100` and CH1 in the channel mask; meaning **UNVERIFIED**); `0x10FC` when `SEQ_CMD[18]` |
| `[95:64]` | trigger time stamp, LSB = 0.5 ns, 1 ns resolution: `8 · (accept cycle + 2)` with the CLK250 cycle counted from configuration, plus `2·⌊t/2⌋` for a 1-channel crossing at sample `t` of its clock (i.e. the 1 ns position of the crossing); 2-/3–4-channel units **UNVERIFIED**; `0x40A0` when `SEQ_CMD[18]` |
| `[127:96]` | upper time-stamp word (0 within the first 2^32 · 0.5 ns); `0x40A4` when `SEQ_CMD[18]` |

The same captured time stamp is what `TSTAMP` (`0x40B0`/`0x40B4`, latched by `TSTAMP_CTL` bit 0) reads,
so `TSTAMP` has the same 0.5 ns unit (**UNVERIFIED** in 2-/4-channel modes).

Window placement: `H` is sample-accurate with respect to the trigger. The first sample of the trigger
channel at or after the threshold crossing is at **window index `PRE_LEN + 7`** (record index `H +
N·(PRE_LEN + 7)` with `N` = active channels per instant: 1, 2 or 4), for every position of the crossing
within the 4 ns clock and in 1-, 2- and 3–4-channel modes (word alignment off, no channel delay). Spec 02
§6.6 places the trigger at window index `PRE_LEN` (plus the fine delay of spec 03 §4); whether the
ADC configuration of specs 06 and 20 (word alignment, `CH_SKEW`) accounts for the 7-sample offset is
**UNVERIFIED**. The 32-bit store written by `PRE_ADJ` (`0x404C`, §1.3) is an operand of the position
logic (bit 31 of the store selects a different operand, Scan mode); changing it does not change `H`,
the trigger enable or the accept in Normal mode; its effect in Average and Scan modes is
**UNVERIFIED**. An open image MAY ignore `PRE_ADJ`.

After an event-triggered accept the SEQ starts a sub-sample search that reads the stage-2 sinc
coefficient sets (`SINC_WORD` targets 2 and 3, §9) and holds the frame close until it finishes; without
those sets loaded a frame with a real threshold crossing in the record may never close, so they MUST be
loaded at start-up (spec 02 §3.5). Its result and its use (header word `[63:32]`, time stamp) are
**UNVERIFIED**; an open image SHOULD NOT make frame completion depend on them.

With `SEQ_CMD[18]` (import record, H2C) the header fields come from the registers `0x10FC`, `0x40A0`,
`0x40A4` instead of the internal sources.

## 9. Play-back read

A play command (`SEQ_CMD` bits 30|29, frames `FR_B…FR_C`) reads, per frame, first the frame header
(burst `0x1F00000 + f`), then the data bursts that contain the samples `H + WIN_START − m … H +
WIN_START + WIN_LEN − 1 + m` with a margin `m` of 9…11 samples (e.g. `H` = 844, `WIN_START` = 4950,
`WIN_LEN` = 685 → bursts `0x121…0x144`; `H` = 708 + 20·`0xF00000`, `WIN_START` = 0, `WIN_LEN` = 1001 →
bursts `0xF00022…0xF0003F, 0xF00000…0xF00015`, `m` = 10 fits both). Reads wrap inside the frame's
`FRAME_BURSTS` bursts like the recording does. So `WIN_START` is relative to the frame's window start `H`,
not to the frame's first burst.

Play-back datapath, in order: memory read → unpack to 12-bit samples → bandwidth FIR `BWF` (§6;
bypass with `BWF_MODE` = 1) → sinc interpolator (`SINC_RATIO` ratio `r`) → linear interpolator →
compression (`DECIM`) → display transform (§9.1) → C2H FIFO. With sinc ratio 1, the linear
interpolator off and no compression, output word `i` of a frame is record sample `H + WIN_START + i + 2`
(coarse delay of the play configuration of spec 03; other delays **UNVERIFIED**).

- **Sinc interpolator**: works on the unsigned 12-bit code and keeps 4 fractional bits (its output is
  `x_s` in units of 1/16 code; the transform of §9.1 uses `16·x_s` in place of `16·x`). With ratio `r`
  every input sample yields `r` outputs (branches); the low bits of `SKEW_FINE` rotate the branch order.
  The DC gain per branch with the coefficient sets of spec 02, the rounding and the saturation are
  **UNVERIFIED**; with the `r` = 2 set the branch gains are expected to be unequal (1.0105 / 0.9643), which
  the coefficient sums do not explain.
- **Compression** (`DECIM` factor `C` > 1): every pair of output words covers `2·C` consecutive
  samples and holds their minimum and maximum, **in the order in which they occur** (the extreme that
  occurs first is emitted first). So the stream carries one word per `C` samples, alternating between
  the two extremes of each `2·C` group.

### 9.1 C2H output stage

The play-back result enters an asynchronous FIFO of **2048 words** (written in CLK250, read in CLK125 by the
C2H engine), each 64 data bits = four 16-bit **lanes**, plus a flag that marks the last word of the play.
Each FIFO word carries four consecutive output words of the stream. With `OUT_CTRL.NO_INFO` (`0x1010` bit 31)
= 1, the setting of spec 02, a play of `n` frames writes exactly `n · N · (M + Z) / 4` FIFO words (e.g. 1 ch
`M` = 416 → 104 words, 2 ch `M` = 520 → 260 words): no header, no padding, no trailer (plays of `n` > 1
frames: **UNVERIFIED**). With `NO_INFO` = 0 **two extra 64-bit words (16 bytes) of frame information
precede the samples** (1-ch play of one frame: 106 words; content below; whether they repeat per frame or
per play is **UNVERIFIED**). So bit 31 = 1 means "no frame head". The frame header of §8.3 is consumed by
the play-back read and is not forwarded.

Lane `k` (`k` = 0…3) is FIFO bits `[16k+15:16k]` and carries **output slot `3 − k`**. On the way out,
`OUT_CTRL` (`0x1010`) bit 30 selects the order of the four lanes within each 64-bit beat (bit 30 = 0: lane 3,
2, 1, 0; bit 30 = 1: lane 0, 1, 2, 3). With `0x1010 = 0xA0000000` (spec 02; bit 30 = 0) the beat is
therefore emitted as **slot 0, 1, 2, 3**, which is the stream order of spec 03 §2 (1 channel: four
consecutive samples; 2 channels: `c_lo, c_hi, c_lo, c_hi`; 3–4 channels: `CH1 … CH4`). The byte order
through the XDMA engine follows from this lane mapping and the parse order of spec 03 (**UNVERIFIED** at
the engine itself). `OUT_CTRL` bit 29 gates the packet end: on the C2H AXI-Stream `TLAST = last_word & tvalid
& tready & ¬OUT_CTRL[29]`, so with bit 29 = 1 (spec 02) no packet end reaches the XDMA engine (a transfer
completes on the descriptor length, spec 14 §4.2). An open image MUST emit the stream of spec 03 §2.1
(`NO_INFO` = 1) and SHOULD implement bit 30 as above; it MAY omit the `NO_INFO` = 0 frame-information
words.

Output word (display domain), per output slot `s` with `PB_GAIN[s]` at `0x14C8 + 0xC·s` and `PB_OFS[s]`
at `0x14D0 + 0xC·s` (per **output slot**: in 2-channel mode the lower channel's values go to slots 0 and 2
and the higher channel's to 1 and 3; in 1-channel mode the active channel's values go to all slots):

```
y = clamp( floor(PB_GAIN[s] · 16·x / 1024) + PB_OFS[s] , 0, 65535 )      # PB_OFS signed 32-bit
```

with `x` the 12-bit unsigned record code (offset binary; with the sinc interpolator active `16·x` is
its 4-fractional-bit output, §9). `floor` is a true floor of the unsigned product (no rounding): e.g.
`PB_GAIN` = `0x45A` (17.40625 per code), `PB_OFS` = −2883: `x` = 1788 → 28239, 1789 → 28256, 1802 →
28483. Examples at `x` = 2048: `0x45A`/−2883 → `0x7FFD`, `0x400`/0 → `0x8000`, `0x200`/+1000 →
`0x43E8`, `0x45A`/+40000 → `0xFFFF` (saturated, not wrapped), `0x446`/−40000 → `0x0000` (saturated).

The `NO_INFO` = 0 frame-information words are, in stream order as u16: `0xFA05, 0x0000, 0x007B | Peak << 12 |
Average << 13, 0x0000, 0xC000, 0x0000, 0x0000, 0x0000` (Peak = `SP_MODE` bit 1, Average = bit 3);
they do not change with the time stamp, `WIN_START`, `WIN_LEN`, `OUT_LEN4`, the sinc ratio or
compression. Other fields and whether they repeat per frame are **UNVERIFIED**. The number of sample
words follows `OUT_LEN0` (`0x11F0`): 512 points → 128 FIFO words in 1-channel mode.

## 10. Not covered

Record-side decimation (sample rates below the maximum, Peak, Average), the sinc tap orientation and
scaling and the linear interpolator (§9), the
sub-sample trigger search (§8.3), the frequency counter / DVM block (`0x2210`, `0x2214`, `0x2244`: units
**UNVERIFIED**) and the SP/TRG read-back multiplexers are not specified here.
