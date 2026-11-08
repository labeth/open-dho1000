# 04 — Timebase and Bands

The time/division setting (`tdiv`, int64 fs/div) together with the active channel set, the memory
depth setting and the acquisition mode determines the sample rate, the record length, the
pre/post-trigger split, the record-memory segmentation (frames) and the display resampling. This
spec gives the rules and the register programming for each. Register names refer to spec 02,
the acquisition cycle to spec 03.

There is no fixed "detent → register value" table: the sample rate is computed from `tdiv` and the
depth, then snapped to a producible rate. The screen is **10 horizontal divisions**. The rules
below give working register values for the full timebase ladder × channel modes × depth settings
(reference values in §10). **Every value here is UNVERIFIED until measured on a unit.**

---

## 1. Product configuration

The unit's identity record (`/dev/vendor_storage`, item "model") holds the model string. When it is
absent, the board's hardware code (`/dev/hdcode_gpio`, spec 13) identifies the variant: `0x2`, `0x3`
→ 4-channel 1000-series board (`HDO1074` class); `0x0`, `0x4` → 4000-series board (`HDO4204`
class); any other code (incl. `0x1`, `0x9`) → 2-channel 1000-series board (`HDO1072` class). The
model strings of the DHO1000 family are `HDO10x2/10x4/1202/1204`.

A DHO1074 has 4 channels and 70 MHz bandwidth (more with a bandwidth licence). Hardware constants
this spec uses:

| Constant | Value | Meaning |
|---|---|---|
| ADC count | 1 | number of ADC chips (lane calibration, ADC SPI select) |
| Lanes per ADC group | 14 | data lanes per ADC group; 2 groups → 28 lanes (spec 20) |
| Core phase step | 500 000 fs | interleave phase step of one ADC core (channel deskew, spec 03 §4) |
| Skew unit | 500 000 fs | base unit (1 channel) of `CH_SKEW`; ×2 / ×4 in 2 / 3–4-channel mode (spec 02 §2) |
| FPGA clock period | 4 000 000 fs | 4 ns acquisition clock (time registers) |
| Samples per burst | 20 | samples per record-memory burst (one 32-byte DDR3 burst, §7) |
| Record memory | 2^25 bursts | 2^25 bursts × 32 B = 1 GiB |
| Average segment | 64 bursts | average-mode segment granularity |
| Average reserve | 10 485 760 bursts | record memory kept free in Average mode (§7.2) |
| AUTO depth cap | 1 000 000 | largest AUTO depth (§4) |

The S2 image has 28 lanes and a 32-bit DDR3 (spec 12/20).

## 2. Timebase ladder

| Item | Value |
|---|---|
| Coarse steps | 1-2-5 sequence (`…, 1, 2, 5, 10, 20, 50, …` × 10^n fs) |
| Fine (vernier) step | `max(roof125(tdiv) / 100, 50 000 fs)`; a fine value snaps to `floor125(tdiv) + k·step` |
| Maximum | 1 000 s/div (`1e18` fs) |
| Minimum | depends on the bandwidth (table) |

| Bandwidth | Minimum tdiv |
|---|---|
| 70, 100 MHz | 5 ns |
| 200 MHz | 2 ns |

A non-1-2-5 `tdiv` (fine mode) is rounded **up** to the next 1-2-5 value for all acquisition
arithmetic (`tdiv_acq = roof125(tdiv)`); the display path uses the exact value.

## 3. Channel modes and maximum sample rate

One ADC with 8 interleaved cores serves all four channels. The number of ADC "groups" per channel
depends only on how many analog channels are acquired (the active-channel mask, bit n = CHn+1):

| Active channels | Groups | Max sample rate per channel (`Fmax`) | Sample interval | Max depth |
|---|---|---|---|---|
| 0 or 1 (any single channel) | 1 | 2 GSa/s | 500 ps | 50 M |
| any 2 | 2 | 1 GSa/s | 1 ns | 25 M |
| 3 or 4 | 4 | 500 MSa/s | 2 ns | 10 M |

