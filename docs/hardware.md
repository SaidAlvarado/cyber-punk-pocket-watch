# Hardware

Status legend: **✅ have it** · **📦 ordered** · **🤔 deciding** · **❌ not started**

## Bill of materials

| Part | Choice | Status | Notes |
|---|---|---|---|
| Radio / main MCU | Nordic **nRF54L15** | 🤔 | DK or module? See below |
| Graphics MCU | **RP2350** | 🤔 | Bare chip, or a Pico 2 for bring-up |
| Display | 1.83" *or* 1.85" Waveshare | 📦 | Both ordered, see comparison |
| **GNSS receiver** | decided in principle, part TBD | 🤔 | Load-switched, phone-absent fallback — [ADR-0007](decisions/0007-onboard-gps.md) |
| **9-DoF IMU** | decided in principle, part TBD | 🤔 | Compass heading — [ADR-0008](decisions/0008-imu-heading.md) |
| Microphone | PDM MEMS mic | ❌ | Must be PDM — connects to nRF54L15 |
| Battery | LiPo, size TBD | ❌ | Depends on enclosure |
| Charging / PMIC | — | ❌ | Consider nPM1300 (pairs with nRF54L15) |
| Load switch (GPS rail) | — | ❌ | Must not cut the GNSS backup rail — see below |
| Buttons | — | ❌ | Count driven by UI design |
| Enclosure | — | ❌ | The "punk" part |

## Display: the open decision

Both ordered, both to be tested on real hardware. **This choice cascades into RAM budget,
rendering approach, and the whole visual design.**

