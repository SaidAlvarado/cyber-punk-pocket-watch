# Architecture

## Overview

Three cooperating pieces: the watch's radio/sensor MCU, the watch's graphics MCU, and an
Android companion app. The phone does all the internet-facing work; the watch does none.

```
      ┌───────────────────────────────────────────┐
      │                 WATCH                     │
      │                                           │
      │  ┌──────────────┐        ┌─────────────┐  │      ┌──────────────┐
      │  │  nRF54L15    │  TBD   │   RP2350    │  │      │   Android    │
      │  │  (Zephyr)    │◄──────►│             │  │      │   phone      │
      │  │              │        │  LVGL +     │  │      │              │
 BLE ─┼──┤  BLE peer    │        │  framebuffer│  │      │  companion   │
      │  │  microphone  │        │             │  │      │  app         │
      │  │  GNSS  ⏻     │        │  QSPI ──────┼──┼──► LCD              │
      │  │  9-DoF IMU   │        │  I2C ───────┼──┼──► touch            │
      │  │  RTC / time  │        │             │  │      │              │
      │  │  power master│        │  dual core  │  │      │              │
      │  │  buttons     │        │             │  │      │              │
      │  └──────────────┘        └─────────────┘  │      └──────┬───────┘
      └───────────────────────────────────────────┘             │
                                                                ▼
                                                       internet services
                                              (weather, transit, STT, Strava,
                                               Obsidian, Google Keep)
```

## Why two MCUs

The nRF54L15 could, on paper, drive the smaller SPI display on its own. We're using two
anyway. The reasons, in order of weight:

1. **Timing isolation.** Zephyr's BLE controller has hard real-time radio deadlines.
   Running LVGL animations and rendering on the same core competes for CPU at exactly the
   wrong moments and shows up as connection jitter, missed connection events, and dropped
   audio. A dedicated render MCU removes that class of bug entirely.
2. **The 1.85" panel requires it.** No QSPI on the nRF54L15, and a 360×360 16bpp
   framebuffer is 259 KB against 256 KB of total SRAM. Not close.
3. **Headroom for smooth, reactive animation.** RP2350 is dual-core at 150 MHz with PIO
   and a large SRAM. Animation quality is a stated goal, not a nice-to-have.

### What it costs

Be honest about the trade:

- A second power rail and a second boot sequence to manage.
- An inter-MCU protocol to design, version, and debug — a new failure surface.
- More idle power. Mitigated by treating the RP2350 as normally-off (see below).
- Two toolchains, two flashing procedures, two debug setups.

Decided in [`decisions/0001-dual-mcu-architecture.md`](decisions/0001-dual-mcu-architecture.md).

## Responsibility split

### nRF54L15 — the always-on core

Runs Zephyr. This chip is awake whenever the watch is on.

- BLE peripheral: the single connection to the phone
- Timekeeping (GRTC), alarms, the pomodoro timer's authoritative clock
- Microphone capture over PDM
- **GNSS receiver** — normally gated off; brought up only when no phone is present
  ([ADR-0007](decisions/0007-onboard-gps.md))
- **9-DoF IMU** — compass heading and wake-on-raise
  ([ADR-0008](decisions/0008-imu-heading.md))
- Button input
- **Power master** — owns the RP2350's enable line, the display backlight, and the GNSS
  load switch
- Persistent state that must survive the screen being off: the clicker counter, pending
  notifications, timer state, magnetometer calibration coefficients

### RP2350 — the render core

- Drives the panel (QSPI or SPI depending on which display wins) and reads touch over I2C
- LVGL for menus, lists, text, layout
- Raw framebuffer drawing for anything rotating or pixel-exact — the compass needle
  above all (see [`decisions/0004-lvgl-with-escape-hatch.md`](decisions/0004-lvgl-with-escape-hatch.md))
- Any heavier computation we want off the radio core
- Normally powered down or dormant; woken by the nRF54L15

### Android app — everything internet-facing

The watch never talks to the internet. Every network feature is the phone's job:
weather polling, transit APIs, speech-to-text, Strava upload, Obsidian and Keep writes.

This keeps the watch firmware small and means new features often need no firmware change
at all.

## Key design decisions

### The microphone lives on the nRF54L15

Not on the RP2350. If the mic were on the render core, every audio sample would have to
cross the inter-MCU link *and then* go out over BLE — doubling the bandwidth requirement
and adding buffering and latency for no benefit. On the nRF54L15, captured audio goes
straight from PDM to the BLE stack.

