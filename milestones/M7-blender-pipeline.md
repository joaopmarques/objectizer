# M7 — Blender rendering pipeline: author new objects

Outcome: an agent can model a NEW object in Blender, run `objectizer render` → `objectizer compile` → `objectizer import`, and get a playable `.iff` whose angles, scale, depth sorting, palette and overall look match Maxis objects. The look is fixed by a calibrated, versioned preset (`maxis-look`).

Numbering note: M7 is numbered after M6 only to keep existing task IDs stable. In execution order it comes **after M2 and alongside M3** (it needs M3-T01 and M3-T03 for multi-tile and draw-group edits); it does not depend on M4, M5 or M6.

Read first: [PLAN.md](../PLAN.md) §4, §6, §6a, [docs/formats.md](../docs/formats.md) §15, [docs/testing.md](../docs/testing.md) §11, [AGENTS.md](../AGENTS.md) §3a.

Depends on: M2 complete; M3-T01 and M3-T03 (multi-tile model, manifest draw-group edits) for M7-T06 onward.

## Decisions fixed by this milestone

1. **Port, don't depend.** objectizer owns its Blender driver script and scene definition, adapting the camera/resolution math of `mixiate/ts1-renderer` and the splitter/depth logic of `mixiate/ts1-compiler` (both MPL-2.0, attribution lines required). Reasons: the add-on is operator/GUI-oriented and tied to specific Blender versions (the community guide reports it broken on Blender 4.2); a headless agent pipeline needs one pinned, testable entry point; the calibrated look preset must be versioned with the code that applies it; and the compile step must share the Rust core's palette, encoder and bundle code. ts1-blender-io is used only as an optional, developer-side import tool for the Sim scale reference (M7-T08), never at runtime.
2. **The scene is built by script from a JSON scene spec, not stored as a hand-edited `.blend`.** The Rust core computes camera, resolution and rotation values (`render::scene`) and writes a scene spec; the Python driver applies it to the user's model file. This keeps the math unit-testable without Blender, diffable in review, and identical across Blender versions. `objectizer render --write-template <file.blend>` generates a ready-to-use template for humans from the same spec; that generated file is not committed.
3. **No GPL dependencies.** ts1-compiler quantizes with imagequant (GPL-3.0-or-commercial). objectizer must not link it; M7-T05 extends the project's own quantizer (M2-T10) with posterize + Floyd–Steinberg instead.
4. **Blender is an external tool**, discovered via `--blender <path>`, env `OBJECTIZER_BLENDER`, then `PATH` / default install locations. One Blender LTS version is pinned in `blender/BLENDER_VERSION` (chosen in M7-T01); other versions produce `W_BLENDER_VERSION`.
5. **Cycles output is not bit-reproducible across machines.** Tests on rendered pixels use tolerances; everything after the render (compile, quantize, encode) is deterministic and tested exactly.

## Task graph

```
T01 ─┬─ T02 ── T03 ─────────────┐
     ├─ T04 ─┬─ T05 ── T06 ─────┼─ T09 ── T10 ── T11 ── T12
     └─ T07 ─┴──────── T08 ─────┘
```

After T01: lanes {T02 → T03}, {T04 → T05 → T06}, {T07} are parallel. T08 needs T03 and T07. T09 needs T03, T06, T07, T08. T04–T06 need no Blender (they are tested on analytically generated render sets).

"Tests pass" as defined in M1. Tasks marked **needs Blender** have criteria in tier T-blender (docs/testing.md §11): run locally with the pinned Blender; skipped with a visible notice when Blender is absent. Tasks marked **needs install** read Maxis sprites from `OBJECTIZER_GAME_DIR` and write them only to the local cache dir.

---

### M7-T01 — Source verification and rendering conventions

