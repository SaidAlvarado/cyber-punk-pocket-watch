# ADR-0004: LVGL, with a raw-framebuffer escape hatch

**Status**: Accepted
**Date**: 2026-09-08

## Context

The interface is meant to be **pixel-art inspired**. That is a hard aesthetic requirement,
not a theme choice — pixel art is defined by hard edges and an exact pixel grid, and any
smoothing destroys the effect.

The obvious library is [LVGL](https://lvgl.io/): mature, widgets, fonts, input handling,
animation, well supported on RP2350-class hardware.

## Decision

**LVGL as the base, with a documented path to draw directly to the framebuffer for
screens that need pixel-exact control.**

A thin abstraction over the display so an individual screen can either build an LVGL
widget tree or take the framebuffer and draw into it by hand.

## Why not LVGL alone

LVGL handles part of pixel art well and part of it badly, and the boundary matters.

**Works fine:**
- Bitmap image assets at 1-bpp or indexed colour, drawn at native scale
- Bitmap fonts converted at `bpp=1`, giving hard-edged glyphs with no anti-aliasing
- Styling away every rounded corner, gradient and shadow

**Fights us:**
- **Vector drawing is anti-aliased.** Arcs, lines and rounded rectangles come out with
  soft edges.
- **Image transforms are smoothed.** Rotation and scaling interpolate rather than doing
  nearest-neighbour.

The compass needle sits squarely in the second category: a rotating shape, redrawn every
frame at arbitrary angles. Through LVGL it will look like a smoothed vector graphic, which
is precisely the wrong aesthetic. It wants either nearest-neighbour rotation or
pre-rendered sprite frames — both of which mean drawing it ourselves.

## Alternatives considered

**LVGL only.** Simplest, fastest to a working UI. Rejected: the compass — a flagship
feature — would look wrong, and there'd be no path to fix it without re-architecting.

**Fully custom renderer, no LVGL.** Total aesthetic control and the best animation
performance. Rejected as too much work for a learning project that also has to teach
Zephyr, design a BLE protocol and build an Android app. Menus, scrolling lists, text
layout and input handling are a lot of unglamorous code to rewrite badly.

## Consequences

**Easier**
- Menus, lists, settings and text get LVGL's widgets for free.
- The compass, pomodoro animation and anything rotating can be pixel-exact.
- The choice is deferrable per screen rather than upfront for the whole UI.

**Harder**
- Two rendering idioms in one codebase — each screen must declare which it uses, and the
  boundary must stay clean or it'll rot.
- The abstraction needs designing before much UI exists. Retrofitting it later is painful.
- Buffer ownership between LVGL and hand-drawn screens must be explicit — LVGL typically
  renders through partial buffers rather than owning a full framebuffer, so "take the
  framebuffer" needs a concrete meaning decided at implementation time.

**Practical guidance**
- Default to LVGL. Drop to the framebuffer only when there's a visible reason.
- Author pixel-art assets at native resolution and avoid runtime scaling entirely; if
  scaling is unavoidable, use integer factors with nearest-neighbour.
- Convert fonts at `bpp=1`. Anti-aliased text will quietly undermine the whole look.