Consequence: **the inter-MCU link never carries audio.** It carries control messages, UI
state, notification text, and GPS fixes. That's a low-rate link.

### Inter-MCU link: deliberately deferred

Because audio doesn't cross it, the link's traffic is small — text, coordinates, heading,
button events, state changes. Both UART and SPI are comfortably capable.

UART is easier to bring up and trivially inspectable with a logic analyser. SPI is far
faster and would make runtime asset transfer practical — pushing sprites from the
nRF54L15's storage rather than baking everything into the RP2350's flash.

**Deferred until the display is chosen and there's real UI to measure.** In the meantime,
design the message layer so the transport underneath can be swapped: framing, encoding
and versioning can all be settled now, against either wire.

Tracked in [`decisions/0006-inter-mcu-link.md`](decisions/0006-inter-mcu-link.md).

### Position: phone first, on-board GNSS as fallback

The phone is the preferred position source — it already has a fix and costs the watch
nothing. The watch's own GNSS receiver stays gated off while the phone is connected and
comes up only when no phone is available, so the watch still navigates when the phone is
flat or left behind.

Two consequences worth carrying in your head: the receiver's **backup rail must stay
powered** even when the main rail is gated, or every fallback begins with a slow cold
start; and **phone-presence needs hysteresis**, because BLE connections flap and the rail
must not thrash. Both in [ADR-0007](decisions/0007-onboard-gps.md).

### Power strategy

The nRF54L15 is the power master, with three gated consumers: the backlight, the RP2350,
and the GNSS rail. The backlight and the RP2350 should both be off whenever the screen is
off; the GNSS rail follows phone presence, not screen state.

Open question worth resolving early: **fully power-gating the RP2350 vs. putting it in a
low-power state.** Full gating saves the most, but the RP2350 boots from external QSPI
flash, and boot + LVGL init on every wrist-raise could mean a visible delay before the
screen is usable. A dormant/sleep state wakes faster but leaks more. This needs measuring
on real hardware — it directly determines whether the watch feels instant or sluggish.

Also note both candidate panels are **IPS LCDs, not OLEDs**: the backlight dominates the
power budget and a mostly-black pixel-art UI saves nothing. Backlight timeout is the
single highest-leverage power setting.

## Data flow examples

**Notification arrives**
```
phone notification → Android NotificationListenerService → BLE characteristic
→ nRF54L15 → UART → RP2350 → render on screen
```

**Voice note**
```
button → nRF54L15 wakes, starts PDM capture
→ audio streamed over BLE → phone buffers it
→ Android SpeechRecognizer → text
→ routed by config to Obsidian or Google Keep
→ confirmation back over BLE → UART → RP2350 → "saved" on screen
```

**Bike compass**
```
destination shared from Google Maps → Android share target → stored

position:  phone GPS fix (preferred)  ─┐
           on-board GNSS (no phone)   ─┴→ nRF54L15
heading:   on-board 9-DoF IMU          →  nRF54L15
                                           │
                        bearing + distance ▼
                                    link → RP2350
                → needle drawn to the framebuffer directly (not LVGL)
```

Position and heading come from different sources and that's inherent, not incidental:
GNSS tells you where you are, the IMU tells you which way you're facing. Neither
substitutes for the other.

## Open architectural questions

| Question | Blocked on | Where |
|---|---|---|
| Which display? | Both arriving; test on hardware | [`decisions/0005-display-selection.md`](decisions/0005-display-selection.md) |
| Inter-MCU transport: UART or SPI? | Deferred — needs real UI to measure | [`decisions/0006-inter-mcu-link.md`](decisions/0006-inter-mcu-link.md) |
| Power-gate the RP2350 or sleep it? | Measuring boot-to-first-frame | this doc, Power strategy |
| Which GNSS part? | Needs backup-supply pin + assisted GNSS | [`hardware.md`](hardware.md#gnss-receiver) |
| IMU with on-chip fusion, or raw + software? | Power vs. effort trade | [`hardware.md`](hardware.md#9-dof-imu--compass-heading) |
| Antenna feasibility in a metal enclosure | Early bench test needed | [`hardware.md`](hardware.md#antenna) |
| Handover behaviour between phone and on-board fixes | Nothing — can be designed now | [`decisions/0007-onboard-gps.md`](decisions/0007-onboard-gps.md) |