- **Goal:** Turn the researched rendering facts into verified, cited conventions before any code depends on them.
- **Inputs:** formats.md §15 (all items currently tagged from a relayed reading of the sources); `mixiate/ts1-renderer` `addons/render_ts1/__init__.py`; `mixiate/ts1-compiler` `src/splitter.rs`, `src/quantizer.rs` and its object XML format; the FreeSO/Volcanic Blender template if obtainable; formats.md §4.5.
- **Outputs:**
  - formats.md §15 updated: every item tagged **[V-src]** with file + line reference, or corrected. Explicitly resolved: camera transform and parenting; ortho-scale and resolution formulas for N×M tiles; clip range; which Blender object rotation corresponds to which of our `rf`/`lf`/`lb`/`rb` directions and to FreeSO's direction bits; render output file formats and bit depths the add-on writes; the exact depth-window formula (3.2 BU window, `DISTANCE_TO_CENTER_FROM_CAMERA` 27.7609, per-tile centre); the 200% supersample + Lanczos3 linear-light downsample; alpha threshold; posterize bit depths (the "R5G6B5" comment vs 3/2/3 bits); transparent colour/index convention; how sprites are cropped and how draw-group offsets are derived per tile.
  - `blender/BLENDER_VERSION` decision recorded in PLAN.md §6a (one current LTS; state which Blender versions were checked).
  - A short "what we port, from where" table in formats.md §15 for the attribution lines.
- **Acceptance:**
  - `grep -n "TODO-M7-T01" docs/formats.md` returns nothing.
  - formats.md §15 contains subsections `Camera`, `Scale and resolution`, `Rotations`, `Render passes`, `Depth mapping`, `Downsampling and alpha`, `Palette and dithering`, `Sprite cropping and offsets` (grep check), each with at least one source URL.
  - Tests pass (docs-only task; guard and link check via `cargo xtask guard-assets`).
- **Depends on:** M2.
- **Parallel with:** M3 tasks.

### M7-T02 — Scene math in the core (`render::scene`)

- **Goal:** Compute everything the renderer needs, in Rust, without Blender.
- **Inputs:** formats.md §15 (Camera, Scale and resolution, Rotations).
- **Outputs:** `objectizer_core::render::scene`: `SceneSpec::for_object(tiles_x, tiles_y, opts)` → camera rotation/location/ortho scale/clip, rotation-origin angles for the four views, render resolution per zoom, supersample factor, pass settings, tile-centre positions; projection helpers `world_to_canvas(point, view, zoom)` and tile-diamond geometry; `SceneSpec` serializes to the JSON the driver consumes (`schema` versioned; JSON schema file `docs/scene-spec.schema.json` generated by a test).
- **Acceptance:**
  - Tests pass.
  - `scene::one_tile_constants`: ortho scale = 4.2353 (±1e-4) and resolution 136×384 for 1×1; +0.7071 ortho scale and +64 px per extra tile for 2×1, 2×2, 3×1 (table-driven).
  - `scene::tile_diamond`: the four corners of the unit tile at z = 0 project to a 128×64 px diamond at large zoom, 64×32 at medium, 32×16 at small, at the canvas position given in formats.md §15, for all four views.
  - `scene::height`: a point 1 BU above the tile centre projects the documented number of pixels above it (value from formats.md §15).
  - Scene-spec snapshot test for 1×1 and 2×1.
- **Depends on:** M7-T01.
- **Parallel with:** M7-T04, M7-T07.

### M7-T03 — Headless Blender driver and `render` command (needs Blender)

- **Goal:** `objectizer render` turns a model `.blend` into a render set.
- **Inputs:** PLAN.md §4 (`render`), §6a (render-set layout); scene spec from M7-T02.
- **Outputs:**
  - `blender/objectizer_render.py` (run as `blender -b <model.blend> -P objectizer_render.py -- <scene-spec.json> <out-dir>`): creates camera, rotation origin and world from the spec; applies the look preset (lights, world, colour management, sample counts); forces Cycles, denoise off, adaptive sampling off, transparent film; renders colour+alpha at the supersample factor and a separate 1-sample, 0-bounce, filter-width-1 depth pass (camera-space Z in BU) for 4 views × 3 zooms; writes the files and manifest of PLAN.md §6a. No add-ons required. The model file is opened read-only; nothing is saved back.
  - `objectizer_core::render::blender`: Blender discovery and version check, process runner with timeout and log capture, `ops::render`; CLI `objectizer render <model.blend> --out <render-dir> [--tiles XxY] [--preset <name|path>] [--blender <path>] [--zooms …] [--views …] [--samples N] [--collection <name>] [--write-template <file.blend>]`.
  - `objectizer-testkit::blender`: generates test model files by script at test time (unit cube, flat unit plane, 0.84 BU "table" box, 2×1 box) — no `.blend` files in git.
