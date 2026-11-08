# 12 — FPGA Board Interface

Everything an FPGA image needs in order to connect to the rest of the DHO1000 board: the device,
its configuration options, clocks, and every connected ball with its function, direction, I/O
standard and capture/launch structure. This covers the single-ADC 2 GSa/s board (DHO1000 /
"S2" family). Differences on the dual-ADC board are summarised in §10. The register map an image
presents over PCIe is not part of this interface (`02-register-map.md`); the PCIe link itself is
detailed in `14-pcie-dma-interface.md`; how the CPU loads the image is `13-fpga-configuration.md`;
the ADC lane capture and training logic is `20-adc-capture.md`.

Ball names are FGG676 package balls. "P/N" pairs are the true/complement balls of one
differential pair; for differential signals the P ball is listed first.

---

## 1. Device

| property | value |
|---|---|
| Part | Artix-7 **XC7A100T-FGG676** or **XC7A75T-FGG676** (board variant; same die, same pinout) |
| IDCODE | XC7A100T `0x03631093`, XC7A75T `0x03632093` |
| Speed grade | **-3** (marking **UNVERIFIED**): the DDR3 memory clock of §5 is 500 MHz (2000 ps), which the Xilinx 7-series memory controller accepts only on -3 (limit 2500 ps on -1/-2), and PCIe Gen2 needs -2 or better |
| Configuration | volatile, slave serial from the CPU at every boot (§2) |
| Bitstream | uncompressed, 3,825,788 bytes of configuration data (955,864 FDRI words) |

The XC7A75T is the XC7A100T die with lower resource totals: site grid, package pinout and configuration
frame layout are identical. An S2-family image must stay within the XC7A75T totals: **47,200 LUTs,
94,400 flip-flops, 105 RAMB36 tiles, 180 DSP48E1**. Placement is not restricted to part of the die; any
slice, BRAM or DSP site may be used. For a design within those totals the XC7A75T bitstream is
the XC7A100T bitstream with the IDCODE word set to `0x03632093` and the next CRC word recomputed
(spec 13 §3). The header part string is not used by the device.

### 1.1 Bank supplies

| bank | VCCO | use |
|---|---|---|
| 13 | **1.5 V** | DDR3 data (SSTL15) |
| 14 | **1.5 V** | DDR3 address/command/clock (SSTL15); configuration DIN (R15) |
| 15 | **UNVERIFIED** (1.5 V on the dual-ADC board, where it carries DDR3 data; treat it as unknown here and use its balls only as inputs) | no function on this board (DDR3 data on the dual-ADC board) |
| 16 | **2.5 V** | ADC lanes, reference clock, LVDS output, low-speed control |
| 34 | **2.5 V** | ADC lanes |
| 35 | **2.5 V** | ADC lanes and ADC data clock |
| 216 (GTP) | MGTAVCC/AVTT | PCIe ×2 |
| 213 (GTP) | — | unused |

Banks 13/14 are 1.5 V because the DDR3 I/O is SSTL15 (not SSTL135) and RESET# is LVCMOS15; bank 14
also carries the configuration DIN (R15). In
serial mode UG470 (Table 2-6) requires VCCO_14 to equal the configuration interface voltage (3.3,
2.5, 1.8 or 1.5 V, the same as VCCO_0); with VCCO_14 = 1.5 V this means VCCO_0 = 1.5 V and
CFGBVS = GND, so the RK3399 SPI1/GPIO0 configuration lines reach the FPGA at 1.5 V levels or
through level shifters. The memory parts may be DDR3L devices run at 1.5 V. VCCO_0, VCCO_14 and
CFGBVS are **UNVERIFIED** by measurement.

The DDR3 banks use external VREF: W18 and W24 (bank 13) are VREF balls and must be left as inputs.
F24 and M17 (bank-15 VREF balls) and F25, G25, J25, J26 must be left as inputs (§8).

### 1.2 Image variants

The bitstream files the boot can load (spec 13 §2) and which sections of this spec (and of specs 13,
14, 20, 21) apply to each. "Holds" = the section applies as written.

