# Worklog

Newest entry first. **This is the first thing to read when returning to the project.**

Each entry: what was done, what was decided, what's next, and anything left broken.

---

## 2026-09-10 — Sensor decisions; workspace restructured

**Decided**
- **On-board GNSS after all** ([ADR-0007](decisions/0007-onboard-gps.md)). Reverses the
  2026-09-08 "phone GPS only" recommendation. Held off by a load switch while the phone is
  connected; comes up only when no phone is available, so the watch still navigates when
  the phone is flat or left behind. Part not chosen yet.
- **Compass heading from a 9-DoF IMU** ([ADR-0008](decisions/0008-imu-heading.md)).
  Closes the open heading question. Part not chosen yet.
- **LCD (not OLED) accepted.** No AMOLED hunt; design the visual style around an LCD's
  contrast and treat the backlight as the dominant power term.
- **Inter-MCU transport deferred** ([ADR-0006](decisions/0006-inter-mcu-link.md), rewritten).
  SPI is a genuine contender, not just UART — its real argument is runtime asset transfer.
  Decide once the display is chosen and there's UI to measure.
- **Workspace restructured.** The repo now lives at `code/repository/` inside a wider
  workspace holding `code/scratch/`, `design/`, `reference/` and `notes/`. Workspace-level
  `CLAUDE.md` sits at the workspace root, outside the repo, since it governs the scratch
  and design folders too.

**Consequences worth remembering**
- GNSS part selection now has a **hard requirement**: a backup-supply pin, so the receiver
  keeps its RTC and ephemeris while the main rail is gated. Without it, every fallback
  activation starts with a slow cold start — precisely when you need it most.
- The Android app gains two jobs: pushing **assisted-GNSS data** and **magnetic
  declination** to the watch.
- New work that didn't exist before: magnetometer calibration (routine + persistent
  storage + UI flow), and a defined handover between phone and on-board position sources.
- **Antenna is now the biggest physical risk.** GNSS at 1.575 GHz needs sky view and a
  ground plane, sitting next to a 2.4 GHz BLE transmitter, inside a possibly metallic case,
  in a pocket. Test this before committing to an enclosure design.

**Next**
1. Displays arrive → test both on a Pico 2, close [ADR-0005](decisions/0005-display-selection.md).
2. **Antenna feasibility test** — get a candidate GNSS module acquiring a fix, then put it
   in a mock-up enclosure and see what survives. Do this early; it can invalidate the case.
3. Select the GNSS part (backup-supply pin + assisted GNSS are the filters).
4. Select the IMU — decide on-chip fusion vs. raw sensors first.
5. Get the NCS toolchain installed; verify [`dev-environment.md`](dev-environment.md) and
   remove the `[UNVERIFIED]` markers.

**Still open**: display choice, inter-MCU transport, RP2350 gate-vs-sleep, GNSS part, IMU
part and fusion approach, whether touch is used at all.

**Nothing is broken. No code written yet.**

---

## 2026-09-08 — Project initialised

**Done**
- Set up the monorepo structure and the documentation set.
- Analysed the seven app ideas against Gadgetbridge's capabilities.
- Looked up the specs for both candidate displays.

**Decided**
- **Custom Android app**, not Gadgetbridge ([ADR-0003](decisions/0003-custom-android-app.md)).
  Gadgetbridge can't do voice notes, the clicker CSV, metro departures, or Strava — and
  supporting a custom device in it isn't less work than writing a focused app.
- **Two MCUs regardless of which display wins** ([ADR-0001](decisions/0001-dual-mcu-architecture.md)).
  Originally justified by the nRF54L15 lacking QSPI; kept for timing isolation from the
  BLE stack and animation headroom.
- **LVGL with a raw-framebuffer escape hatch** ([ADR-0004](decisions/0004-lvgl-with-escape-hatch.md)).
  LVGL anti-aliases its vector drawing and image transforms, which fights pixel art. Menus
  and text on LVGL; the compass needle and anything rotating drawn by hand.
- **Monorepo** ([ADR-0002](decisions/0002-monorepo.md)).
- **Microphone on the nRF54L15**, so audio never crosses the inter-MCU link.
- ~~**No GPS on the watch** — use the phone's.~~ *Reversed 2026-09-10, see above.*

**Found out**
- The **1.83" display is SPI, not QSPI** (ST7789P, 240×284). Only the 1.85" is QSPI
  (ST77916, 360×360, round).
- **Both candidates are IPS LCDs, not OLEDs.** Backlight is always on; dark themes save no
  power and blacks will read as grey.
- Both have capacitive touch (CST816D / CST816S) — not originally part of the plan.

**Next**
1. Displays arrive → test both on a Pico 2, measure refresh rate, judge pixel art by eye,
   close [ADR-0005](decisions/0005-display-selection.md).
2. Decide nRF54L15 DK vs module (recommendation: DK for learning).
3. Get the NCS toolchain installed and verify against
   [`dev-environment.md`](dev-environment.md); remove the `[UNVERIFIED]` markers.
4. Resolve the compass heading source — the hardest open hardware question.

**Open / unresolved**
- Which display ([ADR-0005](decisions/0005-display-selection.md))
- Inter-MCU wire protocol format ([ADR-0006](decisions/0006-inter-mcu-link.md))
- Compass heading: magnetometer vs GPS course-over-ground
- Power-gate the RP2350 or sleep it — needs boot-to-first-frame measured
- Whether touch is used at all, given the interaction model was buttons

**Nothing is broken. No code written yet.**