- **Acceptance:**
  - Tests pass (without Blender: argument, discovery and manifest tests; Blender missing → exit 3 `E_NOT_FOUND` with install hint; wrong version → `W_BLENDER_VERSION`).
  - T-blender `render::file_set`: rendering the unit cube yields exactly the 24 image files of §6a (4 views × 3 zooms × `color` with alpha, `depth`) plus the manifest, with the resolutions of the scene spec, and a manifest that validates against `docs/render-set.schema.json`.
  - T-blender `render::diamond_calibration`: rendering the flat unit plane, the alpha footprint's bounding box is 128×64 px (±1) at large zoom at the predicted canvas position (±1 px), for all four views; same at 64×32 and 32×16.
  - T-blender `render::depth_is_view_z`: depth at the plane's centre pixel equals the predicted camera distance (±0.01 BU); background pixels are flagged as empty.
  - T-blender `render::rotation_order`: an asymmetric marker model (box with one coloured corner post) shows the post in the predicted screen quadrant in each of the four views.
  - `render --dry-run` runs no Blender process and lists `data.writes`.
- **Depends on:** M7-T02.
- **Parallel with:** M7-T04, M7-T05, M7-T07.

### M7-T04 — Compile I: split, depth mapping, downsample

- **Goal:** Turn a render set into per-tile, per-zoom, per-view RGBA + z + alpha planes, deterministically.
- **Inputs:** formats.md §15 (Depth mapping, Downsampling and alpha, Sprite cropping and offsets). Port from ts1-compiler `src/splitter.rs` with attribution.
- **Outputs:** `objectizer_core::render::compile`: read render set (EXR/PNG per §6a); linear-light Lanczos3 downsample of colour/alpha from the supersampled size; alpha threshold → binary coverage (threshold from formats.md §15, overridable; optional soft alpha keeps 5-bit values for the alpha plane); depth → 8-bit z per tile using the 3.2 BU window around each tile centre (nearer = lower, empty = 255); multi-tile split into per-tile canvases; crop each sprite to its coverage bounding box and compute frame `x`/`y` and draw-group offsets. `objectizer-testkit::render`: analytic render-set generator (axis-aligned boxes rasterized in Rust with exact depth) so no Blender is needed.
- **Acceptance:**
  - Tests pass.
  - `compile::depth_window`: for analytic boxes at known distances, z bytes equal the formula's values exactly; off-tile pixels are 255; a point at the tile centre on the ground maps to the documented z value.
  - `compile::split_2x1`: a 2×1 analytic box yields two tile sprite sets whose recomposition (using the computed offsets) equals the downsampled whole image pixel-for-pixel.
  - `compile::downsample_linear`: a 2× supersampled black/white checker downsamples to the linear-light mid-grey (not the gamma-space value), within 1/255.
  - `compile::deterministic`: same render set → identical output hashes on all three OSes (CI).
- **Depends on:** M7-T01.
- **Parallel with:** M7-T02, M7-T03, M7-T07.

### M7-T05 — Compile II: palette and dithering for rendered sprites

