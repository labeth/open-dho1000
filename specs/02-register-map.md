# 02 — Register Map

The CPU talks to the FPGA through one PCIe BAR (the XDMA "bypass" BAR) as a flat array of 32-bit
registers, plus two XDMA streaming engines for bulk sample data (spec 03, spec 14). Every
acquisition, trigger, timebase, calibration and analog action that reaches the FPGA reduces to
32-bit reads and writes of the registers below. This document is the register reference.

**Every value, bit position and behaviour in this spec is UNVERIFIED until measured on a unit.**
"Factory image" means the FPGA image installed on the unit (spec 13); statements about it describe
the hardware the firmware drives when it keeps that image.

Register and block names are ours. The register file has three blocks: **SP** (sample path: ADC
capture, DDR record memory, display decimation/interpolation, sample output), **TRG** (trigger,
incl. frequency counter and DVM), **SEQ** (sequencer: the run/stop/record FSM, auto-trigger,
timestamps). "Shadow" means the driver keeps a copy of the register in RAM and always performs
read-modify-write on the copy, never on the hardware.

---

## 1. Bus & addressing model

### 1.1 The register window

| Item | Value |
|---|---|
| Device node | `/dev/xdma0_bypass` |
| open flags | `O_RDWR \| O_SYNC` (`0x101002`) |
| mmap | `mmap(NULL, 0x8000, PROT_READ\|PROT_WRITE, MAP_SHARED, fd, 0)` — one 32 KiB window, offset 0 |
| Access width | exactly one aligned 32-bit load or store per register access |
| Addressing | `reg_addr = base + offset`; offsets are byte offsets, always multiples of 4 |
| Highest offset used | `0x40D0` |

Rules the implementation MUST follow:

- Open and map once, keep the fd and mapping for the life of the process. `read()`/`write()` on
  this node fail (`EINVAL`); only mmap works (spec 14).
- Serialize every register access behind one process-wide lock. Several registers are
  "select-then-access" pairs (§1.3); an interleaved access from another thread corrupts them.
- Use volatile 32-bit accesses. Never let the compiler split, merge or elide an access: several
  registers are strobes or FIFO-like ports where the number of accesses matters.
- Treat the map as uncached device memory.

### 1.2 Address blocks

| Range | Block | Contents |
|---|---|---|
| `0x1000`–`0x10FF` | SP core | reset/control, acquisition mode, ADC data-capture alignment, ADC SPI bridge, ADC gain, sample rate, memory size, record depth |
| `0x1100`–`0x13FF` | SP display path | main/zoom read window, sinc & linear interpolation, compression, mask test, C2H output control, logic-analyzer (MSO) path |
| `0x1400`–`0x14BF` | SP board I/O | ext-trigger level, probe offset DACs, start-up constants `0x1418`–`0x1424`, probe power & 1-wire, aux out, XADC, DDR PHY calibration taps |
| `0x14C0`–`0x155F` | SP post-processing | per-channel digital gain/offset (main & zoom), frequency response (Bode), bus-decode digitizer |
| `0x2000`–`0x21FF` | TRG | trigger levels, three event engines (A, B, R), trigger select/mode, holdoff, delay, protocol triggers |
| `0x2200`–`0x226F` | TRG aux | frequency counter, DVM, external-trigger status |
| `0x4000`–`0x40DF` | SEQ | version/ID, run control, mode, record/play control, auto trigger, pre/post trigger, timestamps |

Offsets not listed in this document have no known function; leave them alone.

Address decoding in the factory image (spec 21 §1.1): the SP block decodes its offsets fully; the
TRG block decodes only address bits A2–A9, so `0x2000`–`0x23FF` repeats at `0x2400`, `0x2800` and
`0x2C00`; the SEQ block decodes only the bits needed to separate its implemented registers, so other
SEQ offsets alias implemented ones. A driver MUST use only the offsets listed here (aliases MUST NOT
be relied on).

### 1.2.1 Stored bits

Only the bits listed in spec 21 §1.2 exist as storage in the factory image; other bits of a write are
ignored. The tables below describe what a driver **writes**; where a written bit is not stored in
the factory image this is noted ("not stored"). TRG configuration registers read back the last
written value (stored bits only); a driver keeps shadows and does not need this.

Read-back in the factory image (spec 21 §1.3): bits that the addressed register does not drive read
0 (each block's read register is cleared in every cycle without a read); SEQ reads have no side
effects. `FR_B`, `FR_C`, `PRE_LEN`, `POST_LEN`, `AUTO_T1/2`, `SEQ_TIMEOUT`, `FR_GAP` and the
`0x4058` store read back as written; `RUNCTL` reads bits 0, 1, 30, 31 as written plus internal
status bits (a reader MUST mask them); `FR_A`, `SEQ_STAT` and `FR_READY` return status (§6, §6.5).

### 1.3 Access patterns

Five patterns recur. Each register entry below names the pattern it uses.

- **Shadowed RMW** — the driver holds a 32-bit copy, updates a field in the copy, writes the whole
  copy. Initial shadow value is 0 unless stated.
- **Pulse** — set a bit in the shadow and write, then clear it and write (`0→1→0` for a
  three-write pulse where stated). The FPGA acts on the edge; the bit reads back as written.
- **Load strobe** — set a "load" bit, write the parameter registers, clear the load bit. A compatible
  FPGA latches the parameters when the load bit falls.
- **Select-then-write** — a field in the same word selects a channel/lane/bank; the value lands in
  the selected target(s).
- **Read-into-shadow** — for `STAT_CTRL` (`0x1040`), `PRB_WIRE1–4` (`0x1444`–`0x1450`), `EXT_STATE`
  (`0x2264`), `SEQ_STAT` (`0x4034`), `FR_READY` (`0x4080`) and `TSTAMP_CTL` (`0x40B8`) the driver
  loads the value it reads into its write shadow, so the next write carries the read-back bits. For
  `0x1040` an FPGA MUST read back `[4:0]`, `[11:8]`, bits 24–27 and bit 31 as last written (bit 30 =
  done) and MUST ignore bit 30 on writes. For `0x40B8` it MUST read bits 0 and 2 as 0 (they are
  pulses) and MUST ignore bit 1 on writes. For `0x1444`–`0x1450` it MUST read bit 8 = byte ready and
  `[7:0]` = received byte, MUST read the pulse bits 31, 25 and 24 as 0, and MUST ignore `[8:0]` on
  writes. For `0x2264` it MUST ignore `[3:0]` on writes. `0x4034` and `0x4080`: §6, §6.5.

### 1.4 Value units

Driver-side times are signed 64-bit **femtoseconds** (1 ns = 1 000 000 fs). The FPGA counts in two
time bases:

| Tick | Period | Conversion from fs |
|---|---|---|
| FPGA clock | 4 ns (250 MHz) | `ticks = t_fs / 4 000 000` (truncate toward zero) |
| Trigger point tick | 1 ns | `ticks = t_fs / 1 000 000` (the TRG block has 4 slots per channel per 4 ns clock, spec 21 §2.1; slot content per channel mode: spec 05 §1) |

Sample time at full rate is 500 000 fs (2 GSa/s, one ADC, 8 interleaved cores). Wider than 32-bit
time counts are split into a low 32-bit word and a high 16-bit field as stated per register.

---

## 2. SP core (`0x1000`–`0x10FF`)

| Offset | Name | Access | Format | Purpose |
|---|---|---|---|---|
| `0x1000` | SP_RESET | W, shadowed | bitfield §2.1 | Block resets and strobes |
| `0x1004` | SP_MODE | RW, shadowed | bitfield §2.2 | Acquisition mode, channel invert/filter, ADC core order |
| `0x1008` | RM_STATUS | RW | bitfield §2.3 | Record memory (DDR/BRAM) control and status |
| `0x1010` | OUT_CTRL | W, shadowed | bitfield §2.4 | C2H output framing control |
| `0x1014` | CAP_CTRL | W, shadowed | bitfield §2.5 | ADC data-capture control |
| `0x1018` | CAP_TAP | W | §2.5 | Per-lane input-delay tap load |
| `0x101C` | CAP_SLIP | W | §2.5 | Per-lane bit-slip; not needed (§2.5) |
| `0x1020` | CAP_ROTATE | W, shadowed | bit 4 / bit 12 | Core-alignment enable for ADC group 0 / 1 |
| `0x1024`, `0x1028` | CAP_FLAGS | R | 32-bit | Per-lane sticky transition flags for the lane search (spec 20 §5.2); meaningful only with the constant training pattern `0x650F` |
| `0x102C` | CAP_SLIP_ST | R | 32-bit | Bit-slip search status; not needed (§2.5) |
| `0x1030` | CAP_ROT_ST | R | 32-bit | Core-alignment search status |
| `0x1034` | ADCREG_CMD | W | §2.6 | SPI bridge to the ADC chip |
| `0x1038` | ADCREG_DATA | R | `[15:0]` | SPI bridge read-back |
| `0x1040` | STAT_CTRL | RW, shadowed | §2.7 | On-FPGA ADC statistics (average, peak) control/status |
| `0x1044` | STAT_MEAN | R | 32-bit | Average result |
| `0x1048` | STAT_MINMAX | R | 32-bit | Peak result |
| `0x1058` | LANE_TRIM | W, shadowed | 16 × 2-bit | Fine digital gain step per core |
| `0x105C` | LANE_GAIN | W, shadowed | 8 × 2-bit in `[15:0]` | Coarse digital gain per ADC lane (§2.8) |
| `0x1080`, `0x1084` | CH_SKEW | W, shadowed | 2 × 16-bit each | Per-channel sample delay `v = delay / U + B`; `U` = sample interval of the channel mode (500 ps 1 ch, 1 ns 2 ch, 2 ns 3–4 ch), `B` = 480 / 240 / 120 (a constant 240 ns). 3–4 channels: `0x1080 = v(CH1) << 16 \| v(CH2)`, `0x1084 = v(CH3) << 16 \| v(CH4)`. 2 channels: `0x1080 = v(lower) << 16 \| v(higher)`. 1 channel: `0x1080[31:16] = v(active channel)` only. Unwritten halves keep their previous value; write both words every time |
| `0x1098` | BWF_MODE | RW, shadowed | §2.10 | Bandwidth-limit FIR control: `1` = bypass, `2` = coefficient load, `0` = filter on |
| `0x109C` | BWF_COEF | W | `bank[31:30] \| tap[27:23] \| coef[22:0]` | FIR coefficient write port (§2.10) |
| `0x10A0` | BWF_CFG | W | `0x18` / `0x0C` | FIR configuration word: `0x0C` when the acquisition mode is Average **and** the BWF FIR is enabled (coefficients loaded, `BWF_MODE = 0`), else `0x18`. Write it in every horizontal configuration and in every BWF load/bypass |
| `0x10B0` | AVG_COUNT | RW | W: `max(count − 1, 1)` | Hardware averaging count (counts 1 and 2 both give 1); write it in every acquisition configuration, also outside Average (then 1). Read returns averaging progress |
| `0x10B4` | AVG_RESTART | W | pulse `1` then `0` | Restart hardware averaging |
| `0x10D0` | RATE_SEL | W | §2.9 | Sample-rate mode / interleave |
| `0x10D4` | RATE_PHASE | W | `0` | Always write 0 |
| `0x10D8` | RATE_DIV | W | §2.9 | Decimation factor |
| `0x10DC` | RDEC_OFF | W | `[15:0]` | Write `1` in every acquisition configuration while random decimation (anti-aliasing) is **off**; leave it unwritten while it is on. How the hardware returns to the "on" state is **UNVERIFIED** (possibly on `SP_RESET.SP_RST`, which every stop issues) |
| `0x10E0` | FRAME_BURSTS | W | 32-bit | Frame size in record-memory bursts (32 bytes = 20 samples each; spec 04 §7.1). Frame indices in `0x4010`–`0x4018` count in units of this size |
| `0x10E4` | REC_LEN | W | 32-bit | Record depth in samples per channel |
| `0x10F0` | IMP_STAT | R | 32-bit | Status of an H2C waveform import |
| `0x10FC` | IMP_HDR1 | W | 32-bit | Frame-header word `[63:32]` for frames recorded with `SEQ_CMD[18]` (import, spec 21 §8.3); how an import supplies it is **UNVERIFIED** |

### 2.1 `0x1000` SP_RESET

| Bit | Name | Use |
|---|---|---|
| 0 | SP_RST | Pulse (1→0): reset the SP datapath. Issue on every stop |
| 1 | CAP_RESYNC | Three-write pulse (0→1→0): re-synchronise the ADC capture |
| 2 | RM_RST_A | Level: record-memory reset A |
| 3 | RM_RST_B | Level: record-memory reset B |
| 4 | OUT_RST | Three-write pulse: reset the C2H output path. Issue on every stop |
| 5 | PRE_RST | Pulse: reset the pre-processing (gain/offset/filter) stage |

Bring-up writes: bit0 1→0, then bit4 0→1→0.

### 2.2 `0x1004` SP_MODE

| Bits | Name | Encoding |
|---|---|---|
| 0 | ORDER_CAL | Set only during the ADC core-order calibration (spec 10); 0 in normal operation |
| 1 | PEAK | 1 = peak-detect decimation (min/max per decimation bin) |
| 2 | SLOW_PATH | 1 = sample rate below `Fmax/8` (averaging/decimation uses the sinc pre-filter path) |
| 3 | AVG | 1 = hardware averaging (count in `AVG_COUNT`) |
| 12 | (cleared) | Clear it whenever the acquisition mode is written |
| 19:16 | INV | Per-channel invert, bit `16+n` = channel `n+1` |
| 23:20 | BWL20 | Per-channel filter enable, bit `20+n` = channel `n+1` |
| 26 | CORE_ORDER | ADC core order select |
| 27 | SINGLE_CORE | 1 = take data from a single ADC core (calibration) |
| 31:28 | CORE_IDX | Core index for SINGLE_CORE |

Acquisition mode: Peak → bit 1, Average → bit 3, Normal, High-Res and UltraAcquire → neither. Write
Peak as Normal when the sample rate equals the maximum (nothing to decimate). BWL20 is the
per-channel **20 MHz bandwidth limit** (digital filter in the SP block). INV is the per-channel
invert; on boards reporting `BOARD_ID == 0xF` write the invert bits inverted (that front end has
opposite polarity). Both are written as a 4-bit per-*physical-lane* mask: in 1-channel and 2-channel
interleave modes the requested channel's bit is replicated to every lane that carries it (spec 04
§3).

