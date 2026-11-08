# 14 — PCIe / DMA Interface

This document covers the CPU↔FPGA link: the PCIe root port bring-up, the XDMA endpoint and its
BARs, the Linux character devices the firmware uses (register window, sample DMA), their exact
syscall semantics, and the optional memory-to-memory DMA helper (`/dev/dma_auklet`). FPGA register
*contents* are in `02-register-map.md`. The FPGA configuration that has to happen before the link
can train is in `13-fpga-configuration.md`. The analog-control GPIO devices are in
`06-vertical-and-analog.md`.

## 1. Topology and bring-up order

| Item | Value |
|---|---|
| Root complex | RK3399 PCIe controller, driver `pcie-rockchip.ko` (DT compatible `rockchip,rk3399-pcie`): a module build of the Rockchip BSP 4.4 root-port driver (exact BSP revision and any further changes **UNVERIFIED**) that unloads and re-probes cleanly |
| Endpoint | Xilinx 7-series FPGA with the Xilinx DMA/Bridge Subsystem for PCIe (XDMA) |
| Host driver | Xilinx XDMA reference driver v2019.2.51 (`xdma.ko` or `xdma_v1.ko`, §3) |
| PCI vendor ID | `0x10EE` |
| PCI device ID | `0x7022`; subsystem `0x10EE:0x0007`, revision `0x00` (S2/S2B images). An open image may use any of the 50 IDs in the Xilinx reference driver's ID table (`xdma.ko` and `xdma_v1.ko` have the same table): `0x2808`; `0x4808 0x4828 0x4908 0x4A28 0x4B28`; `0x6828 0x6830 0x6928 0x6930 0x6A28 0x6A30 0x6D30`; `0x70xx` and `0x80xx` with `xx` = `11 12 14 18 21 22 24 28 31 32 34 38`; `0x9011 0x9012 0x9014 0x9018 0x9021 0x9022 0x9024 0x9028 0x9031 0x9032 0x9034 0x9038 0x903F`. It should keep `0x7022` |
| Endpoint capability | **Gen2 (5 GT/s) ×2**, lane reversal off, up-configure capable; GTP reference clock 100 MHz; user clock 125 MHz, 64-bit (spec 12 §6) |
| Class code | `0x070001` |
| Max payload supported | 512 bytes |
| Interrupt capability | MSI (64-bit, 1 vector), legacy INTA; no MSI-X |
| Link width / speed | Root side set by the board device tree (`num-lanes`, max link speed; an invalid `num-lanes` falls back to ×1). The driver tries gen2 and falls back to gen1 on a training timeout (mainline driver: Gen1 only, spec 01 §11). Negotiated width and speed are **UNVERIFIED** (≤ gen2 ×2). The PERST# GPIO, `num-lanes` and `max-link-speed` come from the device tree of the system image (not in the update package): **UNVERIFIED**, read `/sys/firmware/fdt` on the unit. |

The PCIe root-port driver is a loadable module. It must be loaded only after the FPGA is
configured and its PCIe reference clock is running, because the link trains (and the bus is
enumerated) once, at probe time. The required order is:

1. Load the FPGA GPIO helper and configure the FPGA (`13-fpga-configuration.md`).
2. Program the LMK04821 clock chip (`06-vertical-and-analog.md` §4).
3. Load `pcie-rockchip.ko`. The link trains and the XDMA endpoint enumerates.
4. Load the XDMA driver; the device nodes `/dev/xdma0_*` appear once ueventd has processed the
   probe (allow about 1 s).
5. `chmod 0777 /dev/xdma0_*` if a non-root process is to open the nodes.

Reconfiguring the FPGA while the link is up (for example loading a self-test bitstream)
drops the endpoint. The XDMA driver does not support that, so an implementation must
unload `xdma` and `pcie-rockchip`, reconfigure, and reload them in the order above (spec 13 §6).
The root-port driver retrains the link
and re-enumerates on every probe, and it drives PERST# from a device-tree GPIO (`ep-gpios`,
number **UNVERIFIED**; on the FPGA side presumably ball E22, spec 12 §7.4). A replacement root-port
driver must likewise survive `rmmod`/`insmod`, assert PERST# and retrain on every probe.

