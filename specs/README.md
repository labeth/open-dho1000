# Specifications

Implementable specs of the Rigol DHO1000 (reference unit DHO1074) hardware and its interfaces,
written so an open firmware on open-dso-core, and later an open FPGA image, can be built from the text. Start with
`00-overview.md` (device, block diagram, index, product configuration, glossary).

**Rules for these documents** (the same for every open-dso scope): concise, complete,
implementable. State what *is* and what the implementation *must do*, in our own words. No history,
no "how we found this", no changelog, no vendor code, strings or symbol names. Register
addresses, value formats, ordering and timing are given explicitly. Register and field names are our
own. Nothing has been checked on a real unit yet: every hardware fact is **UNVERIFIED** until it is,
and each spec says so.

Numbering follows the open-dso family: 00–12 are the shared topics, 13 and up keep the numbers the
family shares (13 FPGA configuration, 14 kernel-driver interface, 16 test environment; 15 is not
used here), and topics specific to this scope start at 20.

| # | Spec | Covers |
|---|------|--------|
| 00 | [`00-overview.md`](00-overview.md) | Device, block diagram, index, product configuration, glossary |
| 01 | [`01-system-architecture.md`](01-system-architecture.md) | Boot order, processes, device nodes, GPIOs, supervision, update/recovery, launching custom software |
| 02 | [`02-register-map.md`](02-register-map.md) | FPGA register map (bypass BAR) |
| 03 | [`03-acquisition-engine.md`](03-acquisition-engine.md) | Record/play-back acquisition FSM, sample transfer |
| 04 | [`04-timebase-and-bands.md`](04-timebase-and-bands.md) | Sample rates, interleave, decimation, memory depth |
| 05 | [`05-triggering.md`](05-triggering.md) | Digital trigger engines and types |
| 06 | [`06-vertical-and-analog.md`](06-vertical-and-analog.md) | AFE, aux DAC, gear lines, LMK04821, ADC registers, start-up order |
| 07 | [`07-display-and-rendering.md`](07-display-and-rendering.md) | Waveform rendering |
| 08 | [`08-front-panel.md`](08-front-panel.md) | Panel MCU protocol, keys, encoders, LEDs |
| 09 | [`09-control-plane.md`](09-control-plane.md) | What the DHO1000 adds to the core's control plane: inputs, per-unit storage, model and bandwidth identity |
| 10 | [`10-calibration.md`](10-calibration.md) | Calibration files and procedures |
| 11 | [`11-host-interface.md`](11-host-interface.md) | SCPI transports, waveform transfer, web remote |
| 12 | [`12-fpga-board-interface.md`](12-fpga-board-interface.md) | Every FPGA ball and its function |
| 13 | [`13-fpga-configuration.md`](13-fpga-configuration.md) | Bitstream selection and loading |
| 14 | [`14-pcie-dma-interface.md`](14-pcie-dma-interface.md) | PCIe/XDMA link and char devices |
| 16 | [`16-test-environment.md`](16-test-environment.md) | Emulation test environments (Tier 1 qemu-user + device model, Tier 2 Android emulator), strict/lenient model |
| 20 | [`20-adc-capture.md`](20-adc-capture.md) | ADC LVDS capture and training |
| 21 | [`21-fpga-processing-logic.md`](21-fpga-processing-logic.md) | FPGA processing logic behind the registers: register file, trigger, coupling filter, FIR, record sequence and memory, play-back, C2H output |
