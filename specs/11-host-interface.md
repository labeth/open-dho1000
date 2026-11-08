# 11 — Host Interface

The DHO1000 offers one SCPI command set over four transports (LAN VXI-11, LAN raw socket, USB-TMC
device, USB-GPIB adapter), plus a browser remote and an FTP file share. All SCPI transports share one
parser, one error queue and one IEEE 488.2 status model. A replacement firmware must present the
transports, identities and byte formats below; it may implement them any way it likes.

The public **DHO1000/DHO4000 Programming Guide** is the normative reference for the commands, their
parameters and ranges. This spec names only commands from that guide (and IEEE 488.2 common
commands), and states the rules that apply across them. Everything here is **UNVERIFIED** on a unit
unless a measurement says otherwise.

---

## 1. Transports

| Transport | Endpoint | Notes |
|---|---|---|
| Raw SCPI socket | TCP **5555**, all addresses | VISA `TCPIP::<ip>::5555::SOCKET`; newline-terminated |
| VXI-11 | ONC RPC portmapper TCP/UDP 111; DEVICE_CORE 0x0607AF, DEVICE_ASYNC 0x0607B0 and DEVICE_INTR 0x0607B1, version 1, on TCP **20712** and UDP **20712**; device name `inst0` | VISA `TCPIP::<ip>::INSTR`. A portmapper must answer on port 111 (spec 01 §3 for the one already running at boot) |
| USB-TMC (USB488) device | rear USB-B device port | `libcomposite.ko` + `usbtmc_dev.ko` (gadget), char dev `/dev/usbtmc_dev`; §1.3 |
| USB-GPIB | USB-GPIB adapter on a front USB-A host port | `usb_gpib.ko` (`/dev/usbgpib0`; equivalent to the mainline host `usbtmc` driver plus ioctl 0x5b0a = set GPIB address). The scope is USB host to the adapter (class FE/03); GPIB address `:SYSTem:GPIB` |
| HTTP web remote | TCP 80 redirected to **8080** | §7 |
| WebSockets | TCP 9001–9004 | §7 |
| FTP | TCP 21, root `/data/UserData/`, writable | user files (setups, waveforms, screenshots); spec 01 §3 for the server already running at boot |

HiSLIP (4880) is not offered.

### 1.1 Raw socket

One SCPI message per line, terminated by `\n`. Responses end with `\n`. Binary responses (§4, §6)
are IEEE 488.2 definite-length blocks followed by `\n`. Several clients may connect at once.

### 1.2 VXI-11

Standard VXI-11 (ONC RPC; record marking on TCP). The portmapper answers port 20712 for every
GETPORT query (procedure 0 answers void, other procedures "no such procedure"). One listening socket
per protocol serves all three programs.

- `create_link`: `device` must be exactly `inst0` (else error 1). Returns an opaque `lid`,
  `abortPort` 20712 and `maxRecvSize` 65536 (TCP) / 8800 (UDP). `lockDevice` takes the instrument
  lock, waiting up to `lock_timeout` ms for another link's lock (error 11 on timeout).
- `device_write`: data longer than `maxRecvSize` → error 5; empty data → no error and nothing
  written. Flag `end` (0x08) marks the end of a message. A lock held by another link gives error 11.
  Result codes: I/O timeout 15, I/O error 17, abort 23.
- `device_read`: `termChrSet` (0x80) and `termChar` are ignored (reason CHR is never set). Each call
  returns at most `requestSize` bytes of the pending response; `reason` = REQCNT (1) when the request
  size is reached, END (4) when the last byte of the response is returned (both may be set). Errors as
  for write.
- `device_readstb` → status byte; `device_trigger` → `*TRG`; `device_clear` → device clear;
  `device_remote` / `device_local`; failures map to error 8, 23, 17 or 15.
  `device_lock` / `device_unlock` (unlock by another link → error 12); `device_enable_srq` stores the
  enable flag and handle; `destroy_link` releases the link and its lock.
- Not supported: `create_intr_chan` (error 8; no SRQ interrupts are sent), `destroy_intr_chan`
  (error 6), `device_docmd` (error 8).
