# 16 — Test Environment

The test environment runs aarch64 DHO1000 userspace (the unit's boot scripts and tools, its
application stack, or a replacement firmware) unmodified on an x86-64 host against a software model
of the board, and records every hardware access. It serves development of replacement software
without a unit, and dynamic checks that software drives the hardware the way specs 02–14 say.

The reference implementation of this environment lives in a private workspace and is not published.
This spec states what such an environment does and how faithful its model is, so that one can be
rebuilt and so that results obtained with it can be judged. The hardware facts it models come from
specs 02–14, 20 and 21 and are **UNVERIFIED** like them; agreement with the model is not evidence about a unit.

## 1. Tiers

| Tier | What runs | Hardware boundary | State of the reference implementation |
|---|---|---|---|
| 1 | aarch64 userspace under qemu user-mode emulation, Android 7.1 arm64 userspace (bionic) in a container | Linux syscall interface (device nodes, sysfs/procfs, module loading, MMIO page faults) | boot script end to end, FPGA load, PLL init, PCIe/XDMA bring-up, application start, register traffic, C2H reads |
| 2a | The unit's application stack in a system emulator | Same protocol (§3) to the same device model | kept with the private reference implementation (§7) |
| 2b | The unit's kernel modules (`xdma.ko` …) in a system emulator | Emulated PCIe endpoint (XDMA IP model) in QEMU | plan (§8) |

## 2. Tier-1 architecture

```
process under test (aarch64)  ──syscalls──▶  patched qemu-aarch64 (linux-user, static)
                                                │ device paths, module calls, MMIO faults
                                                ▼ Unix stream socket (§3)
                                             device-model daemon (simd, Go)  ──▶ JSONL log
```

### 2.1 Front end: patched qemu-user

qemu `linux-user` 10.1.2 with a forwarding layer. It is chosen over LD_PRELOAD and ptrace
because the unit's loader tools are statically linked (no LD_PRELOAD), FPGA registers are accessed
with plain loads/stores to an `mmap`ed BAR (only a page-fault mechanism sees them), and a ptrace
interceptor cannot run on a host of a different architecture. One mechanism thus covers static
glibc tools, bionic executables and libraries, and shell scripts.

The front end must:

1. **Classify paths.** For `openat` of a path under a configured prefix (default `/dev/`, `/sys/`,
   `/proc/devices`, `/proc/modules`, `/proc/bus/`) ask the daemon; the answer is *emulated*,
   *pass* (normal open in the container) or an errno. `newfstatat`, `fstat`, `faccessat`,
   `fchmodat` on such paths are answered the same way.
2. **Represent an emulated open file as one socket connection** to the daemon. The guest fd is
   the connection, so `dup`, `fork`, `exec` and `close` need no bookkeeping. An fd is recognised
   as emulated by comparing `getpeername()` with the peer address learned from a probe
   connection at start-up (the daemon's bind path differs from the in-container path).
3. **Forward file operations** on emulated fds: `read`, `pread64`, `write`, `pwrite64`,
   `readv`, `writev`, `lseek`, `ioctl`, `mmap`, `fstat`, `ppoll`. For `ioctl` the argument bytes
   given by `_IOC_SIZE` are sent when `_IOC_WRITE` is set; the daemon may read and write any
   guest memory during the call (for argument structures carrying pointers) and returns the
   output bytes. For `read` the guest buffer address is sent too (some device nodes of spec 01
   take input bytes in the read buffer).
4. **Trap MMIO.** An `mmap` the daemon answers with *trap* becomes a `PROT_NONE` guest range.
   A host SIGSEGV inside it restores the guest CPU state, decodes the faulting A64 instruction
   (LDR/STR all addressing forms and sizes, sign-extending loads, LDP/STP/LDPSW, SIMD&FP
   loads/stores up to 128 bit, LDAR/STLR, LDXR/STXR (store reports success), PRFM), performs each
   element access through the daemon, applies base write-back, advances the PC and resumes.
   Unsupported encodings abort with the instruction word. Each access carries the guest PC and a
   caller chain (LR, then return addresses from the x29 frame chain). A 128-bit element is sent
   as one access (`lo`, `hi`); the daemon splits it (§4.3).
5. **Follow exec.** `execve` of an aarch64 ELF or a `#!` script re-executes the same qemu with
   `-0 argv0` (script: interpreter, optional argument, script path), so the whole process tree is
   emulated. Each qemu instance reports its argv to the daemon.
6. **Emulate module loading.** `init_module`, `finit_module` (image read from the fd) and
   `delete_module` are sent to the daemon, which decides success.
7. **Inject directory entries.** `getdents64` on a passed-through directory under a prefix
   returns the real entries followed by the daemon's emulated entries (so shell globs such as
   `/dev/xdma0_*` expand).
8. Report executable file mappings (path, address, length, offset) so a PC in the log can be
   attributed to the executable or library it lies in.
