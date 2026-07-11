# PROJECT — Screensaver Clock Rework

Local customization of NSPanel-Easy's screensaver time display. Not intended for upstream PR unless decided later.

## Charter

1. **The one thing this must do:** Display the screensaver time as a single horizontal line (e.g. `11:39`) instead of the stacked three-line `11 / 39 / AM` layout.
2. **What would be wrong if shipped without it:** If the time still wraps to multiple lines at any supported font size, the feature has failed — "horizontal" means one row, always.
3. **Off-limits workarounds:** None that leave the time wrapping onto more than one line in normal operation. Silent narrowing of "horizontal" to "usually horizontal" is not acceptable.
4. **Deployment target / backup:** Sonoff NSPanel (portrait, 320×480), flashed via ESPHome from this repo's YAML packages. Backup = git (repo is version-controlled; every change is revertable).
5. **Verify it is done:** After reflashing, the screensaver shows the time on one line at the chosen font size. (Cannot be tested on this machine — requires an ESPHome build + flash to the panel.)

## Phases

- **Phase 1 — Horizontal alignment** *(in progress)*
  - Decision: show `HH:MM` only, **drop AM/PM**, so it always fits one line.
  - Change: `esphome/nspanel_esphome_datetime.yaml` — screensaver branch no longer replaces `:`/space with `\r`; strips `%p`.
  - Code-only. No Nextion/TFT rebuild.
- **Phase 2 — Color cycling** *(not started)*
  - Cycle `text.pco` through colors on an ESPHome interval timer. Code-only, no TFT rebuild.
- **Phase 3 — DVD-style bounce** *(not started)*
  - Drive `text.x`/`text.y` from ESPHome to bounce the clock off the screen edges.
  - Likely needs an HMI/TFT edit (shrink the `text` box from full-screen to clock-sized, handle trail-clearing) in the Nextion Editor. To be scoped when we get there.

## Notes / Architecture

- Rendering layers: blueprint (settings only) → `datetime.yaml` (builds the string) → HMI `text` component id 4 (Nextion, `hmi/dev/nextion2text/nspanel_portrait/screensaver.txt`).
- The three-line stacking was intentional US-mode behavior (`DISPLAY_MODES.US.TEXT`), stacking to fit large fonts. Phase 1 removes that stacking.