- `device_abort` (DEVICE_ASYNC, same port): aborts the link's operation in progress (error 4 if `lid`
  is not this connection's link).

### 1.3 USB-TMC device

| Field | Value |
|---|---|
| idVendor | **0x1AB1** |
| idProduct | **0x0610** |
| bcdUSB / bcdDevice | 0x0200 / 0x0000 |
| bMaxPacketSize0 | 64 |
| iProduct | model string, e.g. `DHO1074` |
| iSerialNumber | instrument serial number |
| Interface | class 0xFE, subclass 0x03, protocol 0x01 (USBTMC USB488), 3 endpoints |
| Endpoints | bulk OUT, bulk IN (max packet 64 FS / 512 HS / 1024 SS), interrupt IN (max packet 64, bInterval 16 HS) |

With the unit's gadget module the device is created by `insmod libcomposite.ko`, then
`insmod usbtmc_dev.ko iSerialNum=<serial> iProductDesc=<model> ProductId=1552` (1552 = 0x0610).
Bulk messages are standard USBTMC (MsgID, bTag, ~bTag, TransferSize, EOM). An open implementation may
use any USBTMC gadget (e.g. FunctionFS) that presents the same descriptors.

### 1.4 Interface arbitration

A response goes to the interface the query came in on. Commands from different interfaces execute
one at a time.

---

## 2. SCPI message rules

- Syntax: IEEE 488.2 / SCPI. Headers are case-insensitive, long or short form (upper-case letters of
  the long form), leading colon optional, `?` marks a query, several commands may be joined with `;`.
  Numeric suffixes are part of the node: `CHANnel1`–`4`, `MATH1`–`4`, `BUS1`–`4`, `REFerence1`–`10`.
- Status model (IEEE 488.2): `STB`, `ESR`/`ESE`, `SRE`, and the SCPI operation and questionable status registers.
  Overlapped commands complete through `*OPC`/`*WAI`. After power-on `*ESR?` reports bit 7 (PON).
- Compound messages: units joined with `;` are executed in order. A unit without a leading colon
  continues the header path of the previous unit (SCPI tree rule): `:CHANnel1:SCALe?;OFFSet?` reads
  CH1 scale and CH1 offset. Common commands (`*IDN?`) may be mixed in. The responses of all queries
  in one message are joined with `;` and terminated by one `\n`:
  `:CHANnel1:SCALe?;:TIMebase:MAIN:SCALe?` → `5.000000E-01;2.000000E-6\n`.
- Numeric arguments: decimal numbers (NR1/NR2/NR3) with an optional SI multiplier (`EX`, `PE`, `T`,
  `G`, `MA`, `K`, `M` = milli, `U`, `N`, `P`, `F`, `A`) and an optional unit. A firmware must accept
  plain numbers in base units (`0.2`, `1e-5`) and an integer with a bare multiplier (`200m`); clients
  should send only those forms, because a typed unit after the multiplier (`200mV/div`) or a
  multiplier after a fraction is not portable across instruments. Integer parameters truncate toward
  zero (`:WAVeform:POINts 500.7` → 500).
- Out-of-range numbers are clamped to the parameter's limits without an error (`:CHANnel1:SCALe 1000`
  → 100 V/div with the 10X probe; `:TRIGger:EDGE:LEVel 150` → 4.5 div).
- Errors go into the SCPI error queue (`:SYSTem:ERRor[:NEXT]?`), formatted `<code>,"<text>"`; an empty
  queue answers code 0. Codes per failure kind (SCPI standard codes; the text is the SCPI standard
  text or a short equivalent):

  | Failure | Code |
  |---|---|
  | Unknown header | -100 or -113 |
  | Missing argument | -109 |
  | Too many arguments, including any argument on a command or query that takes none (`:RUN 1`, `:CHANnel1:SCALe? 1`) | -108 |
  | Not a number (`abc`, `.5`, `200mV/div`) | -120 |
  | Quoted string for a number (`"1"`) | -158 |
  | Unknown enum keyword | -222 |
  | Invalid bool keyword (`:CHANnel1:DISPlay MAYBE`) | -141 |
  | Value or state refused (e.g. `:WAVeform:SOURce` of a hidden channel) | -200 (ESR bit 4) |
  | Number out of range | none (clamped) |

  The setting is left unchanged on any error. An unknown *query* produces no response (the client
  times out) plus the error entry. Error-queue depth and overflow behaviour are **UNVERIFIED**.
