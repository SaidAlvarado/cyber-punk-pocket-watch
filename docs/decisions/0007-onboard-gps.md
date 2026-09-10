# ADR-0007: On-board GNSS, load-switched, as a phone-absent fallback

**Status**: Accepted (part not yet selected)
**Date**: 2026-09-10

## Context

The bike compass needs position. The phone has a good GNSS receiver, a large battery, and
is normally in the same pocket — so the cheapest design is to use the phone's location
over BLE and put no receiver on the watch at all.

That was the initial recommendation. It was rejected: **the watch should still work when
the phone isn't there.** A phone can be flat, forgotten, or left behind deliberately on a
ride. A navigation aid that fails in exactly that situation isn't much of a navigation aid.

## Decision

**The watch carries its own GNSS receiver, held powered down by a load switch whenever
the phone is connected, and brought online only when no phone is available.**

The nRF54L15 is the power master and owns the load switch, as it owns the RP2350's enable
line and the backlight.

The phone remains the *preferred* position source when present — it is already running,
already has a fix, and costs the watch nothing.

## Alternatives considered

**No receiver; phone GNSS only.** Cheapest in parts, power, and board area. Rejected: no
standalone capability, which is the whole point of the decision.

**Receiver always on.** Simplest firmware, no gating logic, no hysteresis, no
cold-start problem. Rejected on power — a continuously tracking GNSS receiver is one of
the hungriest things that could be put in this enclosure, and it would be doing redundant
work the great majority of the time.

**Receiver powered on only when the compass app is open.** Simpler trigger than
phone-presence. Rejected as strictly worse: it wouldn't save meaningfully more power than
phone-presence gating, and it would impose the acquisition delay every single time the app
is opened, including the common case where the phone is right there with a fix ready.

## Consequences

**Easier**
- The watch is genuinely useful without the phone.
- The phone stays the default source, so the usual case costs nothing.

**Harder — and these need designing, not just noting**

- **Cold starts.** A fully powered-off receiver must re-download ephemeris from the
  satellites, taking tens of seconds under open sky and longer under poor conditions. The
  fallback path would be slowest exactly when it matters. Two mitigations, both required:
  - **Gate only the main rail; keep the backup rail alive.** Preserving the receiver's
    RTC and ephemeris RAM on microamps converts a cold start into a warm or hot start.
    *This makes "has a backup-supply pin" a hard requirement on part selection.*
  - **Push assistance data from the phone.** While connected, the companion app can fetch
    current ephemeris over the internet and upload it over BLE, so the receiver already
    knows where to look when the phone vanishes.
- **Hysteresis.** BLE connections drop and re-establish routinely. Without a hold-off in
  both directions, the rail will thrash and produce a stream of half-acquired fixes.
- **Two position sources to reconcile.** Handover between phone and on-board fixes needs
  defined behaviour — which wins, what happens mid-ride, and how the UI signals which is
  in use. A silent switch between sources of differing accuracy is a confusing bug.
- **Antenna.** GNSS needs sky view and a ground plane; a metal enclosure will degrade or
  kill it, and a pocket watch lives in a pocket. Compounded by BLE at 2.4 GHz sitting next
  to a receiver listening for very weak 1.575 GHz signals. This is the hardest physical
  constraint in the project — see [`../hardware.md`](../hardware.md#antenna).
- More board area, more cost, more power, an extra rail to sequence.

**Commits us to**
- A GNSS part with a backup-supply pin and assisted-GNSS support.
- Assistance-data upload in the Android app and a matching path in the firmware.
- An early antenna feasibility test, before the enclosure design is committed.
