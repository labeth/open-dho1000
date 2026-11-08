# 13 — FPGA Configuration

This spec defines how the CPU selects and loads the FPGA bitstream: the board HW code, the image
table, the slave-serial transport over RK3399 SPI1, the PROGRAM_B / INIT_B / DONE handshake, timing,
retries, the post-configuration steps, and runtime reconfiguration. The FPGA is volatile; it is
configured on every boot (spec 01 §4 step 6). The PCIe side that follows is in spec 14, the clock chip
in spec 06. Nothing here has been measured on a unit: every value is **UNVERIFIED** until it is.

---

## 1. Interfaces

| Signal / node | Implementation |
|---|---|
| CCLK, D01_DIN | RK3399 SPI1, `/dev/spidev1.0` (kernel spidev). SPI mode 0, 8 bits per word, MSB first. Only MOSI is used; it drives FPGA ball R15 `IO_L1N_T0_D01_DIN_14` (spec 12 §2.1). |
| PROGRAM_B | GPIO 2, output, `/dev/spi2k7_ctl` |
| INIT_B | GPIO 1, input, `/dev/spi2k7_ctl` |
| DONE | GPIO 3, input, `/dev/spi2k7_ctl` |
| Board HW code | GPIO 7, 8, 11, 12, `/dev/hdcode_gpio` |

The configuration mode is **Xilinx 7-series slave serial**: one data bit per CCLK rising edge,
each byte MSB first, which is exactly SPI mode 0 MSB-first output. Mode pins M[2:0] are strapped on
the board (expected `111`, UNVERIFIED). The FPGA-side pin list is in spec 12 §2.1.

### 1.1 `/dev/spi2k7_ctl` (from `fpga_gpio.ko`)

Loading the module requests the three GPIOs, sets INIT_B and DONE as inputs and drives PROGRAM_B
**high** (no reset on insmod).

| Operation | ioctl code | Argument | Effect |
|---|---|---|---|
| read INIT_B | `0x80047801` (`_IOR('x',1,int)`) | `int *` | stores the INIT_B level (0/1) |
| set PROGRAM_B | `0x00007802` (`_IO('x',2)`) | the value itself, 0 or 1 | drives PROGRAM_B; values ≥ 2 → `-EINVAL` |
| read DONE | `0x80047803` (`_IOR('x',3,int)`) | `int *` | stores the DONE level (0/1) |

Any other request returns `-EINVAL`. `read()` and `write()` on this node are not part of the
interface and must not be called. The module enables no clock; no configuration step turns on an
FPGA clock.

### 1.2 `/dev/hdcode_gpio` (from `hdcode_gpio.ko`)

`read(fd, &b, 1)` returns one byte `b = hd0 | hd1<<1 | hd2<<2 | hd3<<3` with hd0..hd3 = GPIO 7, 8,
11, 12. Bits 4..7 are 0. The value is fixed by board straps and sampled on every `read()`. Read
exactly 1 byte; longer reads are not safe.

## 2. Bitstream selection

### 2.1 Images

| File | Device | IDCODE | Product | Purpose |
|---|---|---|---|---|
| `SPU_H12S4.bit` | XC7A100T | `0x03631093` | DHO4000 (2 ADC, 4 GSa/s) | normal |
| `SPU_H12S4B.bit` | XC7A75T | `0x03632093` | DHO4000 | normal |
| `SPU_H12S2.bit` | XC7A100T | `0x03631093` | DHO1000 / DHO2000 (1 ADC, 2 GSa/s) | normal |
| `SPU_H12S2B.bit` | XC7A75T | `0x03632093` | DHO1000 / DHO2000 | normal |
| `SPU_H12S4_SelfTest.bit`, `SPU_H12S4B_SelfTest.bit`, `SPU_H12S2_SelfTest.bit` | as named | | | chip/board self-test incl. DDR calibration read-back |
| `SPU_H12S4_bode.bit`, `SPU_H12S2_bode.bit` | A100T | | | Bode variant without high-resolution mode |

There is no `SPU_H12S2B_SelfTest.bit` and no A75T Bode image: the 1-ADC Bode/self-test design does
not fit the XC7A75T. All images have the same configuration payload length (955 864 FDRI words,
the full frame set of the die); every A75T image carries IDCODE `0x03632093`, every A100T image
`0x03631093`.

- `SPU_H12S2B.bit` has the same configuration frames as `SPU_H12S2.bit`; only the IDCODE word and
  CRC word #1 (§3) differ. A75T and A100T 1-ADC boards therefore run the same design (spec 12 §1.2).
  `SPU_H12S4B.bit` is a separate implementation of the 2-ADC design.