- **Goal:** Quantize rendered colour to game palettes with the period look, without GPL code.
- **Inputs:** formats.md §15 (Palette and dithering); `image::quantize` from M2-T10.
- **Outputs:** `image::quantize` gains: optional posterize step (bit depths per formats.md §15), Floyd–Steinberg dithering (serpentine off, fixed scan order, deterministic), a "one palette per object" mode (all 12 views of all tiles share one 255-colour palette + transparent index) and a "one palette per sprite chunk" mode; transparent index and colour per formats.md §15; palette statistics output (colours used, dither-pixel fraction) for calibration.
- **Acceptance:**
  - Tests pass.
  - `quantize::fs_dither_reference`: on a fixed gradient, output indices hash to a recorded value on all three OSes.
  - `quantize::transparent_reserved`: the transparent index is never assigned to an opaque pixel and no palette entry other than it equals the key colour.
  - `quantize::shared_palette`: with the per-object mode, every frame references the same PALT and total colours ≤ 255 + 1.
  - `cargo tree -p objectizer-core` contains no `imagequant` (scripted check in `xtask guard-assets` or a dedicated test).
- **Depends on:** M7-T04, M2-T10.
- **Parallel with:** M7-T03, M7-T07.

### M7-T06 — `compile` command: render set → bundle → playable `.iff`

- **Goal:** Close the loop to a new object.
- **Inputs:** PLAN.md §4 (`compile`), §6 (bundle), §6a; M3-T03 (manifest draw-group edits).
- **Outputs:** `ops::compile`; CLI `objectizer compile <render-dir> --base <source> --out <bundle-dir> [--name …] [--palette per-object|per-sprite] [--alpha binary|soft] [--no-dither] [--thumbnail auto|none]`: takes a **base object** (a Maxis object with the wanted behaviour and the same tile footprint, e.g. a chair for a chair), writes a normal objectizer bundle whose manifest replaces the base's sprites and draw groups with the rendered ones (all four views drawn, no mirror flags; frame order near/med/far × views; sprite chunk and PALT ids allocated or reused per a documented rule), and a generated catalog thumbnail. The documented new-object recipe is then: `clone <base>` → `render` → `compile --base <clone>` → `import <bundle> --into <clone>` → `set-catalog` → `validate`. Footprint mismatch between render set and base → `E_VALIDATION`. Objects with several graphic states (multiple DGRPs per tile, e.g. on/off, open/closed): `compile --state <n>` maps one render set to one state; unrendered states keep the base's graphics and produce `W_STATE_NOT_REPLACED`.
- **Acceptance:**
  - Tests pass.
  - `e2e::new_object_synthetic`: analytic render set (M7-T04 generator) + `testkit::object` base → compile → import → `validate` exits 0; decoded sprites equal the compile-stage planes (L3); every DGRP image references an existing frame; no flip flags set.
  - `e2e::new_object_2x1_synthetic`: same for the 2×1 fixture.
  - Bundle produced by `compile` validates against `docs/bundle.schema.json`; `compile --dry-run` writes nothing.
  - T-blender `e2e::cube_to_iff`: unit cube → render → compile → import into a synthetic base → `validate` exits 0.
- **Depends on:** M7-T05, M3-T01, M3-T03.
- **Parallel with:** M7-T07, M7-T08.

### M7-T07 — Reference set and look metrics (needs install)

