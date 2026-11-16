# M3 — Multi-tile objects and full Transmogrifier parity

Outcome: everything M2 does also works for multi-tile objects; SPR# sprites are supported; smaller zooms and mirrored rotations can be generated; a TMog export folder can be imported; a parity matrix shows nothing TMog does is missing.

Read first: [PLAN.md](../PLAN.md), [docs/formats.md](../docs/formats.md) §3.1, 5, 7, 13.

Depends on: M2 complete (M2-T15 passed).

## Task graph

```
T01 ─┬─ T02 ─┐
     └─ T03 ─┼─ T07 ── T08
T04 ─────────┤
T05 ─────────┤
T06 ─────────┘
```

T01, T04, T05, T06 can start in parallel. T02 and T03 need T01. M4 may start as soon as T04 is done.

---

### M3-T01 — Multi-tile object model

- **Goal:** Group master and tile OBJDs into one logical object.
- **Inputs:** formats.md §3.1; survey data.
- **Outputs:** `object::ObjectDef` gains `tiles: Vec<Tile { objd_id, guid, sub_index, x, y }>` and `master`; grouping by MasterID within a file; SubIndex decoding verified on the corpus (survey addition: histogram of SubIndex values vs tile counts) and documented in formats.md §3.1; `list`/`inspect` show one entry per logical object with `tile_count`; index stores master and tiles. Files containing several multi-tile objects are handled.
- **Acceptance:**
  - Tests pass; `testkit::object` 2×1 and 2×2 fixtures group correctly; orphan tiles (no master) → `W_ORPHAN_TILE`.
  - Corpus: every OBJD with MasterID ≠ 0 belongs to a group with exactly one master, or is counted and listed in the gate record.
  - `grep -n "TODO-M3-T01" docs/formats.md` returns nothing.
- **Depends on:** M2.
- **Parallel with:** M3-T04, M3-T05, M3-T06.

### M3-T02 — `clone` for multi-tile objects

