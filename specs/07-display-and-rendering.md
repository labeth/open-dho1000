# 07 — Display and Rendering

The DHO1000 has a 1280 × 800 touch panel driven by the RK3399's display controller. The FPGA does
not render pixels: it delivers sample records (specs 03, 14), and everything on the screen is drawn
by the CPU and its GPU. This spec gives the display hardware and what an open firmware's renderer
must provide on this unit. The choice of renderer is free.

---

## 1. Panel

| Property | Value |
|---|---|
| Resolution | 1280 × 800, landscape, 10.1" |
| Interface | MIPI DSI from the RK3399 VOP; controller model **UNVERIFIED** (JD9365 class expected; read the panel node of the unit's device tree) |
| Density | 1 dp = 1 px (mdpi) under Android, **UNVERIFIED** (`getprop ro.sf.lcd_density`) |
| Backlight | PWM backlight, `/sys/devices/platform/backlight/backlight/backlight/brightness`, 0–255; the boot sets 255. Low values may switch the panel visibly dark: keep a floor (50 is a safe choice) unless the user asks for less. **UNVERIFIED** |
| GPU | Mali-T860 (RK3399), EGL + OpenGL ES 3.0 under Android; Panfrost on a mainline kernel (spec 01 §11) |
| Touch | reports 1280 × 800 pixel coordinates (spec 08 §8) |
| Refresh | **UNVERIFIED** (60 Hz expected; `dumpsys SurfaceFlinger --latency`) |

---

## 2. Screen geometry

### 2.1 Waveform area

The layout of the screen is the firmware's choice. The waveform area should be as wide as the
screen allows (about 1266 px with 4 px margins), and must work at any size the UI gives it, since
results, measurement and settings panels take space from it.

- **Graticule: 10 horizontal × 8 vertical divisions.** The time of column i of a 10-division area
  of width w is `(i·10/w − 5) × s/div + delay`; the voltage of row j of an 8-division area of
  height h is `(4 − j·8/h) × V/div − offset`.
- Several waveform windows (main, zoom, XY, FFT, math, reference, …) may be shown at once; each
  needs its own drawing area.

### 2.2 Trace colours

The front-panel channel LEDs and the channel keys carry the channel colours (spec 08 §6): CH1
yellow, CH2 cyan, CH3 magenta, CH4 blue. Traces should use the same colours so the panel and the
screen agree. Background: black.

---

## 3. Drawing surfaces

Under Android an application draws through Android surfaces only; there is no application-owned
framebuffer. Two ways work on this unit:

- **Under Android:** draw traces with GLES 3.0 into a `TextureView` or `SurfaceView` surface and the
  rest of the UI with ordinary views. Request an RGBA8888 EGL config with stencil 8 and 4× MSAA,
  and fall back to one without MSAA if the first is refused.
- **Without Android** (mainline kernel, spec 01 §11): a DRM/KMS plane on the VOP, drawn by the CPU
  or by Panfrost.

Keep traces in their own layer, so the rest of the UI can redraw without re-rendering waveforms.

---

## 4. Trace rendering

The hardware delivers display records of 16-bit codes, 32768 = screen centre, 7500 codes per
division (spec 03 §2), possibly several frames per play-back. The renderer must provide:

- **Density display**: hits per pixel accumulated over all frames of a play-back, with vector
  segments between consecutive samples, and an intensity scale that keeps rare paths visible (a
  logarithmic mapping works), adjustable by a waveform brightness setting.
- **Colour grade**: an optional mode that maps density to a colour ramp instead of the channel
  colour.
- **Persistence**: minimum (each frame replaces the last), timed decay (0.1–10 s) and infinite.
  The decay must follow wall-clock time, not the frame rate, since the frame rate varies with the
  timebase.
- **Compression**: when a record holds more points than the area has columns, each column shows
  the minimum-to-maximum envelope of its points, so narrow events survive.
- **Clipping markers**: samples above or below the screen are marked at the top and bottom edges.
- **CLEAR**: clears all density and persistence state.

The frame rate is set by new acquisition data (spec 03); there is no display-rate timer in the
hardware.

---

## 5. Display settings

The display settings are the public SCPI `:DISPlay` subsystem of the DHO1000/DHO4000 programming
guide: `:DISPlay:TYPE` (VECTors), `:DISPlay:GRADing:TIME` (MIN, 0.1 … 10 s, INFinite),
`:DISPlay:WBRightness` and `:DISPlay:GBRightness` (0–100 %), `:DISPlay:GRID` (FULL, HALF, NONE),
`:DISPlay:RULers`, `:DISPlay:COLor`, `:DISPlay:WHOLd`, `:DISPlay:CLEar` (and `:CLEar`). Spec 11
covers the host interface.

---

## 6. Screenshots

There is no hardware screenshot path. Screen captures (`:DISPlay:DATA?`, `:SAVE:IMAGe`, the web
remote) are made by the firmware from its own composition, at 1280 × 800 (spec 11 §6).

---

## 7. Open-firmware notes

- GLES 3.0 on the Mali is available under Android; a CPU renderer into an Android `Surface`, or a
  DRM/KMS plane if Android is replaced, also works.
- Draw order: background and grid → traces → overlays (cursors, markers, labels).

## 8. Open (hardware only)

| Item | Measurement |
|---|---|
| Panel refresh rate | `dumpsys SurfaceFlinger --latency` |
| Display density | `getprop ro.sf.lcd_density` |
| Panel controller and timings | the panel node of `/sys/firmware/fdt` |
| Lowest usable backlight value | step `brightness` down on a unit |
