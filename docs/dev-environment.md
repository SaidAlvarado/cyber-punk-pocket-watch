# Development environment

Everything needed to go from a fresh machine to flashing firmware. **Record exact
versions here as they're confirmed working** — this file is the antidote to returning
after six months and finding nothing builds.

Host: Linux (Manjaro), zsh.

> Commands below marked `[UNVERIFIED]` have not yet been run on this machine. Verify,
> then remove the marker and pin the version that worked.

---

## nRF54L15 — Zephyr via nRF Connect SDK

Use the **nRF Connect SDK (NCS)**, not vanilla upstream Zephyr. NCS is Nordic's
distribution: it bundles Zephyr plus Nordic's drivers, the BLE controller, and the samples
that actually target this chip. Nordic's documentation assumes NCS, and for a part this
new that matters. `[UNVERIFIED]` — confirm the minimum NCS version with nRF54L15 support.

### Concepts worth understanding before starting

Read these once; they explain most of the confusion Zephyr causes at the start.

- **`west`** — the meta-tool. Manages a *workspace* (multiple git repos pinned by a
  manifest), and wraps build/flash/debug. `west` is not a build system; CMake is.
- **Workspace layout** — your app is one directory inside a workspace that also contains
  `zephyr/`, `nrf/`, `modules/`. This is why `build/` and those directories are gitignored:
  they're fetched, not authored.
- **Devicetree** — describes *hardware that exists*. Which peripherals, which pins, which
  addresses. Compiled into the build; not runtime configuration.
- **Kconfig** — selects *software that gets compiled in*. Subsystems, drivers, features.
- The two are separate and both are required. A peripheral in devicetree with its driver
  not enabled in Kconfig silently does nothing — this is the single most common beginner
  trap.
- **Overlays** — `.overlay` files patch devicetree per-board; `prj.conf` and
  `boards/*.conf` set Kconfig. This is how you adapt a board without forking it.

### Setup

`[UNVERIFIED]` — the recommended path is the **nRF Connect for VS Code extension**, which
installs the toolchain and SDK for you and is far less error-prone than doing it by hand
the first time. The manual route:

```sh
# Dependencies (Manjaro)
sudo pacman -S --needed git cmake ninja gperf ccache dtc python python-pip

# west, in a virtualenv to avoid polluting system python
python -m venv ~/.venvs/zephyr
source ~/.venvs/zephyr/bin/activate
pip install west

# Initialise the NCS workspace — pin the tag once confirmed
west init -m https://github.com/nrfconnect/sdk-nrf --mr <TAG> ~/ncs
cd ~/ncs && west update
west zephyr-export
pip install -r zephyr/scripts/requirements.txt
```

The **Zephyr SDK** (the toolchain proper) is separate from the source tree and must be
installed too. `[UNVERIFIED]` — record the version used.

### Build and flash

```sh
# From firmware/nrf54l15/
west build -b <BOARD_NAME> .        # e.g. nrf54l15dk/nrf54l15/cpuapp  [UNVERIFIED]
west flash
west build -t menuconfig             # explore Kconfig interactively
west build -t guiconfig
```

Clean rebuild — needed more often than you'd like, especially after devicetree changes:

```sh
west build -b <BOARD_NAME> . --pristine
```

### Serial console

```sh
# Find the port, then:
picocom -b 115200 /dev/ttyACM0     # or minicom / tio
```

---

## RP2350 — Pico SDK

```sh
sudo pacman -S --needed cmake arm-none-eabi-gcc arm-none-eabi-newlib ninja

git clone https://github.com/raspberrypi/pico-sdk.git ~/pico-sdk
cd ~/pico-sdk && git submodule update --init
export PICO_SDK_PATH=~/pico-sdk
```

Build:

```sh
# From firmware/rp2350/
mkdir -p build && cd build
cmake -DPICO_BOARD=pico2 -G Ninja ..
ninja
```

Flash: hold BOOTSEL while plugging in, then copy the `.uf2` to the mounted volume. Or use
a debug probe with `picotool` / OpenOCD for a faster edit-run loop — worth setting up
early, the BOOTSEL dance gets old fast.

> **RP2350 note**: it can run Arm Cortex-M33 *or* RISC-V Hazard3 cores. Stay on Arm unless
> there's a reason not to — toolchain and library support is better trodden.

---

## Android app

Android Studio. `[UNVERIFIED]` — record the versions once the project exists.

Permissions this app will need, all of which require explicit user action beyond a normal
permission prompt:

- `BLUETOOTH_SCAN` / `BLUETOOTH_CONNECT`
- `ACCESS_FINE_LOCATION` — and background location for the compass
- `RECORD_AUDIO`
- Notification listener access — granted in system settings, not by a dialog
- Battery optimisation exemption — otherwise Doze kills the BLE connection

---

## Useful references

| Thing | Where |
|---|---|
| Zephyr docs | https://docs.zephyrproject.org/ |
| nRF Connect SDK docs | https://docs.nordicsemi.com/ |
| Zephyr devicetree guide | https://docs.zephyrproject.org/latest/build/dts/index.html |
| Pico SDK docs | https://www.raspberrypi.com/documentation/microcontrollers/ |
| LVGL docs | https://docs.lvgl.io/ |
| Display product pages | [1.83"](https://www.waveshare.com/1.83inch-touch-lcd-module.htm) · [1.85"](https://www.waveshare.com/1.85inch-touch-lcd-module.htm) |

---

## Troubleshooting log

Add entries as problems are hit and solved. Future-you will thank present-you.

<!--
### Symptom
What was tried, what actually fixed it.
-->

*(empty)*