- **Goal:** Make "looks like Maxis" measurable.
- **Inputs:** `ops::export` (composites), install index; docs/testing.md §11.
- **Outputs:**
  - `ops::calibrate_refs`; CLI `objectizer calibrate refs [--count 16] [--select <list.json>] --out <cache-dir>`: exports composited RGBA + z of a fixed, documented selection of 10–20 Maxis reference objects (by GUID, spanning materials: painted wood, upholstery, metal, plastic, glass, ceramic; spanning packs: base game and at least two expansions if installed) at all zooms/views into the **local cache dir only**. The selection list (GUIDs and category labels — no pixels) is committed as `presets/reference-set-v1.json`.
  - `ops::calibrate_measure`; CLI `objectizer calibrate measure <image-or-bundle>... --out <metrics.json>`: per sprite and aggregated — luminance histogram (mean, p5/p50/p95), per-channel and hue/saturation histograms in linear and sRGB, contrast, shadow ratio (fraction of opaque pixels below a luminance threshold relative to the sprite's median), top-lit vs side-lit face luminance ratio where planar regions are detectable, edge sharpness (gradient-magnitude statistics at silhouette and interior), opaque coverage fraction of the canvas, palette statistics (colours used, dither-pixel fraction, mean run length), depth-plane statistics (z range, z gradient). Metric definitions and formulas written to `docs/look-metrics.md`.
  - Aggregate target statistics of the reference set (numbers per material category — means, percentiles, histograms with ≤ 32 bins; no per-pixel data) committed as `presets/reference-metrics-v1.json`. This is derived statistics, not artwork; the user approved committing it (decision of 2026-11-16). It must stay aggregate: no per-sprite records, no per-pixel data.
- **Acceptance:**
  - Tests pass; `metrics::known_images`: each metric returns the analytically known value on synthetic images (flat grey, two-tone, gradient, checker, dithered pattern).
  - `calibrate refs --dry-run` writes nothing; without an install → exit 3.
  - With an install: `objectizer calibrate refs --out <dir> --json` exits 0 and reports ≥ 10 references found; `objectizer calibrate measure <dir> --out m.json` validates against `docs/look-metrics.schema.json`.
  - `cargo xtask guard-assets` passes with `presets/*.json` present (allow-list entry limited to JSON under `presets/`, size-capped).
- **Depends on:** M7-T01 (M2 for export).
- **Parallel with:** M7-T02 … M7-T06.

### M7-T08 — Stand-in models, material library, Sim scale reference (needs Blender; Sim part needs install)

- **Goal:** Give the calibration loop something to render and give authors period-appropriate materials.
- **Inputs:** `presets/reference-set-v1.json`; formats.md §15 (Scale); ts1-blender-io (developer-side, optional).
- **Outputs:**
  - `blender/standins.py` + `presets/standins-v1.json`: procedural stand-in models for each reference object (boxes, cylinders, simple chair/table/sofa/counter/lamp shapes with dimensions in BU estimated from the reference sprites' silhouettes via `world_to_canvas` inverse), each tagged with a material from the library. Built at render time; no mesh files in git.
  - `blender/materials.py` + `presets/materials-v1.json`: a small library defined as Principled BSDF parameter sets and simple procedural textures — painted wood, varnished wood, laminate, matte plastic, glossy plastic, brushed metal, chrome, ceramic, glass, fabric/upholstery, leather, rubber. `objectizer render --list-materials`; the driver can append the library into a model file's data on request (`--inject-materials`).
  - Sim scale reference: documented developer procedure (`docs/authoring.md` §Scale) to import an adult Sim with ts1-blender-io from the user's install, measure standing height, seat height and eye level in BU, and record the numbers in `presets/scale-v1.json`; the driver can add a mannequin proxy (capsule/box figure of those dimensions) to a scene with `--with-scale-figure` for human/vision checks. The imported Sim mesh is never saved into the repo.
- **Acceptance:**
  - Tests pass; `standins::schema` and `materials::schema` validate the preset JSON files; every reference in `reference-set-v1.json` has a stand-in entry.
  - T-blender `standins::render_all`: every stand-in renders without error; silhouette bounding box at large zoom is within ±10% (width and height) of the dimensions recorded for its reference in `reference-metrics-v1.json`.
  - T-blender `materials::render_swatches`: each material renders on a test sphere+cube without missing-shader (magenta) pixels (scripted colour check).
  - `presets/scale-v1.json` exists with `adult_height_bu`, `seat_height_bu`, `source` fields; if the Sim import was not performed, the fields are marked `"estimated": true` and the gate (M7-T12) must replace them.
- **Depends on:** M7-T03, M7-T07.
- **Parallel with:** M7-T06.

### M7-T09 — Calibration loop and the frozen `maxis-look` preset (needs Blender, needs install)

- **Goal:** Tune lighting, world and material defaults until rendered stand-ins measure like Maxis sprites, then freeze the result.
- **Inputs:** M7-T07 metrics and references; M7-T08 stand-ins; formats.md §15 (lighting is NOT specified by any source; start from a sun + sky-texture world with ground plane for contact shadow/AO, turbidity 10 as used by the FreeSO template).
- **Outputs:**
  - Look preset format `presets/maxis-look-v<N>.json` (`schema`, version, Blender version, colour management/view transform, sun direction/energy/angle, fill light(s), world type/colour/strength/turbidity, ground plane on/off and shadow-catcher settings, AO, samples, light path bounces, default roughness/specular offsets per material class, quantizer options). Applied by the driver; referenced by name with `--preset maxis-look`.
  - `ops::calibrate_run`; CLI `objectizer calibrate run --refs <cache-dir> --preset <in> [--out <report-dir>]`: renders all stand-ins with the preset, compiles them, measures them, and writes a comparison report (`distance` per metric and material class, weighted total score, per-reference worst offenders, side-by-side contact sheets in the report dir). `objectizer calibrate compare <metrics-a.json> <metrics-b.json>` for ad-hoc checks. The **tuning loop itself is agent-driven**: edit preset parameters → `calibrate run` → read the JSON report → repeat; the procedure, parameter bounds and stopping rule are written in `docs/authoring.md` §Calibration. Each iteration's preset and score are logged to `<report-dir>/history.jsonl`.
  - Frozen `presets/maxis-look-v1.json` plus `presets/maxis-look-v1.report.json` (final scores, no images).
- **Acceptance:**
  - Tests pass; `calibrate::score_math` on synthetic metrics; `preset::schema` validates every file under `presets/`.
  - T-blender + install: `objectizer calibrate run --refs <dir> --preset maxis-look --json` exits 0 and reports, for the frozen preset: mean luminance within ±5% of reference per material class; luminance p5/p95 within ±8%; shadow ratio within ±0.05 absolute; mean saturation within ±10%; silhouette edge-sharpness within ±15%; palette colours-used and dither fraction within ±20%; total score ≤ the threshold recorded in `docs/look-metrics.md`. Thresholds may be revised once by M7-T09 with justification in the hand-over; after that they are fixed.
  - `history.jsonl` shows a non-increasing best score and at least 5 iterations.
  - CI check (no Blender, no install): `presets/maxis-look-v1.report.json` scores satisfy the thresholds and its `preset_sha256` equals the hash of `presets/maxis-look-v1.json` (a preset edited after freezing fails CI until re-calibrated).
- **Depends on:** M7-T03, M7-T06, M7-T07, M7-T08.
- **Parallel with:** nothing.

### M7-T10 — Review gate for authored objects (`review`)

- **Goal:** Every authored object gets an objective and a visual check against its nearest Maxis reference before it reaches the game.
- **Inputs:** M7-T07 metrics; reference cache.
- **Outputs:** `ops::review`; CLI `objectizer review <bundle-or-iff> --refs <cache-dir> --out <review-dir> [--class <material>]`: measures the object's sprites; finds the nearest reference(s) by metric distance (or by `--class`); writes `review.json` (per zoom/view: metric distances, pass/fail per threshold, overall verdict `pass | warn | fail`, plus structural checks: canvas bounds, tile-diamond footprint coverage, depth plane monotonic along the ground, no key-colour pixels, binary alpha consistency) and contact sheets `sheet_<zoom>.png` placing each authored sprite next to its reference at the same scale on the same background. A **vision check** step is defined for agents: the agent (or a vision model it calls) inspects each sheet and appends a `vision` block to `review.json` — `{ "model": …, "verdict": "pass|fail", "issues": [ { "sheet": …, "kind": "angle|scale|lighting|material|palette|silhouette|depth", "note": … } ] }` — using the rubric in `docs/authoring.md` §Review. `objectizer review --check <review.json>` exits 0 only if the metric verdict is not `fail` **and** a `vision` block with verdict `pass` is present.
- **Acceptance:**
  - Tests pass; `review::synthetic_pass_fail`: a synthetic object identical to a synthetic reference passes; one with doubled brightness, one scaled 150%, one with inverted depth each fail with the expected `kind`.
  - `review --check` exits 5 when the `vision` block is missing or failing (test), 0 when present and passing.
  - `review.json` validates against `docs/review.schema.json`; contact sheets are written only under `--out`; sheets containing Maxis reference pixels are never written inside the repository tree (path check + test).
  - With install + Blender: the review of each calibration stand-in rendered with `maxis-look` has metric verdict `pass` or `warn` (recorded in the gate record).
- **Depends on:** M7-T09.
- **Parallel with:** M7-T11.

### M7-T11 — MCP tools, authoring guide, example project

- **Goal:** Make the pipeline usable end to end by an agent with no prior context.
- **Inputs:** PLAN.md §5; all M7 ops.
- **Outputs:** MCP tools `render`, `compile`, `calibrate_refs`, `calibrate_measure`, `calibrate_run`, `review` (long-running tools report progress and honour cancellation); `docs/authoring.md`: modelling rules (1 BU = 1 tile edge, origin at tile centre on the floor, +Z up, max 4 BU tall, which way is "front", footprint for multi-tile, one collection per graphic state, no lights/cameras in the model file, materials from the library, polygon/texture guidance), the new-object recipe, calibration procedure, review rubric, troubleshooting; `examples/side-table/`: a model **built by script** (`examples/side-table/build.py`, no `.blend` committed) plus the exact command list that produces a playable object from it using a base object named by GUID.
- **Acceptance:**
  - Tests pass; MCP `tools/list` snapshot includes the six tools; parity tests against the CLI for `compile`, `calibrate_measure`, `review` on synthetic inputs.
  - `docs/authoring.md` contains sections `Modelling rules`, `Scale`, `Recipe`, `Calibration`, `Review`, `Troubleshooting` (grep check); every command line in it parses with `objectizer … --help`-level validation (doc-test style script `cargo xtask doc-commands`, exits 0).
  - T-blender `examples::side_table`: running the example's command list against a synthetic base produces an `.iff` that passes `validate`.
- **Depends on:** M7-T10.
- **Parallel with:** —

### M7-T12 — M7 gate: authored objects in game **[human gate]** (needs Blender, needs install)

- **Goal:** Prove that authored objects sit correctly in the real game, especially depth sorting.
- **Inputs:** docs/testing.md §6, §11; `docs/authoring.md`.
- **Outputs:** `docs/checklists/M7-ingame.md` with exact commands producing three authored objects with `maxis-look`: a chair (base: a Maxis dining chair), a counter or desk (base: a Maxis counter), and a 2-tile sofa or table. Checks, on each game kind available and in Simitone: appears in catalog with generated thumbnail; footprint matches the tile at all 3 zooms × 4 rotations; **a Sim seated in the chair is drawn correctly in front of the back-rest and behind the arm-rests/front edge as appropriate in every rotation; a Sim standing behind the counter is occluded up to counter height and a Sim in front is not; objects placed on the counter/table surface sit on top; neighbouring objects and walls sort correctly; no z-fighting shimmer when scrolling**; lighting/saturation does not look out of place next to Maxis objects in a furnished room (screenshot side by side, kept out of git); `presets/scale-v1.json` contains measured (not estimated) values. Gate record filled in, including the final calibration scores and any threshold revisions.
- **Acceptance:**
  - `cargo xtask corpus` exits 0; all T-blender tests pass locally with the pinned Blender (`cargo xtask blender-tests`, exit 0, summary pasted in the gate record).
  - `objectizer review --check` exits 0 for the three authored objects.
  - `grep -n "TODO-M7" docs/formats.md` returns nothing.
  - Human in-game pass recorded for every checklist line on at least one real game; an agent must not fill this in.
- **Depends on:** M7-T01 … M7-T11.

## Gate record

_To be filled by M7-T12._