### 2.3 `0x1008` RM_STATUS

| Bit | Dir | Meaning |
|---|---|---|
| 31 | W | 1 = route the record into on-chip BRAM instead of DDR. Not needed; write 0 (an FPGA may treat it as reserved) |
| 3:0 | R | Memory controller state |
| 28 | R | DDR status, used by the DDR calibration (spec 10) |
| 29 | R | DDR initialisation passed |

Treat bit 29 = 1 as "DDR initialised". **UNVERIFIED** which of bits 28/29 means "done" and which
"pass"; an FPGA MUST set bit 29 after a successful DDR initialisation and SHOULD set bit 28.

### 2.4 `0x1010` OUT_CTRL

| Bit | Meaning |
|---|---|
| 31 | NO_INFO: write 1 once at start-up (`0x1010 = 0xA0000000`) and keep it. 1 = no frame-information words in the C2H stream (the stream the CPU parses, spec 03 §2.1); 0 = the factory image inserts two 64-bit words (16 bytes) of frame information before the samples, in stream order as u16: `0xFA05, 0, 0x007B \| Peak << 12 \| Average << 13, 0, 0xC000, 0, 0, 0` (Peak = `SP_MODE` bit 1, Average = bit 3), unchanged by the time stamp, window, sinc ratio and compression; the number of sample words still follows `OUT_LEN0`. The other fields and whether the words repeat per frame of a multi-frame play are **UNVERIFIED**; an open image MAY omit the words (spec 21 §9.1) |
| 30 | Lane order: 0 = each 8-byte beat carries output slots 0, 1, 2, 3 in stream order (spec 03 §2); 1 = the four 16-bit words of every beat reversed. Write 0. An FPGA SHOULD implement it or keep the bit-0 order |
| 29 | NO_TLAST: write 1 once at start-up and keep it. Factory image: `TLAST = last_word & tvalid & tready & ~NO_TLAST` on the C2H AXI-Stream, so with 1 no packet end reaches the XDMA engine (spec 14 §4.2); the same gated strobe clocks a capture of the last word's high bits |
| 28 | OUT_SNAP: three-write pulse; latches the transmitted-sample counter into `0x11E8` |

### 2.5 ADC data capture (`0x1014`–`0x1030`)

Hardware meaning of every field: spec 20. Driver view:

`0x1014` CAP_CTRL (shadowed; values in operation `0x14F010x1`, `0x14F013x5`):

| Bits | Meaning |
|---|---|
| 0 | Capture enable; always write 1 |
| 1 | Bit-slip status latch; not needed, write 0 |
| 2 | FLAG_ARM (spec 20 §5.2): three-write pulse, clear and arm the per-lane error flags of `0x1024/0x1028` |
| 3 | Read-back only; no decoder behind it in the FPGA, always 0 in operation |
| 7:4 | lane-pair sub-index for `0x102C` reads; write 0 |
| 9:8 | sub-index for `0x1030` reads (0–3) |
| 19:12 | CAP_SEL: one-hot ADC (bit 12 = ADC 0); gates the IDELAY loads and the transfer of `0x1020` |
| 23:20 | LANE_EN: per-lane enable (spec 04 §3.1), `0xF` |
| 31:24 | `0x14` (written with the core alignment; keep; meaning **UNVERIFIED**) |

`0x1018` CAP_TAP — select-then-write, three writes: (1) `tap`, (2) `tap | select`, (3) `tap`.
`tap` = bits `[4:0]` (32 taps of 78 ps; `[8:5]` ignored). `select`: all lanes = `0x3FFFF000`; lane
`n` of group 0 = `(1 << (12+n)) | 0x10000000`, of group 1 = `(1 << (12+n)) | 0x20000000`
(`n` = 0…13 = ADC code bit, LSB first). The load happens on the rising edge of the select.

`0x101C` CAP_SLIP / `0x102C` CAP_SLIP_ST (R) — a bit-slip pass is not needed: the S2 image has no
bit-slip logic (spec 20 §5.6; layout of `0x102C` there). Leave `0x101C` unwritten and `0x102C`
unread. An FPGA MAY implement neither register, and reads of `0x102C` SHOULD return 0.

`0x1020` CAP_ROTATE — `[3:0]` rotation group 0 (0–3 on the 1-ADC board), bit 4 enable; `[11:8]`
rotation group 1, bit 12 enable. Per ADC, with `CAP_SEL` = that ADC: write with bits 4 and 12
cleared before the search; afterwards write `(v & 0xF0F0) | R[17:16] << 8 | R[1:0]`, then the same
value with bits 4 and 12 set, where `v` = the previous value and `R` = `0x1030` read with sub-index 0
(spec 20 §5.3–5.4). With rotations 0 the three writes are `0`, `0`, `0x1010`.

`0x1024`/`0x1028` CAP_FLAGS (R) — sticky per-lane error flags (bit `n` = lane `n` of group 0 / 1 in
`[13:0]`); 0 = lane stable since the FLAG_ARM pulse (flag semantics: spec 20 §5.2). `[29:16]` carry
no needed information.

`0x1030` CAP_ROT_ST (R) — `[7:0]` group 0, `[23:16]` group 1 alignment (`[1:0]`/`[17:16]` =
rotation); the two should agree (a mismatch is worth a warning). 0 is a valid result.

Bring-up sequence (the step list is spec 20 §5.2–§5.4 and §5.7, its place in the start-up order
spec 06 §10): `CAP_CTRL = 0x00F00001`, then `0x14F00001`. Lane search with the ADC in data mode 2
(the lane-training pattern, `0x0D = 0x650F`, constant per sample position): per fitted ADC (ADC 0
only on the 1-ADC board), `CAP_SEL` = that ADC (`0x14F01301` for ADC 0), for tap 0…31 load all
lanes, FLAG_ARM pulse (`0x14F01305`), read `0x1024/0x1028`; window search (below). Then ADC data
mode 4, and each lane's tap is loaded with `CAP_SEL` = ADC 0, then again with `CAP_SEL` = ADC 1
(`0x14F02301`). Word alignment, per ADC with `CAP_SEL` = that ADC: read `0x1030` with sub-index 0
(the result) and 1–3 (discarded; `[9:8]` stays 3), then write `0x1020` twice (spec 20 §5.4). The
ADC stays in data mode 4 (operating mode) afterwards.