- **Goal:** Clone with all GUIDs replaced consistently.
- **Inputs:** PLAN.md §8.
- **Outputs:** `ops::clone` accepts multi-tile sources: new GUID for the master and every tile; MasterID/SubIndex unchanged; any intra-file GUID references found by the survey (e.g. fields holding another OBJD's GUID such as lead-object or type-attribute GUIDs) remapped when they pointed at a re-GUIDed OBJD — the list of such fields is determined on the corpus and documented; name set on the master's CTSS.
- **Acceptance:**
  - Tests pass; `clone::multi_tile_guid_map`: N+1 distinct new GUIDs, byte diff confined to GUID fields + CTSS + rsmp.
  - Corpus: clone every multi-tile catalog object; each passes `validate`; no GUID collisions.
- **Depends on:** M3-T01.
- **Parallel with:** M3-T03.

### M3-T03 — Multi-tile bundles

- **Goal:** `export`/`import` for multi-tile objects.
- **Inputs:** PLAN.md §6.
- **Outputs:** manifest `objects[]` lists master and tiles; `draw_groups[]` entries carry the owning tile; sprite directories unchanged (sprite chunks are per file, shared across tiles). `export --tiles all|<x,y>`; `--composite` additionally writes whole-object previews per zoom/rotation assembled from tiles (preview only). Manifest `draw_groups` edits (sprite x/y offsets, flags, sprite/frame reassignment) are applied on import.
- **Acceptance:**
  - Tests pass; L4 and L5 on synthetic multi-tile fixtures.
  - `import::dgrp_offset_edit`: changing one `x` in the manifest changes exactly those DGRP bytes.
  - Corpus: L4 and L5 = 100% over all multi-tile objects.
- **Depends on:** M3-T01.
- **Parallel with:** M3-T02.

### M3-T04 — SPR# decode/encode

- **Goal:** Support the older sprite chunk (needed for walls and for any objects that use it).
- **Inputs:** formats.md §5. Survey addition first: SPR# versions, endianness, opcode statistics, which file kinds use it.
- **Outputs:** `chunks::spr`: lossless container with raw per-frame bytes, `decode()` to colour-index plane, `encode()` following observed Maxis rules, both endiannesses if the corpus has them; harness levels L1/L2/L3 extended to SPR#; bundles treat SPR# frames as colour-only (`chunk: "SPR#"` in the manifest); formats.md §5 updated.
- **Acceptance:**
  - Tests pass; hand-assembled streams for each opcode; proptest encode/decode.
  - Corpus: SPR# L1 100%, L3 100%, L2 ≥ 99% with no `unexplained` class.
  - `grep -n "TODO-M3-T04" docs/formats.md` returns nothing.
- **Depends on:** M2 (harness, bundle). Technically only M1; may be started early if staffing allows.
- **Parallel with:** M3-T01, M3-T05, M3-T06.

### M3-T05 — Auto-zoom and mirrored rotations

- **Goal:** Let the user draw less: derive `med`/`far` from `near`, and use flipped frames for two of four rotations, as TMog does.
- **Inputs:** PLAN.md §6 (import rule 5), formats.md §7 (flip flag), §4.5.
- **Outputs:** `image::downscale` (÷2, ÷4; colour via area-average then re-quantize to the frame palette, transparent-aware; z via min/nearest policy chosen and documented; alpha via average); `import --auto-zoom` regenerates med/far frames (pixels and x/y offsets) from changed near frames; `export --rotations drawn` exports only frames referenced without the flip flag; `inspect` reports which DGRP images are mirrors.
- **Acceptance:**
  - Tests pass; `downscale::dimensions_and_offsets` on synthetic frames (sizes and offsets halve with defined rounding); deterministic output hash.
  - `import::auto_zoom_only_touches_derived`: near frames unchanged → nothing changes; one near frame changed → exactly its med/far counterparts change.
  - Corpus sanity metric (not a gate): mean absolute RGB difference between auto-generated and Maxis med/far frames is reported in the gate record.
- **Depends on:** M2.
- **Parallel with:** M3-T01, M3-T04, M3-T06.

### M3-T06 — TMog export-folder import (`--format tmog`)

- **Goal:** Let users bring existing TMog projects (XML + BMPs) into objectizer.
- **Inputs:** formats.md §13. **Blocked until the user supplies at least two TMog export folders made from different objects** (one single-tile, one multi-tile); these stay outside the repo. Do not consult TMog binaries or decompile anything.
- **Outputs:** written description of the observed XML schema and file naming in formats.md §13; `import --format tmog <folder> --into <iff>`: reads the XML, `_p`/`_z`/`_a` BMPs, maps them to frames, then follows the normal import path; synthetic TMog-style fixtures generated by testkit from the documented schema.
- **Acceptance:**
  - Tests pass; synthetic TMog folder → import → frames equal the source pixels.
  - On each user-supplied sample: `objectizer import --format tmog … --dry-run --json` exits 0 and maps 100% of the BMP files to frames (recorded in the gate record).
  - `grep -n "TODO-M3-T06" docs/formats.md` returns nothing.
- **Depends on:** M2; user-supplied samples.
- **Parallel with:** M3-T01, M3-T04, M3-T05.

### M3-T07 — Batch operations and ergonomics

- **Goal:** Do many objects in one call; make agent workflows shorter.
- **Inputs:** PLAN.md §4.
- **Outputs:** `clone`, `export`, `recolor`, `validate` accept multiple sources or `--from-list <json>`; per-item results in `data.items[]` with individual `ok`/`error`; overall exit 0 only if all succeeded (else the code of the first failure; `--keep-going` continues after failures). MCP tools accept the same arrays. New convenience: `clone --recolor-hue <deg>` (clone + recolor in one step) and `export --open` is explicitly **not** added (no GUI/OS integration in the CLI).
- **Acceptance:**
  - Tests pass; CLI snapshot for a batch with one failing item (exit code and `items[]` shape); MCP parity test.
- **Depends on:** M3-T02, M3-T03.
- **Parallel with:** —

### M3-T08 — M3 gate: parity matrix and in-game **[human gate]**

- **Goal:** Show TMog parity and prove multi-tile objects in game.
- **Inputs:** formats.md §13 (TMog feature list); docs/testing.md §6; a real install of each game kind available.
- **Outputs:**
  - `docs/tmog-parity.md`: table of every TMog user-visible feature (from formats.md §13 and the tutorials) → objectizer command/flag → test name, or "intentionally not supported" with reason.
  - `docs/checklists/M3-ingame.md`: a recoloured 2-tile object (e.g. a sofa), a repainted 4+-tile object (e.g. a bed or pool table), an object using auto-zoom, each on both game kinds where available.
  - Gate record filled in.
- **Acceptance:**
  - `cargo xtask corpus` exits 0 (all hard gates incl. SPR# and multi-tile L4/L5).
  - Every row of `docs/tmog-parity.md` has a test name that exists (`cargo test -- --list | grep` check scripted in `xtask parity-check`, exits 0).
  - `grep -n "TODO-M3" docs/formats.md` returns nothing.
  - Human in-game pass recorded.
- **Depends on:** M3-T01 … M3-T07.

## Gate record

_To be filled by M3-T08._
