# open-dho1000

Specifications for porting [open-dso](https://github.com/labeth/open-dso) to the Rigol DHO1000
oscilloscopes.

The DHO1000 series (DHO1072, DHO1074, DHO1102, DHO1104, DHO1202, DHO1204) are 12-bit bench
oscilloscopes with 2 or 4 channels, 70 to 200 MHz bandwidth, up to 2 GSa/s and a 10.1-inch touch
screen. Inside is a Rockchip RK3399 running Android, a Xilinx Artix-7 FPGA with 1 GiB of DDR3 record
memory behind it, one 12-bit ADC, and a small microcontroller for the front-panel keys and LEDs. The
CPU reaches the FPGA over PCIe and loads it at every boot.

This repository is the specification stage of an open-dso scope. There is no firmware here yet, and
nothing in it has been checked on a unit: every hardware fact is marked **UNVERIFIED** until someone
measures it. The reference unit for the specs is the DHO1074.

## What the specs cover

[`specs/`](specs/README.md) describes the unit's hardware and interfaces in our own words, so that a
firmware, and later an FPGA design, can be written from them:

- the system: boot order, device nodes, GPIOs, update and recovery paths, and how custom software can
  be started without changing the unit (01);
- the FPGA register map and what the logic behind it does: acquisition, timebase, triggering, record
  memory, play-back and the PCIe sample stream (02–05, 14, 21);
- the analog side: front ends, offset DACs, clock chip and ADC, and the ADC's capture into the FPGA
  (06, 20);
- the FPGA itself: every ball, its configuration and the board clocks (12, 13);
- display, front panel, calibration data and the host interfaces (07, 08, 10, 11);
- what the DHO1000 adds to the core's control plane (09), and a test environment that runs software
  against a model of the board (16).

Start with [`specs/00-overview.md`](specs/00-overview.md).

## Starting a port

1. Read [adding a scope](https://github.com/labeth/open-dso/blob/main/docs/adding-a-scope.md): it
   describes the path from specifications to a simulator, a board for the core's HAL and first contact
   with a unit.
2. The firmware is built on [open-dso-core](https://github.com/labeth/open-dso-core), which brings
   the engine, UI, renderer, web server and SCPI. A port implements the core's board interface for
   this unit from these specs.
3. A first port can drive the FPGA image the unit already loads at boot (specs 02–05 and 21 describe
   its register interface). An open FPGA design is a separate, later step, written from specs 12, 20
   and 21.
4. Before touching a unit, back up its calibration, its FRAM and its vendor storage (spec 01 §2).
   Spec 01 §10.7 lists ways to start custom software that leave the unit unchanged, so that a reboot
   returns to the vendor firmware.

Contributions are written from the specifications; see the family's
[provenance rules](https://github.com/labeth/open-dso/blob/main/docs/contributing.md).

## Licence

MIT, see [LICENSE](LICENSE).
