# Features

The app ideas, analysed. Original unedited list preserved in
[`app-ideas-original.md`](app-ideas-original.md).

## Why there's a custom Android app

The question was whether [Gadgetbridge](https://gadgetbridge.org/) could serve as the
phone side instead of writing an app. It can't. Assessment:

| Feature | Gadgetbridge |
|---|---|
| Notification forwarding | ✅ Core feature |
| Media control | ✅ Core feature |
| Rain alert | ⚠️ Forwards weather, but 30-min nowcast logic is still yours |
| Bike compass | ⚠️ Can push location; no share-target for a Google Maps pin |
| Strava upload | ❌ No integration |
| Clicker CSV export | ❌ No generic custom-data mechanism |
| Metro departures | ❌ No transit polling |
| Voice note → STT | ❌ No audio streaming from device at all |

Two things settled it:

1. **The features Gadgetbridge gives free are the cheap ones.**
   `NotificationListenerService` and `MediaSessionManager` are roughly a hundred lines
   each. The four it can't do are the ones that make this watch worth building.
2. **Gadgetbridge is not less work.** Supporting a custom device means writing a device
   coordinator and protocol driver *inside their tree*, in Java, conforming to their
   abstractions — the same effort as a focused app, plus learning someone else's framework.

A third option was considered and rejected: speaking the **Bangle.js** protocol (JSON over
Nordic UART) so Gadgetbridge treats the watch as a Bangle.js. That would inherit
notifications, media control, location push, and Gadgetbridge's HTTP-request proxy —
which would actually cover the metro and rain features. Rejected because it constrains the
protocol to someone else's design and still can't do audio.

Full reasoning in [`decisions/0003-custom-android-app.md`](decisions/0003-custom-android-app.md).

**The real difficulty in the Android app is not BLE.** It's Android's background execution
rules — Doze, foreground service requirements, and the permission dance for notification
access and `RECORD_AUDIO`. Budget accordingly.

---

## The features

### 1. Notification forwarding
Show the most recent phone notification on the watch face.

- **Android**: `NotificationListenerService`, plus filtering (which apps, dedup, rate
  limiting). Needs the special notification-access permission, granted manually in settings.
- **nRF54L15**: receive over BLE, hold current notification, forward to render core.
- **RP2350**: render text, handle overflow and scrolling.
- **Open**: how many notifications are kept? "The last one" is simplest and matches the
  original spec. History needs a scrollable list and storage.

### 2. Bike compass
Beeline-style needle pointing at a destination shared from Google Maps. Trips recorded and
pushed to Strava.

- **Android**: register as a share target for map/geo intents to capture a destination;
  stream location; record the track; Strava OAuth + upload; push assisted-GNSS data and
  magnetic declination to the watch.
- **nRF54L15**: heading from the on-board 9-DoF IMU
  ([ADR-0008](decisions/0008-imu-heading.md)); position from the phone when connected, or
  the on-board GNSS when not ([ADR-0007](decisions/0007-onboard-gps.md)); compute bearing
  and distance; relay to the render core.
- **RP2350**: draw the needle — **raw framebuffer, not LVGL** (see
  [`decisions/0004`](decisions/0004-lvgl-with-escape-hatch.md)).
- **Works without the phone.** This is the only feature that does, and it's why the watch
  carries its own GNSS.
- **Open**: which GNSS part, which IMU, and whether the antenna survives the enclosure —
  see [`hardware.md`](hardware.md). The antenna is the real risk.
- **Needs building anyway**: magnetometer calibration (routine, storage, UI flow), and a
  defined handover between phone and on-board position sources.
- **Note**: bearing-to-destination is a straight line, not a route. That's what Beeline
  does and it works well on a bike, but be deliberate about it.

### 3. Voice notetaking
Button starts recording; audio goes to the phone; phone transcribes; text is routed to a
configured Obsidian note or Google Keep note.

- **nRF54L15**: PDM capture, stream over BLE. The mic lives here specifically so audio
  never crosses the inter-MCU link.
- **Android**: buffer audio, run `SpeechRecognizer` (or a cloud STT), route the result by
  a preset config, confirm back to the watch.
- **RP2350**: recording indicator, then a confirmation.
- **Open**: on-device vs cloud STT — on-device works offline and is private but less
  accurate. Obsidian integration route is unresolved: an intent to the Android app, or
  writing directly to a synced vault folder on disk?
- **The most technically demanding feature.** Build it last.

### 4. Media control
Play/pause, forward, back on the phone's current media.

- **Android**: `MediaSessionManager` — straightforward.
- **Watch**: map buttons to commands. Optionally display track metadata.
- **Easiest of the seven.** Good early confidence-builder once BLE works.

### 5. Clicker counter
A button increments a counter stored on the watch; timestamped clicks exportable as CSV
from the phone.

- **nRF54L15**: increment, timestamp against the RTC, persist to flash so it survives
  reboot and battery pull. Zephyr's NVS or settings subsystem — a good, well-scoped
  first exercise in Zephyr storage.
- **Android**: pull the log, render CSV, hand off via the share sheet.
- **Open**: what happens when storage fills? Decide the cap and whether it wraps or stops.

### 6. Rain alert
Warn if rain is expected within 30 minutes.

- **Android**: poll a nowcast-capable weather API on a schedule, apply the threshold, push
  an alert. Needs an API with genuine short-term precipitation nowcasting — not every
  weather API has this.
- **Watch**: display it; it's really just a special notification.
- **Note**: mostly reuses feature 1's plumbing. Cheap once notifications work.

### 7. Metro departures
Departure times for the nearby station on the watch face.

- **Android**: location → nearest station → poll the transit API → push departures.
- **Watch**: render a small schedule list.
- **Open**: which transit API? Availability and quality vary enormously by city; some
  offer GTFS-Realtime, many don't. Check before committing.

### 8. Visual pomodoro timer
Mentioned in the original brief, not in the ideas file.

- **nRF54L15**: authoritative timer against the RTC so it survives the screen being off.
- **RP2350**: the visual — this is where animation quality matters most.
- **Fully local.** No phone needed, which makes it a good early app for exercising the
  render pipeline and the inter-MCU link without BLE in the way.

---

## Suggested build order

Ordered so each step de-risks the next, and the hard problems arrive after the
foundations are solid.

1. **Bring-up** — blink on both MCUs, get both toolchains working, pick the display.
2. **Display driver** — the chosen panel rendering something from the RP2350.
3. **Inter-MCU link** — a working, versioned protocol over UART.
4. **Pomodoro timer** — first real app; no BLE, exercises timing + rendering + link.
5. **BLE connection** — nRF54L15 ↔ Android app, bidirectional, reconnecting reliably.
6. **Notification forwarding** — first phone-dependent feature.
7. **Media control** — cheap win on the same plumbing.
8. **Clicker counter** — introduces persistent storage and phone-side export.
9. **Rain alert** — reuses the notification path.
10. **Bike compass** — the biggest single feature. Sub-steps: IMU heading + calibration,
    then GNSS bring-up and the phone/on-board handover, then Strava. Do the antenna
    feasibility test long before this, though — it can invalidate the enclosure design.
11. **Metro departures** — needs a viable transit API.
12. **Voice notetaking** — hardest; do it when everything else is stable.