| | `SPU_H12S2.bit` | `SPU_H12S2B.bit` | `SPU_H12S2_bode.bit` = `SPU_H12S2_SelfTest.bit` | `SPU_H12S4.bit` | `SPU_H12S4B.bit` |
|---|---|---|---|---|---|
| part / HW code | XC7A100T / `0x1`, `0x3` | XC7A75T / `0x2`, `0x9` | XC7A100T / DDR self-test reload (spec 13 §6) | XC7A100T / `0x0` | XC7A75T / `0x4` |
| relation to `SPU_H12S2` | — | same configuration frames; IDCODE and CRC word #1 differ | different design | different design (2 ADC) | different implementation of the 2-ADC design |
| §1–§2, spec 13 §3 (device, configuration options, stream) | holds | holds (IDCODE `0x03632093`) | holds | holds | holds (IDCODE `0x03632093`) |
| §3 clocks | holds | holds | holds (same MMCM/PLL settings) | §10 | §10 |
| §4 ADC lanes | holds | holds | holds (N3 differs) | §10 | §10 (lane I/O as `SPU_H12S4`) |
| §5 DDR3 | holds | holds | holds; L22 unused | §10 | §10 |
| §6 PCIe, spec 14 §2 endpoint | holds | holds | **BAR1 = 16 MiB**, slot-clock bit set (spec 14 §2) | holds | holds |
| §7–§9 (low-speed, reserved, internal) | holds | holds | A25/B25/C22/C23 bidirectional; rest not established | §10 | as `SPU_H12S4` at ball level |
| specs 20, 21 and the FPGA behaviour behind specs 02–05 | holds | holds | not established | not established | not established |

An open 1-ADC image that is to run on every DHO1000 unit is needed in both IDCODE variants (selected
by HW code, spec 13 §2.2). If the design stays within the XC7A75T totals (§1), the two files differ
only in the IDCODE word and CRC word #1.

---

## 2. Configuration

The CPU loads the bitstream in **slave serial** mode (DIN on R15 `IO_L1N_T0_D01_DIN_14`, CCLK
driven by the CPU); the protocol is in `13-fpga-configuration.md`. An image must:

