# ADR-0006: Inter-MCU transport and protocol

**Status**: **Open — deliberately deferred.** Decide when there's something real to measure.
**Date**: 2026-09-10 (superseding the initial 2026-09-08 framing, which leaned UART)

## Context

[ADR-0001](0001-dual-mcu-architecture.md) commits to two MCUs, which must talk to each
other.

One earlier decision removes the obvious bandwidth driver: **the microphone lives on the
nRF54L15**, so audio goes straight from PDM to the BLE stack and never crosses this link.

What does cross it:

- Notification text
- Position, bearing and distance for the compass
- Heading from the IMU — the only continuous stream, and only while the compass is open
- Button and touch events
- Timer and app state changes
- Power and wake commands

All small. The heading stream is the only thing with a sustained rate, and even at 100 Hz
a handful of bytes per sample is trivial for either candidate.

## Status: deferred

Both options are viable and the traffic doesn't force the choice. Deferring until the
display is chosen and there's real UI to measure.

### UART with hardware flow control

- Easy to bring up on both sides; hard to get badly wrong.
- **Trivially inspectable with a logic analyser** — which matters more than it sounds
  when debugging across two MCUs with separate debuggers and separate reset domains.
- Comfortably fast enough for everything listed above.
- Not error-detecting on its own; needs a checksum if corruption matters.

### SPI, RP2350 as controller

- Much faster, with headroom for traffic we haven't thought of.
- Makes runtime asset transfer practical — pushing sprites or fonts from the nRF54L15's
  storage rather than baking every asset into the RP2350's flash. **This is the strongest
  argument for SPI**, and whether it matters depends on how the UI design lands.
- More pins, and the controller/peripheral roles need care: the RP2350 driving the bus
  means the nRF54L15 can't initiate, so an attention line is needed for the peripheral to
  say "I have something for you".
- Harder to eavesdrop on while debugging.

### What would decide it

- Does the UI want runtime asset transfer, or can everything be baked into flash? → the
  main question.
- Does the 1.85" panel win, making the RP2350's SRAM tight enough that streaming assets
  becomes attractive?
- Measured heading-update latency through UART once the compass is real.

## Still to decide, whichever transport wins

- **Framing.** Must resync cleanly after a reset on either side — one MCU rebooting
  mid-message is a normal event here, not an exceptional one.
- **Message encoding.** Hand-rolled structs, CBOR, or a small generated codec. A generator
  in `tools/` would keep both sides in sync automatically.
- **Versioning.** The two MCUs are flashed independently and *will* end up mismatched. A
  version handshake at link-up is cheap insurance.
- **Wake signalling.** Probably a dedicated GPIO rather than in-band, so the RP2350 can be
  fully powered down and still be woken deterministically.
- **Flow control and backpressure** when the RP2350 is busy rendering.

## Consequences

- Whatever is chosen must be documented and versioned in the repo — exactly the kind of
  detail that evaporates over a multi-month gap.
- **Design the message layer so the transport underneath it can be swapped.** That's what
  makes deferring this decision cheap rather than merely postponed: framing, encoding and
  versioning can all be settled and built now, against either wire.
- Worth deciding before the pomodoro app (step 4 in the build order), since that's the
  first thing to exercise the link.
