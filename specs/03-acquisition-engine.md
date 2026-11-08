# 03 — Acquisition Engine

The FPGA does all real-time work: ADC capture, calibration gain/offset, trigger, segmented
recording into its DDR3, and resampling of each recorded frame to display length. The CPU
configures it, starts a segmented recording, waits until frames are available, stops, and then
commands a **play-back** of the recorded frames; the SP block streams the resampled display records
to the CPU over the XDMA C2H engine. This document specifies that cycle at register level, the
record format the CPU receives, and the slow (roll/scan) variant. Registers: spec 02. Rates, depths,
frame counts and resampling arithmetic: spec 04. Trigger: spec 05. ADC capture: spec 20.

**Every value and behaviour in this spec is UNVERIFIED until measured on a unit.** "Factory image"
means the FPGA image installed on the unit (spec 13).

---

## 1. Data path

```
ADC (8 cores, 2 GSa/s, 12 bit, 28 LVDS lanes)
  → SP capture: lane IDELAY, word (core) alignment, 14 → 12 bit          (0x1014–0x1030, spec 20)
  → per-channel delay, invert, 20 MHz limit                               (0x1004, 0x1080)
  → decimation / peak / average                                           (0x10D0, 0x10D8, 0x10B0)
  → TRG trigger: 4 slots/channel per 4 ns clock, 12-bit (spec 05 §1, spec 21 §2)
  → SEQ FSM: segmented record into DDR3 frames                            (0x4004–0x4018)
  → play-back: window read, FIR bandwidth filter, sinc / linear
    interpolation, compression, display gain/offset           (0x1098…, 0x1100–0x1174, 0x14C8…)
  → C2H stream to the CPU                                                  (/dev/xdma0_c2h_0)
```

There are two independent display paths per frame, **main** and **zoom**; the zoom path exists only
when zoom is enabled (`SEQ_MODE.ZOOM`). There is no Gray decoder in the capture path (spec 20).
The FIR bandwidth filter (`BWF`) is on the play-back path in the factory image (spec 21 §6, §9), so
the record and the trigger see unfiltered samples; the driver programs it with the acquisition
configuration (spec 02 §2.10).

---

## 2. Sample format delivered to the CPU

