# 01 — System Architecture

This spec defines the platform, the boot and bring-up sequence, the process model, every device
node and GPIO the firmware stack uses, the supervision model, and the update and recovery paths.
Every ordering rule here is load-bearing: the FPGA (spec 13), the clock chip (spec 06) and the
PCIe link (spec 14) only work when brought up in the order given in §4. Nothing in this spec has
been checked on a unit: every fact is **UNVERIFIED** until it is.

---

## 1. Platform

| Item | Value |
|---|---|
| SoC | Rockchip RK3399 (2× Cortex-A72 + 4× Cortex-A53); RAM and eMMC size **UNVERIFIED** (`/proc/meminfo`, `/proc/partitions`) |
| OS | Android 7.1.2, Linux 4.4.126 SMP preempt, arm64 (`vermagic 4.4.126 SMP preempt mod_unload modversions aarch64`) |
| OS version property | `ro.rigol.system.version`, dotted (e.g. `1.1.1`) |
| FPGA | Xilinx Artix-7, XC7A100T or XC7A75T, FGG676, volatile, configured by the CPU at every boot (spec 13); ball-level interface in spec 12 |
| ADC | one RT8847IQ, 8 cores, 2 GSa/s, 28 LVDS lanes into the FPGA (specs 06 §8, 20) |
| Record memory | **1 GiB** DDR3 (1.5 V I/O) on the FPGA's 32-bit memory bus (two ×16 devices; spec 12 §5, spec 04 §1). Not visible to the CPU except through FPGA play-back (spec 03) |
| CPU↔FPGA data path | PCIe, Xilinx XDMA endpoint; register window + DMA (spec 14) |
| CPU↔FPGA configuration path | RK3399 SPI1 + three GPIOs (spec 13) |
| Sample clock | TI LMK04821 programmed by the CPU over a GPIO 3-wire bus (spec 06) |
| Front panel | keyboard/LED MCU on `/dev/ttyS0` (spec 08) |
| Network | Gigabit Ethernet, Motorcomm YT8531S PHY (out-of-tree module, §4) |

GPIO numbers in this document are Linux global GPIO numbers (RK3399: `bank × 32 + pin`, so
GPIO 2 = GPIO0_A2, GPIO 73 = GPIO2_B1, GPIO 125 = GPIO3_D5).

## 2. Storage layout