A channel used only as trigger source still counts as active (spec 05 §3.1). The mask is written to
`SEQ_MODE[3:0]` by every run command. "Words per sample instant" `N` (spec 03 §2) is 1, 2, 4 for
the three rows. A depth setting above the row's maximum is clamped to the maximum.

### 3.1 Lane routing for a channel mode

The SP block has four channel **lanes** (0–3) behind the ADC. Per-channel properties are written as
a 4-bit per-lane mask built from the four per-channel flags `f1..f4`:

| Channel mode | Lane 0 | Lane 1 | Lane 2 | Lane 3 |
|---|---|---|---|---|
| 1 channel (CHk) | `fk` | `fk` | `fk` | `fk` |
| 2 channels (CHa < CHb) | `fa` | `fa` | `fb` | `fb` |
| 4 channels | `f1` | `f2` | `f3` | `f4` |

Used for `CAP_CTRL.LANE_EN`, `SP_MODE.INV` and `SP_MODE.BWL20` (example: CH1 alone with 20 MHz
limit → `BWL20 = 0xF`). A channel-mode change also programs `LANE_GAIN` (`0x105C`, spec 02 §2.8:
`0x00005555` in every mode when all codes are 1, i.e. ×2.5 about mid-scale on every lane in the
factory image), `LANE_TRIM` from the calibration (spec 10) and the ADC chip input mode (spec 06).

Record channel packing (factory image, spec 21 §8.2; word alignment off, `0x1020` = 0). The 8
samples of a 4 ns clock are `t` = 0…7 = `g0.Q4, g1.Q4, g0.Q3, g1.Q3, g0.Q2, g1.Q2, g0.Q1, g1.Q1`
(spec 20 §3.2). 1 channel: record positions in time order. 2 channels: even record positions =
group 0 = the lower-numbered active channel, odd = group 1 = the higher. 3–4 channels: record
position `j` holds CH`(j mod 4) + 1`, with CH1 = `t` 0/4, CH2 = 2/6, CH3 = 1/5, CH4 = 3/7 — the same
assignment as the TRG slots (spec 05 §1, spec 21 §2.1). The play-back then emits the slot order of
spec 03 §2. With the operating word alignment (`0x1020 = 0x1010`) the factory image records `t` =
6, 7, 2, 3, 4, 5, 0, 1 per clock; which ISERDES framing the real ADC delivers at rotation 0, and so
the record order on a unit, is **UNVERIFIED**. Which ADC cores drive which (group, Q) position is
set by the ADC output mode (spec 06).

## 4. Sample-rate and depth selection

Inputs: `tdiv` (§2, rounded up), `Fmax` (§3), the depth setting `Dset`, acquisition mode.