- Every sample is an unsigned **16-bit little-endian** word, offset binary, `32768` = the
  channel's vertical centre. Two code domains exist:
  - **Record domain** (ADC → memory, and the TRG trigger): the AFE gear selected for the V/div
    (spec 06, spec 10) gives `7500 / G` codes per division (after the `LANE_GAIN` stage, i.e. with
    the ×2.5 of code 1 included; spec 21 §2.1), where `G` is the channel's residual
    digital gain from the calibration record (`G = gear_native_scale / requested_scale`, ≥ 1).
    A 12-bit record code `a` (the `LANE_GAIN` stage output, spec 02 §2.8; not the raw ADC code unless
    `LANE_GAIN` = 0) is `(a << 4)` offset binary (`(code − 32768) / 16` is the signed 12-bit value).
  - **Display domain** (C2H output): the play-back path applies `PB_GAIN`/`PB_OFS` (main) and
    `ZM_GAIN`/`ZM_OFS` (zoom) per output slot (spec 02 §4): `y = (PB_GAIN/1024)·x +
    PB_OFS`, with `PB_GAIN = trunc(g·1024)` and `PB_OFS = trunc(32768 − g·32768)`,
    `g = min(G′, 64)` in float32, truncation toward zero, so that the CPU always receives **7500
    codes per division**, 32768 = centre. `G′ = G`; when `G ≥ 2` the driver first moves a factor 2
    into the ADC coarse gain (`LANE_GAIN` code 2, exactly twice code 1; spec 02 §2.8) and uses
    `G′ = G/2`. With GND coupling
    `PB_GAIN = 0` and `PB_OFS = trunc(32768 + 7500·offset/scale)` (flat trace at the offset
    position). In the factory image (spec 21 §9.1), per output slot with `x` the 12-bit record
    code: `y = clamp(floor(PB_GAIN · 16·x / 1024) + PB_OFS, 0, 65535)`, `PB_OFS` signed — the
    output **saturates** at 0 and 65535, it does not wrap (main path; the zoom path is **UNVERIFIED**).
    `floor` is a true floor of the unsigned product `PB_GAIN·16·x` (no rounding; e.g. `PB_GAIN`
    `0x45A`, `PB_OFS` −2883: `x` = 1788 → 28239, 1789 → 28256); with the sinc interpolator active
    `16·x` is replaced by its output, which carries 4 fractional bits (spec 21 §9). The real-valued
    model `y = g·x16 + o` differs from this only by < 1 code. With `DECIM` > 1 the words are min/max
    pairs in time order (spec 02 §3.2).
  - After **stop**, a V/div or offset change can re-program only `PB_GAIN/PB_OFS` and replay the
    recorded frame, so a stopped waveform rescales without re-acquiring:
    `g = min(G′_acq·S_acq/S_new, 64)`, where `G′_acq` is the play-back gain of the acquisition (`G`
    of the acquired scale, halved when it was ≥ 2; `LANE_GAIN` is not rewritten while stopped),
    `PB_GAIN = trunc(g·1024)`, `PB_OFS = trunc(7500·Δ + trunc(32768 − g·32768))`,
    `Δ = (O_new − O_acq)/S_new`, plus `trunc(O_acq/S_new)` added to `Δ` when the channel invert
    changed while stopped. GND coupling while stopped: `PB_GAIN = 0`,
    `PB_OFS = trunc(7500·O_new/S_new + 32768)`.
  - **Residual gain for the trigger** (spec 05 §3.3): the calibration-record gain `G` of the
    *current* scale (spec 10 §3.5 step 4), before the `G ≥ 2` halving, also while stopped.
    `PB_GAIN/1024` equals it only while acquiring with `G < 2`; it is `G/2` for `G ≥ 2`, and
    `G′_acq·S_acq/S_new` after a stopped rescale. Example (default calibration, CH1 acquired at
    50 mV/div with `G` = 1.0880, then rescaled while stopped): 1 V/div → trigger `G` = 1.0406,
    `PB_GAIN` = `0x37`; 100 mV/div → trigger `G` = 2.9112, `PB_GAIN` = `0x22D`.
    For `G ≥ 2` the TRG comparators tap the stream after the `LANE_GAIN` stage and before
    `PB_GAIN`/`PB_OFS` (factory image, spec 20 §3.3, spec 21 §2.1), so they see `15000/G` codes
    per division; trigger codes computed with `7500/G` codes per division would act at `L/2` from
    the channel centre. Compute trigger codes in the domain the comparators see (the tap position
    is established; the effect on a unit is **UNVERIFIED**). Settle on a unit: a scale whose
    record has `G ≥ 2`, level at +2 div, check where the trigger fires.
- Channels are **interleaved sample by sample** in ascending channel order of the acquired set:

| Acquired channels | Words per sample instant `N` | Order |
|---|---|---|
| 1 | 1 | `[c]` |
| 2 | 2 | `[c_lo, c_hi]` (lower channel number first) |
| 3 or 4 | 4 | `[c1, c2, c3, c4]` (all four slots are present in 3-channel mode; the unused one is ignored) |

Each 8-byte C2H beat carries the four output slots 0, 1, 2, 3 in stream order (`OUT_CTRL` bit 30 = 0,
spec 02 §2.4): 1 channel four consecutive samples, 2 channels `c_lo, c_hi, c_lo, c_hi`, 3–4 channels
`c1 … c4`. (Factory image: the slot-to-FIFO-lane mapping is spec 21 §9.1; the byte order through
the XDMA engine follows from it, **UNVERIFIED** at the engine itself.)

### 2.1 Frame layout and read size

For one played-back frame, with `N` from the table, `M` = main points per channel (`OUT_LEN0`),
`Z` = zoom points per channel (`OUT_LEN1`, used only when `SEQ_MODE.ZOOM`):

```
frame = main record: M × N words  (interleaved as above)
        zoom record: Z × N words  (only if zoom enabled)
bytes per frame = 2 · N · (M + Z)
```

A play-back of `n` frames delivers `n` consecutive frames; the CPU reads all of them with **one**
`read()` of `n · 2 · N · (M + Z)` bytes from file position 0 (the CPU never seeks). There
is no header, padding or trailer in the stream: the CPU parses the buffer at fixed strides.

