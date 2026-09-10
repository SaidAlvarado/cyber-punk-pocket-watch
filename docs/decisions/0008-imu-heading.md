# ADR-0008: Compass heading from an on-board 9-DoF IMU

**Status**: Accepted (part not yet selected)
**Date**: 2026-09-10

## Context

The bike compass points a needle at a destination. That needs two things: where you are,
and **which way you are facing**. GNSS answers the first and not the second — standing
still at a junction, a position fix cannot tell you which way you're pointed, which is
precisely the moment you look at the compass.

## Decision

**Heading comes from a 9-DoF IMU on the watch** — accelerometer, gyroscope and
magnetometer.

## Alternatives considered

**Course-over-ground from GNSS.** Free, no extra hardware. Rejected as a primary source:
it only works while moving, degrades badly at low speed, and is useless at a standstill.
Still worth keeping as a **sanity check and fallback while riding**, where it's reliable
and immune to magnetic distortion.

**The phone's fused orientation over BLE.** The phone already does good sensor fusion.
Rejected because the phone's orientation is not the watch's — in a pocket, the phone is
pointing wherever it happens to be lying. It also fails whenever the phone is absent,
which [ADR-0007](0007-onboard-gps.md) has already committed to supporting.

**Magnetometer alone.** Rejected on physics, not cost: a magnetometer gives a correct
heading only when held flat. Tilt it and the reading swings wildly. Correcting for that
requires knowing which way is down, which requires an accelerometer — so a "3-DoF
compass" isn't actually a thing you can build. Hence 9-DoF.

## Consequences

**Easier**
- True heading at a standstill — the compass works when you're stopped at a junction.
- Works with no phone, consistent with [ADR-0007](0007-onboard-gps.md).
- The IMU pays for itself elsewhere: **wake-on-raise** (the natural pocket-watch gesture),
  motion-gated power saving, and step counting if that ever becomes interesting.

**Harder**

- **Calibration is not optional.** Hard-iron and soft-iron offsets are specific to this
  board, this battery and this enclosure. Needs a calibration routine, persistent storage
  for the coefficients, a UI flow for it, and re-testing after any enclosure change.
- **Magnetic environment.** The battery, any speaker, any steel in the "punk" case, and
  the bike itself all distort the field. A handlebar mount puts the watch near a lot of
  metal. Expect this to be an ongoing annoyance rather than a solved problem.
- **True vs. magnetic north.** Magnetic declination varies by location and drifts over
  time. Needs a declination correction — most cheaply pushed from the phone, which knows
  the position and can carry a model.
- Another always-on sensor in the power budget, if used for wake-on-raise.

**Open**

- **On-chip fusion or fusion in software?** A part that outputs an orientation quaternion
  directly removes the fusion filter, the tilt compensation maths and much of the
  calibration work, at the cost of power, area and money. Given this project's goal is
  learning Zephyr rather than learning sensor fusion, on-chip fusion is the pragmatic
  choice — but it isn't decided. See [`../hardware.md`](../hardware.md#9-dof-imu--compass-heading).
- **Which MCU owns it.** Leaning nRF54L15: wake-on-raise needs the always-on core, and
  on-chip fusion would leave nothing to hand to the RP2350 anyway.