Memory-depth settings (points per channel): AUTO, 1 k, 10 k, 100 k, 1 M, 10 M, 25 M, 50 M (larger
values are clamped to the channel mode's maximum, §3).

```
T10 = 10 · tdiv                                # record covers the 10 screen divisions

if Dset != AUTO:
    D  = Dset
    Fs = min(D · 1e15 / T10, Fmax)
else:
    D  = T10 · Fmax / 1e15
    if D > 1 000 000:  D = 1 000 000;  Fs = D · 1e15 / T10
    else:              Fs = Fmax
    D += 16                                     # guard points

if Fs is one of the non-producible rates (table): Fs = replacement
    and, for AUTO only, D = T10 · Fs / 1e15     # (no guard points in this case)
dot_time = 1e15 / Fs                            # fs per sample
```

All divisions are exact 64-bit integer `A·B/C` with GCD reduction; for `tdiv ≥ 500 s/div` form the
product as `tdiv · Fs / 1e15 · 10` to avoid overflow.

| Requested Fs | Used |
|---|---|
| 20 MSa/s | 10 MSa/s |
| 100 MSa/s | 50 MSa/s |
| 200 MSa/s | 125 MSa/s |
| 400 MSa/s | 250 MSa/s |
| 625 MSa/s | 500 MSa/s |
| 1.25 GSa/s | 1 GSa/s |
| 2.5 GSa/s | 2 GSa/s |

Other rates are used as computed. Producible rates are `Fmax / k` for `k ∈ {1, 2, 4, 8}` while
`> 250 MSa/s` (interleave selection) and `250 MSa/s / n` for integer `n ≥ 1` (decimation); the
1-2-5 ladder × depth table only ever produces values in these two sets after the substitution.

## 5. FPGA sample-rate programming

```
if Fs > 250 000 000:
    RATE_SEL (0x10D0) = 0x8000 | ((Fmax / Fs) & 0xFF)
    RATE_DIV (0x10D8) = 1
else:
    RATE_SEL (0x10D0) = 0
    RATE_DIV (0x10D8) = 250 000 000 / Fs
RATE_PHASE (0x10D4) = 0
REC_LEN (0x10E4) = D
PRE_LEN / POST_LEN  (0x4050 / 0x4054)  sample counts per §6
```

Examples: 1 ch 2 GSa/s → `0x8001/1`; 1 ch 500 MSa/s → `0x8004/1`; 2 ch 1 GSa/s → `0x8001/1`;
125 MSa/s → `0/2`; 10 MSa/s → `0/25`; 100 Sa/s → `0/2 500 000`.

### 5.1 Acquisition modes

| Mode | `SP_MODE` bits | Other registers |
|---|---|---|
| Normal | — | — |
| Peak | `PEAK` (bit 1) — written as Normal when `Fs == Fmax` | — |
| Average | `AVG` (bit 3) | `AVG_COUNT = count − 1` (count 2…8192 in powers of two; 1 in the other modes), `PRE_ADJ = 10`, `AVG_START` (spec 03 §4), `BWF_CFG = 0x0C` while the BWF FIR is enabled (else `0x18`), record memory half 0 only (spec 03 §3.3) |
| High-Res | — | a High-Res FIR coefficient set is written through `BWF_COEF` (spec 02 §2.10); no mode bit. An open FPGA may implement High-Res as its own boxcar/FIR |
| UltraAcquire | as Normal | frames per §7.2, `SEQ_TIMEOUT` = UltraAcquire timeout |

In every mode `SP_MODE.SLOW_PATH` (bit 2) `= (Fs < Fmax / 8)`. The bandwidth FIR (`BWF`) is
either bypassed (`BWF_MODE = 1`) or loaded with a coefficient set chosen by bandwidth and sample
rate (spec 06 §9.6; load sequence and per-channel-mode coefficient layout spec 02 §2.10); its state
is not tied to `SLOW_PATH` (loads also occur at decimated rates). `PRE_ADJ` (`0x404C`) = 5, 10 in
Average (no effect on the trigger position in Normal mode in the factory image, Average and Scan
**UNVERIFIED**, spec 21 §8.3; an open FPGA MAY ignore it, spec 02 §6). Decimation
(`RATE_DIV > 1`) keeps one sample per `RATE_DIV` (Normal) or a min/max pair per bin (Peak; bin
width, pair order and the effect on the stored depth **UNVERIFIED**, spec 21 §10).

## 6. Horizontal position and memory window

The trigger offset `t0` (fs, the horizontal position, positive = trigger moved left) splits the
record:

```
mem_start = t0 − D · dot_time / 2              # time of the first sample relative to the trigger
mem_end   = mem_start + D · dot_time
ext   = EXT-trigger latency (fs): 119 ns (1 ch), 400 ns (2 ch), 401 ns (3–4 ch) when the trigger
        source is EXT, else 0
post  = max(ceil((mem_end + ext) / dot_time), 1)     # not clamped to D
pre   = max(D − post, 0)
```

(For `tdiv = 1000 s/div` the window is `t0 − dot_time · D/2 … t0`.) Offset range while running:
`t0 ≥ −5·tdiv`; upper limit `clamp(100·tdiv, 1 s, 1000 s)` for `tdiv < 10 s/div`, else
`max(5·tdiv, 1000 s)` (minus `5·tdiv_zoom` with zoom on). Within it `mem_end ≥ 0`; replace
`mem_end = 0` by 1 fs, so `post ≥ 1`. A `mem_start < −D·dot_time` leaves PRE/POST unchanged.
Examples (1 µs/div, 10 k, 1 GSa/s): `t0 = 0` → 5000/5000; `+2 µs` → 3000/7000; `−2 µs` →
7000/3000; `−20 µs` (applied as `−5 µs`) → 9999/1; `+20 µs` → 0/25000 (POST > D: the trigger lies
before the record); EXT source at 2 GSa/s adds 238 samples to POST (119 ns).

The **screen window** is the time range `[t0 − 5·tdiv, t0 + 5·tdiv]`; in memory points it is
`P = 10 · tdiv · Fs / 1e15` points starting at `S = (mem_start_of_screen − mem_start) / dot_time`
(`WIN_START`). With `t0 = 0` and the record centred, `S = (D − P) / 2`.

## 7. Record memory and frames

The record memory (1 GiB DDR3 on the S2 board) is addressed in **bursts**: one burst is 32 bytes
and holds 20 samples of 12 bits (240 of the 256 bits used; packing order inside the burst is
FPGA-internal and invisible to the CPU). It is split into two **ping-pong halves** used alternately
by the segmented recorder (spec 03 §3.3); each half is divided into equal **frames**.

### 7.1 Frame size (`FRAME_BURSTS`, `0x10E0`)

```
n = D · nch                                      # samples per frame, all acquired channels
if mode == Average:
    s = ((n + 0x17F) / (64 · 20) + 2) · 64 · 2
else:
    s = (n + 0x180) / 20                         # bursts, integer division
s = max(s, 32);  s = next power of two ≥ s;  s = min(s, 16 252 928)   # (2^25 − 2^20)/2; never reached
FRAME_BURSTS = s                                 # bursts per frame
```

(`nch` here = 1, 2, 4 words, §3.) Examples: 1 ch 10 k → 1024; 1 ch 50 M → 4 194 304; 4 ch 1 k →
256; 4 ch 10 M → 2 097 152.

### 7.2 Frames per half

```
frames_hw = 15 728 640 / FRAME_BURSTS            # = (2^25/2 − 2^20) / FRAME_BURSTS, ≥ 1
```

Frame indices (`FR_A/FR_B/FR_C`, spec 02 §6.3) count in units of `FRAME_BURSTS` bursts.
Half 0 is frames `[0, F−1]`; **half 1 starts at frame index `frames_hw`** (byte address
`frames_hw · FRAME_BURSTS · 32`). In Run/Single (ping-pong halves) the data of both halves ends
below burst `0x1E00000` and bursts `0x1E00000–0x1EFFFFF` hold no data. Burst `0x1F00000 + f` holds
the one-burst header of frame `f` (spec 21 §8). The FPGA has no notion of halves: frame `f` of half
1 is addressed exactly like any other frame (e.g. `FRAME_BURSTS` = 64, `f = frames_hw = 0x3C000`:
data from burst `0xF00000`, header at `0x1F3C000`), and play-back reads wrap inside the frame's
`FRAME_BURSTS` bursts (spec 03 §3.3). The waveform recorder may use up to
`(2^25 − 2^20) / FRAME_BURSTS` frames from frame 0 (Average: `(2^25 − 10 485 760 − 2^20) /
FRAME_BURSTS`); its data then extends up to burst `0x1EFFFFF`.

Frames per half `F` (Normal / Peak / Average / High-Res) is a refresh-rate choice: any `F` in
`[1, frames_hw]` works, and `F = 1` in Single. A useful budget keeps record plus play-back time near
40 ms per screen update:

```
P      = screen window in samples (§6), dt = dot_time (fs), K = interpolation factor of the main path (§9; 1 if none)
f_time = 4e13 / (D · dt · nch + max(K,1) · 4e6 · nch · (T10 / dt))     # ≥ 1 ; 4e6 → 4e7 for tdiv ≥ 500 s/div
F      = max(min(f_time, frames_hw), 1)
```

and keeps the play-back of `F` frames (`F · 2 · N · (M + Z)` bytes, spec 03 §2.1) small enough for
the read buffer and the display. UltraAcquire: `F ≤ min(frames_hw, 524 288 000 / (D · nch))`
(spec 03 §7).

## 8. Slow modes: Roll and Scan

| Condition (YT time mode, acquisition mode ≠ Average, no feature active that needs frames) | Mode |
|---|---|
| Auto-Roll on and `tdiv ≥ 50 ms/div` | ROLL |
| Auto-Roll off and `tdiv ≥ 200 ms/div` | SCAN |
| otherwise | normal (triggered frames) |

Programming (both): `D = 1 000 000` (no guard points), Fs by §4, `OUT_LEN0 = 1 000 000`,
`DECIM main = 0x80000000 | 0x40000000 | (10 · N)` (N = 1, 2, 4),
`OUT_LEN4 = K = (100 · tdiv / roof125(tdiv)) / N` points per chunk (100 / N on a 1-2-5 timebase);
data stream per spec 03 §6.

- ROLL: `SEQ_MODE.ROLL` (bit 17) = 1, `SCAN` = 0; `PRE_LEN = D − 1`, `POST_LEN = 1`; auto-trigger
  timeouts `min(20·tdiv, 500 ms)` / half of it; the FSM free-runs and the display scrolls.
- SCAN: `SEQ_MODE.SCAN` (bit 22) = 1, `ROLL` = 0; `PRE_LEN = POST_LEN = D/2`; `SCAN_TPOS`
  (`0x4058`) = trigger position; `SCAN_END` / `SCAN_POS` report the sweep.
- Leaving a slow mode: `ROLL = SCAN = 0`, normal configuration (spec 03 §5).

## 9. Display resampling (main and zoom paths)

Each path turns its window of `P` memory points (§6; zoom: the zoom window) into `M` output points
per channel (`OUT_LEN0`, zoom `OUT_LEN1`):

```
P_c = memory points of the window that lie inside the record   # P_c = P when the window is inside it
if P ≤ 1 000 000:
    M = max(1000, 1000 · ceil(P / 1000)) · P_c / P   # output = window, rounded up to 1000-point blocks,
                                                     # scaled down when the window is clipped
    C = 1
    if P_c < M:                                  # fewer memory points than output points
        r = M / P_c   (fraction)
        sinc = largest s in {200,160,100,80,50,40,32,25,20,16,10,8,5,4,2,1} with s ≤ r
        if Fs < 200 MSa/s or (Peak and Fs != Fmax): sinc = 1
        linear ratio = (P_c · sinc) / M          # ≤ 1
else:                                            # compression
    q = P / 1000                                 # memory points per output block
    C = smallest divisor of q with q / C ≤ 1000
    M = P_c / C
```

A window is clipped when it extends beyond the recorded data: e.g. in the first replay after a
change to a slower timebase (the stored record was taken with the old setting), or after a
horizontal offset change while stopped. Clipped examples (1 ch, 2 GSa/s unless noted):
`P_c`/`P` = 116/200 → M 580, sinc 5; 216/400 → M 540, sinc 2, linear 0.8; 416/1000 → M 416;
1016/2000 → M 1016; 4 ch `P_c`/`P` = 1016/2500 → M 1219; compression 1 000 016 / 2.5 M → C 4,
M 250 004. A clipped `M` need not be a multiple of 8 (§9 `OUT_LEN0 = ceil8(M)`, spec 03 §2.1).
Truncate a non-integer `M` and compute the linear ratio from the unrounded `M` (**UNVERIFIED**
beyond these cases).

Examples (1 ch unless noted): 5 ns/div AUTO (P = 100) → sinc 10, M 1000; 20 ns/div
(P = 400) → sinc 2, linear 800/1000; 500 ns/div 1 k (Fs 125 M, P = 625) → sinc 1, linear
625/1000; 5 µs/div 10 k (P = 6250) → M = 7000, linear 6250/7000; 4 ch 500 ns/div 10 k (P = 2500)
→ M 3000; 500 µs/div 10 M (P = 10 M) → C 10, M 1 000 000; 5 ms/div 10 M (Fs 125 M, P = 6.25 M) →
C 10, M 625 000; 4 ch 500 µs/div 10 M (P = 2.5 M) → C 4, M 625 000; 50 ms/div 10 M (P = 5 M) →
C 5.

The sinc ratio selects prototype set `h_r` (spec 02 §3.5.1). Main and zoom choose their ratios
independently and each loads its own copy; stage 2 always uses `h_200`. The timebase ladder alone
gives r ∈ {2, 4, 5, 10, 20, 40} (5 ns/div: 1 ch → 10, 2 ch → 20, 3–4 ch → 40); larger ratios, up
to 200, occur only when the window has very few points (a small zoom window, or a window clipped by
a large horizontal offset).

Registers per path (main / zoom):

| Register | Value |
|---|---|
| `WIN_START` | `S` (first memory point of the window, relative to the frame's window start `H`, spec 21 §9) |
| `WIN_LEN` | main `P_c + 512·N + 72 + C`; zoom `P_c + 512·N + 72` |
| `SINC_LEN` | `P_c + 40` |
| `SINC_RATIO` (main `0x1130`, zoom `0x1134`) | `(sinc > 1) << 31 | sinc << 16 | sinc << 8 | sinc` (`0x00010101` = ratio 1; zoom off: `0x00000000`) |
| `SINC_RATIO2` (`0x1138`, shared) | `0x00C8C8C8`, bit 31 set when either path has `sinc > 1` |
| `LIN_OFF` / `LIN_INC` | linear enabled when ratio ≠ 1: `LIN_INC = floor((2^32 − 1) · num / den)`; disabled → `0xFFFFFFFF` (spec 02 §3.1) |
| `DECIM` | `0x40000000 | (C > 1) << 31 | C` (output word format with `C > 1`: spec 02 §3.2) |
| `OUT_LEN0` / `OUT_LEN1` | `ceil8(M)` / `ceil8(M_zoom)`; OUT_LEN1 may keep its last value when zoom is off, and is not written in Roll/Scan (spec 02 §3.3) |
| `OUT_LEN4` | 1000 (triggered modes) |

The display record of a frame therefore has at most 1 000 000 points per channel, and at least 1000
unless the window is clipped by the record; it is not a fixed 1000: the renderer (spec 07)
decimates further. In Roll/Scan the window is `10^6` points and `C = 10·N` (§8).

## 10. Reference values

Values the rules above give for AUTO depth, 1 channel:

| tdiv | Fs | D | FRAME_BURSTS | M | RATE_SEL/RATE_DIV |
|---|---|---|---|---|---|
| 5 ns | 2 G | 116 | 32 | 1000 (sinc 10) | 0x8001/1 |
| 50 ns | 2 G | 1 016 | 128 | 1000 | 0x8001/1 |
| 500 ns | 2 G | 10 016 | 1024 | 10 000 | 0x8001/1 |
| 5 µs | 2 G | 100 016 | 8192 | 100 000 | 0x8001/1 |
| 50 µs | 2 G | 1 000 016 | 65 536 | 1 000 000 | 0x8001/1 |
| 100 µs | 1 G | 1 000 016 | 65 536 | 1 000 000 | 0x8002/1 |
| 200 µs | 500 M | 1 000 016 | 65 536 | 1 000 000 | 0x8004/1 |
| 500 µs | 125 M | 625 000 | 32 768 | 625 000 | 0/2 |
| 1 ms | 50 M | 500 000 | 32 768 | 500 000 | 0/5 |
| 5 ms | 10 M | 500 000 | 32 768 | 500 000 | 0/25 |
| 100 ms | 1 M | 1 000 016 | 65 536 | 1 000 000 | 0/250 |
| 200 ms (scan) | 500 k | 1 000 000 | 65 536 | 1 000 000, C 10 | 0/500 |
| 1 s (scan) | 100 k | 1 000 000 | 65 536 | 1 000 000, C 10 | 0/2500 |