Root-port timing (`pcie-rockchip.ko`, each probe): PERST# (DT `ep-gpios`, mandatory in this driver)
is driven low while the controller and PHY are reset (microseconds to a few milliseconds; there is
no 100 ms PERST# delay), then released. The driver then polls up to **500 ms** for the gen1 link
(failure: `PCIe link training gen1 timeout!`, probe fails, no bus) and, with max link speed 2,
retrains and waits up to another 500 ms for gen2 (failure: stays gen1). The FPGA endpoint must
accept a short PERST# pulse and reach L0 within 500 ms of PERST# rising.

## 2. BARs

The XDMA driver classifies BARs by position. It finds the XDMA *config* BAR by reading two
identifier registers in each mapped BAR of at least 32 KiB (both must match; the first such BAR
wins):

- offset `0x2000`: `(id & 0xFFFF0000) == 0x1FC20000` (IRQ block)
- offset `0x3000`: `(id & 0xFFFF0000) == 0x1FC30000` (config block)

Only BARs with nonzero length count. A 64-bit BAR occupies two indices and counts once. With
N such BARs and the config BAR at list position P:

| N | P | user BAR | bypass BAR |
|---|---|---|---|
| 1 | 0 | none | none |
| 2 | 0 | none | 2nd BAR |
| 2 | 1 | 1st BAR | none |
| 3 or 4 | 1 or 2 | 1st BAR | last BAR |

The FPGA register file is reached **only** through the *bypass* BAR (`/dev/xdma0_bypass`);
`/dev/xdma0_user` is not used. Every image of spec 13 §2.1 has N = 2: BAR0 =
XDMA config, BAR1 = the DMA-bypass AXI master (the FPGA register file), BAR2–BAR5 and the expansion
ROM disabled, both BARs 32-bit non-prefetchable memory. The driver therefore reports config 0,
no user BAR, bypass 1. An image must keep this layout (an extra user BAR would turn BAR1 into the
user BAR and remove `/dev/xdma0_bypass`).

| BAR | Function | Size |
|---|---|---|
| 0 (config) | XDMA engine registers (H2C at `0x0000`, C2H at `0x1000`, IRQ block `0x2000`, config `0x3000`, H2C SGDMA `0x4000`, C2H SGDMA `0x5000`, SGDMA common `0x6000`, MSI-X `0x8000`). Used only by the driver. | 64 KiB |
| 1 (bypass) | FPGA register file: 32-bit registers at absolute byte offsets `0x1000`..`0x40D0` (see `02-register-map.md`) | 32 KiB (`0x8000`) |

These sizes hold for the normal images (`SPU_H12S2`, `SPU_H12S2B`, `SPU_H12S4`, `SPU_H12S4B`). The
1-ADC self-test image loaded by the runtime reload (`SPU_H12S2_SelfTest.bit` = `SPU_H12S2_bode.bit`,
spec 13 §6) presents BAR1 as **16 MiB** (`0x01000000`) and sets the Link Status *Slot Clock
Configuration* bit; BAR0 and the link capability are unchanged (other differences: spec 12 §1.2).
Drivers and software must not depend on BAR1 being exactly 32 KiB; mapping `0x8000` bytes works for
both.

The FPGA register window must decode at least `0x0000`..`0x7FFF`, because userspace maps exactly
`0x8000` bytes from offset 0. All accesses are aligned 32-bit loads and stores. An FPGA must not
depend on byte enables.

## 3. Driver variants

`xdma.ko` and `xdma_v1.ko` are built from the same source with different kernel configurations
for two OS image generations. They have identical behaviour and interfaces. `load_pcie.sh` picks
`xdma_v1.ko` when the system property `ro.rigol.system.version` with the dots removed is
less than `108`, and `xdma.ko` otherwise. Neither variant depends on the FPGA revision.

Both are the Xilinx reference driver with these deviations:

| Item | Upstream reference driver | Shipped `xdma.ko` / `xdma_v1.ko` |
|---|---|---|
| `poll_mode` default | 0 (interrupt completion) | **1** (completion by descriptor write-back polling, kernel threads; no DMA interrupts needed) |
| read/write on `xdma0_bypass_{h2c,c2h}_N` | push u32s to the descriptor-bypass FIFO / dump descriptor lists | linear 32-bit MMIO copy (§4.4) |

The module is loaded with no parameters, so the defaults apply:

| Parameter | Value |
|---|---|
| `poll_mode` | 1 |
| `interrupt_mode` | 0 (MSI-X if the endpoint offers it, otherwise legacy INTx; MSI only with `interrupt_mode=1`) |
| `enable_credit_mp` | 1 |
| `desc_blen_max` | `0x0FFFFFFF` |
| `sgdma_timeout` | 10 s |

An open-source replacement may use the upstream driver with `poll_mode=1`. Only the two bypass sub-nodes (unused) behave differently.

## 4. Character devices

Major number dynamic. Device node names and minors:

| Node | Minor | Present when | Needed |
|---|---|---|---|
| `/dev/xdma0_control` | 1 | always | no |
| `/dev/xdma0_events_0`..`_15` | 10..25 | always | no |
| `/dev/xdma0_h2c_N` | 32+N | H2C engine N exists | **N=0** |
| `/dev/xdma0_c2h_N` | 36+N | C2H engine N exists | **N=0** |
| `/dev/xdma0_bypass_h2c_N` | 64+N | bypass BAR and engine N | no |
| `/dev/xdma0_bypass_c2h_N` | 68+N | bypass BAR and engine N | no |
| `/dev/xdma0_bypass` | 100 | bypass BAR | **yes** (registers) |
| `/dev/xdma0_user`, `/dev/xdma0_xvc` | 0, 2 | user BAR | no |

N ranges over 0..3; only engines whose identifier register reads `0x1FC0xxxx` / `0x1FC1xxxx`
are created.

### 4.1 `/dev/xdma0_bypass`: register window

- Open with `O_RDWR | O_SYNC` (`0x101002`).
- `mmap(NULL, 0x8000, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0)`. The mapping is uncached device
  memory onto the bypass BAR, starting at BAR offset `vm_pgoff << 12`. `-EINVAL` if
  `offset + length` exceeds the BAR size.
- Access: `*(volatile uint32_t *)(base + off)` with `off` the absolute register byte offset.
  Writes are posted, and nothing in software orders them beyond program order to device memory.
  A read-back is a full round trip.
- `read()` / `write()` on this node always fail with `-EINVAL`. `ioctl` is not implemented.

### 4.2 `/dev/xdma0_c2h_0`: card→host DMA (trace/sample read-out)

- Open `O_RDWR` or `O_RDONLY`. C2H channel 0 is an **AXI-Stream** engine: its identifier register
  (BAR0 `0x1000`) reads `0x1FC1_8xxx` (bit 15 = 1, target 1, channel 0), so the driver runs it in
  streaming mode and a second concurrent open fails with `-EBUSY`. An open image MUST configure the
  XDMA core for AXI-Stream (the choice applies to all channels; H2C channel 0 is AXI-Stream as well).
  The identifier value itself is **UNVERIFIED** (one BAR0 read or the `dmesg` engine line on the unit).
- `read(fd, buf, len)` performs one synchronous scatter-gather DMA of `len` bytes into the user
  buffer, which is pinned. There is no driver-side buffer and no mmap. For an AXI-MM engine the
  card source address is the file position; for this stream engine the file position is not used.
  Each read takes the next bytes of the stream (`03-acquisition-engine.md` §2.2).
- Read sizes: exactly the byte count of one play-back command (`03-acquisition-engine.md`
  §2.1; 2 000 B to 8 000 000 B per frame in normal acquisition, up to 500 MiB in UltraAcquire); in
  Roll/Scan, `chunks · k · 2` bytes per read (`k` = 100 · tdiv / roof125(tdiv) words per chunk,
  `03-acquisition-engine.md` §6), which can be 0 bytes. The shipped driver returns `-EINVAL` for a
  0-byte read into a page-aligned buffer (nothing to pin) and otherwise submits a zero-length entry;
  use a page-aligned buffer (`03-acquisition-engine.md` §7) and skip 0-byte reads. A replacement
  driver must not start a transfer for a 0-byte read and should return 0.
- Return: bytes transferred or a negative errno: `-EINVAL` (alignment/direction), `-ETIMEDOUT`/`-EIO`
  (no completion within `sgdma_timeout`), `-ENOMEM`. This driver version has no end-of-packet
  flush: a read completes only when every descriptor of the transfer is closed, and the count is
  the sum of the bytes written per descriptor. An AXI-Stream packet end (TLAST) closes the current
  descriptor early and leaves a gap in the buffer. The FPGA must therefore deliver exactly the
  requested byte count and must not end a packet before it. The per-play last-word flag of the C2H
  FIFO drives TLAST only when `OUT_CTRL` bit 29 = 0; set bit 29 = 1 so TLAST stays deasserted (spec 02
  §2.4, spec 21 §9.1). An open image MUST NOT assert TLAST while bit 29 = 1.
- Alignment (streaming engine): `buf` must be a multiple of the engine's address alignment and
  `len` a multiple of its length granularity.
- One transfer is split into descriptors of at most `desc_blen_max` bytes and at most 2048
  descriptors.

### 4.3 `/dev/xdma0_h2c_0`: host→card DMA

Same as §4.2 with `write()` and the direction reversed (the card destination address is the
file position). Open it write-only (`O_WRONLY`) for trace/waveform upload (record-memory import,
`03-acquisition-engine.md` §8); normal acquisition does not use it.

### 4.4 Engine ioctls (on `h2c_N` / `c2h_N`)

| Request | Code | Argument |
|---|---|---|
| `PERF_START` | `0x40087101` | `struct xdma_performance_ioctl *` with `version=1`, `transfer_size` |
| `PERF_STOP` | `0x40087102` | same struct, filled on return |
| `PERF_GET` | `0x80087103` | same struct, filled on return |
| `ADDRMODE_SET` | `0x40047104` | `int *`: 0 = incrementing address, 1 = fixed (non-incrementing) |
| `ADDRMODE_GET` | `0x80047105` | `int *` |
| `ALIGN_GET` | `0x80047106` | `int *`: engine address alignment |

`struct xdma_performance_ioctl` (40 bytes, little-endian, no padding): `u32 version, transfer_size, stopped,
iterations; u64 clock_cycle_count, data_cycle_count, pending_count`. Other codes return `-EINVAL`.
None of these ioctls is needed.

`/dev/xdma0_control` (not needed): `read`/`write` of one 32-bit word at the file
position in the config BAR, `mmap` of the config BAR, ioctls `0xC0287801` (INFO, magic
`0x586C0C6C`), `0x7802` (OFFLINE), `0x7803` (ONLINE).

`/dev/xdma0_bypass_{h2c,c2h}_N` (unused): `count` must be a multiple of 4. Words are copied to or
from bypass-BAR offset `bypass_offset*4 + i` for `i = 0, 4, …`. The file position is ignored.
A read stops with `-EINVAL` if a word reads as 0.

## 5. Interrupts

Completion of DMA does not use interrupts (`poll_mode=1`). User interrupts (`/dev/xdma0_events_N`)
are not needed. An implementation therefore needs no MSI/MSI-X setup beyond
what the driver does at probe, and the FPGA does not need to raise user IRQs. The endpoint (spec 12 §6)
offers MSI with one vector and INTA (no MSI-X), so with the default `interrupt_mode` 0 the driver
uses **legacy INTA**: it requests the (shared) INTx line at probe. MSI is used only with
`interrupt_mode=1`. With `poll_mode=1` no DMA interrupt is needed either way.
Apart from the configuration pins (spec 13), no FPGA ball other than the PCIe lanes and PERST#
connects the FPGA to the CPU (spec 12 §7).

## 6. `/dev/dma_auklet`: memory-to-memory DMA helper (optional)

The boot script does not load `dma_auklet.ko` (spec 01 §4). The module is a dmaengine memcpy helper
for copies between buffers in CPU DDR (for example de-interleaving sample buffers). It is not part
of the FPGA path, and a CPU `memcpy` does the same job, so a replacement firmware does not need it.

| Item | Value |
|---|---|
| Match | platform device, DT compatible `rigol,dma_auklet` (channels from the node's DMA list) |
| Node | `/dev/dma_auklet` (`O_RDWR`) |
| Buffer | `mmap(NULL, 0x10000000, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0)`: 256 MiB of physically contiguous memory. Source pool = `base`, destination pool = `base + 0x08000000` (128 MiB each). |
| Transfer | copies `length` bytes from `src_pool + src_offset` to `dst_pool + dst_offset` with the SoC DMA controller |

ioctls (argument = pointer to `u32` unless noted):

| Code | Meaning |
|---|---|
| `0x80086164` | set the byte offset into the source pool |
| `0x80086166` | set the byte offset into the destination pool |
| `0x80086165` | set the number of bytes to copy |
| `0x40086162` | start the copy asynchronously |
| `0x40086161` | wait for completion of the started copy |
| `0x80086163` | start the copy and wait (synchronous) |

A copy is: set the length (offsets default to 0), then start and wait. Each returns 0 on success.
An implementation can replace the whole device with `memcpy` between two 128 MiB buffers.