`M` and `Z` in the read size are the point counts of the horizontal configuration, while the FPGA
is programmed with `OUT_LEN0 = ceil8(M)` and `OUT_LEN1 = ceil8(Z)` (spec 02 §3.3) and streams
`2 · N · (OUT_LEN0 + zoom·OUT_LEN1)` bytes per frame. When `M` is a multiple of 8 (every
configuration of spec 04 §9 whose window lies inside the record) the read is exactly the stream.
Otherwise a read of `2 · N · M` bytes takes `ceil8(M) − M` points per channel (e.g. 4 or 5) less
than the FPGA sends; this occurs in the single-frame replay of a record whose window is clipped by
the record (see below). An implementation must send `2 · N · (OUT_LEN0 + zoom·OUT_LEN1)` bytes per
frame and must not stall on an unread tail: the output-path reset of the next stop (`SP_RESET` bit
4, §3.2) discards it. The stride of a multi-frame play with `ceil8(M) ≠ M` is **UNVERIFIED**.
(`OUT_CTRL.NO_INFO` = 1 is set once at start-up and kept; in the factory image it suppresses the
frame-information words, so the stream is exactly `n · N · (M + Z)` words; with `NO_INFO` = 0 the
factory image would insert two 64-bit words (`0xFA05, 0, 0x007B | Peak << 12 | Average << 13, 0,
0xC000, 0, 0, 0` as u16) before the samples, spec 02 §2.4, which a fixed-stride parse would
misread. The record-memory frame header is consumed by the play-back read and never forwarded.)

`M` and `Z` are between 1000 and 1 000 000 for a window inside the record (spec 04 §9). A replay
whose window extends beyond the recorded data (e.g. the first replay after a change to a slower
timebase, or a replay after a horizontal offset change while stopped) has proportionally fewer
points (down to e.g. `M` = 116, spec 04 §9). An FPGA MUST accept any `OUT_LEN0`/`OUT_LEN1` ≥ 8 at a
play command. Read sizes (one frame):

| Setting | N | M | Z | bytes |
|---|---|---|---|---|
| 1 ch, 5 ns/div, AUTO | 1 | 1000 | — | 2 000 |
| 1 ch, 2 µs/div, 10 k | 1 | 10 000 | — | 20 000 |
| 2 ch, 5 µs/div, AUTO | 2 | 50 000 | — | 200 000 |
| 4 ch, 5 ns/div, AUTO | 4 | 1000 | — | 8 000 |
| 4 ch, 50 ms/div, 1 M | 4 | 1 000 000 | — | 8 000 000 |
| 1 ch, 50 µs/div, zoom 5 µs/div | 1 | 1 000 000 | 100 000 | 2 200 000 |

### 2.2 C2H engine

The CPU never sets the C2H file position and consecutive reads return new data, so the FPGA
side of `/dev/xdma0_c2h_0` behaves as a **stream** (AXI-ST, or AXI-MM with a fixed FIFO address):
the play-back result is pushed into the engine and the `read()` consumes it. A `read()` of 0
bytes may occur in slow mode (§6) and must return 0.

---

## 3. Run control

### 3.1 Run commands

Every run command `run_cmd(mask, command, n, a, b, c, interval, timer)` is:

```
SEQ_MODE &= ~bit17; write;  SEQ_MODE &= ~bit22; write          # leave roll/scan
<command-specific SEQ_CMD sequence, spec 02 §6.3>
SEQ_MODE[3:0] = mask; write
RUNCTL |= RUN; write
```

A run command writes no `SEQ_STAT` and no `SEQ_FUNC`. `SEQ_FUNC` (auto trigger, mask actions) is
written by the configuration (spec 05 §10, spec 02 §6.4) before the stop/run cycle and keeps its
value across it; `SEQ_STAT` is cleared only when the driver reads status (spec 02 §6.5). An FPGA MUST
arm on the `RUN` write without a preceding status clear. `SEQ_MODE` bit 23 (`ANA_EN`) is already 1
from initialisation and is carried by every write (spec 02 §6.2).

| Command | SEQ programming |
|---|---|
| Run | record-sequence: `FR_GAP = 0`, `SEQ_TIMEOUT = timer`, `FR_A = FR_B = b`, `FR_C = b + n − 1`, SEQ_CMD bits 25, 29 |
| Single | as Run with `n = 1` and `SEQ_TIMEOUT = 0` (also in UltraAcquire) |
| Play | play: `FR_GAP = interval`, `SEQ_TIMEOUT = 0`, `FR_A = a`, `FR_B = b`, `FR_C = c`, SEQ_CMD bits 29, 30 |
| Play last | `RUNCTL.REPLAY` 0→1 (scan: replay the last sweep) |
| Record (waveform recorder) | as Run with the recorder's frame range and interval |
| Record loop | no `SEQ_CMD` programming (mask + RUN only) |
| Import | H2C import into the record memory |
| Import play | play imported frames (**UNVERIFIED**) |

