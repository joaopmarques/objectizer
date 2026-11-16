# M6 — Cross-platform GUI

Outcome: a desktop application for macOS, Linux and Windows that covers the M2–M5 workflows for people who don't use a terminal or an agent, centred on a browsable, searchable catalog of **every object in the user's game** (base game and all expansions inside FAR archives, plus Downloads). It contains no format logic: every action calls `objectizer-core::ops`. The search index and previews behind the catalog are core-library features exposed through the CLI and MCP first (M6-T08, M6-T09), so agents get the same search and the GUI stays a thin layer.

Read first: [PLAN.md](../PLAN.md) §3.

Depends on: M2 complete (minimum). **M6-T08 and M6-T09 are core/CLI/MCP tasks with no GUI dependency: they can and should be done right after M2 (M6-T09 after M3-T03), long before the rest of M6.** Screens for multi-tile, floors/walls and skins require M3, M4, M5 respectively and are feature-gated until those milestones pass.

## Task graph

```
T08 ── T09 ─────────────┐                 (core, CLI, MCP — no GUI)
T01 ── T02 ─────────────┴─ T10 ─┬─ T03 ─┐
                                ├─ T04 ─┼─ T06 ── T07
                                └─ T05 ─┘
```

T08 → T09 is independent of T01 → T02. T10 needs T02, T08, T09. T03, T04, T05 are parallel after T10 (they open from the catalog view).

---

### M6-T01 — Framework decision spike

- **Goal:** Choose egui/eframe or Tauri with evidence.
- **Inputs:** PLAN.md recommendation (egui/eframe: pure Rust, one binary, no web stack, direct access to `objectizer-core` types, good fit for pixel-exact sprite viewing). Counter-case for Tauri: richer UI toolkit, easier theming, but adds a JS toolchain and an IPC layer that duplicates the request/response structs.
- **Outputs:** `docs/gui-decision.md` with a comparison table (binary size, build complexity on the 3 OSes, pixel-exact image rendering with nearest-neighbour zoom, file dialogs, accessibility, testability headless in CI, packaging/signing) and a decision; two throwaway prototypes (each: open an IFF, show one decoded frame at 4× nearest-neighbour) kept on a branch, not merged.
- **Acceptance:**
  - `docs/gui-decision.md` exists with a `## Decision` heading naming one framework and filled rows for all listed criteria (grep check).
  - The user has approved the decision (recorded in the doc by the human). **[human gate]**
- **Depends on:** M2.

### M6-T02 — App shell and install picker

- **Goal:** Start the app and find the game. (The object catalog itself is M6-T10.)
- **Inputs:** `docs/gui-decision.md`; PLAN.md §3.1 (ops rule), §4 (`locate`, `list`).
- **Outputs:** crate `objectizer-gui`; view-model layer (`objectizer-gui::vm`, plain Rust, no UI types) that wraps `ops`; install selection (uses `ops::locate`, manual folder pick); navigation frame with placeholder panes; background task runner with progress and cancellation (used first for `ops::index_build`); error display using `ErrorCode`.
- **Acceptance:**
  - Tests pass on all three OSes in CI (`cargo test -p objectizer-gui` — view-model tests against synthetic installs; UI smoke test headless if the chosen framework supports it, else behind a feature flag and documented).
  - `cargo build -p objectizer-gui --release` succeeds on the three CI OSes.
  - `grep -rn "chunks::" crates/objectizer-gui/src` returns nothing outside `vm` imports of public `ops` types (the GUI contains no format logic) — enforced by a test.
- **Depends on:** M6-T01.

### M6-T08 — Persistent object index and search (`index`, `list`) — core, CLI, MCP

- **Goal:** One local, fast, queryable index of every object in the game, shared by the CLI, MCP and GUI.
- **Inputs:** `install::Index` from M2-T05 (this task supersedes its cache with a richer, persistent one); PLAN.md §4 (`index`, `list`), §4a (index record and query model).
- **Outputs:**
  - `objectizer_core::index`: `IndexRecord` per logical object (PLAN.md §4a: guid, all tile guids, name and description per language present, internal name, source file and FAR entry, source group `base | expansion:<n> | shared | downloads | other`, origin `maxis | custom`, price, catalog kind/category/sub-sorts, catalog-visible flag, tile count and footprint, sprite chunk kinds, has-thumbnail, file size, content hash); built by scanning every FAR, loose IFF and Downloads; stored as one versioned file in the platform cache dir (`<cache>/objectizer/index/<install-id>.bin` + small JSON header); incremental rebuild keyed by path + size + mtime; safe concurrent readers, single writer with lock file; corrupt or old-schema index is rebuilt automatically.
  - Query engine (in memory, no external database): case- and accent-insensitive token/substring search over name, description, internal name, file name and GUID hex; filters `--name`, `--guid`, `--category`, `--kind`, `--source <group|path-glob>`, `--origin maxis|custom`, `--multi-tile`/`--single-tile`, `--price-min`, `--price-max`, `--visible`/`--hidden`, `--lang`; `--sort name|price|source|guid`, `--limit`, `--offset`; total count always returned.
  - CLI: `objectizer index build [--force]`, `index status`, `index clear`; `objectizer list` (object mode) consumes the index and gains `--query <text>` and the filters above; `list` builds the index on first use and prints a notice on stderr. MCP: `index_build`, `index_status`, and the same new arguments on `list`.
  - Origin rule documented: `custom` = under Downloads or a loose file not shipped by the game (determined by location; a content-hash allow-list of Maxis files is **not** shipped).