* keep **R15** unused as user I/O (it is the CPU's DIN line);
* not use encryption (`CTL0.DEC = 0`) and not set readback security;
* use these startup options or equivalent:

| register | value | meaning |
|---|---|---|
| COR0 | `0x02003FE5` | startup clock CCLK; GWE cycle 6, GTS cycle 5, DONE cycle 4, DONE pipe on, DONE not actively driven, no wait for MMCM lock or DCI match |
| COR1 | `0x00000000` | |
| CTL0 | `0x00000501` | GTS_USR_B=1, PERSIST off (configuration pins become user I/O), GLUTMASK_B=1, fallback disabled (bit 10 = 1), no AES |
| CTL1 | `0x00000000` | |
| TIMER, WBSTAR | 0 | no watchdog, no warm boot |
| USERID | `0xFFFFFFFF` | |

The bitstream is not compressed and carries a CRC. The bus-width detection words
`0x000000BB 0x11220044` precede the sync word; they are ignored in serial mode.

### 2.1 Configuration pins

| pin | connection | notes |
|---|---|---|
| M[2:0] | board straps | must select slave serial (`111`); strapping **UNVERIFIED** (check the straps on the board) |
| CCLK | RK3399 SPI1 clock | driven by the CPU, only while data is sent; 10 MHz normal, 49 MHz on some load paths (spec 13 §4) |
| D01_DIN (R15) | RK3399 SPI1 MOSI | MSB of each byte first; no readback, MISO unused |
| PROGRAM_B | RK3399 GPIO 2 (GPIO0_A2), output | high except during the reset pulse |
| INIT_B | RK3399 GPIO 1 (GPIO0_A1), input | |
| DONE | RK3399 GPIO 3 (GPIO0_A3), input | |
| CSI_B (P19) | DDR3 address/command output of the image | not a configuration input in slave serial; it is wired to the memory, so the SPI1 chip select cannot be on it. R14 (D00_MOSI), P18 (FCS_B), N23 (RDWR_B) and N24 (DOUT_CSO_B) are DDR3 address/command outputs as well |
| PUDC_B (P15) | board strap | §8 |

Pull-ups and level shifting on the configuration lines are **UNVERIFIED**. The four board-ID straps
that choose the image (`/dev/hdcode_gpio`, spec 13 §2) are CPU GPIOs, not FPGA balls.

Because the CPU stops CCLK when the file ends, an image must keep `StartUpClk = CCLK`, no wait for
MMCM lock or DCI match during startup, and trailing NOOP padding (spec 13 §3).

### 2.2 State after configuration

The FPGA leaves startup **before** the CPU programs the LMK04821 clock chip and before the PCIe root
port is loaded (spec 01 §4). At that time the reference clock (§3.1) and the ADC data clock may be
absent or unstable. An image must hold all logic clocked from the reference clock, the DCO and the
DDR3 PLL/MMCM in reset until the corresponding PLL/MMCM reports lock, and must tolerate the
reference clock starting after configuration. The PCIe reference clock (100 MHz) does not come from
the LMK04821 (it has no 100 MHz output, spec 06 §4.4); whether it runs before configuration ends is
**UNVERIFIED**.

---

## 3. Clocks

### 3.1 Clock inputs

| balls (P/N) | signal | standard | rate | use |
|---|---|---|---|---|
| D19 / C19 (`IO_L13P/N_T2_MRCC_16`) | reference clock | LVDS_25 in | **125 MHz** (LMK04821 CLKout2; frequency **UNVERIFIED**, measure) | global buffer; input of the reference PLL (§3.2); clock of the LVDS output (§7) |
| G5 / F5 (`IO_L12P/N_T1_MRCC_35`) | ADC data clock (DCO) | LVDS_25 in | **500 MHz** | I/O clocks of the ADC lanes (§4) |
| F11 / E11 (`MGTREFCLK0P/N_216`) | PCIe reference clock | GTP refclk | **100 MHz** | GTP quad 216 PLL0 (§6); source not the LMK04821 (RK3399 or a separate oscillator: **UNVERIFIED**) |

The reference clock is 125 MHz because the reference PLL's ×8/5 output (§3.2) is the 200 MHz
IDELAYCTRL reference. It is the clock chip's divide-by-16 output of its 2 GHz VCO
(`06-vertical-and-analog.md` §4.4). No FPGA input is known to carry the clock chip's 200 MHz
output (CLKout4) or a SYSREF/SYNC copy. Whether an unused ball is wired to them, and the
consumer of CLKout4, are **UNVERIFIED**. The ADC sample clock (2 GHz) goes from the clock chip directly to the ADC; the FPGA
sees it only as the 500 MHz DCO.

### 3.2 Generated clocks

A clock plan that produces every frequency the board interfaces need (**UNVERIFIED** on a unit):

| block | input | VCO | outputs | use |
|---|---|---|---|---|
| reference PLL | reference clock 125 MHz | 1000 MHz (M = 8, D = 1) | **200 MHz** (÷5); **20 MHz** (÷50) | IDELAYCTRL reference (all four banks) and DDR3 controller input; slow serial outputs (§7) |
| DDR3 PLL | 200 MHz | 1000 MHz (M = 5) | 500 MHz ×2 (÷2, memory clock / phaser reference); 31.25 MHz (÷32, 2/30 duty, phaser sync); 125 MHz (÷8) | DDR3 PHY (§5) |
| DDR3 MMCM | 125 MHz (DDR3 PLL) | 1000 MHz (M = 8) | **125 MHz** (feedback output) = controller user clock; 62.5 MHz (÷16) | DDR3 controller and its user side |
| PCIe pipe MMCM | GTP TXOUTCLK 100 MHz | 1000 MHz (M = 10) | 125 MHz and 250 MHz (pipe clock, selected per link speed); **125 MHz** user clock | PCIe core and the register/DMA side |
| ADC regional | DCO 500 MHz | — | BUFIO 500 MHz; BUFR ÷2 = **250 MHz** | ISERDES CLK / CLKDIV in banks 34, 35 |
| acquisition clock | BUFR 250 MHz → global buffer | — | **250 MHz** | all acquisition logic (the "4 ns" FPGA clock) |
| bank-16 lane MMCM | acquisition clock 250 MHz | 1000 MHz (M = 4) | 500 MHz (÷2), 250 MHz (÷4) on global buffers | ISERDES CLK / CLKDIV of the four bank-16 lanes |

Clock domains seen at the balls: acquisition 250 MHz (ADC lanes, A18, the A25/B25/C22/C23/E20
group), PCIe user 125 MHz (most low-speed control, §7), 20 MHz (slow serial group, §7),
reference clock 125 MHz (LVDS output). The reference clock and the ADC data clock have no fixed
phase relationship visible to the FPGA.

---

## 4. ADC interface

One ADC (RT8847IQ, `06-vertical-and-analog.md`) delivers **28 LVDS data lanes** in two groups
(14 lanes each) plus a data clock. Every lane runs at **1 Gb/s DDR** against the 500 MHz DCO, so
the lanes together carry 28 Gb/s = 2 GSa/s × 14 bits.

### 4.1 Capture

For every lane the image must:

* receive with `IBUFDS` LVDS_25 (100 Ω internal termination, `DIFF_TERM = TRUE`, `IBUF_LOW_PWR =
  FALSE`; the same on the DCO G5/F5 and on N3/N2);
* pass the P-side signal through an **IDELAYE2**, `IDELAY_TYPE = VAR_LOAD`, initial value 0,
  `HIGH_PERFORMANCE_MODE = TRUE`, 32 taps at a 200 MHz reference (78 ps per tap), `DELAY_SRC =
  IDATAIN` with **no inversion** (`IS_IDATAIN_INVERTED = 0`; no lane is inverted anywhere in the
  capture path, `20-adc-capture.md` §2). `C` = the lane's CLKDIV clock, `CE = INC = 0`; tap value and load
  strobe come from registers. The host calibrates the tap of each lane at run time
  (`20-adc-capture.md` §5, `10-calibration.md`);
* deserialise with an **ISERDESE2**, `NETWORKING`, `DATA_RATE = DDR`, `DATA_WIDTH = 4`, master,
  `IOBDELAY = BOTH`, `NUM_CE = 2` (both CE high), `BITSLIP` tied 0, CLK = the 500 MHz I/O clock,
  CLKB = inverted CLK, CLKDIV = the 250 MHz clock. Each lane then yields 4 bits per 4 ns
  acquisition clock: 112 bits = 8 samples × 14 bits per clock.

The complete per-lane primitive settings, bit/sample mapping and training registers are in
`20-adc-capture.md` §2–§5.

Clocking per bank: lanes in bank 35 and 34 use their bank's BUFIO (500 MHz) and BUFR ÷2; the bank-34
BUFIO/BUFR are fed from the same DCO (via the multi-region clock buffer: **UNVERIFIED**). The four
bank-16 lanes are clocked from the bank-16 lane MMCM (§3.2), which re-creates 500/250 MHz from the
acquisition clock.

### 4.2 Lane balls

All lanes are LVDS_25 inputs, listed as P ball / N ball.

| bank | lanes (P / N) |
|---|---|
| 35 (23) | A3/A2, B4/A4, B5/A5, C1/B1, C2/B2, D3/C3, D4/C4, E1/D1, E5/D5, E6/D6, F2/E2, F3/E3, F8/F7, G2/G1, G4/F4, H6/G6, H7/G7, H8/G8, H9/G9, J4/H4, J6/J5, K7/K6, L8/K8 |
| 34 (1) | M7/L7 |
| 16 (4) | B20/A20, B22/A22, C21/B21, E21/D21 |
| 34 (extra) | **N3/N2**: the **EXT trigger comparator** input. Deserialised like a lane (same ISERDES settings, bank-34 clocks) but **without IDELAY** and not inverted; it never enters the sample words. Its 4 bits per 250 MHz cycle (one per 1 ns slot) are the EXT comparator state, trigger source index `0x31` (spec 05 §3.4, spec 21 §5.5). An image that supports the EXT trigger must receive it. |

The lanes form two groups of 14; the lane index within a group is the ADC code bit. The full
ball-to-group/bit table is `20-adc-capture.md` §1. In short: the bank-16 lanes are bits 12 and 13
of both groups (B20 = group 0 bit 13, E21 = group 0 bit 12, C21 = group 1 bit 13, B22 = group 1
bit 12), M7 (bank 34) is group 0 bit 0, and bits 0 and 1 of both groups (M7, H7, H6, G4) are
dropped when samples are trimmed to 12 bits. The ADC is configured over an SPI bridge implemented
in the FPGA (§7.1).

---

## 5. DDR3 memory

A **32-bit DDR3** interface (1.5 V I/O; Xilinx 7-series memory PHY, four byte lanes) on banks 13
and 14: two ×16 devices. The software addresses the record memory as **1 GiB** = 2^25 bursts of
32 bytes (`04-timebase-and-bands.md` §1, §7), which on a 32-bit bus means two **4 Gb ×16** devices
(256M×16-class, 15 row address bits A0–A14, consistent with the 23 address/command outputs of
§5.2). Chip type (DDR3 or DDR3L run at 1.5 V) and markings **UNVERIFIED**. One 32-byte burst
(one 4:1 PHY word: 8 beats × 32 bits) holds 20 samples × 12 bits; packing inside the burst is
internal to the image.

| property | value |
|---|---|
| Memory clock | **500 MHz** (DDR3-1000) |
| PHY ratio | 4:1; controller user clock **125 MHz** |
| Data | 32 DQ, 4 DQS pairs, **no DM** |
| Address/command | 23 single-ended outputs + differential CK + RESET# |
| I/O standard | SSTL15 (DQ, addr/cmd), DIFF_SSTL15 (DQS, CK), full-strength fixed drive, fast slew |
| DQ/DQS termination | IN_TERM `UNTUNED_SPLIT_50` (on-die, 50 Ω split) |
| DQ input | IDELAYE2 `VARIABLE`, high performance; ISERDES `MEMORY_DDR3` 1:4; OSERDES DDR 4:1 with tristate |
| Addr/cmd output | OSERDES SDR 4:1 |
| VREF | external (W18, W24) |

### 5.1 Data balls (bank 13)

| byte lane (bank tile) | DQS P/N | DQ balls |
|---|---|---|
| T0 | AB26 / AC26 | U25 U26 V26 W26 W25 Y26 Y25 AA25 |
| T1 | AB24 / AC24 | V23 W23 Y22 Y23 AA22 AA23 AA24 AB25 |
| T2 | T20 / U20 | T19 U19 U21 V21 W20 Y20 W21 Y21 |
| T3 | T17 / T18 | T14 T15 U14 V14 U15 U16 V16 V17 |

Which byte lane is memory byte 0, and the DQ bit order inside a lane, are **UNVERIFIED** (any
order works for a memory interface as long as each DQ stays in its DQS lane).

### 5.2 Address/command balls (bank 14)

| balls | signal |
|---|---|
| N21 / N22 (`IO_L13P/N_T2_MRCC_14`) | CK / CK# |
| P24 | RESET# (LVCMOS15, 12 mA, fast) |
| K25 K26 L20 L22 L25 M20 M24 M25 N14 N16 N19 N23 N24 P14 P18 P19 P21 P23 R14 R16 R17 R20 R21 | A0–A14, BA0–BA2, RAS#, CAS#, WE#, CKE, ODT (23 signals; there is no CS# output). Ball-to-signal assignment **UNVERIFIED** |

---

## 6. PCIe

| property | value |
|---|---|
| Link | **Gen2 (5 GT/s) ×2**, endpoint, lane reversal disabled, up-configure capable |
| Transceivers | GTP quad 216, channels 0 and 1 (PIPE mode, 20-bit, 8b/10b) |
| Reference | MGTREFCLK0_216 (F11/E11), 100 MHz; quad PLL0 VCO 2.5 GHz |
| User clock | 125 MHz, 64-bit interface |
| BAR0 | 64 KiB, 32-bit, non-prefetchable memory (DMA engine control) |
| BAR1 | 32 KiB, 32-bit, non-prefetchable memory (DMA bypass window = the FPGA registers) |
| BAR2–BAR5, expansion ROM | disabled |
| Class code | `0x070001` |
| Max payload | 512 bytes |
| Interrupts | MSI (64-bit address, 1 vector) and legacy INTA; MSI-X off |
| Capabilities | PM, MSI, PCIe, DSN, VSEC |
| Vendor/device ID | `0x10EE` / `0x7022`; subsystem `0x10EE` / `0x0007`; revision `0x00` (see `14-pcie-dma-interface.md` §1) |

| signal | P | N |
|---|---|---|
| channel 0 TX | B7 | A7 |
| channel 0 RX | B11 | A11 |
| channel 1 TX | D8 | C8 |
| channel 1 RX | D14 | C14 |

PCIe **lane 0 = channel 0** (GTPE2_CHANNEL_X0Y4), **lane 1 = channel 1** (X0Y5). Lane reversal is
disabled, so the board routes lane 0 to the root port's lane 0. Quad 213 and channels 2/3 of quad 216
are unused.

---

## 7. Low-speed I/O

All balls in this section are in 2.5 V banks. Single-ended outputs are **LVCMOS25, 12 mA, slow
slew**; inputs are LVCMOS25. Functions are presumed from the clock domain and the registers that
control them, and are **UNVERIFIED** unless noted.

### 7.1 Control serial port (PCIe-user clock domain)

| ball | dir | structure | presumed function |
|---|---|---|---|
| F19 | in/out | bidirectional | serial data (bidirectional); output enable set by the first shifted bit (bit 23 = read) |
| F18 | out | | serial clock (**UNVERIFIED**) |
| G15 | out | `0x1034` bit 24 | chip select of ADC 0, low during a frame |
| F15 | out | `0x1034` bit 25 | chip select of ADC 1 (no second ADC on this board), low during a frame |

This 4-wire port with one bidirectional data line is the FPGA's ADC configuration bridge
(register `0x1034`/`0x1038`; frame format in `20-adc-capture.md` §6 and `02-register-map.md`,
ADC registers in `06-vertical-and-analog.md` §8).

### 7.2 Slow serial group (20 MHz domain)

Eight outputs in the 20 MHz domain (reference PLL ÷50):

| ball | bank |
|---|---|
| B24, B26, C24, D23, D24 | 16 |
| H3, J8 | 35 |
| N8 | 34 |

**B24 carries a gated 20 MHz clock** (presumed serial clock); the other seven are data and select
lines in the same domain. Presumed: the serial interface to the FPGA-controlled DACs (probe/AFE offsets, external-trigger
level; registers `0x1404`–`0x1418`, refresh period `0x1428`). **UNVERIFIED**.

### 7.3 Acquisition-domain lines

| ball | dir | notes |
|---|---|---|
| A25, B25, C22, C23, E20 | out | one signal in the 250 MHz domain on all five; A25, B25, C22, C23 with the weak pull-up enabled, **E20 without**. On the dual-ADC board (and with the Bode image) A25/B25/C22/C23 are bidirectional with separate drivers and tristate control (E20 stays an output). Function **UNVERIFIED**; an image should drive them as one signal and keep the pull-ups on A25/B25/C22/C23 until identified |
| A18 | in | **AC-line (mains-sync) trigger** input, trigger source index `0x30` (spec 05 §3.1, spec 21 §5.5): asynchronous. An image synchronises it into the 250 MHz domain and accepts an edge only if the previous level lasted ≥ 4096 cycles (16.384 µs), which rejects noise on a slow mains-derived signal. The board circuit that drives it is **UNVERIFIED** |

### 7.4 Other register-controlled lines

| ball | dir | notes |
|---|---|---|
| A17 | out | one of three acquisition-domain (250 MHz) signals, chosen by three register bits (presumed trigger/aux output with source select; register **UNVERIFIED**) |
| C26 | out | register-controlled output, PCIe-user domain; function **UNVERIFIED** |
| D20 | out | register-controlled output, PCIe-user domain; function **UNVERIFIED** |
| E22 | in | **active-low** asynchronous reset of the PCIe-side logic (PCIe user and pipe clock domains, with synchronised release). Consistent with PCIe **PERST#** (spec 14 §1). Which RK3399 GPIO drives it is **UNVERIFIED** (DT `ep-gpios`) |

Output candidates from the register map: probe power (`0x142C`), auxiliary output (`0x1430`). **UNVERIFIED**.

### 7.5 LVDS output

| balls (P/N) | standard | structure |
|---|---|---|
| G17 / F17 (`IO_L2P/N_T0_16`) | LVDS_25 | a **gated copy of the 125 MHz reference clock** (or a strobe aligned to it), from an output DDR register in the reference-clock domain. Presumed ADC synchronisation input. **UNVERIFIED**; spec 06 §4.4 presumes LMK04821 CLKout7/CLKout11 as the ADC synchronisation inputs; which signal reaches which ADC pin is board routing (**UNVERIFIED**) |

---

## 8. Reserved and unused balls

An image should leave these as inputs with no pull:

| balls | reason |
|---|---|
| W18, W24 | bank 13 VREF |
| F24, M17 | bank 15 VREF |
| F25, G25, J25, J26 | bank 15, no function on this board |
| H17 (bank 16), U4 (bank 34) | no function on this board |
| P15 (`PUDC_B`) | configuration pull-up select pin; input with pull-up |

R15 (D01_DIN, §2) is unused by the image: after configuration it has the tool-default weak
pull-down. Do not drive it.

All other user balls are unused. Their state (tool default weak pull-down) is **UNVERIFIED** as
safe; do not drive them.

---

## 9. Internal resources an image may rely on

* XADC: available on the dedicated VP/VN balls N12/P11 only. The auxiliary analog inputs are not
  available (their balls are ADC lanes).
* Nothing on the board needs a soft processor, ICAP or STARTUPE2 (**UNVERIFIED**).

---

## 10. Dual-ADC board (S4 family) differences

* **64-bit DDR3** (SSTL15 like the S2 board): four more byte lanes: bank 14 T3 (DQ M26 N26 P26 R22 R23 R26 T22 T23, DQS
  T24/T25) and bank 15 T0 (J14 J15 J16 K15 L14 L15 M14 M15, DQS K16/K17), T1 (G20 G21 H18 H19 J18
  J19 L17 L18, DQS K20/J20), T2 (E23 F23 H21 H22 H23 H24 J23 J24, DQS G22/F22). Bank 14 then also
  needs its VREF balls N18 and P25 reserved; the address/command balls are unchanged.
* **57 lanes** (two ADCs × 28 + N3/N2): banks 35 (23), 34 (24), 16 (10). The second ADC's DCO is on
  D18/C18 (`IO_L12P/N_T1_MRCC_16`); D18 and G5 take a fixed IDELAY of 12 taps (**UNVERIFIED**). Bank 16 lanes use their
  own DCO, so the bank-16 lane MMCM is absent.
* A second gated reference-clock LVDS output on B19/A19 (`IO_L10P/N_T1_16`).
* PCIe, clocks, configuration options and the low-speed groups are the same; the A25/B25/C22/C23
  lines are bidirectional.
* The A75T image of the 2-ADC board (`SPU_H12S4B.bit`) uses the same balls, I/O standards, ADC lane
  I/O, PCIe and PLL configuration as `SPU_H12S4.bit`; whether the DDR3 DQ inputs need on-die
  termination there is **UNVERIFIED**.
