# ADR-0005: Which display panel

**Status**: **Open** — both panels ordered, decision deferred until they can be tested
**Date**: 2026-09-08

## Context

Two Waveshare modules were bought so the choice could be made against real hardware rather
than spec sheets.

| | **1.83" Touch LCD** | **1.85" Touch LCD** |
|---|---|---|
| Resolution | 240 × 284 | 360 × 360 |
| Shape | Rectangular, rounded corners | **Round** |
| Panel | IPS LCD | IPS LCD |
| Colour | 262K | 262K |
| Driver IC | ST7789P | ST77916 |
| Host interface | **SPI** | **QSPI** |
| Touch IC | CST816D (I2C) | CST816S (I2C) |
| Framebuffer @16bpp | 136 KB | 259 KB |
| Approx. density | ~186 PPI | ~275 PPI |
| Link | [product page](https://www.waveshare.com/1.83inch-touch-lcd-module.htm) | [product page](https://www.waveshare.com/1.85inch-touch-lcd-module.htm) |

Both fit comfortably in the RP2350's 520 KB SRAM, so this no longer affects the MCU count
— see [ADR-0001](0001-dual-mcu-architecture.md), which fixed the two-MCU design
independently.

## The trade-off

**For the 1.85" (round, 360×360)**
- Round is the correct pocket-watch shape. This is a real argument, not a cosmetic one.
- 2.6× the pixels — a far better canvas for a detailed pixel-art face.

**Against it**
- 2.6× the data per frame; QSPI bandwidth and refresh rate need measuring.
- 259 KB framebuffer is half the RP2350's SRAM, so double buffering is tight.
- **A round panel means every layout must respect a circular safe area.** LVGL will
  happily lay text into corners that physically do not exist. This is a persistent,
  recurring annoyance rather than a one-off problem.

**For the 1.83" (rectangular, 240×284)**
- Half the framebuffer; room for double buffering.
- Plain SPI — simpler to bring up.
- A rectangular grid that pixel art and text both sit on naturally.
- Lower density means chunkier native pixels, which is arguably *better* for pixel art —
  less fighting the "too crisp to read as pixel art" problem.

## How to decide

When both arrive, on a Pico 2:

1. Bring each up and measure **achievable full-frame refresh rate** and
   **time-to-first-pixel from cold**.
2. Render the **same pixel-art mockup** on both.
3. Judge by eye, in the hand, at arm's length.

**Make the aesthetic call looking at the real panels.** Density, contrast and how pixel
art actually reads at these sizes do not come across on a spec sheet.

## Notes carried forward regardless of choice

- **Both are IPS LCDs, not OLEDs — accepted.** The original plan assumed OLED. The
  backlight is always on, so dark themes save no power and blacks read as grey rather
  than true black. Sourcing a QSPI AMOLED instead was considered and rejected; the visual
  design should be built around an LCD's contrast, and the backlight should be treated as
  the dominant term in the power budget.
- **Both include capacitive touch**, which was not part of the original interaction model.
  Decide deliberately whether touch is used. Buttons work with gloves and without looking,
  which suits the bike compass; touch suits menus. Using both is fine — but nothing needed
  while riding should *require* touch.