Window search over the 32 flag words of the sweep:

1. Reference lane = lane 2 of group 0. Find the **first run of at least 9 consecutive taps with
   equal flags** (all 0 or all 1) that is bounded by a flag change on both sides (a run touching
   tap 0 or tap 31 does not count). Its centre `c = end + 1 − ⌊(len + 1)/2⌋` and the flag value
   `v` at `c` are the reference. Without such a run the search has failed; continue with `c = 0`,
   `v = 0`.
2. Every lane (both groups): normally the run of value `v` that contains tap `c`, loaded at
   `max(end − ⌊len/2⌋, 0)`; the fallback cases and the failure case are in spec 20 §5.2 step 4.

Consequently all-zero flags are not a passing answer: they fail step 1 and every lane gets tap
15 (the middle of a 32-tap all-zero run), because the result is loaded even after a failed search.
An FPGA or device model must return flags that form a data eye: set within a few taps after each
data transition, repeating every bit period (1 ns at 1 Gb/s = 12.8 taps of 78 ps), so that the
reference lane shows a bounded stable run of ≥ 9 taps.

### 2.6 `0x1034` / `0x1038` ADC SPI bridge

The ADC's configuration registers are reached through the FPGA, not through a Linux SPI device.

| Bits of `0x1034` | Meaning |
|---|---|
| 25:24 | ADC select, one-hot: `01` = ADC 0, `10` = ADC 1 |
| 23 | 1 = read, 0 = write |
| 22:16 | ADC register address (`0x00`–`0x67`; `0x6E` is also written once, spec 06 §8.1) |
| 15:0 | Write data |

- Write: `0x1034 = sel | addr << 16 | (value ^ MASK[addr])`, then wait ≥ 200 µs.
- Read: `0x1034 = sel | 0x00800000 | addr << 16`, wait ≥ 100 µs, `value = (0x1038 & 0xFFFF) ^ MASK[addr]`.
- `MASK[]` is a fixed 104-entry XOR table for `0x00`–`0x67` (the ADC's reset defaults; `0x6E` has
  mask 0); values in spec 06 §8.1. A driver works in "difference from default" space; a device
  model must apply the same XOR.
- Per-core calibration-done: reading ADC register `0x16 + 8·core` (core 0…7) returns **bit 6 = 1**
  when the core's quantiser calibration, started by setting bit 14 of register `0x13 + 8·core`,
  is complete. The bit is read raw (the mask of those registers is 0). At every start-up set bit 14
  for all eight cores of each fitted ADC, then poll the cores in turn (about every 100 µs), clearing
  bit 14 of each core once it reports done. Give the whole poll a bounded budget (2 s is ample) and
  report a time-out without other consequence. A device model or FPGA must set the bit only after
  the start bit of that core was written. ADC 1 (select `0x02000000`) is not fitted on 1-ADC (S2)
  boards; what `0x1038` returns for it there is **UNVERIFIED** (an idle-high data line would read
  `0xFFFF`), so do not poll it on those boards.

### 2.7 `0x1040` STAT_CTRL

| Bits | Meaning |
|---|---|
| 4:0 | log2 of the averaging period in 4 ns ticks (max 16) |
| 11:8 | Core select (`k` = 0…7; `8·adc + core` on a 2-ADC board): selects the core whose result `0x1044`/`0x1048` return |
| 24 | MM_LATCH: two-write pulse (0→1) to latch the peak result into `0x1048` |
| 25, 27 | MM_EN (write both with the same value) |
| 26 | MM_CLR: pulse 1→0 clears the peak detector |
| 30 | R: measurement done |
| 31 | START: write 0 then 1 to start a measurement; the first of the two writes also clears bit 27 (bit 25 kept) |

Used only by ADC calibration (spec 10): results in `0x1044` (average) and `0x1048` (peak).
`0x1048` = `max[31:16] | min[15:0]`; each half is a raw 12-bit code (shift left by 4 for the record
domain).

Measurement: write `[4:0]`, `[11:8]` = 0 with START 0 then 1; poll bit 30 (read-into-shadow, §1.3).
Result read, per core `k` = 0…7: write `[11:8] = k` (other bits unchanged, i.e. the value last read),
then read `0x1044` (average) and/or `0x1048` (peak). One measurement therefore covers all cores, and
`[11:8]` selects which core's result the two result registers return. Treat `0x1044` =
`0xEEEEEEEE` as "not ready" and read that core again; an FPGA MUST NOT return that value as a
valid average.

### 2.8 `0x105C` LANE_GAIN / `0x1058` LANE_TRIM

`0x105C` LANE_GAIN (S2 image): `[15:0]` = eight 2-bit coarse gain codes, field `k` = bits
`[2k+1:2k]`. Field `k` acts on one ADC lane: fields 0–3 on the group-0 samples `t` = 0, 2, 4, 6
(`t = 2k`), fields 4–7 on the group-1 samples `t` = 1, 3, 5, 7 (`t = 2(k−4)+1`), with `t` as in
spec 21 §2.1. Assign fields by channel mode:

| Channel mode | Fields |
|---|---|
| 1 channel | all eight = the active channel |
| 2 channels | 0–3 = lower-numbered channel, 4–7 = higher |
| 3–4 channels | CH1 → 0, 2; CH2 → 1, 3; CH3 → 4, 6; CH4 → 5, 7 |

(This is exactly the lane → channel assignment of the record and TRG paths.) Write code 1
(`0x00005555`, the normal gain) and code 2 for a channel whose calibration gain `G ≥ 2` (spec 03
§2). In the factory image, on the 12-bit lane sample `x`, code 1 gives `y ≈ 2048 + 2.5·(x − 2048)`
and code 2 gives `y ≈ 2048 + 5·(x − 2048)`, saturated to 0…4095; the exact function, with the fine
code and the two extra LSBs, is in spec 21 §2.1. The ratio code 2 / code 1 is exactly 2, and the
×2.5 of code 1 is part of the calibrated gain chain (the calibration's `G` values hold with code 1
written). Code 0 bypasses the stage (`y = x`, fine code ignored); code 3 equals code 2; neither is
needed. An open FPGA MUST implement codes 1 and 2 as in spec 21 §2.1, including the factor 2.5,
or the stored calibration does not apply. `[31:16]` carry no field in the S2 layout (a start-up
write of `0x55555555` is harmless); an FPGA MUST ignore `[31:16]`. The S4 layout (four bytes
`0x55·g`) does not apply to the DHO1074.

`0x1058` LANE_TRIM packs sixteen 2-bit fine-gain codes (field `n` at bits `2n+1:2n`). In the S2
image field `n` = 0–7 trims the lane of `LANE_GAIN` field `n`: codes 1/2/3 add 1/256, 2/256 or 4/256
to that lane's factor (2.5 or 5); there is no effect at `LANE_GAIN` code 0. Fields 8–15 are stored
without effect on the sample path.

### 2.9 Sample rate (`0x10D0`, `0x10D8`)

Let `Fs` be the requested per-channel sample rate in Sa/s and `Fmax` the maximum for the current
channel mode (2 GSa/s with 1 channel active, 1 GSa/s with 2, 500 MSa/s with 3–4; spec 04).

| Condition | `0x10D0` | `0x10D8` |
|---|---|---|
| `Fs > 250 000 000` | `0x8000 \| (Fmax / Fs) & 0xFF` | `1` |
| `Fs ≤ 250 000 000` | `0` | `250 000 000 / Fs` (integer) |

Write `0x10D4` = 0 after both. Bit 15 of `0x10D0` selects "interleave mode" (rate derived from
the ADC by lane selection); otherwise the 250 MSa/s per-lane stream is decimated by `0x10D8`.

### 2.10 BWF bandwidth-limit FIR (`0x1098`–`0x10A0`)

A 24-tap, linear-phase FIR that implements the digital bandwidth limits (model bandwidth, BW-limit
settings; selection of the coefficient set per bandwidth and sample rate in spec 06 §9.6) and the
High-Res filter. The driver programs it with the channel/ADC configuration and with High-Res (spec
03 §5, spec 04 §5.1). In the factory image (spec 21 §6) the FIR sits on the **play-back path**,
between the DDR read and the sinc interpolator; the record memory and the TRG comparators get
unfiltered samples, so this FIR does not band-limit the trigger signal.

`BWF_MODE` (`0x1098`) values:

| Value | Meaning |
|---|---|
| `1` | FIR bypassed (filter off; also at start-up and in Import mode) |
| `2` | coefficient load in progress |
| `0` | FIR on with the loaded coefficients |

Load sequence (filter on): `BWF_MODE = 2`, `BWF_CFG = 0x18`, 96 writes to `BWF_COEF`,
`BWF_MODE = 0`, `BWF_CFG = 0x18`. Filter off: `BWF_MODE = 1`, `BWF_CFG = 0x18`. Factory image:
`BWF_COEF` words are taken only while `BWF_MODE` = 2 (words written in mode 0 are ignored); an
unloaded FIR in mode 0 outputs signed 0, i.e. mid-scale code 2048, so the play-back stream is a
constant (the transform of spec 03 §2 at `x` = 2048). A driver MUST keep this order (`2`, 96 ×
`BWF_COEF`, final mode).

`BWF_COEF` (`0x109C`) word: `[31:30]` bank 0–3, `[29:28]` 0, `[27:23]` tap 0–23, `[22:0]`
coefficient (23-bit two's complement). Write bank 0 taps 0…23, then banks 1, 2, 3 (4 × 24 = 96
words per load). A coefficient set `h[0…23]` is symmetric (`h[k] = h[23−k]`), has 22 fractional bits
and sums to exactly `2^22` (unity DC gain). The word for bank `b`, tap `t` carries, by channel mode
(spec 04 §3):