Clamp `n` to `c − b + 1` and to ≥ 1. `timer` (`SEQ_TIMEOUT`, `0x406C`) is the UltraAcquire
timeout (`ticks | (timer ≥ 4 ns) << 31`), 0 otherwise.

### 3.2 Stop

```
stop(full):
    poll SEQ_STAT.IDLE (bit 13) up to 10 × 1 ms             # continue anyway on timeout
    stop_word(0)                                            # 3 writes: clear RUN, bit 31; spec 02 §6.1
    if full: stop_word(1) (bit31 pulse);  FR_READY bit31 pulse (0→1→0)  # clear the frame-ready flag
    else:    RUNCTL.HALT_SAMPLING pulse
    SP_RESET bit0 pulse (SP reset);  SP_RESET bit4 pulse (output-path reset)
    stop_word(0)
```

with ≥ 1 µs between steps. A full stop precedes every reconfiguration, every re-arm and every
play-back.

### 3.3 Normal (triggered) cycle — register timeline

The record memory is used as **two ping-pong halves**: half 0 = frames `[0, F−1]`, half 1 =
frames `[B, B+F−1]` with `B = frames_hw` (spec 04 §7.2) — e.g. 1 ch, 10 k, 500 ns/div:
`FRAME_BURSTS = 1024`, `B = 15 360`, `F = 100`. Average mode uses half 0 only. `F` = frames per half
(spec 04 §7.2).

One cycle (`h` = half being recorded):

```
 1  stop(full)
 2  run_cmd(mask, Run, F, ·, base[h], base[h]+F−1):
       SEQ_MODE (2 writes) ; SEQ_TIMEOUT = timer ; FR_GAP = 0
       SEQ_CMD  = (shadow & 0x89F9FFFF) | 0x30000000          # load window open
       FR_A = FR_B = base[h] ; FR_C = base[h]+F−1
       SEQ_CMD &= ~bit28                                      # latch
       SEQ_CMD |= 0x22000000                                  # record-sequence + enable
       SEQ_MODE[3:0] = mask ; RUNCTL |= RUN
    -- FPGA records triggered frames into base[h], base[h]+1, … (at most F) --
 3  loop (CPU, about every 10 ms):
       ready = read(FR_READY).bit30       # read only; never write FR_READY while polling
       newest = read(FR_A) ; first = read(FR_B)
       until ready
    n = min(newest − first + 1, F)        (n < 0: an error, treat as 0)
 4  stop(full)                             # record of this half ends; the other half becomes current
 5  (replay of a stored half only, e.g. after a stop: FRAME_BURSTS = size of that half)
 6  stop(full)
 7  run_cmd(mask, Play, n, base[h], base[h], base[h]+n−1):
       FR_GAP = 0 ; SEQ_TIMEOUT = 0
       SEQ_CMD = (shadow & 0x89FFFFFF) | 0x30000000 ; FR_A = FR_B = base[h] ; FR_C = base[h]+n−1
       SEQ_CMD &= ~bit28 ; SEQ_CMD = (shadow & ~bit26) | 0x60000000          # play + enable
       SEQ_MODE[3:0] = mask ; RUNCTL |= RUN
 8  read(c2h, buf, n · 2 · N · (M + Z))   # §2.1; the FPGA plays frames base[h] … base[h]+n−1
 9  de-interleave; hand the n frames to the consumers
10  h ^= 1 ; go to 1 (re-arm immediately after the read; the next poll follows about 10 ms later,
    UltraAcquire: at least the UltraAcquire timeout)
```

Frame-counter contract the FPGA must meet:

| Register | Read value (factory image, spec 21 §7.2) | During record-sequence |
|---|---|---|
| `FR_A` (`0x4010`) read | loaded by the `FR_A` write; set to the index of each frame when that frame is committed (header written); not changed by a stop | after `k ≥ 1` committed frames: `first + k − 1` |
| `FR_B` (`0x4014`) read | as written (20 bits) | `first` |
| `FR_READY` (`0x4080`) bit 30 read | set at every frame commit; cleared by a write with bit 31 = 1 (the bit-31 pulse of stop(full)); writing bit 30 = 1 does not set it | 1 = at least one frame committed since the last clear |
| `SEQ_STAT` (`0x4034`) | spec 02 §6.5 | RUN during the pre-trigger fill, WAIT until the trigger, T'D / AUTO after |

