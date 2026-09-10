# Punk Pocket Watch

A custom smart pocket watch: pixel-art display, BLE companion link to an Android phone,
built around an nRF54L15 running Zephyr.

> **Coming back after a few months?** Read [`docs/worklog.md`](docs/worklog.md) first —
> the newest entry tells you exactly where you left off. Then skim
> [`docs/decisions/`](docs/decisions/) to remember *why* things are the way they are.

## Status

**Phase: architecture and parts selection.** No firmware written yet. Both candidate
displays are on order and untested.

See [`docs/hardware.md`](docs/hardware.md) for what has actually been bought.

## What it's meant to do

Seven app ideas, in rough priority order — full detail and feasibility analysis in
[`docs/features.md`](docs/features.md):

1. **Notification forwarding** — last phone notification on the watch face
2. **Bike compass** — Beeline-style needle pointing at a destination shared from Google Maps; trips logged to Strava
3. **Voice notetaking** — button records audio, phone transcribes it, text lands in Obsidian or Google Keep
4. **Media control** — play/pause/skip the phone's audio
5. **Clicker counter** — button increments a counter; timestamped CSV exportable from the phone
6. **Rain alert** — phone polls a weather service, warns of rain within 30 minutes
7. **Metro departures** — nearby train departure times on the watch face

Plus a visual pomodoro timer.

## Architecture at a glance

```
   ┌──────────────┐         ┌──────────────┐        ┌─────────────┐
   │  nRF54L15    │  TBD    │   RP2350     │  QSPI  │  LCD panel  │
   │  Zephyr      │◄───────►│  display     │───────►│  + touch    │
   │              │         │  rendering   │        └─────────────┘
   │  BLE ◄───────┼── phone │  LVGL        │
   │  mic (PDM)   │         └──────────────┘
   │  GNSS ⏻      │
   │  9-DoF IMU   │
   │  power master│
   └──────────────┘
```

Two MCUs so that render load can never disturb the BLE stack's real-time radio timing —
see [`docs/decisions/0001-dual-mcu-architecture.md`](docs/decisions/0001-dual-mcu-architecture.md).
The watch carries its own GNSS, gated off whenever the phone is present, so navigation
still works with no phone.

## Documentation map

| Document | What's in it |
|---|---|
| [`docs/worklog.md`](docs/worklog.md) | Session-by-session log. **Start here.** |
| [`docs/architecture.md`](docs/architecture.md) | How the pieces fit, and the reasoning |
| [`docs/hardware.md`](docs/hardware.md) | BOM, part status, pin maps, display comparison |
| [`docs/dev-environment.md`](docs/dev-environment.md) | Toolchain setup and flashing commands |
| [`docs/features.md`](docs/features.md) | The app ideas, analysed for feasibility |
| [`docs/zephyr-learning.md`](docs/zephyr-learning.md) | Zephyr concepts notebook |
| [`docs/decisions/`](docs/decisions/) | ADRs — why each choice was made |

## Repo layout

```
firmware/nrf54l15/   Zephyr application — BLE, GNSS, IMU, microphone, power
firmware/rp2350/     Display rendering, LVGL, animations
android/             Companion Android app (Kotlin)
hardware/            Schematics, pin maps, BOM
docs/                Documentation, including architecture decision records
tools/               Helper scripts — flashing, asset conversion, protocol codegen
```

This repo sits inside a larger local workspace (`code/repository/`) that also holds design
sketches, UI mockups, datasheets, and a `code/scratch/` area for study exercises and
throwaway spikes. Only what's here is published.

## Licence

Not yet chosen.