| Channel mode | Coefficient at (bank `b`, tap `t`) | Structure |
|---|---|---|
| 1 channel | `h[4·(t mod 6) + b]` | 4 polyphase components of 6 taps (bank = phase), repeated in the four 6-tap cascades of each bank |
| 2 channels | `h[2·(t mod 12) + (b mod 2)]` | 2 polyphase components of 12 taps (banks 0, 2 even phase; 1, 3 odd phase), repeated in both halves of each bank |
| 3–4 channels | `h[t]` | the full 24-tap set, identical in all four banks |

(Because `h` is symmetric, `h[k]` and `h[23−k]` are interchangeable in this table.) The mapping
depends only on the channel mode, not on the sample rate or decimation. Evaluated as a plain 24-tap
FIR at the sample rate of the spec 06 §9.6 column that selects them, the sets give the −3 dB
corners listed there, which match their bandwidth assignment; an open FPGA image MUST apply a
loaded set as this 24-tap FIR at the channel's sample rate (the per-tap sample routing of the
factory image is **UNVERIFIED**, spec 21 §6.2). Hardware structure of the 96 coefficient registers:
spec 21 §6.

---

## 3. SP display path (`0x1100`–`0x13FF`)

The SP block reads a window out of the record memory, resamples it to display length and streams
it to the CPU. Two independent paths exist: **main** (index 0) and **zoom** (index 1). Spec 03 §4
and spec 04 §9 give the arithmetic; this table gives the registers.

| Offset (main / zoom) | Name | Access | Format | Purpose |
|---|---|---|---|---|
| `0x1100` / `0x1104` | WIN_LEN | W | 32-bit | Number of memory points to read for the window: `P + 512·N + 72 + C` (main), `P + 512·N + 72` (zoom); `P` = window points, `N` = words per sample instant, `C` = compression |
| `0x1108` / `0x110C` | WIN_START | W | 32-bit | First memory point of the window, relative to the first sample `H` of the frame's `PRE + POST` window (frame header, spec 21 §8.3/§9), not to the frame's first burst |
| `0x1110` / `0x1114` | AVG_START | W | `[31:10]` = `floor(S·N/40)`, `[7:0]` = word remainder | Average mode only: window start in 40-word blocks (spec 03 §4) |
| `0x1118` / `0x111C` | SINC_LEN | W | `n + 40` | Sinc-interpolator input length |
| `0x1120` / `0x1124` | SKEW_COARSE | W, select-then-write | `[15:0]` delay, `[19:16]` output-slot one-hot, bit 31 = 0, bit 30 = 1 | Per-slot coarse skew (whole samples); write 0 for every slot in every horizontal configuration |
| `0x1128` / `0x112C` | SKEW_FINE | W, select-then-write | as above, bit 31 = 1, bit 30 = 1 | Per-slot fine skew (interpolator phase, spec 03 §4) |
| `0x1130`, `0x1134`, `0x1138` | SINC_RATIO main/zoom/stage2 | W | bit31 enable, `[23:16]`,`[15:8]`,`[7:0]` = ratio (same value in all three bytes) | Sinc interpolation ratio: main, zoom; shared stage 2 = `0x00C8C8C8` (200), bit 31 set when either path interpolates. Ratio 1 = `0x00010101`; zoom off = `0x00000000` |
| `0x1140` | SINC_SEL | W | one-hot `[15:0]` | Select coefficient bank for `0x1144` |
| `0x1144` | SINC_WORD | W | `0x10000000 \| target << 24 \| addr << 16 \| coef16` (`[25:24]` target 0 main, 1 zoom, 2/3 stage 2) | Sinc coefficient write port (load sequence §3.5) |
| `0x1148` | AVG_STRIDE | W | `640 / N + 32` | Averaging burst spacing (`0x2A0` 1 ch, `0xC0` 4 ch) |
| `0x1154` | LIN_OFF | W, shadowed | §3.1 | Interpolator enables |
| `0x1158` / `0x1160` | LIN_PH | W | `0` | Linear-interpolator start phase |
| `0x115C` / `0x1164` | LIN_INC | W | `floor((2^32 − 1) · M / N)` or `0xFFFFFFFF` | Linear-interpolator step (32-bit fraction) |
| `0x1170` / `0x1174` | DECIM | W, shadowed | §3.2 | Display decimation (compression) factor |
| `0x1184` | MSK_DECIM | W | `0x40000000 \| en << 31 \| factor[27:0]` (as DECIM, §3.2) | Compression for the mask-test path; write `0x40000001` (off) in every configuration without mask test |
| `0x1190` | MSK_CTRL | RW, shadowed | §3.4 | Pass/fail mask control |
| `0x1194` | MSK_TABLE | W | §3.4 | Mask table / threshold write port |
| `0x119C` | MSK_OUT | W | bitfield | Mask output (aux-out pulse, polarity, select) |
| `0x11A0`, `0x11A4` | MSK_FRAMES | R | 64-bit (hi, lo) | Frames tested |
| `0x11B4`–`0x11CC` | MSK_FAILS | R | 48-bit counts | Failed frames / crossing counters |
| `0x11D0` | ZONE_CTRL | W | bitfield | Zone-trigger control |
| `0x11D8`, `0x11DC` | ZONE_TABLE1/2 | W | table port | Zone-trigger tables |
| `0x11E8` | OUT_LEVEL | R | 32-bit | C2H stream data produced and not yet read, in units of 4 16-bit words; latched by the `0x1010` bit-28 pulse (spec 03 §6) |
| `0x11F0`, `0x11F4`, `0x11FC` | OUT_LEN0/1/4 | W | 32-bit | C2H transfer description (§3.3) |
| `0x12C0`, `0x12C4`, `0x12CC` | OUT_LEN2/3/5 | W | 32-bit | C2H transfer description (§3.3) |
| `0x1204` | SCAN_POS | R | bit31 valid, `[15:0]` position | Scan (slow sweep) write position |
| `0x1208` | REC_COUNT | R | bit31 flag, `[29:0]` count | Number of points/frames written to memory |
| `0x1210` | CLK_METER | R | 32-bit | ADC data-clock frequency counter; healthy = 62 500 ± 6 |
| `0x1280`–`0x12DC` | logic-analyzer registers | W/R | — | Logic-analyzer (MSO) path; not used without the MSO option |
| `0x131C` | SINC_ALIGN | W, shadowed | `[3:0]` = a, `[11:4]` = b, `[15:12]` kept (0), OR `0x02580000` | Sinc "early" alignment. A constant, not a per-configuration value: write it once at start-up as `0x02580044` (a = b = 4). Field meanings **UNVERIFIED** |
| `0x1328` | REC_DELAY | W, shadowed | `[31:7]` coarse, `[6:0]` fine | Acquisition delay. Not needed; an FPGA may treat it as reserved |

`SKEW_COARSE` / `SKEW_FINE` select field `[19:16]` = one-hot **output slot**: 1 channel: always
slot 0 (`0x10000`), carrying the active channel; 2 channels: slot 0 = lower, slot 1 = higher
channel; 3–4 channels: slot `n` = CH`n+1`. Each update of a path is one write with select 0
followed by one write per slot. The select-0 write carries the stale `[15:0]` of the last write
(coarse and fine share one shadow) and selects nothing; an FPGA MUST ignore it. Meaning of bits
31/30 **UNVERIFIED** (bit 31 distinguishes fine from coarse as written).

### 3.1 `0x1154` LIN_OFF

| Bit | Meaning |
|---|---|
| 0 | Main linear interpolator **disabled** (written as `!enable`) |
| 1 | Zoom linear interpolator disabled |
| 2 | Written `!enable` of the path programmed last (main, then zoom → follows the zoom path; `0x7` = both off, `0x6` = main on, `0x1` = zoom on) — **UNVERIFIED** hardware meaning |
| 8 | ×2 interpolation, main (meaning **UNVERIFIED**). Write 0 |
| 9 | ×2 interpolation, zoom (meaning **UNVERIFIED**). Write 0 |

Write `LIN_PH` = 0 before each `LIN_INC`; `LIN_INC = 0xFFFFFFFF` when disabled.

### 3.2 `0x1170` / `0x1174` DECIM

`value = 0x40000000 | (en << 31) | (shadow & 0x30000000) | (factor & 0x0FFFFFFF)`. `en = 1` when
`factor > 1`. `0x40000001` = pass-through. Preserve bits 29:28 from the shadow (**UNVERIFIED**
meaning). In Roll/Scan the main factor is `nch · c · 10`.

With `C = factor > 1` the stream carries `M = P / C` words per channel per frame (one 16-bit word
per channel per `C` memory points). Every pair of consecutive output words of a channel covers `2·C`
consecutive memory points and holds their minimum and maximum in the order in which they occur (the
extreme whose first occurrence is earlier is emitted first; factory image, spec 21 §9, main path of
1-channel plays; 2-/4-channel plays, the zoom path and the Roll/Scan stream are assumed to follow
the same rule, **UNVERIFIED**). A driver may treat them as `M` ordinary points per channel at the
fixed stride of spec 03 §2.1 or pair them. An FPGA MUST emit exactly `M` words per channel and
SHOULD use this min/max order.

### 3.3 C2H transfer description (OUT_LEN)

The six OUT_LEN registers are indexed 0–5 (`index → offset`: 0 `0x11F0`, 1 `0x11F4`, 2 `0x12C0`,
3 `0x12C4`, 4 `0x11FC`, 5 `0x12CC`):

| Index | Meaning | Values |
|---|---|---|
| 0 | main-path output points per channel per frame | `ceil8(M)`, `M` = 1000 … 1 000 000, fewer for a window clipped by the record (spec 04 §9); `1 000 000` (or ≤ 50 M) for a raw-memory export |
| 1 | zoom-path output points per channel per frame | `ceil8(Z)`, `Z` = 1000 … 1 000 000, fewer for a clipped window |
| 4 | output chunk size | `1000` in triggered modes; Roll/Scan: `K = (100 · tdiv / roof125(tdiv)) / N` points per chunk (100, 50, 25 on a 1-2-5 timebase) |
| 2, 3, 5 | logic-analyzer (MSO) equivalents | not written without the MSO option |