9. With the daemon unreachable, run without emulation (warning on stderr).

### 2.2 Rootfs and container

- Android 7.1.1 (API 25) arm64 `system/` from the public AOSP emulator image
  `sys-img/android/arm64-v8a-25_r02.zip` (`/system/bin/linker64`, `/system/lib64`, toybox,
  `mksh` as `/system/bin/sh`), plus `property_contexts` and `default.prop` from its ramdisk.
- `/system/bin/bash`: a static aarch64 bash (the unit's scripts start with `#!/system/bin/bash`,
  spec 01).
- `/system/bin/su`: accepts `su -c "cmd"` (spec 01 §3).
- Property service: `propd` creates the property area under `/dev/__properties__`, loads
  `default.prop`, `/system/build.prop` and a property file for the unit, serves `setprop` on
  `/dev/socket/property_service` and persists `persist.*` in `/data/property/persist.txt`.
  The unit's property file sets `ro.rigol.system.version=1.1.1` (selects `spi2fpga` + `version.txt`
  and `xdma.ko`, spec 01 §4; the real unit's value is **UNVERIFIED**).
- The update-package tree (spec 01 §10.1) is bind-mounted read-only at
  `/rigol/{tools,shell,driver,FPGA,resource,app}`; `/rigol/data` is a per-run writable copy of the
  package's `data/`; `/data` is per run.
- Rootless podman: `--rootfs <dir>:O` (changes discarded), `--network none`, the container's
  minimal `/dev`, uid 0 inside a user namespace, ≤2 CPUs, 3 GB.

### 2.3 Software under test and its log

Tier 1 has no Android runtime: what runs there is native code (a replacement firmware, the unit's
boot scripts and tools). Tier 1 also has no
`logd`. A preloaded shim replaces the NDK logging calls (`__android_log_write`, `__android_log_print`,
`__android_log_vprint`, the `_buf_` variants and `__android_log_assert`) and writes each call as one
line `P<prio>	<tag>	<text>` to `/dev/simd_alog`; the daemon logs it in order with the hardware
accesses as an `alog` event (§6). Without that node it prints to stderr.

## 3. Wire protocol

Little-endian. Message = `u32 payload_length, u32 type, payload`. Strings = `u32 len, bytes`.
Transport: Unix stream socket (Tier 1); in Tier 2 a TCP connection that a forwarder turns
into one Unix connection each (`MMAP` mode 2 (shm, SCM_RIGHTS) is therefore unavailable in
Tier 2; every node of the model uses mode 1).
Each emulated open file is one connection whose first message is `OPEN`; `STAT`, `EXEC`,
`MODULE`, `LIBMAP`, `LIST` use a fresh connection each. Every request is answered with
`RET {i64 ret, bytes}`; negative `ret` = `-errno`.

| Type | Name | Payload | RET |
|---|---|---|---|
| 0x01 | OPEN | u32 pid, u32 guest open flags, u32 mode, str path, str exe | 0 emulated (connection stays open as the file), 1 pass, <0 errno |
| 0x02 | STAT | u32 pid, u32 op (0 stat, 1 access, 2 chmod), u32 arg, str path, str exe | 0 + {u32 st_mode, u32 rdev, u64 size}, 1 pass, <0 errno |
| 0x03 | EXEC | u32 pid, u32 ppid, str exe, u32 argc, str argv[argc] | 0 |
| 0x04 | MODULE | u32 pid, u32 op (0 load, 1 unload), str params-or-name, bytes image | 0 / errno |
| 0x05 | LIBMAP | u32 pid, u64 addr, u64 len, u64 off, str path | 0 |
| 0x06 | LIST | str directory | data = (u8 d_type, str name)* |
| 0x10 | READ | i64 off (−1 = file position), u32 len, u64 guest buffer | ret = return value, data = bytes for the buffer |
| 0x11 | WRITE | i64 off, bytes | bytes written |
| 0x12 | IOCTL | u32 cmd, u64 arg, u64 pc, bt, bytes (in) | ret, data = out bytes copied to `arg` |
| 0x13 | LSEEK | i64 off, u32 whence | new position |
| 0x14 | MMAP | u64 addr, u64 len, u32 prot, u32 flags, u64 off | 0 + {u32 mode (1 trap, 2 shm), u64 shm_off}; shm fd as SCM_RIGHTS |
| 0x15 | MMIO | u64 offset (file offset of mapping + delta), u32 size, u32 write, u64 lo, u64 hi, u64 pc, bt | 0 + {u64 lo, u64 hi} (read data) |
| 0x18 | MMIO_POST | as MMIO with write = 1 | none (posted write; processed in connection order) |
| 0x16 | POLL | u32 events | revents |
| 0x17 | FSTAT | — | {u32 st_mode, u32 rdev, u64 size} |
| 0x90 | MEMRD (daemon→client, during IOCTL/READ) | u64 addr, u32 len | client answers 0x1F MEM {i32 status, bytes} |
| 0x91 | MEMWR (daemon→client) | u64 addr, bytes | client answers 0x1F MEM {i32 status} |