- **Acceptance:**
  - Tests pass.
  - `index::build_synthetic`: a synthetic install (testkit: 3 FARs, loose IFFs, Downloads, one multi-tile, duplicate GUID across files) indexes to the expected records; duplicate GUIDs are both listed with `W_DUPLICATE_GUID`.
  - `index::incremental`: touching one file re-reads only that file (read counter); deleting a file removes its records.
  - `index::query_table`: table-driven tests for every filter and sort, including accent-insensitive match and GUID prefix match.
  - `index::scale`: a generated index of 20,000 records answers a `--query` + two filters in < 50 ms in release mode (measured in a `#[ignore]`d benchmark test run by `cargo xtask bench-index`, threshold asserted) and loads from disk in < 300 ms.
  - CLI snapshots for `index status --json` and `list --query … --json`; MCP parity test.
  - Corpus: `objectizer index build --json` exits 0 on a real install; object count and per-source-group counts recorded in the gate record; `cargo xtask guard-assets` confirms nothing is written inside the repository.
- **Depends on:** M2-T05 (M3-T01 for multi-tile grouping; before M3-T01 lands, tiles are grouped by MasterID only).
- **Parallel with:** M6-T01, M6-T02.

### M6-T09 — Sprite previews and thumbnail cache (`preview`) — core, CLI, MCP

- **Goal:** Any object, any zoom, any rotation, as a PNG — fast enough to scroll a catalog.
- **Inputs:** `ops::export --composite` code path (M2-T08), whole-object composition for multi-tile (M3-T03), `chunks::bmp` (M2-T04).
- **Outputs:**
  - `objectizer_core::preview`: `render_preview(source, zoom, rot, opts) -> RGBA` compositing all draw-group sprites of all tiles with z-aware layering, for the object's default graphic state (or `--state <n>`); `catalog_thumbnail(source)` returning the game's own thumbnail when present, else a generated far-zoom preview.
  - Thumbnail cache in the platform cache dir (`<cache>/objectizer/thumbs/<content-hash>_<zoom>_<rot>.png`), size-capped with least-recently-used eviction, invalidated by content hash; never inside the repository; `index clear --thumbs` empties it.
  - CLI: `objectizer preview <source> [--zoom near|med|far] [--rot rf|lf|lb|rb] [--state n] [--scale 1|2|4] [--sheet] --out <png>` (`--sheet` = all 12 views in one contact sheet). MCP: `preview` (returns the file path, and image content when the client supports it).
- **Acceptance:**
  - Tests pass; `preview::matches_export_composite`: for single-tile synthetic objects the preview equals the `export --composite` image byte-for-byte; `preview::multi_tile_assembly`: a 2×1 synthetic object's preview equals an independently computed reference.
  - `preview::cache_hit`: second request does no sprite decoding (decode counter); changing the file invalidates the entry; cache honours its size cap.
  - `preview --dry-run` writes nothing; CLI snapshot; MCP parity.
  - Corpus: preview at far zoom generated for every indexed object without error (count recorded in the gate record); mean time per uncached far-zoom preview reported.
- **Depends on:** M6-T08, M2-T08, M3-T03.
- **Parallel with:** M6-T01, M6-T02.

### M6-T10 — Catalog view: browse and search every object