- `SPU_H12S2_SelfTest.bit` is byte-identical to `SPU_H12S2_bode.bit`.

### 2.2 Version table `/rigol/resource/conf/version.txt`

The boot-time loader `spi2fpga` (spec 01 §4 step 6) picks the image from this table. Format, as the
loader needs it:

- one entry per line, `<code>@<file>`, `<code>` in hex (`0x..`), `<file>` relative to `/rigol/FPGA/`;
- every line ends with LF only (a CR before the LF becomes part of the file name, which then cannot
  be opened); lines of length ≤ 1 are skipped;
- file names have at most 35 characters;
- the **first** line whose code equals the HW code wins.

The table as shipped:

| HW code | Image |
|---|---|
| `0x0` | `SPU_H12S4.bit` |
| `0x4` | `SPU_H12S4B.bit` |
| `0x2` | `SPU_H12S2B.bit` |
| `0x3` | `SPU_H12S2.bit` |
| `0x1` | `SPU_H12S2.bit` |
| `0x9` | `SPU_H12S2B.bit` |

If no line matches, or the table or `/dev/hdcode_gpio` cannot be opened, the loader uses
`/rigol/FPGA/SPU_H12S2.bit`. On an A75T board this default does not configure (IDCODE mismatch,
DONE stays low).

The HW code is per unit. Codes `0x2` and `0x3` are the 4-channel 1-ADC board and codes `0x1` and
`0x9` the 2-channel 1-ADC board, so a DHO1074 has code `0x2` (`SPU_H12S2B.bit`, XC7A75T) or `0x3`
(`SPU_H12S2.bit`, XC7A100T) (**UNVERIFIED**; which is more common is unknown). An open image must
build for both parts. The HW code can be read with `dd if=/dev/hdcode_gpio bs=1 count=1`; the
scope app's system-information page probably shows it in decimal (**UNVERIFIED**).

## 3. Bitstream file

- The `.bit` file is sent **unmodified, header included**. The FPGA ignores everything before the
  sync word; the header contains no sync pattern. No bit reversal, no byte swapping, no stripping.
- Equivalent: send only the configuration stream starting at the first `0xFFFFFFFF` dummy word
  (bus-width pattern, sync word `0xAA995566`, packets) through the end of the file.
- Configuration options an image must use (spec 12 §2): unencrypted, no bitstream compression, CRC
  enabled, `COR0 = 0x02003FE5` (startup clock = **CCLK**, DONE released in phase 4, GTS phase 5,
  GWE phase 6, no wait for DCM/MMCM lock or DCI match, DONE_PIPE on), `COR1 = 0`,
  `CTL0 = 0x00000501` (GTS_USR_B = 1, PERSIST = 0, SBITS = 0, DEC = 0, GLUTMASK_B = 1, bit 10 = 1 =
  fallback disabled, the Vivado default; fallback only exists in master modes), `CTL1 = 0`,
  `TIMER = 0`, `WBSTAR = 0`, `USERID = 0xFFFFFFFF`.
- The packet sequence is the standard 7-series one (Xilinx UG470): register writes including
  `IDCODE`, then `FDRI` with 955 864 words (9464 frames), a CRC check, `GRESTORE`, `LFRM`, `START`,
  a second CRC check, `DESYNC`, and NOOP padding (400 words, 1600 bytes, is enough). The CRC is the
  7-series configuration CRC (CRC-32C, reflected, over the 32 data bits plus the 5-bit register
  address of every register write; reset by `RCRC` and after each check). CRC word #1 covers every
  register write after `RCRC` up to the end of FDRI, including the IDCODE write: changing the
  IDCODE for the other part (spec 12 §1) means recomputing CRC word #1.
- Because the startup sequence is clocked by CCLK and the host only toggles CCLK while sending, the
  trailing NOOP padding is what clocks the FPGA through startup. **A self-built image must use
  `StartUpClk = CCLK`, no `LCK_cycle` / `Match_cycle` wait, and keep at least 400 words of trailing
  padding**; otherwise DONE may never rise.

## 4. Loading procedure

1. Open `/dev/spidev1.0` (O_RDWR) and `/dev/spi2k7_ctl` (O_RDWR).
2. Select the image (§2).
3. **Reset**: set PROGRAM_B 0; wait ≥ 1 µs (the device minimum is 250 ns); set PROGRAM_B 1.
4. **Wait for INIT_B**: poll INIT_B, sleeping 1 ms after each poll, until it reads 1; give up
   after about 5 s. On timeout the load has failed.
