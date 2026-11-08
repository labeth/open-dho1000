# 00 — Overview

The device, its major blocks, the scope of these specs, and the glossary. Read
`01-system-architecture.md` next. Nothing in these specs has been checked on a unit yet: every
hardware fact is **UNVERIFIED** until it is.

---

## 1. Device

| property | value |
|---|---|
| Product | Rigol DHO1000 series; reference unit **DHO1074** (4 channels, 70 MHz, 2 GSa/s max) |
| ADC | 12-bit, one ADC package in the S2 boards (2 GSa/s, 1 ch; 1 GSa/s, 2 ch; 0.5 GSa/s, 3–4 ch) |
| CPU | Rockchip RK3399 (2× A72 + 4× A53); RAM and eMMC size UNVERIFIED |
| OS | Android 7.1.2 (API 25), Linux 4.4.126, arm64 |
| FPGA | Xilinx Artix-7 XC7A100T-FGG676 (HW codes 0x1/0x3) or XC7A75T (HW codes 0x2/0x9); same die and pinout; the S2 and S2B images are bit-identical apart from the IDCODE word (and the CRC word that covers it) (spec 12 §1.2, spec 13 §2.1); a DHO1074 is `0x2` or `0x3` |
| FPGA memory | DDR3 (SSTL15 I/O, 1.5 V banks), 32-bit, 1 GiB (2^25 bursts of 32 B; chip type and density UNVERIFIED) (spec 12, spec 04 §1) |
| CPU↔FPGA | PCIe Gen2 ×2, Xilinx XDMA endpoint: register window through the bypass BAR (32 KiB), sample data through one C2H stream engine (spec 14) |
| Display | 10.1" 1280 × 800 touch; waveforms rendered on the CPU GPU (GLES 3.0), not in the FPGA |
| Front panel | keys, 7 encoders, 16 LEDs (19 LED indices; LA, G1, G2 not fitted) behind a panel MCU on `/dev/ttyS0` (spec 08) |

## 2. Block diagram

```
          ┌────────────── RK3399 (Android 7.1) ───────────────┐
 panel ──▶│ ttyS0  keys and LEDs                               │
 MCU      │        firmware: UI, acquisition control, host I/F │
          │        GPIO bit-bang: AFE SPI, LMK04821, aux DAC,  │
          │        DAC gear, afe_rms, hdcode, FPGA cfg pins    │
          │ SPI1 ─────────── slave-serial config ─────────┐    │
          │ PCIe Gen2 ×2 ── XDMA: bypass BAR regs, C2H ─┐ │    │
          └─────────────────────────────────────────────┼─┼────┘
                                                        ▼ ▼
 CH1..4 ─▶ RT1642 AFE ─▶ 12-bit ADC ══ 28 LVDS lanes ══▶ Artix-7 ──▶ DDR3 
                         ▲ clocks from LMK04821          (SP: capture/resample,
                                                          TRG: digital trigger,
                                                          SEQ: record FSM)
```

## 3. Scope of these specs