- `*IDN?` → `RIGOL TECHNOLOGIES,<model>,<serial>,<software version>\n` (format from the programming
  guide), e.g. `RIGOL TECHNOLOGIES,DHO1074,<serial>,<version>`. Model and serial come from the unit's
  identity (spec 01 §2, `/dev/vendor_storage`).

### 2.1 Command set

The firmware implements the commands of the public programming guide that its features support, with
the guide's parameters and ranges. Commands that are not in the public guide are not part of this
interface. A remote-control open firmware needs only the subset of §3.

### 2.2 Reply formats

| Reply | Format | Example |
|---|---|---|
| Integer values (`:ACQuire:AVERages?`, `:WAVeform:POINts?`) | NR1 | `500` |
| Booleans | `1` / `0` | |
| Enumerations | short form | `CHAN1`, `NORM`, `VECT`, `FULL`, `DEF` |
| Channel scale, offset, position | C `%E` (`%.6E`, signed exponent of at least 2 digits) | `5.000000E-01`, `1.000000E+02`, `0.000000E+00` |
| Time, sample rate, trigger level and the other scaled reals | "NR3EX", below | `2.000000E-6` |
| Memory depth (`:ACQuire:MDEPth?`), `yincrement` | `%.3E` | `1.000E+04` |

**NR3EX:** the value is printed with 7 significant digits (float32 precision; float32 rounding shows in
the last digit) as `d.ddddddE<exp>`, where the exponent is a plain decimal integer: no leading zeros,
`+` only for exponents ≥ 1, no sign for 0. Zero prints as `0.000000` (no exponent). Examples:
`2.000000E-6` (2 µs/div), `1.000000E0` (1 s/div), `1.000000E+1` (10 s/div), `5.000000E+8`
(500 MSa/s), `2.500000E-1` / `-2.500000E-1` (±0.25 V trigger level), `0.000000` (level 0 V),
`-8.999999E-6` (−9 µs).

---

## 3. Core command subset (normative for open firmware)

Commands on CHANnel2–4 behave as on CHANnel1.