`bt` = `u32 n, u64[n]`: LR followed by frame-chain return addresses (n ≤ 6). The file position
of emulated files is kept by the daemon; XDMA SG-DMA nodes do not advance it on read/write.

Control socket (`simd -ctl PATH`, Unix stream, text lines): `key CODE [CMD]` queues the
front-panel frame `{5A, CMD (default 01), CODE, crc7(CMD, CODE), A5}` (spec 08) for reading from
`/dev/ttyS0`; reply `ok` or `err …`.

## 4. Device model (what is emulated, and how faithfully)

The daemon has two models of the FPGA and board: the **strict model** (default, §4.3) and the
**lenient model** (`simd -lenient`, §4.4). Both share the node set of §4.2 and the unit of §4.1.
Every hardware behaviour the model implements comes from specs 02–14, 20 and 21 and is as
**UNVERIFIED** as those specs; a run against the model proves nothing about a unit.

### 4.1 Emulated unit

Register traffic recorded in the emulator holds for this unit only:

| Item | Emulated value |
|---|---|
| Hardware code (`/dev/hdcode_gpio`) | `0x3` → `SPU_H12S2.bit` (XC7A100T, bitstream IDCODE `0x03631093`; the model compares bits 27:0 only), S2 board with one ADC (ADC 0 fitted, ADC 1 not). `-hwcode 0x2` emulates the XC7A75T board (`SPU_H12S2B.bit`, IDCODE `0x03632093`, selected automatically) |
| Product identity | `/dev/vendor_storage` empty: no model string, no serial number |
| Options | no licence file: 70 MHz bandwidth, no options |
| `BOARD_ID` (`0x401C`) | 0 (front end without the inverted `INV` of `0xF` boards) |
| `SEQ_ID` (`0x4000`) | `0x02140005` (placeholder) |
| FPGA DNA (`0x4048:0x4044`) | `0x0123456789ABCDEF` (synthetic) |
| Calibration | only the update package's defaults (`data/default/`: `adc_vert.cal`, `cal_afe.hex`, `cal_lsb.hex`, spec 10); no per-unit `/rigol/data/cal_*.hex`, so no ADC gain/offset/phase table |
| XADC (`0x1438`) | strict: die 45 °C, rail 1.0 V (`0x05550A1A`); lenient: 0 |
| Board monitors | constant: hwmon 42/43 °C, thermal zones 45/46 °C, adc128d818 inputs 1000, IIO raw 2048 |
| AFE (RT1642) | STATUS and e-fuse words read 0: revision 0, identical trims on all four channels |
| LMK04821 reference | OSCin = 10 MHz (strict lock computation) |
| FRAM (I²C 0x50) | strict: blank (0xFF) at every daemon start; lenient: no memory |
| `ro.rigol.system.version` | `1.1.1` |

### 4.2 Nodes

Nodes appear when the module that creates them is loaded (module name taken from the `.ko`
`.gnu.linkonce.this_module` / `.modinfo`), and disappear on unload. Device state outside the FPGA
(clock chip, ADC chips, FRAM) persists across module reloads and FPGA reconfiguration. Paths not
listed are passed to the container (`/dev`) or are absent (`/sys`, ENOENT).