5. **Configure SPI**: bits per word 8 (`SPI_IOC_WR_BITS_PER_WORD` `0x40016B03`), mode 0
   (`SPI_IOC_WR_MODE` `0x40016B01`), clock (`SPI_IOC_WR_MAX_SPEED_HZ` `0x40046B04`). 10 MHz takes
   ≈ 3.1 s per image, 49 MHz ≈ 0.63 s; any rate up to the slave-serial limit of the part should work
   electrically (**UNVERIFIED** on the board's wiring above 10 MHz; use 10 MHz for retries).
6. **Stream**: read the whole file (≤ 16 MiB) into memory and `write()` it to spidev in chunks of
   **4096 bytes** (the default spidev `bufsiz`; larger writes fail), then the remainder. Check every
   `write()` return.
7. Wait **10 ms**.
8. **Check DONE**. 1 = configured.
9. On DONE = 0: repeat from step 3 (reset) at 10 MHz, at most 3 attempts in total. The reset is
   required: after a CRC or IDCODE error the FPGA ignores further data until PROGRAM_B is pulsed.
10. Close both fds.

INIT_B going low during streaming signals a configuration CRC error; an implementation may poll it
between chunks and abort early.

A replacement firmware that relies on the boot-time load must read DONE itself before it uses the
FPGA; the boot continues whether or not configuration succeeded.

### 4.1 Boot-time loaders

On OS ≥ 1.1.1 the boot script runs `spi2fpga` with the version table (§2.2); on older OS images it
runs `spi2fpga_v1` with the fixed path `SPU_H12S4.bit`, which is the wrong image for a DHO1000
(spec 01 §4, §9). A replacement that loads its own image uses §4 directly.

## 5. After configuration

1. Program the LMK04821 (spec 06). The FPGA runs from this point without its LMK-derived clocks
   until step 1 completes; its design must keep clock-dependent logic in reset until its clocks lock.
2. Insert `pcie-rockchip.ko`: the root port trains the link and enumerates the XDMA endpoint.
3. Insert `xdma.ko` (OS < 1.0.8: `xdma_v1.ko`), wait ≥ 1 s, `chmod 0777 /dev/xdma0_*` (spec 14).
4. Register access through `/dev/xdma0_bypass` is now possible; read the FPGA version and status
   registers first (spec 02) to confirm the image.

## 6. Runtime reconfiguration

Loading another image while the system runs (for example a self-test image for a DDR check) drops
the PCIe endpoint. The sequence that keeps the system usable:

1. Stop acquisition; close all FPGA device handles.
2. `rmmod xdma`, then `rmmod pcie_rockchip` (as root).
3. Load the image with §4.
4. Only if DONE = 1: insert `pcie-rockchip.ko` and `xdma.ko` again (or run
   `/rigol/shell/load_pcie.sh`), then wait for `/dev/xdma0_bypass` and re-open the device handles.
5. If configuration failed, load the normal image (§2) and re-run the PCIe bring-up; otherwise the
   FPGA stays unconfigured and `/dev/xdma0_*` stay absent until reboot.

Facts that matter here:

- The only 1-ADC self-test file, `SPU_H12S2_SelfTest.bit`, is an XC7A100T image: it does not
  configure an XC7A75T board (HW codes `0x2`, `0x9`).
- On XC7A100T boards it is the Bode design (§2.1), a different design from `SPU_H12S2.bit`
  (spec 12 §1.2). Its PCIe endpoint presents BAR1 as 16 MiB instead of 32 KiB (spec 14 §2); software
  that maps `0x8000` bytes still works. Whether it contains DDR self-test logic is **UNVERIFIED**.
- After a self-test image, return to the normal image by reconfiguring again (this section) or by
  rebooting.

Any reconfiguration must follow the unload → configure → reload order (spec 01 §8, spec 14 §1).

## 7. Failure behaviour

| Symptom | Cause |
|---|---|
| INIT_B stays low after the reset pulse | PROGRAM_B not released, power/config rail fault, or INIT_B held low externally |
| DONE = 0 after streaming | wrong image for the part (IDCODE), corrupted file, CRC error, missing trailing padding |
| `/dev/xdma0_*` never appear | FPGA not configured, or `pcie-rockchip.ko` inserted before configuration |
| default image loaded on an A75T board | HW code not in `version.txt` (§2.2) |

A failed configuration does not stop the OS boot: the system comes up without the FPGA.