| # | Spec | Covers |
|---|------|--------|
| 01 | `01-system-architecture.md` | Boot/bring-up order, processes, device nodes, GPIOs, supervision, update and recovery |
| 02 | `02-register-map.md` | Every FPGA register in the bypass BAR (SP 0x1000, TRG 0x2000, SEQ 0x4000 blocks) |
| 03 | `03-acquisition-engine.md` | Record → play-back acquisition FSM, frame handshake, C2H sample transfer and format |
| 04 | `04-timebase-and-bands.md` | Sample-rate/interleave selection, decimation, memory depth, frame counts |
| 05 | `05-triggering.md` | Digital trigger (TRG block): thresholds, A/B/R engines, all trigger types, holdoff, EXT |
| 06 | `06-vertical-and-analog.md` | RT1642 AFE frames, aux DAC, DAC gear, LMK04821 clocking, afe_rms overload, ADC (RT8847) register interface, vertical setting → register values, start-up order |
| 07 | `07-display-and-rendering.md` | GLES density renderer, persistence, waveform area |
| 08 | `08-front-panel.md` | Panel MCU UART protocol, key/encoder/LED tables |
| 09 | `09-control-plane.md` | What the DHO1000 adds to the core's control plane: inputs, per-unit storage (FRAM, vendor storage, calibration), model and bandwidth identity |
| 10 | `10-calibration.md` | Calibration file container, per-file payloads and how they are applied; the calibration procedures |
| 11 | `11-host-interface.md` | SCPI transports (VXI-11, 5555, USBTMC, USB-GPIB), waveform transfer, web remote |
| 12 | `12-fpga-board-interface.md` | Every used FPGA ball: ADC lanes, DDR3 (SSTL15), PCIe, clocks, low-speed I/O |
| 13 | `13-fpga-configuration.md` | HW code → bitstream selection, slave-serial load, PROGRAM_B/INIT_B/DONE handshake |
| 14 | `14-pcie-dma-interface.md` | PCIe bring-up, XDMA BARs and char devices, the optional `/dev/dma_auklet` copy engine |
| 16 | `16-test-environment.md` | Emulation test environments (Tier 1 qemu-user + device model, Tier 2 Android emulator): device model, emulated-unit profile, strict/lenient model, what runs can and cannot establish |
| 20 | `20-adc-capture.md` | ADC LVDS capture: lanes, deserialisation, sample assembly, training registers |
| 21 | `21-fpga-processing-logic.md` | FPGA processing logic behind the registers: register file, trigger comparators and engines, coupling filter, FIR, record sequence, record memory, play-back and C2H output |

## 4. Product configuration

The unit's model and serial number are items in `/dev/vendor_storage` (spec 01 §2). The 4-bit
HW code (spec 13 §1.2) tells the board variants apart: `0x2` and `0x3` are 4-channel 70 MHz boards
(DHO1074); `0x0` and `0x4` belong to the 4-channel DHO4000 boards; the other codes are 2-channel
DHO1000 boards (UNVERIFIED: only the DHO1074 codes matter to these specs). The acquisition
parameters of the DHO1074 board are: one ADC, 14-bit lane groups (28 LVDS lanes), record memory of
2^25 bursts of 32 bytes (1 GiB), 20 samples per burst (spec 04 §1, UNVERIFIED).

## 5. Glossary

| term | meaning |
|---|---|
| SP | FPGA signal-processing unit: ADC capture, alignment, record memory access, resampling, output |
| TRG | FPGA trigger-processing unit (fully digital trigger on ADC codes) |
| SEQ | FPGA system-control unit: run/stop FSM, frame recording, status |
| Bypass BAR | XDMA BAR1, 32 KiB, mapped by the CPU through `/dev/xdma0_bypass` as the register file |
| C2H / H2C | XDMA card-to-host / host-to-card DMA stream engines |
| HW code | 4-bit board code from `/dev/hdcode_gpio`, selects the bitstream |
| AFE | analog front end (RT1642, one per channel) |
| GEL | the unit's firmware update container (unsigned gzip'd tar, spec 01 §10.1) |
| CLK250 | 250 MHz acquisition clock (ADC DCO / 2): the "4 ns" clock of the TRG, SEQ and record path |
| CLK125 | 125 MHz PCIe user clock: register file and C2H side |
| slot | one of the 4 TRG sample positions per channel per CLK250 cycle (1 ns each) |
| burst | 32 bytes of record memory = 20 × 12-bit samples |
| frame | `FRAME_BURSTS` bursts holding one triggered record, plus one header burst (spec 21 §8) |
| record / display domain | 16-bit codes before (7500/G codes per div) / after (7500 per div) the play-back gain (spec 03 §2) |
| lane | context-dependent: LVDS data pair (specs 12, 20); SP channel lane 0–3 (specs 02, 04); 16-bit C2H FIFO lane (spec 21 §9.1) |