So `R(FR_A) − R(FR_B) + 1` = frames completed whenever bit 30 = 1. Before the first commit the
difference is also 1, so check bit 30 first; the count is relative to `first`, not an absolute
frame index. An FPGA MUST return these values as described and MUST NOT compensate. Do not issue a
play when `ready = 1` and the difference is 0; an FPGA SHOULD tolerate a play with `FR_C = FR_B − 1`
(output nothing; the following stop resets the C2H path).

Recording runs continuously from the arm: the FSM writes samples into frame `FR_B` at once,
wrapping inside the frame's `FRAME_BURSTS` bursts while waiting for the pre-trigger fill and the
trigger, then records `POST_LEN` samples, closes the frame with a one-burst header at burst
`0x1F00000 + f` (header `[31:0]` = `H`, the absolute sample index of the first sample of the
`PRE + POST` window; `[95:64]` = trigger time stamp, LSB = 0.5 ns, spec 02 §6; spec 21 §7–§8) and
re-arms into the next frame. In the factory image the trigger-channel crossing lies at window index
`PRE_LEN + 7` (spec 02 §6.6), and after an event-triggered accept the frame close waits for a
sub-sample search over the stage-2 sinc sets (spec 02 §3.5, spec 21 §8.3). The recorder stops
writing at `FR_C` (it must not wrap: a wrap would make `newest < first`, which the cycle treats as
an error). Play-back reads frames `FR_B … FR_C` in order; per frame it reads the header first and
then the bursts that contain the samples `H + WIN_START − 10 … H + WIN_START + WIN_LEN − 1 + 10`
(margin 9…11 samples), wrapping inside the frame's `FRAME_BURSTS` bursts (spec 21 §9). The FPGA has
no notion of ping-pong halves: data and header of frame `f` are at bursts `f · FRAME_BURSTS …` and
`0x1F00000 + f` in either half. Re-arm (step 1–2) and the read (step 8) must not depend on the
display; a driver may hold the next cycle until the display has released the previous trace
buffer.

Errors: `read()` ≤ 0 → drop the frames and re-arm. Not ready → no data this time; poll again (the
FSM keeps recording).

### 3.4 Single

`Single` is Run with `n = 1` (`FR_C = FR_B`): arm, poll until ready, play 1 frame, read, clear
`SEQ_STAT`, then stop without re-arming. `SEQ_STAT` bits 8/9 (bit 8: the sequence reached `FR_C`,
spec 02 §6.5) are not needed.

### 3.5 Stop key

A Stop request ends the cycle at the next step boundary: stop(full), no re-arm. The last
played-back frames remain in the CPU buffer; vertical changes re-program the display gain/offset
and replay (§2).

### 3.6 Status