`ceil8(x) = (x + 7) & ~7`. In triggered modes every horizontal configuration writes OUT_LEN4 = 1000,
OUT_LEN0 and OUT_LEN1. In Roll/Scan it writes only OUT_LEN4 (= `K`) and OUT_LEN0 (= `ceil8(10^6)`);
OUT_LEN1 may keep its last value, also while zoom is off. An FPGA MUST ignore OUT_LEN1 unless
`SEQ_MODE.ZOOM` (behaviour of the zoom path in Roll/Scan with zoom on is **UNVERIFIED**). An FPGA
MUST accept transient values outside 1000 … 10^6 between configurations; only the values present at
a play command matter, and those can be below 1000 (any value ≥ 8, spec 03 §2.1).

### 3.4 Mask test

`0x1190` MSK_CTRL: bits 30 and 29 are written only together, as a pulse (set both, write, clear
both, write), to reset the mask counters; bit 28 is the "read result" pulse (0 → 1 → 0, three writes)
issued before the counters `0x11A0`–`0x11CC` are read; `[23:20]` = source select: `1 << ch`
(ch 0–3) with 3–4 channels, `1` with one channel, and with two channels `1` = lower / `2` = higher
channel of the pair. Bits 30/29/28 are only ever pulsed, never held as levels. `0x1194` MSK_TABLE
writes: rule rows `0xFC000000 | row << 16 | hi << 8 | lo` for 1000 rows (`row` steps 0x10 per entry
in bits 31:16), "greater" rule `bit27 | value << 8`, "less" rule `bit26 | value`. The mask feature
is optional for an open firmware; it is listed so a device model accepts the writes.

### 3.5 Sinc coefficient load (`0x1140` / `0x1144`)

Each sinc interpolator is a polyphase FIR with `r` branches of 16 taps (`r` = the path's sinc
ratio, spec 04 §9). Its coefficients come from the symmetric `16·r`-tap prototype `h_r` (§3.5.1).

`SINC_WORD` word:

| Bits | Field |
|---|---|
| 31:29, 27:26 | 0 |
| 28 | 1 |
| 25:24 | target `p`: 0 main, 1 zoom, 2 stage 2, 3 stage-2 second set (targets 2 and 3 are read by the SEQ sub-sample trigger search, spec 21 §8.3; what that search computes is **UNVERIFIED**) |
| 23:16 | address `a` in the selected bank, 0 … r − 1 |
| 15:0 | coefficient, 16-bit two's complement |

**Load of set `h_r` into target `p`:** for bank `b` = 0 … 15, write `SINC_SEL = 1 << b`. Then for
`t` = 0 … r − 1, write `SINC_WORD = 0x10000000 | p << 24 | a << 16 | (h_r[t + r·b] & 0xFFFF)`, with
`a = (b + t) mod r` for p = 0, 1, 2 and `a = t` for p = 3. Bank `b` therefore holds tap `b` of every
branch: tap `b` of branch `φ` is `h_r[φ + r·b]`, stored at address `(φ + b) mod r` (p = 3: at `φ`).
A load is 16 + 16·r writes. A driver MUST issue them in this order (whether the FPGA uses the
address field or auto-increments is **UNVERIFIED**).

**When to load:**
- At initialisation, once: `SINC_RATIO2 = 0x80C8C8C8`, load `h_200` into p = 2; then
  `SINC_RATIO2 = 0x80C8C8C8` again, load `h_200` into p = 3.
- On every horizontal configuration (spec 03 §4 step 2): for zoom, then main, write the path's
  `SINC_RATIO`. If its ratio r > 1, load `h_r` into p = 1 (zoom) or p = 0 (main). Then write
  `SINC_RATIO2` (no load). Reload even when r is unchanged: skipping an unchanged reload relies on
  the FPGA keeping coefficients across `SINC_RATIO` writes and SP resets (**UNVERIFIED**). The FPGA
  MUST keep the p = 2 / p = 3 sets for the whole session, because they are loaded only once: in the
  factory image the SEQ block reads them in a sub-sample search after every event-triggered accept
  and holds the frame close until it finishes, so without them a frame may never be committed (spec
  21 §8.3; an open image SHOULD NOT make frame completion depend on them).
- r = 1 or zoom off: nothing is loaded.

An open FPGA with its own interpolator MUST accept these writes. It may ignore their content.

### 3.5.1 Prototype coefficient sets

There is one set per sinc ratio r ∈ {2, 4, 5, 8, 10, 16, 20, 25, 32, 40, 50, 80, 100, 160, 200}
(the sinc list of spec 04 §9 without 1). Set `h_r` has `N = 16·r` taps:

```
c      = (N − 1) / 2
w[n]   = 0.35875 − 0.48829·cos(2πn/(N−1)) + 0.14128·cos(4πn/(N−1)) − 0.01168·cos(6πn/(N−1))
g[n]   = sinc((n − c) / r) · w[n]                 sinc(x) = sin(πx)/(πx), sinc(0) = 1
h_r[n] = round(r · 32767 · g[n] / Σ_k g[k])        n = 0 … N − 1, round half away from zero
```

This is a 4-term Blackman-Harris window (symmetric form) on a sinc with its cutoff at the Nyquist
frequency of the input rate, scaled to unity DC gain per branch in Q1.15, evaluated in IEEE double
(or single) precision. These are the reference sets below. An implementation MAY differ from them
by ±1 LSB.

A replacement set MUST have these properties so the factory FPGA image behaves as it does with the
reference sets:

| Property | Requirement | Reference sets |
|---|---|---|
| Length | 16·r taps (16 per branch, r branches) | — |
| Word | signed 16-bit, `|h| ≤ 32767` | max `|h|` = 32766 |
| Symmetry | `h[n] = h[N−1−n]`: linear phase, group delay (N − 1)/2 output samples = 8 − 1/(2r) input samples (the window/delay arithmetic of spec 03 §4 assumes it) | exact |
| Branch DC gain | `Σ_b h[φ + r·b]` = 32767 ± 4 for every branch φ | 32763 … 32770 (r = 2: 32768) |
| Worst-case branch gain | `Σ_b |h[φ + r·b]|` ≤ 1.82 · 32767 | 1.585 · 32767 (r = 2) … 1.819 · 32767 (r = 200) |

and SHOULD match the reference response (`Fs` = sample rate entering the interpolator; stopband
= every image band from 0.75·Fs up to r·Fs/2):

| Response | Target | Reference sets (all r) |
|---|---|---|
| Passband ripple, 0 … 0.2·Fs | ≤ 0.001 dB | ≤ 0.0006 dB |
| −0.1 dB / −3 dB / −6 dB points | ≥ 0.34 / 0.46 / 0.50 ·Fs | 0.346–0.351 / 0.461–0.462 / 0.500 ·Fs |
| Stopband (from 0.75·Fs) | ≤ −85 dB | −86.1 dB (r = 4) … −99.4 dB (r = 200); −80 dB reached at 0.72·Fs |
| Sine interpolation error, f ≤ 0.2·Fs | ≤ −84 dBc | −84.9 … −97.5 dBc |

Mathematically, branch φ produces output sample `r·m + φ` from input samples `m … m − 15`. The
FPGA's tap-to-sample orientation, its output scaling (taken as `>> 15`, because branch sums are
2^15 − 1), its accumulator rounding and its saturation are **UNVERIFIED**.

## 4. SP board I/O and post-processing (`0x1400`–`0x155F`)

| Offset | Name | Access | Format | Purpose |
|---|---|---|---|---|
| `0x1404` | EXT_THRESH | W | DAC code | External-trigger comparator level |
| `0x1408`–`0x1414` | PRB_OFS1–4 | W | DAC code | Active-probe offset DAC, ch1–4 |
| `0x1418`–`0x1424` | BOARD_K1–4 | W | `0x8000` | `0x1418 + 4·ch` (ch 0–3): write `0x8000` exactly once, at start-up. It is **not** the vertical offset (that is AFE register `0x28`, spec 06 §9.3); function **UNVERIFIED** |
| `0x1428` | DAC_REFRESH | W, shadowed | `[15:8]` = A / 5·10^7, `[7:0]` = B / 5·10^7 (A, B in fs, 50 ns units); `[31:16]` not written | DAC refresh polling interval. Write once at start-up: `0x0000646E` (A = 5 µs, B = 5.5 µs) |
| `0x142C` | PRB_POWER | W | bit 0 | Probe power enable (one bit for all probe ports); write 1 at start-up |
| `0x1430` | AUXOUT_SEL | W, shadowed | `[2:0]` source | Rear AUX OUT source: write 0 = trigger out, 3 = pass/fail. Hardware decode (spec 21 §5.1): 0, 4–7 trigger pulse (60 ns = 15 CLK250 cycles), 1 SEQ trigger-accept flag, 2 constant low, 3 pass/fail |
| `0x1438` | XADC | R | `[11:0]` temperature code, `[27:16]` supply code | FPGA die temperature (`code · 503.975/4096 − 273.15` °C) and a supply rail (`code · 3/4096` V) |
| `0x1440`, `0x1454` | PRB_WIRE_TIM | W | — | Probe 1-wire timing |
| `0x1444`–`0x1450` | PRB_WIRE1–4 | RW, shadowed, read into shadow (§1.3) | write: bit 31 reset pulse; bit 24 command strobe pulse with command fields bit 30, `[15:9]`, `[23:16]`; bit 25 acknowledge pulse, issued after a read with bit 8 = 1. Read: bit 8 byte ready, `[7:0]` received byte | Probe 1-wire command/data, ch1–4 (field meanings beyond this **UNVERIFIED**) |
| `0x1470`–`0x1494` | DDR_TAPSET | W | — | DDR PHY tap overrides (from `cal_ddr.hex`, spec 10) |
| `0x1498`–`0x14B8` | DDR_TAPSTAT | R | — | DDR PHY calibration results (first-edge taps, write-level coarse, capture counts, DQS/DQ idelay counts) |
| `0x14C8`, `0x14D4`, `0x14E0`, `0x14EC` | PB_GAIN1–4 | W | `trunc(gain · 1024)` (0 … 65536, gain ≤ 64) | Play-back path digital gain per output slot, main (spec 03 §2) |
| `0x14D0`, `0x14DC`, `0x14E8`, `0x14F4` | PB_OFS1–4 | W | signed 32-bit code | Play-back path offset per output slot, main: `y = gain·x + offset` |
| `0x14F8`–`0x1504` | ZM_OFS1–4 | W | signed 32-bit code | Zoom-path offset per output slot |
| `0x1508`–`0x1514` | ZM_GAIN1–4 | W | `trunc(gain · 1024)` | Zoom-path gain per output slot |
| `0x1518` | OUT_CLAMP | W | `[15:0]` min, `[31:16]` max | Output clamp. Not needed; an FPGA may treat it as reserved |
| `0x151C`–`0x1524` | FRA_CFG | W | — | Frequency-response (Bode) option |
| `0x1528`, `0x152C` | FRA_DATA1/2 | R | — | Frequency-response results |
| `0x1530` | BDIG_CFG | W | bitfield | Bus-decode digitizer config, reset, cal control, math sources |
| `0x1534`–`0x1540` | BDIG_GAIN/OFS A/B | W | — | Bus-decode digitizer calibration |
| `0x1544`, `0x1548` | BDIG_THR A/B | W | — | Bus-decode thresholds |
| `0x154C` | BDIG_LEN | W | — | Bus-decode output length |
| `0x1550`, `0x1554` | BDIG_MINMAX A/B | R | — | Bus-decode min/max |

