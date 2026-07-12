# PROJECT — Screensaver Clock Rework

Local customization of NSPanel-Easy's screensaver time display. Not intended for upstream PR unless decided later.

## Charter

1. **The one thing this must do:** Display the screensaver time as a single horizontal line (e.g. `11:39`) instead of the stacked three-line `11 / 39 / AM` layout.
2. **What would be wrong if shipped without it:** If the time still wraps to multiple lines at any supported font size, the feature has failed — "horizontal" means one row, always.
3. **Off-limits workarounds:** None that leave the time wrapping onto more than one line in normal operation. Silent narrowing of "horizontal" to "usually horizontal" is not acceptable.
4. **Deployment target / backup:** Sonoff NSPanel (portrait, 320×480), flashed via ESPHome from this repo's YAML packages. Backup = git (repo is version-controlled; every change is revertable).
5. **Verify it is done:** After reflashing, the screensaver shows the time on one line at the chosen font size. (Cannot be tested on this machine — requires an ESPHome build + flash to the panel.)

## End-state vision

A DVD-logo-style **bouncing clock** on the screensaver: the time (e.g. `11:39`) drifts
around the screen, bounces off the four edges, and changes color — all slow and calm, with
speeds adjustable live from Home Assistant. Reference: https://bouncingdvdlogo.com/ (but slower).

### Home Assistant controls (new ESPHome entities on each panel)
- **Select — "Screensaver Color Mode"**: `Continuous shift` · `On bounce` · `Off/static`
  - *Continuous shift* → color cycles the spectrum on its own timer (Color Speed applies).
  - *On bounce* → classic DVD: color changes only when it hits an edge.
- **Number — "Screensaver Motion Speed"**: drift speed (both moving modes).
- **Number — "Screensaver Color Speed"**: spectrum cycle rate (Continuous mode).

### Rendering approach (confirmed feasible, code-only)
Drive the animation from an ESPHome interval while the screensaver is shown. Each tick:
move the clock a small step, bounce velocity at edges, advance color, then redraw via Nextion
serial draw commands — `xstr` to paint the time at the computed x/y, `fill` to wipe the prior
position. The full-screen `text` component is hidden while bounce mode is active.
No TFT rebuild expected. (`xstr`/`fill` already used inside the HMI — button.txt, switch.txt.)
Movement is stepwise at a low frame rate; fine because the target is deliberately slow.

## Phases

- **Phase 1 — Horizontal alignment** *(DONE — commit d5c3389, pushed to fork main)*
  - Show `HH:MM` only, **drop AM/PM**, so it always fits one line.
  - `esphome/nspanel_esphome_datetime.yaml` — screensaver branch strips `%p`, no `\r` splitting.
  - Code-only. No Nextion/TFT rebuild.
- **Phase 2 — Color engine + HA controls** *(DONE — pending on-device test)*
  - New package `esphome/nspanel_esphome_addon_screensaver_motion.yaml`, included via `core.yaml`
    so all panels get it automatically. Adds HA entities: Color Mode select
    (Off / Continuous shift / On bounce), Color Speed number, Motion Speed number
    (Motion Speed defined now, used in Phase 3).
  - 250 ms interval cycles hue -> RGB565 -> `text.pco` while screensaver shows the clock in
    Continuous mode; only redraws on actual color change. Default mode Off = no behaviour change.
  - Note: in Phase 2 only "Continuous shift" is visible; "On bounce" needs motion (Phase 3).
  - Code-only, no TFT rebuild.
- **Phase 3 — Bounce motion** *(ABANDONED — hardware limitation)*
  - Built a full `xstr`-based bounce engine, but it flickered badly on-device and could not
    be made smooth. Root cause: the NSPanel display is a Nextion **Discovery** series
    (NX4832F035) — it can't move components at runtime and has no double-buffering, so every
    reposition repaints the clock box and briefly blanks it. Inherent to the hardware.
  - Decision (user): **remove motion entirely.** Reverted `datetime.yaml` to the plain
    Phase-1 centered clock and stripped the animation engine from the addon.

- **Final state — stationary clock + optional color cycle**
  - `nspanel_esphome_addon_screensaver_motion.yaml` now only cycles the clock's color in
    place (recoloring via `text.pco` is a managed redraw = no flicker).
  - HA controls: Select "Screensaver Color Mode" (Off / Cycle color) + Number
    "Screensaver Color Speed". Motion Speed entity removed.
  - Watch item: if in-place recolor itself flickers, slow the 200 ms interval or reduce
    update frequency further.

## Notes / Architecture

- Rendering layers: blueprint (settings only) → `datetime.yaml` (builds the string) → HMI `text` component id 4 (Nextion, `hmi/dev/nextion2text/nspanel_portrait/screensaver.txt`).
- The three-line stacking was intentional US-mode behavior (`DISPLAY_MODES.US.TEXT`), stacking to fit large fonts. Phase 1 removes that stacking.