| | **1.83" Touch LCD** | **1.85" Touch LCD** |
|---|---|---|
| Resolution | 240 × 284 | 360 × 360 |
| Shape | Rectangular, rounded corners | **Round** |
| Panel | IPS LCD | IPS LCD |
| Colour | 262K | 262K |
| Driver IC | ST7789P | ST77916 |
| Host interface | **SPI** | **QSPI** |
| Touch IC | CST816D (I2C) | CST816S (I2C) |
| Framebuffer @ 16bpp | 136 KB | **259 KB** |
| Fits RP2350's 520 KB SRAM | Easily — room for double buffering | Yes, single buffer, ~50% of SRAM |
| Product page | [waveshare.com](https://www.waveshare.com/1.83inch-touch-lcd-module.htm) | [waveshare.com](https://www.waveshare.com/1.85inch-touch-lcd-module.htm) |

### How to choose

**Round is the obvious pocket-watch shape**, and 360×360 gives a much better canvas for a
detailed pixel-art face. It's the aesthetically correct answer.

Against it: 2.6× the pixels to push every frame, and a round panel means every layout
must respect a circular safe area — LVGL will happily lay out into corners that don't
exist. Text at the edges is the recurring pain.

The 1.83" is the pragmatic one: less than half the framebuffer, plain SPI, room for
double buffering, and a rectangular grid that pixel art and text both sit on naturally.
Lower pixel density also means chunkier native pixels — arguably *better* for pixel art,
since you fight the "too crisp to read as pixel art" problem less.

**Suggested test when they arrive:** drive both from a Pico 2, measure achievable
full-frame refresh rate and time-to-first-pixel, then render the same pixel-art mockup on
each and judge it by eye. The aesthetic call should be made looking at the real panels.

Tracked in [`decisions/0005-display-selection.md`](decisions/0005-display-selection.md).

### Notes on both

- **These are LCDs, not OLEDs** — accepted. The backlight is always on, so dark pixel-art
  themes save no power and blacks will read as grey rather than true black. Design the
  visual style knowing this.
- **Both include capacitive touch.** Not originally planned — the interaction model was
  buttons. Worth deciding deliberately whether touch is used at all: buttons work with
  gloves and without looking, which suits a bike-mounted compass; touch suits menus.
  Using both is fine, but nothing needed while riding should *require* touch.
- Both expose a 15-pin GH1.25 and an 18-pin FPC connector, with an onboard level
  translator accepting 3.3 V or 5 V.

## GNSS receiver

**Decided**: the watch carries its own GNSS receiver, held powered-down by a load switch
whenever the phone is connected, and brought online only when no phone is available.
Rationale and alternatives in [`decisions/0007-onboard-gps.md`](decisions/0007-onboard-gps.md).

Part not yet chosen. Selection criteria, roughly in priority order:

1. **A backup-supply pin** (`V_BCKP` or equivalent) that keeps the receiver's RTC and
   ephemeris RAM alive on microamps while the main rail is gated off. **This is the single
   most important requirement** — see the cold-start trap below.
2. **Assisted-GNSS support**, so the phone can push fresh ephemeris over BLE while it's
   still connected.
3. Low tracking power — this rail will be on for the whole ride when it's on at all.
4. UART interface (NMEA plus a binary protocol for configuration and AGNSS upload).
5. Physical size, and antenna type — chip antenna vs. patch is an enclosure decision.

### The cold-start trap

A GNSS receiver that has been fully powered off does a **cold start**: it must download
ephemeris from the satellites themselves, which takes tens of seconds under an open sky
and much longer under poor conditions. If the load switch cuts everything, then *every*
fallback activation — exactly when you've lost your phone and need navigation — begins
with a long wait.

Two mitigations, and the design should use both:

- **Keep the backup rail alive.** Gate only the main supply. Preserving the receiver's
  RTC and last-known ephemeris turns a cold start into a warm or hot start.
- **Push assistance data from the phone.** While connected, the companion app can fetch
  current ephemeris over the internet and hand it to the receiver over BLE. When the
  phone disappears, the receiver already knows where to look.

Together these mean the fallback path is fast in practice, which is what makes the whole
gated-GPS idea viable rather than merely frugal.

### Power-state hysteresis

"Phone present" will flap — BLE connections drop and re-establish routinely. Powering the
GNSS rail up and down on every blip wastes energy and produces a useless stream of
half-acquired fixes. Needs a hold-off: only power up after the phone has been gone for
some interval, and only power down after it's been reliably back for some interval.
Values to be tuned on real hardware.

### Antenna

**The hardest physical constraint in the project.** GNSS at 1.575 GHz needs sky view and a
ground plane. A metal "punk" enclosure will substantially degrade or entirely kill
reception, and a pocket watch spends most of its life in a pocket.

Also note **BLE at 2.4 GHz and GNSS at 1.575 GHz coexisting** in a small enclosure — the
BLE transmitter is a strong nearby source and GNSS signals are extremely weak. Antenna
placement, separation, and possibly filtering need real thought at layout time.

Worth an early sanity test: get a candidate module acquiring a fix, then put it inside a
mock-up of the intended enclosure and see what survives.

## 9-DoF IMU — compass heading

**Decided**: heading comes from a 9-DoF IMU on the watch (accelerometer + gyroscope +
magnetometer). Rationale in [`decisions/0008-imu-heading.md`](decisions/0008-imu-heading.md).

Part not yet chosen. The main fork is **on-chip sensor fusion or not**:

- **Fusion on-chip** (e.g. a Bosch BNO-series part) outputs an orientation quaternion
  directly. Far less work, no fusion maths to get right, and calibration routines are
  handled in the sensor. Costs more power, board area and money.
- **Raw sensors + fusion in software** (e.g. a 6-axis IMU alongside a separate
  magnetometer) is cheaper and lower-power, and gives full control — but you own the
  fusion filter, the tilt compensation, and the hard/soft-iron calibration.

For a project whose goal is learning Zephyr rather than learning sensor fusion, on-chip
fusion is the pragmatic choice. Revisit if power measurements rule it out.

`[UNVERIFIED]` — no parts evaluated yet. Note that many "9-axis" parts are an accel+gyro
die and a magnetometer die in one package rather than a single integrated sensor; check
what you're actually buying.

### Why 9-DoF and not just a magnetometer

A magnetometer alone gives a correct heading only when held perfectly flat. Tilt it and
the reading swings wildly. **Tilt compensation needs the accelerometer** to establish
which way is down — that's the accel's real job here, not motion detection. The gyroscope
smooths the result and rejects the noise you get on a bike.

### Other things the IMU pays for

- **Wake on raise** — the natural gesture for a pocket watch, and it needs an always-on
  sensor, which is an argument for putting the IMU on the nRF54L15.
- Step counting, if that ever becomes interesting.
- Motion-gated power saving: nothing moving means nothing needs updating.

### Calibration is not optional

Magnetometers need hard-iron and soft-iron calibration, and the offsets are specific to
this board, this battery, this enclosure. Nearby magnets, the speaker, the battery and any
steel in the case all distort the field. Budget for:

- A calibration routine, and somewhere to store the coefficients persistently.
- A UI for it — the classic "rotate the device in a figure-of-eight" flow.
- Re-testing after any enclosure change.

### Which MCU owns it

**Recommendation: the nRF54L15.** Wake-on-raise needs the always-on core, and if the part
does fusion on-chip there is no number-crunching left to hand to the RP2350 anyway. Even
raw-sensor fusion at 100 Hz is cheap. `[UNVERIFIED]` — revisit if fusion turns out to be
expensive in practice.

## MCU form factor — to decide

**nRF54L15**: the DK is the sane way to learn Zephyr — onboard debugger, broken-out pins,
every sample targets it. A module is what a wearable actually needs. Recommendation:
**develop on the DK, port to a module later.** Don't do both at once while learning Zephyr.

**RP2350**: a Pico 2 is the fastest path to a working display driver. Move to a bare chip
only when the design is settling.

## Pin maps

Not yet assigned — waiting on the display decision and MCU form factor.

<!-- Fill in once hardware is on the bench:

### nRF54L15
| Signal | Pin | Notes |
|---|---|---|

### RP2350
| Signal | Pin | Notes |
|---|---|---|
-->

## Power budget

Not yet estimated. Fill in measured numbers as they're taken — guesses here are worse
than blanks.

Three gated consumers, in expected order of appetite:

| Rail / consumer | Active | Idle / gated | Notes |
|---|---|---|---|
| Backlight | ? | off | Expect this to dominate |
| GNSS | ? | backup rail only | Only on when the phone is absent |
| RP2350 | ? | ? | Gate-vs-sleep decision still open |
| IMU | ? | low-power motion mode | Always on if used for wake-on-raise |
| nRF54L15 | ? | ? | Always on |