PB_GAINk / PB_OFSk / ZM_GAINk / ZM_OFSk (k = 1 … 4) apply to output slot `k` of each group of four
stream words: 1 channel: all four slots carry the active channel's values; 2 channels: slots 1, 3 =
lower-numbered channel, slots 2, 4 = higher; 3–4 channels: slot k = CHk.

---

## 5. TRG — trigger (`0x2000`–`0x226F`)

### 5.1 Levels

| Offset | Name | Format |
|---|---|---|
| `0x2004`, `0x200C`, `0x20B8`, `0x20C0` | THR_A ch1–4 | `[15:0]` upper Schmitt threshold `U`, `[31:16]` lower threshold `Lo` (record-code domain) |
| `0x2008`, `0x2010`, `0x20BC`, `0x20C4` | THR_B ch1–4 | same |

The trigger is fully digital: each channel has two Schmitt comparators, A (thresholds THR_A) and B
(THR_B). Default assignment (all types without two levels): THR_A = (`code(L)`, `code(L−h)`),
THR_B = (`code(L+h)`, `code(L)`); THR_A and THR_B are therefore different words. Two-level types
(slope, runt, window) put the upper level in THR_A and the lower in THR_B. Spec 05 §3. Write 16-bit
codes; the factory image stores only `[15:4]` and `[31:20]` (12-bit thresholds compared with
`sample > threshold`, spec 21 §2.2).

### 5.2 Event engines A, B, R

Three identical engines. **A** is the main trigger event, **B** the second event of sequential
triggers (A→B delay / Nth edge), **R** the reset event.

| A | B | R | Name | Format |
|---|---|---|---|---|
| `0x2014` | `0x202C` | `0x2044` | EV_IN1 | 2 bits per input for inputs 0–15 (`[2i+1:2i]`) |
| `0x2018` | `0x2030` | `0x2048` | EV_IN2 | 2 bits per input for inputs 16–20 in `[9:0]`; `[11:10]` = combine operator (write 0 = AND; not stored in the factory image, which always ANDs) |
| `0x201C` | `0x2034` | `0x204C` | EV_CFG | §5.3 |
| `0x2020` | `0x2038` | `0x2050` | EV_TMIN | low 32 bits of lower time bound, 1 ns ticks |
| `0x2024` | `0x203C` | `0x2054` | EV_TMAX | low 32 bits of upper time bound, 1 ns ticks |
| `0x2028` | `0x2040` | `0x2058` | EV_THI | `[15:0]` bits 47:32 of EV_TMIN, `[31:16]` bits 47:32 of EV_TMAX |