`SEQ_STAT` (spec 02 §6.5) gives the status shown to the user (RUN / AUTO / WAIT / T'D); it is read
for display only and cleared after reading (`(a<<31)|(b<<30)|last_read` then `last_read`). The
play-back handshake uses `FR_READY` bit30 and the frame counters, not the status word.
`IDLE` (bit 13) = 1 while no record or play sequence runs (idle before `RUN`, after the frame at
`FR_C` has been committed, after a stop); 0 from `RUN` until the sequence ends, **also between
frames**, and during a play until its last word has been written to the C2H FIFO (factory image,
spec 21 §7.2). Polling it ≤ 10 × 1 ms *before* clearing `RUN` (§3.2) and proceeding either way
costs ≈ 10 ms whenever a record sequence that has not reached `FR_C` is stopped (step 4 of §3.3
unless all `F` frames were recorded); a driver may skip the poll. The stop after a play (step 1 of
the next cycle) does not wait: the `read()` has already taken the play's output, and an unread tail
(§2.1) is smaller than the 2048-word C2H FIFO. (Bit 13 for the degenerate play with
`FR_C = FR_B − 1`, §3.3, is **UNVERIFIED**.) An FPGA MAY report 1 earlier; a driver tolerates any
value.

---

## 4. Window and resampling programming (per configuration)

Every horizontal configuration, after the parameter computation (spec 04), writes:

1. `WIN_LEN/WIN_START` main = (`P + 512·N + 72 + C`, `S`), zoom = (`Pz + 512·N + 72`, `Sz`),
   where `P`/`Pz` = memory points in the main/zoom window, `S`/`Sz` = first memory point relative to
   the frame's window start `H` (§3.3), `C` = main compression factor (spec 04 §9). In Average mode
   also `AVG_START` main (and zoom):
   `AVG_START = (floor(S·N/40) << 10) | ((S − (floor(S·N/40)·40)/N) · N & 0xFF)`
   (start expressed in 40-word = 2-burst blocks plus the word remainder; example `S = 4950`,
   `N = 1` → `0x0001EC1E`).
2. Sinc: `SINC_LEN` = window points + 40 per path; `SINC_RATIO` per path enabled when the sinc
   ratio `r > 1` (coefficients loaded through `SINC_SEL/SINC_WORD`); stage 2 (`0x1138`) enabled
   with ratio 200 when either path interpolates.
3. Linear: `LIN_INC = floor((2^32−1)·num/den)` for the fractional ratio, enabled when ≠ 1.
4. `BWF_CFG` = `0x0C` when the acquisition mode is Average and the BWF FIR is enabled, else
   `0x18`.
5. Per-slot fine delay (spec 02 §3): `SKEW_FINE[k] = trunc((3·dt + a + r + f_k) / dt′) +
   trunc(S_f · s)`, `dt` = dot time (fs), `s` = sinc ratio of the path, `dt′ = dt / s`,
   `a = −floor(dt/2)` when the acquisition mode is High-Res or the BWF FIR is enabled, else 0;
   `r` = PRE/POST residual (spec 02 §6.6, ≤ 0), `f_k` = fine remainder of channel `k` (below),
   `S_f` = fractional part of the window start in memory points (float32 product). Example 1 ch
   2 GSa/s, FIR on, `r = f = S_f = 0`: sinc 10/5/2/1 → 25/12/5/2; FIR off, sinc 1 → 3.
   `SKEW_COARSE` = 0 for every slot (main, and zoom when zoom is on).
6. `DECIM` main/zoom = compression factor (enabled when > 1).
7. `OUT_LEN4 = 1000`, `OUT_LEN0 = ceil8(M)`, `OUT_LEN1 = ceil8(Z)` (spec 02 §3.3).

The per-channel sample delays (`CH_SKEW`) come from the acquisition configuration (§5): the channel
deskew (user + probe delay) plus the interleave phase of the channel's ADC cores (multiples of the
500 ps core phase step relative to the trigger source channel), split into a coarse part (multiple
of 2 ns) written to `CH_SKEW` and a fine remainder used in step 5.

---

## 5. Configuration order

```
configure(state):
    parameter computation for main (and zoom)                 (spec 04 §4–§9)
    if the channel set / impedance changed: ADC and channel-mode setup (channel mode, FIR
        bandwidth filter: BWF bypass or coefficient load, spec 02 §2.10; set choice spec 06 §9.6)
    if state ∈ {Run, Single, Record, Record loop}: acquisition setup (below)
    horizontal setup (§4); display gain/offset (§2); mask setup (FR_GAP = 15 ms)
    if Scan mode: SCAN_TPOS
    sequencer setup (zoom enable, roll/scan bits, auto-trigger timeouts)

acquisition setup:
    SP_MODE.CORE_ORDER = 1; SLOW_PATH
    CH_SKEW; SP_RESET.PRE_RST pulse (bit 5)
    INV (BOARD_ID-dependent polarity); BWL20 (20 MHz limit)
    sample rate (spec 04 §5); SP_MODE mode; AVG_COUNT; PRE_ADJ = 5 (10 in Average)
    RDEC_OFF; REC_LEN; PRE_LEN/POST_LEN; FRAME_BURSTS; DVM source/gate
```

Apply a configuration with the FSM stopped (stop(full) first) and follow it with a run command.
Auto-trigger timeouts (sequencer setup): `m = min(20·tdiv, 500 ms)`, `AUTO_T1 = max(m, 10 ms)`,
`AUTO_T2 = max(m/2, 100 ms, holdoff)` (4 ns ticks), `SEQ_FUNC` bit 0 = 1 in Auto sweep. Factory-image
semantics (spec 05 §10, spec 21 §7.1): timer 1 runs from FPGA configuration and restarts on every
TRG event, accepted or not; once it has expired, each frame auto-triggers `AUTO_T2[27:16]·65536 + R +
1` armed cycles after its trigger enable (`R` pseudo-random per frame).