The unit keeps its instrument software on a writable partition mounted at `/rigol` (the "user
partition"). The Android system image only supplies the boot hook that runs
`/rigol/shell/startApp.sh`; everything else is in `/rigol` or in installed APKs.

| Path | Contents |
|---|---|
| `/rigol/shell/` | boot and update scripts: `startApp.sh`, `start_rigol_app.sh`, `load_pcie.sh`, `do_update.sh`, `do_extract.sh`, `force_update_gel.sh`, `copy_logs_to_udisk.sh` |
| `/rigol/driver/` | out-of-tree kernel modules (`*.ko`, §5) |
| `/rigol/tools/` | helper programs run at boot, among them `spi2fpga` (FPGA load, spec 13), `spi2pll` (LMK04821, spec 06) and `pmapService` (portmapper, spec 11) |
| `/rigol/FPGA/` | bitstreams (`SPU_H12S{2,4}{,B}.bit`, `*_bode.bit`, `*_SelfTest.bit`) and `readme.txt` (spec 13 §2) |
| `/rigol/resource/conf/` | `version.txt` (HW code → bitstream, spec 13 §2), `init_all_clock.txt` (LMK04821 register list, spec 06) |
| `/rigol/data/` | per-unit calibration (`cal_*.hex`, `vendor.bin`, …, spec 10), `probe/`; **not** shipped by updates except `default/` |
| `/rigol/data/default/` | factory-default calibration and optional `init_all_clock.txt` fallback |
| `/data/logs/tools_log/` | `boot_up.txt` (all boot-script output), `clock_init_log.txt`, `dmesg_<n>.txt`, `logcat_<n>.txt` (n = boot count mod 10) |
| `/data/UserData/` | user files; exported read-write by FTP |
| `/mnt/tmp/` | update staging directory (§10) |
| `/mnt/media_rw/<id>/` | mounted USB sticks |

Irreplaceable per-unit content: the calibration in `/rigol/data/` (outside `default/`), the 8 KiB
I2C FRAM on `/dev/i2c-4` address 0x50 (spec 09; whether installed option licences are also kept
there is **UNVERIFIED**), and `/dev/vendor_storage` (serial number, model, MAC). Back up all three
before any modification, and never write them from an open firmware.

## 3. Processes

Running after boot, and what a replacement firmware does with them (§10.7):

| Process (package) | Started by | Runs as | Role | Replacement |
|---|---|---|---|---|
| `startApp.sh` → `start_rigol_app.sh` | system image boot hook | root | one-shot hardware bring-up (§4), then USB hooks | keep (it ends by itself) |
| `pmapService` | boot script | root | ONC-RPC portmapper (TCP+UDP port 111) for VXI-11 | keep if VXI-11 is served |
| `tcpsvd 0:21 ftpd ftpd -w /data/UserData/` | boot script | root | anonymous, writable FTP server on all interfaces | stop unless wanted |
| `logcat` writers | boot script | root | rotating logs | keep |
| `com.rigol.launcher` | Android (HOME app) | system uid | home screen and recovery screen (§10.4); holds `/dev/ttyS0` and `/dev/afe_rms` open; keeps the scope app running (§7) | must stop reading `/dev/ttyS0` and must not restart the scope app (§7) |
| `com.rigol.scope` | `com.rigol.launcher` | system uid | the vendor instrument application; holds every FPGA and analog device node | stop |
| `com.rigol.webcontrol` | `com.rigol.launcher` | system uid | web remote control | stop if the replacement serves the web port |

## 4. Boot and bring-up sequence

The boot script `start_rigol_app.sh` does the following, in this order. A failing step only logs;
nothing aborts. Kernel modules are inserted without parameters.

| # | Action | Creates / effect |
|---|---|---|
| 1 | logs everything to `/data/logs/tools_log/boot_up.txt`; advances the boot counter that selects the log slot (`mod 10`) | — |
| 2 | brings `eth0` down, inserts `motorcomm.ko`, brings `eth0` up | Ethernet PHY driver; **no network before this step** |
| 3 | inserts `dac_gpio.ko` | `/dev/dac_gear_change` |
| 4 | inserts `hdcode_gpio.ko` | `/dev/hdcode_gpio` (board HW code, spec 13 §2) |
| 5 | inserts `fpga_gpio.ko` | `/dev/spi2k7_ctl`; drives PROGRAM_B high (no reset pulse, no clock enable) |
| 6 | configures the FPGA with `spi2fpga`: on OS ≥ 1.1.1 it selects the bitstream by HW code from `/rigol/resource/conf/version.txt`; on older OS versions `spi2fpga_v1` always loads `/rigol/FPGA/SPU_H12S4.bit` | FPGA configured (spec 13) |
| 7 | inserts `spi2pll_gpio.ko`, programs the LMK04821 | `/dev/spi_3wires`; LMK04821 programmed and checked (spec 06) |
| 8 | `load_pcie.sh`: inserts `pcie-rockchip.ko` (unless loaded), then `xdma.ko` (OS < 1.0.8: `xdma_v1.ko`), waits 1 s, opens up `/dev/xdma0_*` | PCIe link trains, endpoint enumerates, `/dev/xdma0_*` (spec 14) |
| 9 | inserts `spi2afe_gpio.ko` | `/dev/spi_simulate` |
| 10 | inserts `ilitek_ts.ko`, then `focaltech_ts.ko` | touch screen (ilitek must be inserted first) |
| 11 | inserts `beeper_gpio.ko` and `pwm_fan.ko` | `/dev/pwm-1` (beeper), `/dev/pwm_fan` (also a hwmon `pwm1` and a thermal cooling device) |
| 12 | creates `/rigol/data/probe`, `/mnt/tmp`, `/data/UserData` | — |
| 13 | inserts `afe_rms_gpio.ko` | `/dev/afe_rms` |
| 14 | starts the FTP server and the portmapper, sets the system time from `/dev/rtc1` | §3 |
| 15 | sets the time zone and the backlight, starts the rotating `logcat` writers | — |
| 16 | waits 20 s | — |
| 17 | USB hooks: `fetch_auklet_logs.txt` on a stick → copies the logs to it (§10.6); `force_update_auklet.txt` → runs `force_update_gel.sh` (§10.3) | — |

The boot script opens the device nodes it creates to all users (mode 0777): `/dev/dac_gear_change`,
`/dev/hdcode_gpio`, `/dev/spi*`, `/dev/xdma0_*`, `/dev/pwm-1`, `/dev/pwm_fan`, `/dev/afe_rms`,
`/dev/vendor_storage`, and `/rigol/data/*`.

`dma_auklet.ko`, `usbtmc_dev.ko`, `usb_gpib.ko` and `libcomposite.ko` are not loaded at boot. They
are inserted (as root) when the function is used: `libcomposite.ko`, then `usbtmc_dev.ko
iSerialNum=<serial> iProductDesc=<string> ProductId=<id>` for the USB-TMC device port; `usb_gpib.ko`
(no parameters) for a USB-GPIB adapter on the host port. `dma_auklet.ko` is optional (spec 14 §6).

No boot step enables an FPGA clock explicitly: `fpga_gpio.ko` touches only the three configuration
GPIOs. The FPGA reference clock appears when the LMK04821 is programmed in step 7 (spec 06 §4.4);
the 100 MHz PCIe reference clock does not come from the LMK04821 (source **UNVERIFIED**).

**Required order for any replacement firmware.** The hardware constraints behind steps 5–8 are:

1. FPGA configuration needs only `fpga_gpio.ko` and `spidev1.0` (spec 13).
2. The LMK04821 is programmed **after** the FPGA is configured. The FPGA design must therefore hold
   every LMK-clocked block in reset until its clock is present and locked (spec 12 §2.2).
3. `pcie-rockchip.ko` must be inserted only after the FPGA is configured (and its PCIe reference
   clock runs): the root port trains the link and scans the bus once, at probe (spec 14).
4. The XDMA driver goes last. The register window is usable once `/dev/xdma0_bypass` exists.

## 5. Device nodes

Access to `/dev/ttyS0`, `/dev/rtc1`, `/dev/i2c-1`, `/dev/i2c-4`, `/dev/usbtmc_dev`, `/dev/usbgpib0`
and `/dev/dma_auklet` for a non-root process comes from the system image (ueventd rules,
**UNVERIFIED**); a replacement that runs as root does not depend on it.

| Node | Provider | Purpose | Spec |
|---|---|---|---|
| `/dev/spi2k7_ctl` | `fpga_gpio.ko` (misc) | FPGA PROGRAM_B / INIT_B / DONE | 13 |
| `/dev/spidev1.0` | kernel spidev (RK3399 SPI1) | FPGA configuration data stream | 13 |
| `/dev/hdcode_gpio` | `hdcode_gpio.ko` (misc) | 4-bit board HW code; read exactly 1 byte | 13 |
| `/dev/spi_3wires` | `spi2pll_gpio.ko` (misc) | LMK04821 3-wire bus + reset/sync/status GPIOs | 06 |
| `/dev/spi_simulate` | `spi2afe_gpio.ko` (misc) | AFE + DAC bit-banged SPI | 06 |
| `/dev/dac_gear_change` | `dac_gpio.ko` | DAC gear GPIOs | 06 |
| `/dev/afe_rms` | `afe_rms_gpio.ko` | 50 Ω overload detector status | 06 |
| `/dev/xdma0_bypass` | `xdma.ko` | FPGA register window (mmap of the first 32 KiB of BAR1) | 14, 02 |
| `/dev/xdma0_c2h_0`, `/dev/xdma0_h2c_0` | `xdma.ko` | sample read-out (stream) / record-memory import | 14, 03 |
| `/dev/dma_auklet` | `dma_auklet.ko` (not loaded at boot) | CPU-side memory-to-memory DMA; optional, `memcpy` does the same | 14 |
| `/dev/pwm-1` | `beeper_gpio.ko` | beeper PWM | 08 |
| `/dev/pwm_fan` | `pwm_fan.ko` (DT `pwm-fan`) | fan speed | — |
| `/dev/ttyS0` | kernel UART | front-panel MCU, 115200 8N1 | 08 |
| `/dev/vendor_storage` | Rockchip kernel | serial number, model, MAC | — |
| `/dev/rtc1` | kernel | battery RTC | — |
| `/dev/i2c-1` | kernel i2c-dev | battery pack (raw access); the bus also carries an ADC128D818 at 0x37, read through its kernel driver (sysfs `1-0037`) | — |
| `/dev/i2c-4` | kernel i2c-dev | 8 KiB FRAM at 0x50 (spec 09); the bus also carries a two-channel temperature sensor at 0x1f, read through hwmon (`4-001f`) | 09 |
| `/dev/usbtmc_dev`, `/dev/usbgpib0` | `usbtmc_dev.ko`, `usb_gpib.ko` | USB-TMC device, USB-GPIB adapter | 11 |

Other sensors and controls: `/sys/devices/platform/backlight/backlight/backlight/brightness`,
`/sys/bus/i2c/devices/4-00xx/hwmon/hwmon*/temp*_input`, `/sys/bus/i2c/drivers/adc128d818/1-0037/in*_input`,
`/sys/bus/iio/devices/iio:device0/in_voltage*_raw`, thermal zones 0/1.

## 6. Board GPIOs owned by the boot modules

| GPIO | Name | Dir | At insmod | Module | Function |
|---|---|---|---|---|---|
| 1 | INIT_B | in | input | `fpga_gpio.ko` | FPGA INIT_B |
| 2 | PROGRAM_B | out | high | `fpga_gpio.ko` | FPGA PROGRAM_B |
| 3 | DONE | in | input | `fpga_gpio.ko` | FPGA DONE |
| 7, 8, 11, 12 | HW code 0..3 | in | input | `hdcode_gpio.ko` | board HW code bits 0..3 |
| 76 | LMK CS | out | high | `spi2pll_gpio.ko` | LMK04821 3-wire bus chip select |
| 75, 74 | LMK SCLK, SDIO | out | low | `spi2pll_gpio.ko` | LMK04821 3-wire bus clock, data |
| 73 | LMK reset | out | low | `spi2pll_gpio.ko` | LMK04821 reset (pulsed before programming) |
| 125 | LMK status | in | input | `spi2pll_gpio.ko` | LMK04821 status / lock input |
| 66 | LMK SYNC | out | low | `spi2pll_gpio.ko` | LMK04821 SYNC, set high after programming |

All levels are raw pin levels.

The AFE/DAC/RMS module GPIOs are in spec 06.

## 7. Supervision

No hardware watchdog is in use. What a replacement has to take into account:

- `com.rigol.scope` and `com.rigol.webcontrol` are started by `com.rigol.launcher` early in the
  Android boot, possibly before the bring-up of §4 has finished, and are started again if they stop.
  A replacement that stops `com.rigol.scope` must therefore also stop `com.rigol.launcher`, or the
  scope app comes back within seconds.
- `com.rigol.launcher` holds `/dev/ttyS0` (front-panel keys, spec 08) and `/dev/afe_rms` open.
  While it runs, a replacement does not get the panel keys reliably, and a `/dev/afe_rms`
  implementation must answer its status ioctl quickly.

## 8. Runtime reconfiguration

Reloading the FPGA while the PCIe link is up (for example to load a different image) must follow
this pattern: close all `/dev/xdma0_*` fds, `rmmod xdma`, `rmmod pcie_rockchip`, configure,
`insmod pcie-rockchip.ko`, `insmod xdma.ko`, wait for `/dev/xdma0_bypass`. The self-test image
(spec 13 §2.1; on 1-ADC boards the same file as the S2 Bode image) is a different design from the
normal image, and its BAR1 is 16 MiB instead of 32 KiB (spec 12 §1.2, spec 14 §2). After a failed
reload, reload the PCIe drivers anyway so that the next attempt starts from a known state
(spec 13 §6).

## 9. Constraints and traps

- **Order of bring-up** (§4) is mandatory; inserting `pcie-rockchip.ko` before the FPGA is
  configured leaves the bus empty until the module is reloaded.
- **OS < 1.1.1 always loads `SPU_H12S4.bit`** (4-channel A100T image) regardless of board code; on
  a DHO1000 or an A75T board that is the wrong image.
- The xdma char devices are registered during `insmod xdma.ko`; the `/dev` nodes appear shortly
  after (ueventd). A non-root process can open them only after `load_pcie.sh` has changed their mode
  (about 1 s after the insertion).
- Use one USB stick at a time for the boot USB hooks (§4 step 17): with two sticks mounted they do
  not fire.
- Network (and therefore network adb) is only available after step 2 of §4. A boot script that
  breaks before that step leaves the unit without Ethernet.
- The FTP server exports `/data/UserData` read-write to anyone on the network.
- Logs: the full boot transcript is `/data/logs/tools_log/boot_up.txt`; FPGA and PLL results are
  in it (and in `clock_init_log.txt`). Read them first when bring-up fails.

---

## 10. Update and recovery

### 10.1 Package format

A `.GEL` file is a gzip-compressed POSIX tar with relative paths and these top-level directories:

| Dir | Destination |
|---|---|
| `app/` | APKs, installed with `pm install -r` |
| `FPGA/` | `/rigol/FPGA/` |
| `driver/` | `/rigol/driver/` |
| `tools/` | `/rigol/tools/` |
| `shell/` | `/rigol/shell/`; **`shell/do_extract.sh` is also executed from the package** |
| `resource/` | `/rigol/resource/` |
| `data/` | `/rigol/data/` (packages normally carry only `data/default/`) |

There is no signature, checksum, model check or version check.

### 10.2 Update procedure

`/rigol/shell/do_update.sh <gel> [mode]` runs as root. It extracts `<gel>` into `/mnt/tmp` (without
emptying it first) and then executes **`/mnt/tmp/shell/do_extract.sh [mode]`** from the package; that
script's exit status is the result. Everything after the extraction is therefore decided by the
package: the package must store `shell/do_extract.sh` executable (mode 0755) with a
`#!/system/bin/bash` line. A full update package copies its directories into `/rigol`, installs its
APKs and restarts the unit; mode `1` additionally uninstalls `com.rigol.scope` first, which drops
the app's settings.

### 10.3 Entry points

| Entry | Trigger | Command | Mode |
|---|---|---|---|
| Scope app, update menu | user picks a `*.GEL` whose path contains `HDO4000`, `HDO1000`, `DHO4000` or `DHO1000`; at least 200 × 10^6 bytes free on `/data` | `do_update.sh <path>` | none |
| Recovery screen, *Upgrade* | §10.4 | `do_update.sh <first removable disk>/DHO4000Update.GEL 1` | 1 |
| Boot USB trigger | `force_update_auklet.txt` on a stick at boot + 20 s, and `DHO4000Update.GEL` (else `HDO4000Update.GEL`) on the same stick root | `force_update_gel.sh` deletes the trigger file, then `do_update.sh <gel> 1` | 1 |

The file name `DHO1000Update.GEL` is **not** accepted by the boot trigger or the recovery screen;
those paths only look for `DHO4000Update.GEL` / `HDO4000Update.GEL`.

### 10.4 Recovery screen

The home app is foreground during the first 1–10 s of the Android boot, before the scope app
starts (§7). Pressing **RUN/STOP** (panel key code 33) in that window shows the recovery screen,
with two choices: a USB update with mode 1 (§10.3), or starting the scope app. If the scope app is
not installed, the home app stays foreground and RUN/STOP reaches it at any time.

### 10.5 Risks to the device

- **Arbitrary root code.** Every entry point executes the package's own `shell/do_extract.sh` as
  root. A package can do anything.
- **No atomicity.** Files are copied in place; power loss mid-copy leaves `/rigol` mixed.
- **Mode 1 uninstalls the scope app first.** If the install then fails, the unit boots to the
  home app with no scope app; recover with a good package via the recovery screen.
- **Bricking surface.** Breaking `/rigol/shell/start_rigol_app.sh` before §4 step 2 removes
  Ethernet (no network adb); breaking `/rigol/shell/do_update.sh` removes all update paths
  including recovery; a wrong `/rigol/resource/conf/version.txt` or bitstream leaves the FPGA
  unconfigured (`/dev/xdma0_*` never appear). The Android system image is not touched by updates,
  so the home app and its recovery screen remain, as long as `do_update.sh` and `busybox` work.
- **Data loss.** `data/` in a package overwrites calibration files of the same name in
  `/rigol/data/`. A replacement package must never carry `data/`.

### 10.6 Logs to USB

A stick with `fetch_auklet_logs.txt` in a mounted root at boot + 20 s receives
`/data/logs/tools_log/` in a directory named `YYYY.MM.DD_HH.MM.SS`; the trigger file is deleted.

### 10.7 Launching custom software

There is no USB autorun hook (no `startup.sh`-style hook). The available paths, from least to most
invasive:

1. **adb, no change to the unit.** With adb reachable (system-image setting, not controlled by an
   update package), run binaries from `/data/local/tmp` as root. Nothing on `/rigol` changes; a
   reboot restores the vendor firmware.
2. **One-shot package, no persistent change.** A `.GEL` whose `shell/do_extract.sh` only starts
   the custom software (and copies nothing into `/rigol`) runs once, as root, from the scope app's
   update menu, the recovery screen, or the boot USB trigger (`force_update_auklet.txt` +
   `DHO4000Update.GEL`). The boot trigger deletes `force_update_auklet.txt`; a launcher package that
   wants to run again on the next boot must re-create that file on the stick. Removing the stick
   restores the vendor firmware. The script runs 20 s into the boot script, after the full bring-up
   of §4, so the FPGA, PLL and PCIe are already up; the vendor application is usually running by
   then and must be stopped together with the home app (§7).
3. **Persistent hook.** Editing `/rigol/shell/start_rigol_app.sh` (or adding a line that checks for
   a file on the stick) gives a permanent autorun. This modifies the unit; put the addition after
   §4 step 17 so bring-up, network and the update paths are unaffected, and keep a copy of the
   original script.

A custom firmware replacing the vendor application needs from the boot only §4 steps 2–9; it then
owns `/dev/xdma0_*`, `/dev/spi_*`, `/dev/spidev1.0`, `/dev/spi2k7_ctl` exclusively once
`com.rigol.scope` and `com.rigol.launcher` are stopped. None of these paths has been tried on a
unit (**UNVERIFIED**).

---

## 11. Porting to a mainline kernel

The kernel, its device tree and the boot chain belong to the Android system image; the update
package carries none of them. Before replacing anything: dump `/sys/firmware/fdt`, read and save
every `/dev/vendor_storage` item (serial number, model, MAC; Rockchip BSP only, no mainline
driver), the 8 KiB FRAM (i2c-4, 0x50) and `/rigol/data` (§2). How a replacement kernel is booted
(U-Boot, Rockchip loader mode) is **UNVERIFIED**.

A replacement kernel needs these values from the unit's device tree (**UNVERIFIED** until read):
the RK3399 PCIe node (`ep-gpios` = PERST#, `num-lanes`, `max-link-speed`,
`vpcie3v3`/`vpcie1v8`/`vpcie0v9` supplies, PHY, clocks), the SPI1 node with its spidev child (chip
select, `spi-max-frequency`, SPI clock rate), the `rigol,dma_auklet` node (DMA channels),
`pwm-fan`, backlight, UART0, the I2C buses, the touch controller and the display panel.

GPIO numbers in this spec are BSP-4.4 global numbers (bank × 32 + pin); on mainline use
(controller, line): gpio0 lines 1, 2, 3 (INIT_B, PROGRAM_B, DONE) and 7, 8, 11, 12 (HW code);
gpio2 lines 2, 9, 10, 11, 12 (LMK SYNC, reset, SDIO, SCLK, CS); gpio3 line 29 (LMK status);
gpio3 line 25 (beeper, GPIO 121).

Mainline `pcie-rockchip-host` is a module with `remove`, so the §8 unload/reload works. It caps
the link at 2.5 GT/s whatever `max-link-speed` says (`invalid max-link-speed, limited to 2.5
GT/s`), half the Gen2 ×2 rate; Gen2 needs a local patch. It holds PERST# low for 100 ms and waits
100 ms after release before the 500 ms link poll. An open endpoint must train at Gen1 ×2 and
accept both this PERST# timing and the short PERST# pulse of the unit's own kernel (spec 14 §1).

| Function | Hardware (spec) | Mainline | Notes |
|---|---|---|---|
| FPGA configuration | SPI1 CS0; PROGRAM_B gpio0-2, INIT_B gpio0-1, DONE gpio0-3 (spec 13) | `FPGA_MGR_XILINX_SPI`, node `xlnx,fpga-slave-serial` with `prog_b-gpios` (active low), `init-b-gpios` (active low) and `done-gpios` (active high), all three required; or `SPI_SPIDEV` + GPIO character device | SPI mode 0 (spec 13 §4); do not copy `spi-cpha` from the binding example. The FPGA manager waits ≤ 1 s for INIT_B and adds ≥ 8 CCLKs after DONE (compatible with spec 13 §3). It has no userspace load trigger: load from the kernel (fpga-region with `firmware-name`, or a small module calling `fpga_mgr_load()`). spidev refuses a node with `compatible = "spidev"`; bind it through `driver_override`. Keep 4096-byte writes or raise `bufsiz` |
| HW code | gpio0 lines 7, 8, 11, 12, inputs (spec 13 §1.2) | GPIO character device | bit order of spec 13 §1.2 |
| LMK04821 | 3-wire bus (reset, SDIO, SCLK, CS = gpio2 lines 9–12), SYNC gpio2-2, status gpio3-29 (spec 06 §4) | no LMK04821 driver (`LMK04832` is a different part); userspace bit-bang through the GPIO character device | program after FPGA configuration and before the root port probes (§4) |
| AFE, aux DAC, offset gears, 50 Ω overload | GPIO lines of spec 06 §2–§6 | GPIO character device (or `SPI_GPIO` for the AFE bus) | the nodes `/dev/spi_simulate`, `/dev/dac_gear_change`, `/dev/afe_rms` have no mainline equivalent |
| PCIe root port | RK3399 PCIe, ×2, PERST# = DT `ep-gpios` (spec 14 §1) | `PCIE_ROCKCHIP_HOST` (module), `PHY_ROCKCHIP_PCIE`, `PCI_MSI`; node `rockchip,rk3399-pcie` with `ep-gpios`, `num-lanes = <2>`, PHY, `vpcie*` supplies, and a first child `interrupt-controller` node (needed for INTx) | Gen1 only (see above). Probe it only after FPGA configuration and LMK programming |
| XDMA | endpoint `10EE:7022`, BAR1 = registers (spec 14) | `XILINX_XDMA` is a dmaengine back end that a parent PCI driver must instantiate (pattern: mgb4); it does not create `/dev/xdma0_*`. Drop-in: the out-of-tree Xilinx `dma_ip_drivers` XDMA module | with the out-of-tree module, load `poll_mode=1 enable_st_c2h_credit=1` (its defaults are 0 and 0; its `interrupt_mode` values: 0 auto, 1 MSI, 2 legacy, 3 MSI-X; the C2H interface of spec 14 assumes these settings) and open C2H without `O_TRUNC` (in that module it enables end-of-packet flush, which spec 14 §4.2 does not assume) |
| Ethernet | RK3399 GMAC + Motorcomm YT8531S-family PHY | `DWMAC_ROCKCHIP`, `MOTORCOMM_PHY` | PHY mode and delays from the unit's DT |
| Front-panel MCU | UART on `/dev/ttyS0`, 115200 8N1 (spec 08) | `SERIAL_8250_DW`, DT alias `serial0` | which RK3399 UART: from the DT, **UNVERIFIED** |
| I2C devices | i2c-1: battery pack, ADC128D818 @0x37; i2c-4: FRAM @0x50 (8 KiB), 2-channel temperature sensor @0x1f | `I2C_RK3X`, `I2C_CHARDEV`, `SENSORS_ADC128D818`; FRAM raw through i2c-dev (an `at24` binding would claim 0x50) | temperature-sensor chip and battery address **UNVERIFIED** |
| SoC ADC, thermal, RTC | SARADC (`iio:device0`), thermal zones 0/1, battery RTC = second RTC (`/dev/rtc1`) | `ROCKCHIP_SARADC`, `ROCKCHIP_THERMAL` | RTC chip **UNVERIFIED** |
| Fan, beeper, backlight | `pwm-fan` PWM; beeper bit-banged on GPIO 121 (gpio3 line 25); PWM backlight | `SENSORS_PWM_FAN` (hwmon, not `/dev/pwm_fan`); beeper: `INPUT_GPIO_BEEPER` or userspace GPIO (not `/dev/pwm-1`); `BACKLIGHT_PWM` | |
| Touch screen | Ilitek or FocalTech controller (two panel variants, §4 step 12) | `TOUCHSCREEN_ILITEK`, `TOUCHSCREEN_EDT_FT5X06` | controller model and I2C address from the DT, **UNVERIFIED** |
| Display, GPU | RK3399 VOP, Mali-T860 (spec 07) | `DRM_ROCKCHIP`, `DRM_PANFROST` | panel interface and timings from the DT, **UNVERIFIED** |
| USB device (USB-TMC) | `usbtmc_dev.ko` on `libcomposite` (spec 11) | no USB-TMC gadget function; implement in userspace with FunctionFS (`USB_CONFIGFS_F_FS`) | |
| USB host (USB-GPIB adapter) | `usb_gpib.ko`, binds interface class 0xFE/0x03 (USBTMC/USB488) | `USB_TMC` (`/dev/usbtmcN`) | |
| `/dev/dma_auklet` copy engine | optional (spec 14 §6) | not needed | `memcpy` does the same |