The trigger types of spec 05 need, for engines B and R, only `EV_CFG[23:0]` (and B's time bounds).
Engine B and R `EV_IN1/2` (`0x202C`, `0x2030`, `0x2044`, `0x2048`), engine R's time bounds
(`0x2050`–`0x2058`) and engine R's `TQ_MODE` (`0x204C[26:24]`) are then left unwritten and keep their
configuration-time values. In the factory image these registers have no reset input; their
configuration-time values are `EV_IN1/2` = 0, `EV_TMIN` = 1, `EV_TMAX` = 10, `EV_THI` = 0 and
`EV_CFG` = `0x00220000` (so `TQ_MODE` = 0), the same for all three engines. An FPGA MUST hold these
values from configuration until the registers are written (an SP/SEQ reset need not touch them).

### 5.3 EV_CFG (`0x201C` / `0x2034` / `0x204C`)

| Bits | Field | Meaning |
|---|---|---|
| 5:0 | CMP1 | comparator index of the trigger (end) event (spec 05 §3.1) |
| 7:6 | EDGE1 | edge of CMP1: 0 rising, 1 falling, 2 either edge, 3 never (spec 21 §4) |
| 13:8 | CMP2 | comparator index of the reference (start) event |
| 15:14 | EDGE2 | edge of CMP2 |
| 18:16 | OSEL1 | output select 1 (protocol decoders: clock/data line) |
| 19 | OINV1 | invert output 1 |
| 22:20 | OSEL2 | output select 2 |
| 23 | OINV2 | invert output 2 |
| 26:24 | TQ_MODE | time-qualifier comparison (spec 05 §5.2) |

Write `[15:0]` together (upper half preserved), then `[23:16]` (other bits preserved); write
TQ_MODE together with the time bounds (EV_TMIN, EV_TMAX, EV_THI written in that order after EV_CFG).
Spec 05 §5.1 is the authoritative field definition. In the factory image engine A stores `[26:0]`,
with OINV1/OINV2 (bits 19/23) in hold-type flops; engines B and R store `[26:0]` (spec 21 §1.2).

### 5.4 Trigger control

| Offset | Name | Access | Format |
|---|---|---|---|
| `0x205C` | TRG_SEL | W, shadowed | §5.5 |
| `0x2060` | AB_DLY_LO | W | A→B delay, 4 ns ticks, low 32 bits — not needed (the Delay trigger uses engine A's time bounds) |
| `0x2064` | AB_DLY_HI | W | `[15:0]` bits 47:32 — not needed |
| `0x2068` | R_IDLE_LO | W | reset (idle) timeout, 4 ns ticks, low 32 (Nth edge) |
| `0x206C` | R_IDLE_HI | W | `[15:0]` bits 47:32 |
| `0x2070` | A_COUNT | W | event-A count (Nth edge, 32-bit) |
| `0x2074` | B_COUNT | W | event-B count — not needed |
| `0x2080` | HO_COUNT | W | holdoff by event count — not needed |
| `0x2084` | HO_TIME_LO | W | holdoff time, 4 ns ticks, low 32 |
| `0x2088` | HO_TIME_HI | W, shadowed | `[15:0]` bits 47:32 of holdoff (16-bit update, `[31:16]` kept); `[18:16]` holdoff type, write 4 (time); the type is not stored in the factory image (time holdoff only) |
| `0x208C` | TRG_COUPLING | W, shadowed | `[1:0]` coupling filter (0 DC, 1 HF reject, 2 LF reject, 3 AC); `[3:2]` channel index of the trigger source (0–3 = CH1–CH4); `[17:4]` coefficient, write `0x800` (read back only, no effect in the factory image); shadow keeps `[31:18]` (spec 05 §4) |
| `0x2100`–`0x21F0` | protocol triggers | W | UART `0x2100–0x2104`, I2S `0x2108–0x2114`, LIN `0x2118–0x2138`, CAN `0x213C–0x2164`, FlexRay `0x2168–0x2174`, SPI `0x2178–0x2188`, I2C `0x218C–0x21A0`, video `0x21A4–0x21AC`, 1553B `0x21B0–0x21BC`, SENT `0x21D0–0x21F0` (spec 05 §8.14). `0x2174` carries two FlexRay fields (bit dividers and cycle min/max/mask) in one word; the last write wins, so a driver must write the combined value |
| `0x221C` (write) | UART_BIT_HI | W, shadowed | UART bit-time divider bits 30:21 in `[9:0]` (UART trigger only; same offset as the counter read-back FC_T0_L) |
| `0x2264` | EXT_STATE | RW | R `[3:0]` EXT comparator state (nonzero = input above `EXT_THRESH`); write: clears bit 16 (written value ignored), issued on every noise-reject change; the written value carries the last read `[3:0]` (read-into-shadow, §1.3); bit-16 function **UNVERIFIED** |

The factory image has no clock-enabled storage for `0x221C` (write side), `0x2264` bit 16, and the
FlexRay, MIL-1553B and SENT registers (spec 21 §1.2; **UNVERIFIED** whether they are absent or held
in hold-type flops).

### 5.5 `0x205C` TRG_SEL

| Bits | Meaning |
|---|---|
| 4:0 | TRG_KIND: 0 = engine A (single-edge Edge and all single-engine types), 1 = engine A or B, 3 = A/B/R sequence (Nth edge), 4–0xE protocol/video decoders (spec 05 §6) |
| 5 | FORCE: pulse 1→0; the FPGA acts on the rising edge (one force per 0→1, spec 21 §5.2) |
| 6–10 | ABR sequence enables (5 flags → bits 6..10); the Nth-edge trigger sets them (`1,1,1,0,1`: bits 6, 7, 8, 10). A driver need not clear them afterwards, so an FPGA MUST ignore them unless TRG_KIND = 3 |
| 11 | ABR_CLR: pulse 1→0 = reset the A/B/R sequence FSM — not needed |
| 12 | TRG_PATH: written per trigger type/source (spec 05 §6). Hardware: it does **not** select the comparator input; it is one term of an arm-time flag that switches a SEQ-side trigger-timing path (spec 21 §5.3) |

Update the shadow as a 16-bit read-modify-write of `[15:0]` (`[31:16]` kept). Stored bits in the
factory image: `[13:0]`.

### 5.6 Video trigger

`0x21A4 = (mode & 3) | (line & 0x7FF) << 2`; `0x21A8 = (t1 / 4 ns) | (t2 / 4 ns) << 16` (times in
FPGA clock cycles, truncated); `0x21AC = a[10:0] | lines[10:0] << 11 | trilevel << 22 |
sd_interlaced << 23`. Per-standard constants: spec 05 §8.4.

### 5.7 Counter and DVM (`0x2200`–`0x2258`)

| Offset | Name | Access | Format |
|---|---|---|---|
| `0x2200`, `0x2204` | FC_INIT A/B | W | write `0x8080` (`0x8000` once at start-up); meaning **UNVERIFIED** |
| `0x2208` | FC_RUN | W | bit 0 = counter running (1 only while enabled with a channel source) |
| `0x220C` | FC_SRC | W, 8-bit shadow | `[3:0]` one-hot source channel (bit `n` = CH`n+1`) while running, else 0 |
| `0x2210` | FC_CFG | W, 16-bit shadow | bit 0, bit 2: control (start-up writes bit 0 = 1, bit 2 = 0; clear-result writes `(bit0, bit2)` = `(0, 1)` then `(1, 0)`); bit 1: 1 in counter mode 2; bits 3, 4: write 0; `[9:5]` and `[14:10]`: counter source id (both fields carry the same value; 1–4 = CH1–CH4, other ids **UNVERIFIED**) |
| `0x2214` | FC_GATE | W | gate time in 4 ns ticks (`t_fs / 4 000 000`) |
| `0x2218` | FC_LATCH | W | write `1` before every result read: an FPGA MUST latch the results on each write, not on an edge |
| `0x221C`/`0x2220` | FC_T0 L/H | R | 63-bit: `[31:0]` in L, bits 62:32 in H`[30:0]`; H bit 31 = result-valid flag (`0x221C` is also written by the UART trigger, §5.4) |
| `0x2224`/`0x2228` | FC_T1 L/H | R | 64-bit: `[31:0]` in L, bits 63:32 in H |
| `0x222C`/`0x2230` | FC_N L/H | R | 64-bit, as FC_T1 |
| `0x2234`/`0x2238` | FC_PERIOD L/H | R | 63-bit, H bit 31 = result-valid flag (as FC_T0) |
| `0x223C`/`0x2240` | FC_EVENTS L/H | R | 64-bit, as FC_T1 |
| `0x2244` | DVM_CTRL | W, 16-bit shadow | bit 0 restart pulse (1 then 0), `[5:1]` = `n` (average over `2^n` samples), `[9:8]` = source channel 0–3; bits 6, 7, 10–15 kept. Working values: `0x39`, `0x38` (`n` = 28) |
| `0x2248` | DVM_LATCH | W | pulse 1 → 0 latches the results |
| `0x224C`/`0x2250` | DVM_ACC | R | 48-bit sum of samples: `[31:0]` = `0x224C`, bits 47:32 = `0x2250[31:16]` (`0x2250[15:0]` unused); average = `SUM / 2^n` |
| `0x2254`/`0x2258` | DVM_ACC2 L/H | R | 64-bit sum of squared samples (`0x2258 << 32 \| 0x2254`); mean square = `SQ / 2^n`; AC RMS = `sqrt(SQ/2^n − (SUM/2^n)^2)` |

---

## 6. SEQ — sequencer (`0x4000`–`0x40DF`)

| Offset | Name | Access | Format | Purpose |
|---|---|---|---|---|
| `0x4000` | SEQ_ID | R | 32-bit | FPGA image version |
| `0x4004` | RUNCTL | W, shadowed | §6.1 | Run / stop |
| `0x4008` | SEQ_MODE | W, shadowed | §6.2 | Channel mask, roll/scan/zoom, processing enables |
| `0x400C` | SEQ_CMD | W, shadowed | §6.3 | Record / play / sequence control |
| `0x4010` | FR_A | RW | frame index | W: first frame of a play range; R: frame progress = loaded by the write, then the index of each frame as it is committed, not changed by a stop; with `FR_B`, frames completed (spec 03 §3.3) |
| `0x4014` | FR_B | RW | frame index | W: first frame to record / play; R: as written (`[19:0]`) |
| `0x4018` | FR_C | W | frame index | Last frame of the range (reads back as written) |
| `0x401C` | BOARD_ID | R | 32-bit | Board hardware version |
| `0x4020` | SEQ_FUNC | W, shadowed | §6.4 | Auto trigger and mask-test actions |
| `0x4034` | SEQ_STAT | RW | §6.5 | Acquisition status / status clear |
| `0x4044`, `0x4048` | DNA | R | DNA = `0x4048 << 32 \| 0x4044` | 57-bit device DNA |
| `0x404C` | PRE_ADJ | W | 32-bit | Pre-trigger compensation; no register of its own in the factory image and no effect on the trigger position, trigger enable or accept time in Normal mode (Average and Scan **UNVERIFIED**; spec 21 §1.3, §8.3); an open FPGA MAY ignore it |
| `0x4050` | PRE_LEN | W | §6.6 | Pre-trigger length |
| `0x4054` | POST_LEN | W | §6.6 | Post-trigger length |
| `0x4058` | SCAN_TPOS | W, shadowed | bit31 flag, `[30:0]` position | Trigger position in scan mode (reads back as written) |
| `0x4060` | AUTO_T1 | W | 4 ns ticks | Auto-trigger timeout (first) |
| `0x4064` | AUTO_T2 | W | 4 ns ticks | Auto-trigger timeout (repeat) |
| `0x4068` | FR_GAP | W | 4 ns ticks | Record / play inter-frame interval: 0 in Run/Play; waveform recorder: user interval; a configuration may write 15 ms (99 ms with mask test), which the next run command overwrites |
| `0x406C` | SEQ_TIMEOUT | W, shadowed | bit31 enable, `[30:0]` 4 ns ticks | UltraAcquire record timeout (0 = off) |
| `0x4080` | FR_READY | RW | bit 30 R frame available: set at every frame commit, cleared by a written bit 31 = 1; bit 31 reads back as written; `[3:0]` / `[7:4]` R frame starts / commits mod 16, bits 27:26 R 1 (no needed meaning); factory-image read-back per spec 21 §7.2; write = clear pulse (below) | Frame-ready flag and clear |
| `0x40A0`, `0x40A4` | IMP_HDR2/3 | W | 32-bit | Frame-header words `[95:64]` / `[127:96]` for frames recorded with `SEQ_CMD[18]` (import, spec 21 §8.3); how an import supplies them is **UNVERIFIED** |
| `0x40B0`, `0x40B4` | TSTAMP L/H | R | 64-bit, LSB = 0.5 ns | Trigger time stamp (latched by `0x40B8` bit0). Factory image: a counter free-running from FPGA configuration at 8 counts per CLK250 cycle, captured at the SEQ trigger accept (+2 cycles); the same capture as frame header `[95:64]` (spec 21 §8.3; the unit holds for 1 ch at 2 GSa/s). **UNVERIFIED**: time vs sample counts in 2-/4-ch modes and at decimated rates, and what the bit-0 pulse latches. Read it right after a run command when needed (waveform recorder) |
| `0x40B8` | TSTAMP_CTL | RW, shadowed (read-into-shadow, §1.3) | bit0 latch pulse (0→1→0), bit1 R first-valid, bit2 clear-first pulse | Timestamp control |
| `0x40C0`, `0x40C4` | TSTAMP0 L/H | R | 64-bit | Timestamp of the first frame (unit assumed as TSTAMP, **UNVERIFIED**) |
| `0x40D0` | SCAN_END | R | non-zero = done | Scan sweep complete |

`FR_READY` access: poll by reading only (the read goes into the shadow; never write while
polling). Clear the frame-available flag with three writes `v`, `v | 0x80000000`, `v`, where `v` =
the last value read with bit 31 cleared (typically `0x40000000`). An FPGA MUST act only on the 0→1
edge of bit 31 and MUST ignore bits 30:0 of writes.

Storage in the factory image (spec 21 §1.2, §1.3): `FR_GAP`, `PRE_LEN`, `POST_LEN`, `AUTO_T1/2`
are 32-bit. `PRE_ADJ` has no register of its own: the partial address decode stores a `0x404C`
write into the `POST_LEN` store, the `SEQ_TIMEOUT` store (`0x406C`, also written by
`0x405C`/`0x407C`) and the `SCAN_TPOS` store (`0x4058`, also written by `0x4048`/`0x405C`).
Therefore write `POST_LEN` and `SEQ_TIMEOUT` after `PRE_ADJ` in every run command, and in Scan mode
write `SCAN_TPOS` after `PRE_ADJ`; otherwise the `0x4058` store keeps the `PRE_ADJ` value (5, or 10
in Average). That store is an operand of the frame-position logic, but its value (5 or 64, for example)
does not change the header `H`, the trigger enable or the accept (1 ch, Normal; Average/Scan
**UNVERIFIED**). An open
FPGA MUST store `SCAN_TPOS` and `SEQ_TIMEOUT` and MAY ignore `PRE_ADJ`.

### 6.1 `0x4004` RUNCTL

| Bit | Name | Use |
|---|---|---|
| 0 | RUN | 1 = acquisition FSM running. Set last by every run command; cleared by stop |
| 2 | REPLAY | Write 0 then 1 to request replay of the last frame; cleared by stop/init |
| 30 | HALT_SAMPLING | Three-write pulse: stop sampling only (keep the record) |
| 31 | FLUSH | Set together with RUN=0 for a full stop; cleared on the next write |

`stop_word(f)`, three writes: (1) clear RUN (bit 0) and bit 31 → write; (2) set bit 31 if `f` →
write (the same value again when `f` = 0); (3) clear bits 31 and 2 → write. Values: `0`, `0`, `0`
(`f` = 0) and `0`, `0x80000000`, `0` (`f` = 1). A full stop (spec 03 §3.2) is `stop_word(0)`,
`stop_word(1)`, …, `stop_word(0)`. Run: `RUNCTL |= 1` (one write, last of every run command).
Init (once at start-up): write `bit31 = flag`, then write with bits 31 and 2 cleared. Stored in the
factory image: bits 0, 1, 30, 31 (clock-enabled); bit 2 is not stored: writing 1 issues a one-shot
request that the SEQ FSM takes and clears (its effect **UNVERIFIED**); spec 21 §1.2.

### 6.2 `0x4008` SEQ_MODE

| Bits | Meaning |
|---|---|
| 3:0 | Active-channel mask (bit n = channel n+1) — written by every run command |
| 17 | ROLL: roll mode |
| 22 | SCAN: scan (slow-sweep) mode |
| 23 | ANA_EN: process analog channels. Set it to 1 with the last write of the driver initialisation (after bits 24 and 26 are written 0) and keep it: every later SEQ_MODE write preserves it, so every run command carries it (`0x0080000m` plus any roll/scan/zoom bits) |
| 24 | LA_PROC: process logic channels |
| 25 | ZOOM: zoom path enabled |
| 26 | LA_ACQ: acquire logic channels |

Entering Roll or Scan: first write `SEQ_CMD &= 0xBDF9FFFF` (clears bits 17, 18, 25, 30; no load
strobe), then clear SEQ_MODE bits 17, 20, 21, 22 (mask `0xFF8DFFFF`). Run commands clear bits 17
and 22 before re-arming and re-set them through the roll/scan configuration. Stored in the factory
image: `[3:0]`, 16, 17, 22, 23, 25; the logic-analyzer bits 24 and 26 and bits 20, 21 are not stored
(logic-analyzer acquisition is not implemented in this image).

### 6.3 `0x400C` SEQ_CMD

| Bit | Meaning |
|---|---|
| 16 | Cleared by Import; meaning **UNVERIFIED** |
| 17 | Import-play / import-end marker |
| 18 | Import (H2C) record |
| 24 | Sequence play; not needed (preserve) |
| 25 | Record-sequence (segmented acquisition; also UltraAcquire and the waveform recorder) |
| 26 | Option flag (play/record variant); not stored in the factory image (no effect) |
| 27 | Meaning **UNVERIFIED**; preserve |
| 28 | LOAD: set while `0x4010`–`0x4018` are written, then cleared (load strobe) |
| 29 | ENABLE |
| 30 | PLAY |
| 31 | RECORD; not needed (preserve) |

Stored SEQ_CMD bits in the factory image: 16, 17, 18, 24, 25, 27–31. `FR_A`, `FR_B` and `FR_C` are
20-bit (`[19:0]`; the largest index needed, `(2^25 − 2^20)/FRAME_BURSTS − 1` for the waveform
recorder, fits). Frame indices count in units of `FRAME_BURSTS` bursts (spec 04 §7): frame `f`
starts at record-memory byte address `f · FRAME_BURSTS · 32`; its one-burst header is stored at
burst `0x1F00000 + f` (spec 21 §8). The two ping-pong halves start at frame 0 and at frame
`frames_hw = 15 728 640 / FRAME_BURSTS`.

Every command is: (1) RMW the shadow clearing bits 25, 26, 28, 29, 30 (mask `0x89FFFFFF`;
record-sequence and import use `0x89F9FFFF`, which also clears 17, 18; bits 24, 27, 31 are
preserved) and setting bits 28 and 29 → write; (2) write `FR_A`, `FR_B`, `FR_C`; (3) clear bit 28 →
write; (4) apply the command bits below → write. Write `FR_GAP` / `SEQ_TIMEOUT` before step 1 (not
for Import and Import end).

| Command (spec 03 §3.1) | FR_A / FR_B / FR_C | Step-4 bits set | Step-4 bits cleared |
|---|---|---|---|
| Record (Run, Single, waveform recorder): segmented acquisition | `s`, `s`, `s+n−1` | 25, 29 | — |
| Play frames | `a`, `b`, `c` | 29, 30 | 26 |
| Import | `s`, `s`, `s+n−1` | 18 | 16 |
| Import end | — (no load) | 17 | 18 |
| Import play | `0`, `0`, `0` | 17, 29 | 18, 26 |

A free-running recorder loop programs no `SEQ_CMD` bits (only the channel mask and `RUN`). Import
play is **UNVERIFIED**.

### 6.4 `0x4020` SEQ_FUNC

| Bit | Meaning |
|---|---|
| 0 | AUTO: 1 = auto trigger (generate a trigger when none arrives within AUTO_T) |
| 1 | Mask zone-trigger enable |
| 12 | Mask: stop on fail |
| 13 | Mask: save on fail (set in normal operation: `SEQ_FUNC = 0x2001`) |
| 14 | Mask: save on pass |
| 15 | Mask test enable |

Stored in the factory image: bits 0, 1, 8–10, 12–15 (bits 8–10: meaning **UNVERIFIED**; write 0).

### 6.5 `0x4034` SEQ_STAT

Read bits (factory image, spec 21 §7.2; all other bits read 0):

| Bit | Meaning |
|---|---|
| 3:0 | FSM state code: 0 idle, 1 run start, 2 recording a frame (pre-fill, armed, post-trigger), 0xE/0xF frame commit; ≠ 0 during a play (play states, an end state at its end) |
| 4 | WAIT: trigger enable: 1 from the end of the pre-trigger fill until the frame is committed (also during the post-trigger part after the accept); 0 during the pre-trigger fill and between frames |
| 5 | Only a 4-cycle pulse at the `RUNCTL` bit-31 stop pulse; reads 0 otherwise (no usable stopped state) |
| 6 | Sticky: a frame was triggered by an event or FORCE (T'D) |
| 7 | Sticky: a frame was triggered by any source, including auto (AUTO) |
| 8 | Sticky: a record or play sequence ran to `FR_C` |
| 12 | Sticky: a record sequence ran to `FR_C` (a play does not set it) |
| 9 | 0 in record and play sequences (other use **UNVERIFIED**) |
| 10, 11 | The current frame has accepted its trigger (10: any source; 11: event or FORCE): set at the accept, cleared when the frame is committed, also for the last frame of a sequence |
| 13 | IDLE: 1 while no record or play sequence runs (idle before `RUN`, after the frame at `FR_C` has been committed, after the last word of a play has been written to the C2H FIFO, after a stop); 0 from `RUN` until the sequence ends, **also between frames**, and during a play until its last word is in the C2H FIFO. A driver may poll it ≤ 10 × 1 ms before a stop and proceed either way (spec 03 §3.6) |

A play does not change bits 4, 10, 11, `FR_READY` bit 30 or the frame progress (`FR_A` read).
The sticky bits 6, 7, 8 and 12 are cleared by a write with bit 31 = 1 (the clear below); the effect
of written bit 30 alone is **UNVERIFIED**. An open FPGA MUST provide bits 4, 6 and 7 as above; bit
13 SHOULD follow the rule above (a driver tolerates any value).

Status for the user, in priority order: bit 6 → triggered (T'D), else bit 7 → AUTO, else bit 4 →
WAIT, else RUN; the run state (RUN/STOP) itself comes from the driver, since bit 5 is not a usable
stopped indication. Because bit 4 is 0 between frames and during a play, a play shows the sticky
bits of the preceding record, or RUN after a clear. While a record or play sequence runs and bit 8
has been cleared since the last sequence end, `(status & 0x2320) == 0`.

Write: `SEQ_STAT = (a << 31) | (b << 30) | (last_read & 0x3FFFFFFF)` then `last_read & 0x3FFFFFFF`
(clear pulse with `a = b = 1`). Clear right after a status read when the sticky bits should restart
(e.g. at each status poll, about every 300 ms). No run command needs a status clear between the
stop and `RUN`, and an FPGA MUST NOT require one. The low bits written back are the last value read
and are ignored.

### 6.6 Pre/post trigger

`PRE_LEN` (`0x4050`) and `POST_LEN` (`0x4054`) are sample counts (per channel, at the acquisition
rate). For a record of `D` samples with dot time `dt` (fs) whose first sample lies at time
`t_start` relative to the trigger (spec 04 §6; negative when the trigger is inside the record):

```
t_end  = t_start + D · dt                      # time of the record end relative to the trigger
ext    = EXT-trigger latency compensation (fs): 119 ns (1 ch), 400 ns (2 ch), 401 ns (3–4 ch)
         when the trigger source is EXT, else 0
post   = max(ceil((t_end + ext) / dt), 1)      # samples after the trigger (may exceed D)
pre    = max(D − post, 0)                      # samples before the trigger
POST_LEN = post ;  PRE_LEN = pre
residual = (t_end + ext) − post · dt           # fs, < dt; used for sub-sample placement on screen
```

Driver view: the trigger instant is at window index `PRE_LEN` of the frame's `PRE + POST` window
(per-channel samples), plus `residual` and the fine delay of spec 03 §4 step 5. Factory image (spec
21 §8.3): the first sample of the trigger channel at or after an analog threshold crossing is at
window index **`PRE_LEN + 7`**, for every crossing phase within the 4 ns clock, in 1-, 2- and
3–4-channel modes at the maximum sample rate (word alignment off, no channel delay); the window
start `H` follows the crossing sample-accurately. With the factory image a driver must therefore
place the trigger 7 samples later than `PRE_LEN`. The offset with the operating ADC configuration
(word alignment `0x1020 = 0x1010`, `CH_SKEW`), at decimated rates and for EXT sources is
**UNVERIFIED**. An open FPGA SHOULD place the crossing at window index `PRE_LEN`.

Write `REC_LEN` (`0x10E4`) = `D` together with them. In Roll mode `PRE_LEN = D − 1`,
`POST_LEN = 1`; in Scan mode both are `D/2` (spec 04 §8).

## 7. Bring-up reads

| Read | Healthy value / use |
|---|---|
| `0x4000` | FPGA image version (informational) |
| `0x401C` | hardware version (selects board variant) |
| `0x4044/0x4048` | FPGA DNA (unique device identifier) |
| `0x1210` | 62 494 … 62 506 (ADC data clock present) |
| `0x1008` bit 29 | 1 = DDR init passed (§2.3; bit 28 SHOULD also be 1) |
| `0x1038` after an ADC read of reg `0x16 + 8n` | bit 6 = 1 (ADC core calibration done, §2.6) |
| `0x1024`/`0x1028` | per-lane flags that form a data eye over the IDELAY sweep (§2.5); all-zero flags fail the reference search and load tap 15 on every lane |
| `0x1030` | `[7:0]` equal to `[23:16]` (else the two groups are not aligned alike); the rotation in `[1:0]` is written back to `0x1020` |

---

## 8. Register write order summary

- **Never write a run command without a preceding stop** (spec 03 §3): `RUN` 1 while parameters change
  is not supported.
- **Parameters before strobes**: write data registers, then pulse/load bits, then `RUNCTL.RUN`.
- **Time values**: convert fs → ticks with truncation toward zero; 48-bit values are written low
  word first, then the word carrying the high bits.
