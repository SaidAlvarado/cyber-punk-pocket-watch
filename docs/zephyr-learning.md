# Zephyr learning notes

A notebook for Zephyr concepts as they're learned. Written for the version of me that
comes back in four months having forgotten all of it.

The mental model below is the minimum needed to stop being confused. Everything past it
gets added as it's encountered.

---

## The mental model

Zephyr is not Arduino and not bare-metal-with-helpers. It's an RTOS with a build system
that assembles a firmware image out of *declarations* about your hardware and *selections*
about your software.

Four things do most of the work:

### `west` — the meta-tool

Manages a **workspace**: a collection of git repositories pinned together by a manifest
file. It also wraps build, flash and debug. `west build` calls CMake; `west` itself is not
the build system.

The practical consequence: your application is a small directory inside a much larger
workspace containing `zephyr/`, `nrf/`, `modules/`. Those are *fetched*, not authored,
which is why they're gitignored here.

### Devicetree — what hardware exists

A declarative description of the hardware: which peripherals are present, at which
addresses, on which pins, with which properties. Inherited from Linux.

- Resolved entirely **at build time**. It is not runtime configuration.
- Boards ship a `.dts`; you patch it with a `.overlay` rather than editing theirs.
- Nodes get referenced from C via macros (`DT_NODELABEL`, `DEVICE_DT_GET`, …).

### Kconfig — what software gets compiled in

Selects subsystems, drivers, and features. Set in `prj.conf`, board-specific `.conf`
files, and explorable with `west build -t menuconfig`.

### The trap that catches everyone

**Devicetree and Kconfig are separate and you need both.**

A peripheral declared in devicetree whose driver isn't enabled in Kconfig will compile
fine and silently do nothing. The reverse — driver enabled, no devicetree node — usually
fails more loudly, but not always helpfully.

When something doesn't work and there's no error: check both, in that order.

---

## Concepts to learn, in the order they'll be needed

Tick them off as they're actually understood, not just read about.

- [ ] Workspace layout and `west update`
- [ ] Devicetree syntax: nodes, properties, labels, `&` references
- [ ] Overlays — adapting a board without forking it
- [ ] Kconfig: symbols, dependencies, why a symbol won't enable
- [ ] GPIO via the devicetree API
- [ ] Threads, and why you usually want a work queue instead
- [ ] Work queues and the system work queue
- [ ] Logging subsystem — and its buffering/deferred-mode surprises
- [ ] Timers, `k_timer` vs kernel uptime vs the RTC
- [ ] Power management: device PM and system PM, and how they interact
- [ ] Bluetooth: GATT services, characteristics, notifications
- [ ] Bluetooth connection parameters and their power implications
- [ ] NVS / settings subsystem for persistent state
- [ ] PDM audio capture
- [ ] Shell subsystem — very useful for interactive debugging

---

## Notes

Add entries as things are learned. Prefer writing down the thing that was *confusing*, not
the thing that was obvious in hindsight.

<!--
### Topic
What confused me, what the actual model is, and the snippet that made it click.
-->

*(empty — nothing built yet)*

---

## Gotchas encountered

<!--
### Symptom
Cause, and fix.
-->

*(empty)*