---

## 6. Slow modes (Roll / Scan)

In Roll and Scan (spec 04 §8) the FSM free-runs and the SP block streams compressed display points
continuously; there are no frames and no play-back.

```
run:      stop(full); free_run(mask)               # SEQ_MODE[3:0] = mask, RUNCTL.RUN = 1 (no SEQ_CMD)
config:   k = 100 · tdiv / roof125(tdiv)   # words per chunk, 100 on a 1-2-5 timebase
          OUT_LEN4 = K = k / N ;  OUT_LEN0 = 1 000 000 ;  DECIM main = 10 · N (enabled)
poll:     OUT_CTRL bit28 pulse (0→1→0)             # latch OUT_LEVEL
          avail_words = OUT_LEVEL (0x11E8) · 4
          chunks = avail_words / k
          if OUT_LEVEL == 0: retry later
          read(c2h, buf, chunks · k · 2)            # may be 0 bytes
          de-interleave k/N points per channel per chunk; append to the roll buffer
Scan:     SCAN_POS bit31 = position valid, [15:0] = position; SCAN_END (0x40D0) ≠ 0 = sweep complete;
          at completion replay the last sweep (Play last) and start a new sweep
```

A roll buffer of 1000 chunks (`1000 · K` points per channel) covers the screen. The FPGA must count
`OUT_LEVEL` in units of 4 words of produced-but-unread stream data and must deliver exactly the
requested byte count. Output rate per channel = `Fs / C` points per second
(**UNVERIFIED** against hardware: measure `OUT_LEVEL` growth over 1 s at 200 ms/div).

Leaving a slow mode: `ROLL = SCAN = 0`, clear the roll buffer, reconfigure normally.

---

## 7. Buffers

- A play-back of `n` frames is read with one `read()` (§2.1) into one contiguous, page-aligned
  buffer; the UltraAcquire frame count (spec 04 §7.2) is limited so that it fits. The limit there
  assumes a buffer of **524 288 000 bytes** (500 MiB).
- The optional CPU-side copy engine `/dev/dma_auklet` (spec 14 §6) offers a 128 MiB source pool;
  de-interleaving otherwise is plain `memcpy`/loops on the CPU. An open firmware MUST derive the
  frame limit from the buffer it actually uses.

---

## 8. Other uses of the C2H path

| Use | Programming | Read size |
|---|---|---|
| Raw memory export (measurement on full record, RMS of 1 Mpts) | `WIN_LEN/START` main = (`512·N + 10^6`, start), `OUT_LEN0 = 1 000 000`, Play of one frame, `FR_A = FR_B = FR_C = base[h]` of the most recently completed half, with `FRAME_BURSTS` of that half; then stop(full) + reconfigure | `N · 2 · 10^6` bytes |
| Memory analysis (bus decode on memory) | window of ≤ 50 M points centred in the record | `N · 2 · pts` bytes |
| Bus-decode digitizer (BDIG) | `BDIG_*` thresholds/sources, `BDIG_LEN = pts/8` | `pts/4` bytes (1-bit samples packed) |
| Waveform import (H2C) | Import command, `write(h2c, …)` | — |

---

## 9. Behaviour a device model / open FPGA must provide

Minimal contract for the acquisition cycle of this spec to run indefinitely:

1. `IDLE` reads 1 when no record or play sequence runs (0 between frames is allowed; §3.6);
   `SEQ_STAT` WAIT/T'D/AUTO per spec 02 §6.5.
2. Record-sequence: frames complete at the trigger rate, `FR_READY` bit30 and
   `FR_A`/`FR_B` read-back per §3.3, no wrap past `FR_C`.
3. Play: `(FR_C − FR_B + 1) · 2 · N · (OUT_LEN0 + zoom·OUT_LEN1)` bytes on C2H; the CPU
   may read less (`2 · N · M` with `M < OUT_LEN0` in the replay of a clipped window, §2.1),
   and the remainder is discarded by the next output-path reset.
4. Roll/scan: `OUT_LEVEL` latch on `OUT_CTRL` bit 28, stream of `N`-interleaved compressed points.
5. ADC SPI bridge read-back with the per-core calibration-done bit (spec 02 §2.6).

The Tier-1 device model implements this contract (spec 16 §4); that shows the contract is
self-consistent, not that the factory FPGA image behaves this way.
