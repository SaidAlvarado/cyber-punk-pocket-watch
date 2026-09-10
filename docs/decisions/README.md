# Architecture Decision Records

Why things are the way they are. The point of these is that in four months the *what* is
recoverable from the code, but the *why* is gone forever unless it's written down.

## Rules

- One decision per file, numbered sequentially, never renumbered.
- **Never delete an ADR.** If a decision is reversed, mark the old one `Superseded by
  ADR-XXXX` and write a new one. The record of a changed mind is more useful than a clean
  directory.
- Record what was *rejected* and why. That's usually the part worth having.
- An ADR can be `Open` — a decision that's been framed but not made, with the criteria
  for making it written down.

## Index

| # | Decision | Status |
|---|---|---|
| [0001](0001-dual-mcu-architecture.md) | Two MCUs: nRF54L15 + RP2350 | Accepted |
| [0002](0002-monorepo.md) | Single monorepo | Accepted |
| [0003](0003-custom-android-app.md) | Custom Android app, not Gadgetbridge | Accepted |
| [0004](0004-lvgl-with-escape-hatch.md) | LVGL with a raw-framebuffer escape hatch | Accepted |
| [0005](0005-display-selection.md) | Which display panel | **Open** |
| [0006](0006-inter-mcu-link.md) | Inter-MCU transport and protocol | **Open** (deferred) |
| [0007](0007-onboard-gps.md) | On-board GNSS, load-switched, phone-absent fallback | Accepted |
| [0008](0008-imu-heading.md) | Compass heading from a 9-DoF IMU | Accepted |

## Template

```markdown
# ADR-XXXX: Title

**Status**: Proposed | Accepted | Open | Superseded by ADR-YYYY
**Date**: YYYY-MM-DD

## Context
What's the situation and what forces are in play?

## Decision
What was chosen.

## Alternatives considered
What else was on the table, and why it lost.

## Consequences
What this makes easier, what it makes harder, what it commits us to.
```