| Command | Forms | Values |
|---|---|---|
| `:RUN` / `:STOP` / `:SINGle` / `:TFORce` | set | — |
| `:AUToset` | set | run autoset |
| `:CLEar` | set | — |
| `*IDN?` `*RST` `*CLS` `*OPC[?]` `*ESR?` `*STB?` `*ESE[?]` `*SRE[?]` `*WAI` `*TST?` `*SAV` `*RCL` | | IEEE 488.2 |
| `:CHANnel1:DISPlay` | set/query | bool |
| `:CHANnel1:SCALe` | set/query | V/div (reply `5.000000E-01`) |
| `:CHANnel1:OFFSet` | set/query | V (reply `0.000000E+00`) |
| `:CHANnel1:POSition` | set/query | V |
| `:CHANnel1:COUPling` | set/query | DC, AC, GND |
| `:CHANnel1:BWLimit` | set/query | OFF, ON, 20M (ON = 20M on DHO1000); 250M is for models with that limit |
| `:CHANnel1:PROBe` | set/query | ratio, as in the guide |
| `:CHANnel1:INVert` | set/query | bool |
| `:CHANnel1:UNITs` | set/query | WATT, AMPere, VOLTage, UNKNown |
| `:CHANnel1:VERNier` | set/query | bool (fine) |
| `:CHANnel1:IMPedance` | set/query | OMEG (FIFTy not supported on DHO1000) |
| `:TIMebase:MAIN:SCALe` / `:OFFSet` | set/query | s |
| `:TIMebase:MODE` | set/query | MAIN, XY, ROLL |
| `:TIMebase:DELay:ENABle` / `:SCALe` / `:OFFSet` | set/query | zoom window |
| `:ACQuire:TYPE` | set/query | NORMal, PEAK, AVERages, HRESolution, ULTRa |
| `:ACQuire:AVERages` | set/query | int (2^n) |
| `:ACQuire:MDEPth` | set/query | AUTO, 1k, 10k, 100k, 1M, 10M, 25M, 50M, 100M, 125M, 200M, 250M, 500M; the query answers points in `%.3E` (`1.000E+04`) |
| `:ACQuire:SRATe?` | query | Sa/s |
| `:TRIGger:MODE` | set/query | EDGE, PULSe, SLOPe, VIDeo, PATTern, DURation, TIMeout, RUNT, WINDow, DELay, SETup, NEDGe, RS232, IIC, … (guide) |
| `:TRIGger:SWEep` | set/query | AUTO, NORMal, SINGle |
| `:TRIGger:STATus?` | query | TD, WAIT, RUN, AUTO, STOP |
| `:TRIGger:EDGE:SOURce` | set/query | CHANnel1–4, ACLine, EXT |
| `:TRIGger:EDGE:SLOPe` | set/query | POSitive, NEGative, RFALl |
| `:TRIGger:EDGE:LEVel` | set/query | V |
| `:TRIGger:COUPling` | set/query | DC, AC, LFReject, HFReject |
| `:TRIGger:HOLDoff` | set/query | s |
| `:TRIGger:NREJect` | set/query | bool |
| `:WAVeform:*` | | §4 |
| `:DISPlay:DATA? [PNG\|BMP\|JPG]` | query | §6 |
| `:DISPlay:CLEar` | set | — |
| `:DISPlay:TYPE` | set/query | VECTors |
| `:DISPlay:GRADing:TIME` | set/query | MIN, 0.1, 0.2, 0.5, 1, 2, 5, 10, INFinite |
| `:DISPlay:WBRightness` / `:GBRightness` | set/query | 0–100 % |
| `:DISPlay:GRID` | set/query | FULL, HALF, NONE |
| `:DISPlay:COLor` / `:RULers` / `:WHOLd` | set/query | bool |
| `:SYSTem:ERRor[:NEXT]?` | query | `<code>,"<text>"` |
| `:SYSTem:VERSion?` | query | SCPI version string (`3.0`) |
| `:SYSTem:OPTion:STATus? <opt>` | query | §5 |
| `:SYSTem:LOCKed` | set/query | bool (panel lock) |
| `:SYSTem:PON` | set/query | DEFault, LATest |
| `:SAVE:SETup <path>` / `:LOAD:SETup <path>` | set | string; complete before the next command is parsed |
| `:SAVE:WAVeform <path>` | set | string |
| `:SAVE:IMAGe <path>` / `:SAVE:IMAGe:FORMat` | set | PNG, BMP, JPG |

---

## 4. Waveform transfer

### 4.1 Setup

| Command | Values | Default |
|---|---|---|
| `:WAVeform:SOURce` | CHANnel1–4, MATH1–4 (digital channels are not available on DHO1000) | CHAN1 |
| `:WAVeform:MODE` | NORMal \| MAXimum \| RAW | NORM |
| `:WAVeform:FORMat` | BYTE \| WORD \| ASCii | BYTE |
| `:WAVeform:POINts` | NORMal: 1–1000 · RAW: 1 – memory depth · MAXimum: as NORMal or RAW (below) | 1000 |
| `:WAVeform:STARt`, `:WAVeform:STOP` | 1-based first/last point of the read window | 1 / points |

- **NORMal** reads the on-screen record: **1000 points** across the 10 horizontal divisions.
- **RAW** reads acquisition memory. It is meant for STOP; in RUN it also returns the whole memory
  (10 000 points) without an error; whether that data is consistent while acquiring is
  **UNVERIFIED**. MATH sources support NORMal only.