| Node / file | Created by | Model | Fidelity |
|---|---|---|---|
| `/dev/spidev1.0` | boot | spidev ioctls (mode, bits, speed, `SPI_IOC_MESSAGE` with pointer deref, non-standard nr 0x64 accepted); all data bytes feed the FPGA configuration engine | protocol exact (spec 13) |
| `/dev/spi2k7_ctl` | `fpga_gpio` | `0x7802` PROGRAM_B (arg 0/1, ≥2 EINVAL), `0x80047801` INIT_B, `0x80047803` DONE, others EINVAL; read returns a fixed identification string | exact interface; INIT_B rises 1 ms after PROGRAM_B release |
| FPGA configuration engine | — | 7-series packet processor: sync `0xAA995566`, type-1/type-2 packets, register writes, configuration CRC (CRC-32C reflected over 32 data + 5 address bits, checked on CRC writes, reset by RCRC and after each check), IDCODE compare (bits 27:0; expected IDCODE from the hardware code: `0x2`, `0x4`, `0x9` (the codes that select a `…B.bit` image) → XC7A75T `0x03632093`, every other code → XC7A100T `0x03631093`; `-idcode` overrides), commands. A CRC or ID error drives INIT_B low. DONE, startup and reset: §4.3 / §4.4 | checked against `SPU_H12S2.bit`: both CRC checks match |
| `/dev/hdcode_gpio` | `hdcode_gpio` | read → one byte, the hardware code (`-hwcode`, default 0x3) | interface exact; value per unit `0x2` (XC7A75T) or `0x3` (XC7A100T), spec 13 §2.2; model default `0x3` |
| `/dev/spi_3wires` | `spi2pll_gpio` | LMK04821 register file; write 3 bytes `{a_hi, a_lo, d}` → returns 0; read with `buf = {0x80\|a_hi, a_lo}` → `buf[0]` = value, returns 0; identification registers `0x003`–`0x006`, `0x00C`, `0x00D` = 06 D0 5B 24 51 04, read-only; `0x7800` RESET pin, `0x7801` status pin (`*int`), `0x7802` SYNC pin | interface exact; lock: §4.3 / §4.4 |
| `/dev/spi_simulate` | `spi2afe_gpio` | `0x7800` slave select, `0x7801` mode, `0x7802` aux-DAC switch; AFE write 3 bytes `{addr, hi, lo}`; read with `buf[0] = 0x80\|addr` → 16-bit BE value, returns n; per-slave 128 × 16-bit register file; slave 4 (DAC) writes logged raw | register file only; STATUS/e-fuse read 0 (**UNVERIFIED** values) |
| `/dev/dac_gear_change`, `/dev/afe_rms`, `/dev/pwm-1`, `/dev/pwm_fan` | their modules | ioctls logged (pointer args dereferenced), `afe_rms` `0x7800` → 0 (no overload), `pwm_fan` `0x7802` → 128 | logging stubs |
| `/dev/xdma0_bypass` | `xdma` after `pcie_rockchip` with DONE=1 | BAR of FPGA registers; `mmap` trapped (every access logged with PC and callers); read/write EINVAL | register file + acquisition model (§4.3) |
| `/dev/xdma0_control` | same | register file (pread/pwrite/mmap) | stub |
| `/dev/xdma0_c2h_0` | same | play-back stream (spec 03 §2.1), roll/scan stream counted by `OUT_LEVEL`, H2C-written card memory at card address = file position (not advanced), synthetic samples (u16 display domain; content §4.3 / §4.4); `'q'` ioctls (PERF, ADDRMODE, ALIGN=1) | sizes per §4.3; sample values synthetic |
| `/dev/xdma0_h2c_0` | same | write(n) stores into sparse card memory | functional |
| `/dev/xdma0_events_0..15`, `/dev/xdma0_bypass_{h2c,c2h}_0` | same | periodic event (`-irq-period`) / logging | stub |
| `/dev/ttyS0` | boot | front-panel MCU UART: writes decoded as `{5A, cmd, data, crc7, A5}` LED frames (CRC-7/MMC checked); reads deliver injected key frames; termios ioctls accepted | LED path exact; key injection API only |
| `/dev/vendor_storage` | boot | read request → EINVAL (empty), write logged | stub |
| `/dev/i2c-4`, `/dev/i2c-1` | boot | strict: I²C adapter with the FRAM at 0x50 on `i2c-4` (§4.3); lenient: `I2C_RDWR` returns 0 and leaves the message buffers untouched (a read message returns whatever the caller's buffer held), `read()` returns zeros | strict: FRAM functional; no other I²C device |
| `/dev/rtc1`, `/dev/usbtmc_dev`, `/dev/usbgpib0`, `/dev/dma_auklet` | boot / modules | logged; `read()` and `_IOC_READ` ioctls return zeros | stub; `/dev/dma_auklet` is not created at boot (as on the device, spec 01 §4) |
| `/proc/devices`, `/proc/modules` | — | generated from the loaded modules (`xdma` listed once its nodes exist) | sufficient for `load_pcie.sh` |
| backlight `brightness`, thermal zones, hwmon temps, adc128d818 `in*_input`, IIO `in_voltage*_raw` | — | sysfs attributes with constant values (§4.1); writes stored | constant |
| `/dev/simd_mark`, `/dev/simd_alog` | boot | test-environment nodes, not on the unit: writes logged as `mark` (test markers) / `alog` (Android log of the software under test, §2.3) | — |

PCIe: loading `pcie-rockchip.ko` brings the link up only if the FPGA is configured (DONE);
`xdma.ko` then creates the nodes above; otherwise it loads without creating nodes.

Bypass-BAR read-backs pinned in both models (spec 02 §7): `0x4000`, `0x401C`, `0x4044/0x4048`
(§4.1), `0x1008` reads the last write | `0x30000000` (DDR status pass). A JSON script (`-script`)
pins values or sequences per offset. Every other offset reads back the last 32-bit value
written (the FPGA stores only some bits and aliases some TRG/SEQ offsets, spec 21 §1; the model
does not).

Acquisition commands (both models, spec 03 §9): the SEQ record / record-sequence / play /
sequence-play / import commands are decoded from the `SEQ_CMD` (`0x400C`) load-strobe sequence
and started by RUN (`RUNCTL` `0x4004` bit0); `REPLAY`; frame progress, `FR_READY` and
`SEQ_STAT` per §4.3 / §4.4;
roll/scan: points at `Fs / DECIM` per channel per second, `OUT_CTRL` bit28 latches `OUT_LEVEL`
(4-word units), `SCAN_POS[15:0]` = points modulo 1000 and `SCAN_END` after 1000 points (model
choices); `TSTAMP_CTL` latch stores the latch time in 4 ns ticks since daemon start (not a frame
time stamp).

### 4.3 Strict model (default)

- **FPGA configuration and reset.** DONE rises after START, a DESYNC with no CRC/ID error, and at
  least 512 further CCLK cycles (64 bytes; the bitstreams of spec 13 send 12 800). Bytes clocked
  before INIT_B has risen are ignored. PROGRAM_B low resets the FPGA: the BAR register files and the
  acquisition state return to power-on (pinned read-backs stay). If `pcie_rockchip` is loaded at
  that moment the endpoint leaves the link: bypass reads return all-ones, writes are dropped,
  C2H reads fail with EIO, until `pcie_rockchip` is unloaded; loading it again with DONE = 1
  trains the link.
- **Register granularity.** Registers are 32 bit: a 64-bit BAR access is two 32-bit accesses
  (low word at the lower address), a 128-bit access two 64-bit accesses.
- **Clock chip.** RESET (`0x7800`, 1) returns the chip to "unprogrammed". PLL2 locks when
  `0x138`, `0x140`, `0x160`–`0x162`, `0x166`–`0x168` and `0x173` were written since, the VCO,
  OSCin, PLL2 and its prescaler are powered, `fPD2 = 10 MHz · (2 if PLL2_REF_2X_EN) / PLL2_R`
  ≤ 155 MHz and `fVCO = fPD2 · PLL2_P · PLL2_N` lies in the range of the VCO selected by
  `VCO_MUX` (VCO0 1930–2075 MHz, VCO1 2920–3080 MHz). The status pin (`0x7801`) follows
  `PLL2_LD_MUX` / `PLL2_LD_TYPE` of `0x16E` (power-on: PLL2 DLD, open drain; PLL1 never locks);
  `0x183` bit 1 = PLL2 lock. CLKout6 (ADC sample clock) = `fVCO / DCLKout6_DIV` (bypass: `fVCO`)
  when its group is powered and its format is not off, else absent. SYNC has no effect.
- **ADC clock check.** `CLK_METER` (`0x1210`) = `round(62 500 · f(CLKout6) / 2 GHz)`, 0 without
  a clock.
- **ADC chips** (`0x1034`/`0x1038`). A register file per fitted ADC, initialised to the
  power-on defaults (wire value = MASK, logical 0; spec 06 §8.1); writes take effect at once (no
  transfer time). Calibration-done (bit 6 of `0x16 + 8k`, read raw) is 1 from 1 ms after bit 14
  of `0x13 + 8k` went 0 → 1, latched until the next start. ADC 1 is not fitted: writes are
  dropped, reads return the idle level of the shared data line, `-adc1 absentFFFF` (default,
  `0xFFFF`) or `absent0` (0, which makes the ADC core calibration of spec 06 time out); the S2 image
  has no internal pull on that line, so the unit's value is **UNVERIFIED**. `-adc1 present` models
  ADC 1 like ADC 0. A read with no chip selected returns the idle level. Sync phase: each `CAP_RESYNC`
  pulse (`SP_RESET` bit 1 rising) latches bit 1 of register `0x07` = 1 if the ADC clock runs and
  the sync delay `0x06[10:8]` ≤ 3, else 0 (the edge position is a model choice), so the
  sync-delay calibration of spec 06 §8.3 selects d = 3.
- **Lane capture** (spec 20 §5). A tap per lane (2 groups × 14), loaded on the rising edge of
  `select(lane) ∧ group ∧ CAP_CTRL[12]` (so loads with `CAP_SEL` = ADC 1 are ignored).
  Flags are cleared while `0x1014` bit 2 is 1, armed at its falling edge, and accumulate the
  lane's current flag. A lane flags when the ADC clock runs, capture is enabled (`0x1014` bit 0)
  and ADC 0 outputs the training pattern (data mode code `0x5`) and its tap lies within 3 taps
  after a data transition: transitions every 12.8 taps (1 ns), offset 4.4 taps plus a fixed
  lane skew of −2…+2 taps. In any other data mode every lane flags; without the ADC clock none
  does. `0x1024`/`0x1028` = flags in `[13:0]` and `[29:16]`. `0x1030` = rotation 2 in both groups
  (`0x00020002`) while the ADC clock runs and capture is enabled (`0x1014` bit 0), else 0;
  `0x102C` = 0.
- **Trigger and frame timing** (record-sequence, spec 21 §7, spec 05 §7/§10). The first frame
  arms at RUN, each further frame re-arms at the commit of the previous one; the trigger is
  enabled after the pre-trigger fill (`PRE_LEN` samples at the record rate decoded from
  `RATE_SEL`/`RATE_DIV` and the channel count). The accept is the earliest of: the next input
  event that passes the holdoff (`HO_TIME` + 2 ticks of 4 ns since the previous passing event;
  one event triggers one frame), a pending FORCE (`TRG_SEL` bit 5 rising edge, consumed by one
  frame) and the auto trigger; never before the enable. The frame is committed `POST_LEN`
  samples after the accept. The input signal is a parameter: `-trig 0` = one event every
  `-frame-ms` ms (fixed phase from daemon start), `-trig 1`/`2` = no events. Trigger type,
  source, level and coupling have no effect (there is no signal).
- **Auto trigger** (spec 21 §7.1). Timer 1 runs while `SEQ_FUNC[0]` = 1 (1 at power-on and after
  an FPGA reset, so it runs from configuration). It restarts at every input event (accepted or
  not: also during the pre-trigger fill, between frames, between runs and when the holdoff
  suppresses the event), at a write that sets `SEQ_FUNC[0]`, and when a hold ends; it is held at
  0 while `SEQ_FUNC[0]` = 0, while a FORCE is pending and from an event or forced accept until
  that frame ends (commit or stop); RUN, a re-arm and an auto-triggered frame do not restart it.
  Its flag sets `(⌊AUTO_T1 / 2^15⌋ + 1) · 2^15` ticks after a restart. Timer 2 counts per frame
  while the trigger is enabled and the flag is set (it keeps its count while the flag is
  cleared); the auto trigger fires when it reaches `AUTO_T2[27:16] · 2^16 + R + 1` ticks
  (`AUTO_T2[31:28]` and `[15:0]` ignored; `R` = a 16-bit pseudo-random value drawn per
  frame). With input events closer together than timer 1's period no auto trigger fires.
- **Status and frame progress** (spec 21 §7.2, spec 02 §6.5). `SEQ_STAT` (`0x4034`): `[3:0]` =
  2 while a record sequence runs, 3 while a play runs (a non-zero play state; the FPGA's code
  is not established), else 0; bit 4 WAIT = trigger enabled: 1 from the end of the pre-trigger
  fill until the frame is committed (also during the post-trigger part); bit 5 reads 0; bits 6
  (event or FORCE accept), 7 (any accept), 8 (a record or play sequence ran to its end) and 12
  (a record sequence ran to `FR_C`) are sticky until a `SEQ_STAT` write with bit 31 = 1;
  bits 10/11 = the current frame accepted its trigger (any / event or FORCE): set at the accept,
  cleared at the commit of that frame, also the last one; bit 13 IDLE = 1 only while no
  record or play sequence runs: 0 from RUN until the commit of the frame at `FR_C` (also
  between frames, so a stop that polls bit 13 times out mid-sequence) and, for a play (play,
  sequence-play, import-play commands), from RUN until the C2H read that takes the play's last
  word, which sets bit 8; a stop ends either. A play does not change bits 4, 10, 11,
  `FR_READY` bit 30 or the frame progress. `FR_A` (`0x4010`) reads the written value until
  the first commit, then the index of each committed frame (`FR_B + k − 1`), unchanged by a
  stop; `FR_B` reads as written (20 bits). `FR_READY` (`0x4080`) bit 30 is set at every
  commit and cleared only at the 0 → 1 edge of written bit 31; bit 31 reads as written, bits 27:26 read
  1, the frame start/commit counters in `[7:0]` read 0.
- **C2H.** A play delivers `frames · 2 · N · (OUT_LEN0 + zoom·OUT_LEN1)` bytes; a longer read
  returns only these (short read), a shorter one discards the rest (logged). A read with no
  pending play, no roll/scan stream and no H2C data at its address blocks for
  `-c2h-timeout-ms` (10 000, the XDMA `sgdma_timeout`) and fails with ETIMEDOUT. Content: the
  model's input is a sine of 2 divisions with 2 + k periods per 1000 main points on stream
  channel k, generated in the record domain (spec 03 §2) at `7500 / G′` codes (16 bit) per
  division, i.e. `±15 000 / (16 · G′)` around 2048 as 12-bit codes, where `G′ = PB_GAIN[s] / 1024`
  of the output slot when the record started (RUN of a record command or a roll/scan stream;
  `G′` = 1 for `PB_GAIN` = 0). It passes the output stage of spec 21 §9.1 per output slot
  `s` = word index mod 4:
  `y = clamp(⌊GAIN[s] · 16x / 1024⌋ + OFFSET[s], 0, 65535)` with `PB_GAIN`/`PB_OFS` (main,
  also roll/scan) or `ZM_GAIN`/`ZM_OFS` (zoom), so a replay shows 7500 codes per division
  and a stopped rescale (`PB_GAIN` re-programmed after the record) scales the trace as spec 21
  §9.1 says the hardware does. With `OUT_CTRL.NO_INFO` = 0 the FPGA inserts 16 bytes of frame
  information (layout: spec 02 §2.4, spec 21 §9.1; whether it repeats per frame **UNVERIFIED**); the
  model does not insert them and logs `frm_head` in the play's `c2h` event.
- **I²C.** `i2c-4` address 0x50 is an 8 KiB FRAM: a write message sets the 16-bit big-endian
  address from its first two bytes and stores the rest, a read message returns bytes from the
  address; the address increments and wraps; content blank (0xFF) at daemon start and kept for
  the daemon's lifetime. `I2C_RDWR` returns the message count; a message to any other address
  (and every address on `i2c-1`) fails with ENXIO; `I2C_FUNCS` reports plain I²C.

### 4.4 Lenient model (`-lenient`)

The lenient model (`-lenient`; Tier 2 runs with it): DONE at DESYNC; PROGRAM_B keeps the
link and every register; LMK status always 1 and `0x1210` pinned to 62 500 regardless of
programming; ADC register file starting at wire 0 with bit 6 of `0x16 + 8k` forced on for both
ADCs whether or not a calibration was started, and `0x07` read as written (0: the sync-delay
calibration of spec 06 §8.3 finds no edge and leaves d = 7); no tap state, `0x1024`–`0x1030` read
the register file (0, so the lane-training reference search of spec 20 finds no reference and every
lane gets tap 15); one frame every `-frame-ms` ms from RUN; `SEQ_STAT` = IDLE always 1 plus
STOPPED (bit 5) when not running, else WAIT before the first frame and then T'D or AUTO from `-trig`
(0 T'D, 1 AUTO, 2 none), regardless of `SEQ_FUNC`, FORCE, pre/post-trigger and holdoff;
`FR_A` reads `FR_B + n − 1` and `FR_READY` bit 30 = frames done since RUN (cleared by
the bit-31 pulse); C2H returns the requested length of fixed display-domain sines (±15 000
codes, independent of `PB_GAIN`/`PB_OFS`) and idle reads return synthetic data; a 64-bit access is one register, the
upper half of a 128-bit access is dropped; I²C as in §4.2; XADC 0.

### 4.5 Not modelled

Configuration-time (INIT) values of the registers (spec 02 §5.2): apart from `SEQ_FUNC[0]`
= 1 (strict), unwritten registers read 0. Trigger and timebase physics (no input signal, no
comparator, no sample-rate effect on data),
real ADC samples (strict: C2H content follows `PB_GAIN`/`PB_OFS` but not V/div, AFE gear,
`LANE_GAIN`, filters or the trigger configuration; lenient: no dependence at all), the
`NO_INFO` = 0 frame-information words, the `FR_READY` counters, the transient state codes
(`SEQ_STAT[3:0]` 1, 0xE, 0xF) and the bit-5 stop pulse, cycle-level timing (arm and re-arm
latencies, the 23-cycle trigger pipeline), DDR (status bits fixed, no `RM_RST_A`/`RM_RST_B` or reload effect), ADC bridge transfer
times, AFE e-fuse and comparator bits, write-only and unstored register bits, register aliases,
interrupts, touch controller and Ethernet (Tier 2 uses the emulator's own), USB gadget /
USB-GPIB, CPU timing (under qemu a poll loop bounded by `clock()` runs roughly an order of magnitude
slower per poll than on the unit, so CPU-time budgets expire after fewer polls).

### 4.6 What emulator runs can and cannot establish

A run against the model **can** establish: the register and node access sequences of the software
under test, their order and the values it computes from settings, for the unit of §4.1; whether the
software accepts a given model behaviour (its own checks, visible in the `alog` events, §2.3); the
protocol of the nodes marked exact in §4.2; and, with the strict model, that the software depends on a
modelled rule (e.g. tap loads with `CAP_SEL` = ADC 1 would overwrite the ADC 0 taps if the FPGA did
not gate them).

It **cannot** establish:

- FPGA-side semantics beyond the sources of §4.3: frame progress, `FR_READY` bit 30,
  `SEQ_STAT` and the auto-trigger timers follow spec 21 §7.1–§7.2 (its timer-2 period and the
  cross-frame behaviour are themselves **UNVERIFIED**), including plays (IDLE 0 until the play's
  last word, bit 8 at its end; the model ends a play at the C2H read that takes its last word, the
  FPGA when that word enters its output FIFO, so a play the CPU never reads keeps IDLE 0 in the
  model until the next stop); a run only shows that the software accepts them.
- Roll/scan rates and `SCAN_POS`/`SCAN_END` behaviour, time-stamp units: model choices
  (§4.2).
- Trigger behaviour and frame rates: the model has no signal; frame timing follows the
  programmed timing registers and the `-trig`/`-frame-ms` parameters only.
- Calibration results: the IDELAY taps and the ADC sync delay the software picks follow from the
  model's eye and sync-phase edge; AFE trims (e-fuse words 0) and offset/gain calibration
  (synthetic samples) are artefacts of the model.
- Wall-clock limits of the software's CPU-time loops (§4.5) and anything about bridge or bus
  timing.
- The read-back of write-only or unstored bits, and any property of another unit identity
  (other hardware codes, `BOARD_ID` 0xF, options or bandwidth licences, calibrated units,
  2-ADC boards).

## 5. Running

A Tier-1 run takes the software under test (a command inside the rootfs), the model and its
parameters, and gives the access log (§6) and the run's `/data` (with `boot_up.txt` and
`clock_init_log.txt` when the unit's boot script ran, spec 01 §2, the daemon's stderr and the
`alog` text). The commands of the reference implementation are not part of this spec.

Daemon flags: `-strict` (default) / `-lenient` (§4.4); `-trig 0|1|2` and `-frame-ms N` (input
signal, §4.3); `-adc1 absentFFFF|absent0|present`; `-c2h-timeout-ms N`; `-hwcode` (default 0x3),
`-idcode` (default from `-hwcode`, §4.2), `-script FILE`, `-wavefreq`, `-irq-period`, `-ctl PATH` (§3).

### 5.1 SCPI access in Tier 1

The container has no network (`--network none`) but has loopback. Software under test that serves
the raw-socket SCPI port of spec 11 §1.1 listens on `127.0.0.1:5555` inside it; a client runs in the
same container:

1. Start the software under test in the background.
2. Poll `nc -z 127.0.0.1 5555` until it accepts.
3. Send all commands over **one** `nc 127.0.0.1 5555` connection, `*IDN?` first and after every
   command, about 0.6 s apart. The `*IDN?` reply separates the replies, so commands without a
   reply (sets, unknown queries) stay aligned. Save the reply stream to a file under `/data`.

Caveats: `mksh` needs `TMPDIR` on a writable directory (e.g. `/data/tmp`) for here-documents; the
rootfs `nc` (`/system/xbin/nc`) does not end an idle read with `-w`, so stop it explicitly after
the last reply; under qemu-user `$!` of a background pipeline is not the `nc` pid, so find the pid
another way (e.g. `pidof nc`).

## 6. Log record

JSON object per line. Common fields: `ev` (kind), `seq`, `t` (seconds since daemon start),
`pid`, `exe`. `mmio` records: `bar`, `op` (`r`/`w`), `off`, `size`, `val`, `pc`, `sym`
(`object+offset`), `bt` (caller chain, attributed the same way), `note` (pinned register). Reads of one
register are logged in full for the first 64, then every 1000th (`nread`). Other kinds: `exec`,
`libmap`, `open`, `path`, `module`, `fpga`, `fpgactl`, `spidev`, `hdcode`, `lmk`, `afe`, `uart`,
`dma`, `sysfs`, `dev`, `pcie`, `acq` (`run`, `stop`, `c2h` with the expected size and
`mismatch`, `play-last`; strict: `trigger` with the kind input/auto/force, `force`, `idelay`
with the loaded taps, and per-kind trigger counts in `stop`), `mark`, `alog` (`prio`, `tag`, `text`:
the Android log of the software under test, §2.3). The `start` record names the model (`strict`,
`adc1`, `trig`, `frame_ms`).

## 7. Tier 2a: the unit's application stack

The reference environment can also run the unit's complete application stack in a system emulator,
against the same device model over the same protocol (§3), so that a replacement firmware's register
traffic can be compared with the unit's. How that is set up depends on the unit's own software and is
kept with the private reference implementation; it is not needed to write or test a replacement
firmware, which runs in Tier 1.

## 8. Tier 2b plan: the unit's kernel modules

- Modules: vermagic `4.4.126 SMP preempt mod_unload modversions aarch64`, with `__versions`
  CRCs (`xdma.ko`: 103 symbols). Loading needs a 4.4.126 kernel built from rockchip-linux
  `develop-4.4` with a configuration whose exported-symbol CRCs match (the unit's kernel
  configuration is not available; CRC mismatches are reported per symbol by the loader and
  indicate which structures differ), or a kernel with `CONFIG_MODULE_FORCE_LOAD`.
- `xdma.ko` is a generic PCI driver (vendor 0x10EE): it can bind on `qemu-system-aarch64 -M virt`
  (generic ECAM host) to an emulated endpoint: a QEMU PCIe device with the XDMA config BAR
  (engine identifiers, SG descriptor fetch, write-back status for `poll_mode=1`) and the bypass
  BAR, whose register accesses and DMA are forwarded to the daemon. `pcie-rockchip.ko` is
  RK3399-specific and is not needed on `virt`.
- The GPIO bit-bang modules (`fpga_gpio`, `spi2pll_gpio`, `spi2afe_gpio`, …) need RK3399 GPIO
  banks; on `virt` they would need a gpio-mockup/pl061 mapping of the same GPIO numbers.
- Effort: the XDMA endpoint model is the large item; the unit's modules and software above it run
  unchanged.