- **Goal:** The GUI's main screen — the whole game's object catalog, searchable and actionable.
- **Inputs:** `ops::index_*`, `ops::list`, `ops::preview`, `ops::inspect`; `docs/gui-decision.md`.
- **Outputs:**
  - Virtualized grid and list modes showing thumbnail, name, price, source badge (pack / Downloads), custom-vs-Maxis badge, multi-tile badge; smooth scrolling over the full index with thumbnails loaded lazily in the background.
  - Search box (as-you-type, debounced, same query engine as `list --query`) and a filter panel: name, GUID, catalog category and sub-sort, price range, expansion/source file, custom vs Maxis, single vs multi-tile, catalog-visible vs hidden; sort control; result count; saved filter state per install.
  - Detail pane for the selected object: sprite preview with zoom and rotation controls (and graphic state when several), catalog text, price, GUID(s), source path / FAR entry, tile footprint.
  - Actions from the view (single or multi-selection where the op supports batches): **Inspect**, **Clone**, **Export**, **Recolor** — each opens the corresponding flow (M6-T03, M6-T04, M6-T05) pre-filled with the selection; "Copy as CLI command" for the current search and for each action; "Reveal source file".
  - Index lifecycle UI: first-run build with progress, incremental refresh on start and on demand, clear cache.
- **Acceptance:**
  - Tests pass on all three CI OSes (view-model tests).
  - `gui::catalog_query_parity`: for a table of search/filter states, the view-model's result ids and order equal `objectizer list --json` with the equivalent arguments on a synthetic install; the "Copy as CLI command" string, when executed, returns the same ids.
  - `gui::catalog_virtualization`: with a 20,000-record synthetic index, the view-model requests previews only for the visible range plus a bounded look-ahead (request counter), and a filter change produces a new result set in < 100 ms (release-mode benchmark test).
  - `gui::catalog_actions`: each action invokes the matching `ops` request with the selected source(s) (mock ops layer).
  - No format logic in the GUI crate (the test from M6-T02 still passes); thumbnails and index live only in the platform cache dir (path assertion).
- **Depends on:** M6-T02, M6-T08, M6-T09.
- **Parallel with:** —

### M6-T03 — Sprite viewer

- **Goal:** Inspect an object visually.
- **Inputs:** `ops::inspect` and `ops::export` responses; PLAN.md §6.
- **Outputs:** viewer with zoom level and rotation selectors, channel toggle (colour / z / alpha / composite), nearest-neighbour magnification, per-tile and whole-object composite for multi-tile (when M3 is present), palette panel, frame metadata.
- **Acceptance:**
  - View-model tests: selecting zoom/rotation yields the frame set the DGRP specifies (compared with `ops::inspect` output on synthetic fixtures); composite image equals `export --composite` output byte-for-byte.
- **Depends on:** M6-T10.
- **Parallel with:** M6-T04, M6-T05.

### M6-T04 — Clone, recolor and catalog editor

- **Goal:** The core edit flows without a terminal.
- **Inputs:** `ops::clone`, `ops::recolor`, `ops::set_catalog` and their `--dry-run` responses.
- **Outputs:** clone dialog (name, output location, GUID preview); recolor panel (hue/saturation/lightness sliders, live preview, palette swatch import/export); catalog editor (name, description, price, thumbnail picker with preview); every action has a "what will change" preview driven by `--dry-run` responses, and writes only on confirm.
- **Acceptance:**
  - View-model tests: each action produces the same output file bytes as the equivalent CLI command on synthetic fixtures (`gui::parity_clone`, `gui::parity_recolor`, `gui::parity_set_catalog`).
  - Preview never writes (temp-dir hash unchanged test).
- **Depends on:** M6-T10.
- **Parallel with:** M6-T03, M6-T05.

### M6-T05 — Export/import round-trip flow

- **Goal:** Work with an external image editor.
- **Inputs:** `ops::export`, `ops::import`, `ops::validate`; PLAN.md §6 import rules.
- **Outputs:** "Export for editing" (bundle to a chosen folder, button to reveal it), folder watcher that shows which frames changed since export, import with `--palette keep|new` and auto-zoom options, validation results list, one-click "install to Downloads".
- **Acceptance:**
  - View-model tests: changed-file detection matches manifest hashes; import parity with the CLI (`gui::parity_import`).
- **Depends on:** M6-T10.
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
- **Outputs:** release workflow producing a macOS universal `.app` in a `.dmg`, a Windows `.zip`/installer, a Linux AppImage or tarball; unsigned by default with documented signing steps; `docs/checklists/M6-gui.md` covering the catalog view (search, filters, previews, actions) and the five flows on each OS; README updated with download/run instructions.
- **Acceptance:**
  - Release workflow run produces the three artefacts (CI job exits 0, artefact names asserted).
  - `cargo xtask guard-assets` passes on the packaged tree (no game assets bundled).
  - Human checklist passed on macOS and at least one other OS; gate record filled in.
- **Depends on:** M6-T01 … M6-T06, M6-T08, M6-T09, M6-T10.

## Gate record

_To be filled by M6-T07._