- **MAXimum** (preamble type 1) takes its geometry when `:WAVeform:MODE` or `:WAVeform:SOURce` is set:
  if the instrument is running then (`:TRIGger:STATus?` not STOP), the NORMal screen geometry
  (≤ 1000 points, `xincrement = 10 × s/div / 1000`), e.g. `0,1,1000,1,2.000000E-8,-1.000000E-5,…`;
  otherwise the RAW geometry (memory depth, `1 / sample rate`, memory start time), e.g.
  `0,1,10000,1,2.000000E-9,-1.000000E-5,…`. A later RUN/STOP does not change it until MODE or
  SOURce is set again.
- Window: `POINts` reads `STOP − STARt + 1`. Setting `POINts n` sets `STOP = STARt + n − 1`;
  setting `STARt` keeps `STOP` (STARt 101 after POINts 500 → POINts 400, STOP 500). The limit is
  1000 in the screen geometry (NORMal, MATH sources, MAXimum set while running) and the memory depth
  in the RAW geometry: `STARt` below 1 becomes 1; a `STARt`, `STOP` or resulting `STOP` above the
  limit is refused (`-200`); an accepted `STOP` is then clamped to the points actually available.
  `:WAVeform:DATA?` and the preamble use this window.
- Setting `:WAVeform:MODE` to a different mode resets the window to `STARt 1`, `STOP` = all points
  of the new geometry (RAW with POINts 500 and STARt 101, then `MODE NORMal` → 1000 points).
  Setting the same mode again, `:WAVeform:SOURce` or `:WAVeform:FORMat` keeps a window that was
  set explicitly; parts not set explicitly follow the new geometry.
- `:WAVeform:SOURce` of a channel that is not displayed is refused (`-200`); the source keeps its old
  value. If the selected source is hidden later, `:WAVeform:DATA?` answers the empty block
  `#9000000000\n` with no error entry.

### 4.2 `:WAVeform:DATA?`

- **BYTE / WORD:** an IEEE 488.2 definite-length block. Up to 500 000 000 data bytes the header is
  `#9` + 9-digit length (`#9000001000`). Longer blocks (WORD RAW above 250 Mpts) use the header
  `#B` + 11-digit zero-padded length (e.g. `#B01000000000`), so clients must accept `B` as a digit
  count of 11 (**UNVERIFIED**). No data gives `#9000000000`.
  - **WORD:** 2 bytes per point, **little-endian, unsigned 16-bit**. Screen centre (0 V with zero
    offset) = **32768**; **7500 codes per vertical division**.
  - **BYTE:** 1 byte per point = the **high byte of the WORD value** (`word >> 8`). Centre 128,
    7500/256 ≈ **29.297 codes per division** (not an integer).
- **ASCii:** comma-separated volts computed from the WORD codes, format `%+e` (`+2.364666e-01`), no
  block header, `\n` at the end (1000 points → 14 000 bytes). The preamble shows the WORD
  yincrement and yreference 32768 for format 2.
- RAW returns the whole `STARt…STOP` window in **one block** (1 M points BYTE = 1 000 012 bytes, WORD
  = 2 000 012 bytes). Clients may still step `STARt`/`STOP` for smaller reads.

### 4.3 `:WAVeform:PREamble?`

Ten comma-separated fields (field names from the programming guide):

```
<format>,<type>,<points>,<count>,<xincrement>,<xorigin>,<xreference>,<yincrement>,<yorigin>,<yreference>
```

| Field | Meaning | Output format |
|---|---|---|
| format | 0 = BYTE, 1 = WORD, 2 = ASC | int |
| type | 0 = NORMal, 1 = MAXimum, 2 = RAW | int |
| points | points in the read window | int |
| count | averages in average mode, else 1 | int |
| xincrement | seconds per point | NR3EX (§2.2) |
| xorigin | time of point 1 relative to the trigger (does not follow `STARt`) | NR3EX |
| xreference | reference point index (always 0) | NR3EX zero: `0.000000` |
| yincrement | volts per code | `%.3E` |
| yorigin | `B − 32768` (WORD, ASC) or `trunc((B − 32768) / 256)` toward zero (BYTE), B = the channel's WORD code for 0 V | int |
| yreference | code of the screen centre: **128 (BYTE) / 32768 (WORD, ASC)** | int |

Example (CH1 0.5 V/div, 2 µs/div, 10 kpts):

