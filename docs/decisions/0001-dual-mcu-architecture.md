# ADR-0001: Two MCUs — nRF54L15 + RP2350

**Status**: Accepted
**Date**: 2026-09-08

## Context

The watch needs to maintain a BLE connection to an Android phone, capture audio, keep
time, manage power — and separately render a smooth, animated, pixel-art interface.

The original reasoning was purely mechanical: the chosen display needs QSPI and the
nRF54L15 has none. That reasoning turned out to be conditional. One of the two candidate
panels (the 1.83") is plain SPI at 240×284, whose 136 KB framebuffer would fit in the
nRF54L15's 256 KB of SRAM. A single-MCU design was therefore genuinely on the table.

## Decision

**Two MCUs, regardless of which display is chosen.**

- **nRF54L15** — Zephyr. BLE, microphone, timekeeping, buttons, power management. Always on.
- **RP2350** — display, LVGL, animation, and any heavier computation. Normally powered down.

## Alternatives considered

**Single nRF54L15, driving the 1.83" SPI panel directly.** Fewer parts, one toolchain, one
power rail, no inter-MCU protocol. Rejected because it forces render work onto the core
running the BLE stack. Zephyr's controller has hard real-time radio deadlines; animation
work competing for that core produces connection jitter, missed connection events and
audio dropouts — a category of bug that is miserable to diagnose and would surface late.
It also locks the display choice to the smaller panel forever.

**Single RP2350 with a separate BLE module.** Moves the problem rather than solving it, and
gives up Zephyr — which is a stated learning goal for this project.

## Consequences

**Easier**
- BLE timing is structurally protected from render load.
- Animation gets a dual-core 150 MHz MCU with PIO and 520 KB SRAM to itself.
- Either display remains viable; the panel decision stays open.
- Clean separation of concerns, which suits a project picked up intermittently.

**Harder**
- A second power rail and boot sequence to sequence and debug.
- An inter-MCU protocol to design, version and maintain — a new failure surface. See
  [ADR-0006](0006-inter-mcu-link.md).
- Higher idle power. Mitigated by treating the RP2350 as normally-off, which introduces
  its own wake-latency question (see [`../architecture.md`](../architecture.md)).
- Two toolchains, two flashing procedures, two debug setups.

**Commits us to**
- The microphone living on the nRF54L15, so audio never crosses the link.
- The nRF54L15 acting as power master, owning the RP2350's enable line.
