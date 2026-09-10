# ADR-0002: Single monorepo

**Status**: Accepted
**Date**: 2026-09-08

## Context

Four distinct codebases: Zephyr firmware, RP2350 firmware, an Android app, and hardware
design files. They could live in separate repositories or one.

The dominant constraint is not technical. This is a solo project worked on in bursts with
multi-month gaps. **Context recovery is the main cost**, not build hygiene.

## Decision

One repository, split by directory:

```
firmware/nrf54l15/   firmware/rp2350/   android/   hardware/   docs/   tools/
```

### The repo is not the whole workspace

The repo lives at `code/repository/` inside a larger local workspace that also holds
design sketches, UI mockups, datasheets, and `code/scratch/` for study exercises and
one-off spikes.

That split is deliberate: **the repo is what gets published and must stay coherent**,
while the workspace around it is free to be messy. Large binaries (datasheets, raster
assets) and half-finished experiments stay out of git without needing to be tidied first.

Workspace-level `CLAUDE.md` sits at the workspace root, outside the repo, because it
governs work in the scratch and design folders too.

## Alternatives considered

**Separate repos per component.** Cleaner CI, independent versioning, smaller clones.
Rejected because it multiplies the places where context hides. Coming back after six
months, finding four repos at four different commits with no shared history of *why* is
exactly the failure this project is trying to avoid.

## Consequences

**Easier**
- One place to look. One `git log` telling the whole story.
- A protocol change spanning firmware and Android is one atomic commit.
- Documentation sits next to the code it describes.

**Harder**
- CI must be path-filtered so an Android change doesn't rebuild firmware.
- The Zephyr workspace does not naturally nest inside an app repo — `west` expects the
  application to sit *inside* a workspace containing `zephyr/`, `nrf/` and `modules/`.
  Those are gitignored here; the workspace is initialised outside the repo and the app
  directory referenced into it. Confirm the exact arrangement during toolchain setup and
  document it in [`../dev-environment.md`](../dev-environment.md).
