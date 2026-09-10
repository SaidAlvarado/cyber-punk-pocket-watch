# ADR-0003: Custom Android app, not Gadgetbridge

**Status**: Accepted
**Date**: 2026-09-08

## Context

Every interesting feature needs the phone: notification forwarding, weather, transit,
speech-to-text, Strava, Obsidian, Google Keep. The watch has no internet access by design.

[Gadgetbridge](https://gadgetbridge.org/) is a mature open-source Android app that talks to
many wearables, and using it would avoid writing an app. The question was whether it could
cover the feature list.

## Decision

**Write a custom Android app.**

## Alternatives considered

### Gadgetbridge with a custom device driver

Assessed against the seven planned features:

| Feature | Gadgetbridge |
|---|---|
| Notification forwarding | ✅ Core feature |
| Media control | ✅ Core feature |
| Rain alert | ⚠️ Forwards weather; 30-min nowcast logic still ours |
| Bike compass | ⚠️ Pushes location; no share-target for a Google Maps pin |
| Strava upload | ❌ |
| Clicker CSV export | ❌ No generic custom-data mechanism |
| Metro departures | ❌ |
| Voice note → STT | ❌ No audio streaming from device at all |

Rejected for two reasons:

1. **The features it provides free are the cheap ones.**
   `NotificationListenerService` and `MediaSessionManager` are roughly a hundred lines
   each. The four it cannot do are the ones that make this watch worth building.
2. **It isn't less work.** Supporting a custom device means writing a device coordinator
   and protocol driver *inside the Gadgetbridge tree*, in Java, conforming to their
   abstractions. That's comparable effort to a focused app, plus the cost of learning
   someone else's framework, plus a dependency on their release cycle.

### Impersonate a Bangle.js

Gadgetbridge's Bangle.js support speaks JSON over Nordic UART and includes an HTTP-request
proxy, which would have covered metro departures and rain nicely, alongside notifications,
media control and location push.

Genuinely clever, and the closest call of the three. Rejected because it constrains the
wire protocol to someone else's design for the lifetime of the project, and still cannot
stream audio — so voice notes, the most distinctive feature, would remain impossible.

### Standard BLE profiles only (ANCS/AMS, plain GATT)

No app at all. Rejected: Android's ANCS support is poor, and it rules out every feature
requiring phone-side computation — which is most of them.

## Consequences

**Easier**
- One place for all phone-side logic; a coherent protocol designed for this watch.
- New features frequently need no firmware change at all.
- Full control over the audio path, which is what makes voice notes possible.

**Harder**
- All of it must be written and maintained. Per the project's working agreement, the
  Android app is Claude's responsibility.
- **The real difficulty is not BLE — it's Android's background execution model.** Doze,
  foreground service requirements, notification-access and `RECORD_AUDIO` permissions, and
  battery-optimisation exemptions. Expect this to consume more time than the protocol.
- Rebuilding the long tail of notification filtering and dedup UX that Gadgetbridge has
  already refined over years. Early versions will be noticeably worse at this.