```
0,0,1000,1,2.000000E-8,-1.000000E-5,0.000000,1.707E-02,0,128        BYTE NORMal
1,0,1000,1,2.000000E-8,-1.000000E-5,0.000000,6.667E-05,0,32768      WORD NORMal
1,2,10000,1,2.000000E-9,-1.000000E-5,0.000000,6.667E-05,0,32768     WORD RAW (STOP)
0,2,1000000,1,5.000000E-10,-2.490000E-4,0.000000,3.413E-03,29,128   BYTE RAW, 1 M, 0.1 V/div, offset 0.5 V, time offset 1 µs
```

Rules:

- `yincrement = V/div / 7500` (WORD) and `V/div / 7500 × 256` (BYTE), printed with 4 significant digits.
- NORMal: `xincrement = 10 × s/div / 1000`; `xorigin = time offset − 5 × s/div`.
- RAW / MAXimum: `xincrement = 1 / sample rate`; `xorigin = time offset − (memory depth / 2) ×
  xincrement`, the time of memory point 1, independent of `:WAVeform:STARt` (e.g. STARt 1001,
  STOP 2000 gives the same xorigin as the full read).
- yorigin example: offset −0.3 V at 0.5 V/div → WORD `-4500`, BYTE `-17` (−17.58 truncated).
- In the 1 M example the offset had been set to 0.5 V at 0.5 V/div (yorigin 29) before V/div was
  changed to 0.1 V/div; yorigin stayed 29, i.e. the trace position was kept (offset rescaled with
  V/div). **UNVERIFIED**: whether the offset is rescaled or clamped on a V/div change.

Conversion:

- `volt[i] = (code[i] − yorigin − yreference) × yincrement`
- `t[i] = xorigin + (STARt − 1 + i − xreference) × xincrement`, i = 0-based index in the block
  (NORMal and RAW alike: the NORMal xorigin also stays at the left screen edge when STARt > 1)

Each field is also available on its own: `:WAVeform:XINCrement?`, `:XORigin?`, `:XREFerence?`,
`:YINCrement?`, `:YORigin?`, `:YREFerence?`. They print as in the preamble, except `:XREFerence?`,
which answers NR1 `0` (the preamble field is `0.000000`).

---

## 5. Options query

- `:SYSTem:OPTion:STATus? <opt>` → `1`/`0`; `<opt>` as listed in the programming guide (DHO1000
  options there: `DHO1000-BW7T10`, `DHO1000-BW7T20`, `DHO1000-BW10T20`, `DHO1000-RLU`).

Open firmware answers this from its own configuration. Install/uninstall commands are out of scope.

---

## 6. Screen image

`:DISPlay:DATA? [PNG|BMP|JPG]` (also `:SAVE:IMAGe:DATA?`) returns
`#9<9-digit length><image file bytes>\n`. The argument selects the file format; **without an argument
the format is PNG**. The image is the full 1280 × 800 screen.

`:SAVE:IMAGe` options (`:SAVE:IMAGe:COLor`, `:INVert`, `:HEADer` = model + date watermark,
`:FORMat`) apply to the saved image.

---

## 7. Web remote

The unit serves a browser remote: HTTP on TCP 8080 (port 80 redirected to it) and WebSockets on TCP
9001 (live display), 9002 (remote touch and keys), 9003 (screenshot and recording) and 9004 (an SCPI
bridge to the raw socket of §1.1). The page and its scripts are served by the instrument itself, so
no client outside the instrument depends on these endpoints. An open firmware's web UI can be
completely different and may use other ports; it should not leave the ports above open with nothing
behind them.

---

## 8. Open (hardware only)

| Item | Measurement |
|---|---|
| `*IDN?` version field on a retail unit | `*IDN?` over TCP 5555 |
| USB488 capability bits (`GET_CAPABILITIES` response) and actual enumeration strings | `lsusb -v -d 1ab1:0610` on a host PC |
| VXI-11 behaviour of §1.2 with common clients (NI-VISA, `python-vxi11`): lock, abort, empty write | `vxi11` client / Wireshark against the unit |
| `#B` long-block header of §4.2 | WORD RAW read above 250 Mpts |
