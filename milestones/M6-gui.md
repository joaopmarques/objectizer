# M6 — Cross-platform GUI

Outcome: a desktop application for macOS, Linux and Windows that covers the M2–M5 workflows for people who don't use a terminal or an agent. It contains no format logic: every action calls `objectizer-core::ops`.

Read first: [PLAN.md](../PLAN.md) §3.

Depends on: M2 complete (minimum). Screens for multi-tile, floors/walls and skins require M3, M4, M5 respectively and are feature-gated until those milestones pass.

## Task graph

```
T01 ── T02 ─┬─ T03 ─┐
            ├─ T04 ─┼─ T06 ── T07
            └─ T05 ─┘
```

T03, T04, T05 are parallel after T02.

---

### M6-T01 — Framework decision spike

- **Goal:** Choose egui/eframe or Tauri with evidence.
- **Inputs:** PLAN.md recommendation (egui/eframe: pure Rust, one binary, no web stack, direct access to `objectizer-core` types, good fit for pixel-exact sprite viewing). Counter-case for Tauri: richer UI toolkit, easier theming, but adds a JS toolchain and an IPC layer that duplicates the request/response structs.
- **Outputs:** `docs/gui-decision.md` with a comparison table (binary size, build complexity on the 3 OSes, pixel-exact image rendering with nearest-neighbour zoom, file dialogs, accessibility, testability headless in CI, packaging/signing) and a decision; two throwaway prototypes (each: open an IFF, show one decoded frame at 4× nearest-neighbour) kept on a branch, not merged.
- **Acceptance:**
  - `docs/gui-decision.md` exists with a `## Decision` heading naming one framework and filled rows for all listed criteria (grep check).
  - The user has approved the decision (recorded in the doc by the human). **[human gate]**
- **Depends on:** M2.

### M6-T02 — App shell, install picker, object browser

- **Goal:** Start the app, find the game, browse objects.
- **Inputs:** `docs/gui-decision.md`; PLAN.md §3.1 (ops rule), §4 (`locate`, `list`).
- **Outputs:** crate `objectizer-gui`; view-model layer (`objectizer-gui::vm`, plain Rust, no UI types) that wraps `ops`; install selection (uses `ops::locate`, manual folder pick); object list with search, category and source filters, thumbnails; background indexing with progress; error display using `ErrorCode`.
- **Acceptance:**
  - Tests pass on all three OSes in CI (`cargo test -p objectizer-gui` — view-model tests against synthetic installs; UI smoke test headless if the chosen framework supports it, else behind a feature flag and documented).
  - `cargo build -p objectizer-gui --release` succeeds on the three CI OSes.
  - `grep -rn "chunks::" crates/objectizer-gui/src` returns nothing outside `vm` imports of public `ops` types (the GUI contains no format logic) — enforced by a test.
- **Depends on:** M6-T01.

### M6-T03 — Sprite viewer

- **Goal:** Inspect an object visually.
- **Inputs:** `ops::inspect` and `ops::export` responses; PLAN.md §6.
- **Outputs:** viewer with zoom level and rotation selectors, channel toggle (colour / z / alpha / composite), nearest-neighbour magnification, per-tile and whole-object composite for multi-tile (when M3 is present), palette panel, frame metadata.
- **Acceptance:**
  - View-model tests: selecting zoom/rotation yields the frame set the DGRP specifies (compared with `ops::inspect` output on synthetic fixtures); composite image equals `export --composite` output byte-for-byte.
- **Depends on:** M6-T02.
- **Parallel with:** M6-T04, M6-T05.

### M6-T04 — Clone, recolor and catalog editor

- **Goal:** The core edit flows without a terminal.
- **Inputs:** `ops::clone`, `ops::recolor`, `ops::set_catalog` and their `--dry-run` responses.
- **Outputs:** clone dialog (name, output location, GUID preview); recolor panel (hue/saturation/lightness sliders, live preview, palette swatch import/export); catalog editor (name, description, price, thumbnail picker with preview); every action has a "what will change" preview driven by `--dry-run` responses, and writes only on confirm.
- **Acceptance:**
  - View-model tests: each action produces the same output file bytes as the equivalent CLI command on synthetic fixtures (`gui::parity_clone`, `gui::parity_recolor`, `gui::parity_set_catalog`).
  - Preview never writes (temp-dir hash unchanged test).
- **Depends on:** M6-T02.
- **Parallel with:** M6-T03, M6-T05.

### M6-T05 — Export/import round-trip flow

- **Goal:** Work with an external image editor.
- **Inputs:** `ops::export`, `ops::import`, `ops::validate`; PLAN.md §6 import rules.
- **Outputs:** "Export for editing" (bundle to a chosen folder, button to reveal it), folder watcher that shows which frames changed since export, import with `--palette keep|new` and auto-zoom options, validation results list, one-click "install to Downloads".
- **Acceptance:**
  - View-model tests: changed-file detection matches manifest hashes; import parity with the CLI (`gui::parity_import`).
- **Depends on:** M6-T02.
- **Parallel with:** M6-T03, M6-T04.

### M6-T06 — Floors, walls, skins screens

- **Goal:** Surface M4/M5 features.
- **Inputs:** M4 and M5 `ops` (`floor`, `wall`, `skin`).
- **Outputs:** floor/wall creator (image in, live tile preview at 3 zooms, name/price/sound), skin browser and retexture flow; each feature-gated on its milestone.
- **Acceptance:**
  - View-model parity tests with `floor create`, `wall create`, `skin retexture` on synthetic fixtures.
- **Depends on:** M6-T03, M6-T04, M6-T05; M4 and M5 for the respective screens.

### M6-T07 — Packaging and M6 gate **[human gate]**

- **Goal:** Ship installable builds for the three OSes and prove the GUI flows.
- **Inputs:** docs/testing.md §9 (guard-assets); the chosen framework's packaging docs.
- **Outputs:** release workflow producing a macOS universal `.app` in a `.dmg`, a Windows `.zip`/installer, a Linux AppImage or tarball; unsigned by default with documented signing steps; `docs/checklists/M6-gui.md` covering the five flows on each OS; README updated with download/run instructions.
- **Acceptance:**
  - Release workflow run produces the three artefacts (CI job exits 0, artefact names asserted).
  - `cargo xtask guard-assets` passes on the packaged tree (no game assets bundled).
  - Human checklist passed on macOS and at least one other OS; gate record filled in.
- **Depends on:** M6-T01 … M6-T06.

## Gate record

_To be filled by M6-T07._
