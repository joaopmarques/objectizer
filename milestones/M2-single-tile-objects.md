# M2 — Single-tile objects: clone, export, import, recolor, catalog, MCP

Outcome: an agent can run `clone` → `export` → edit PNGs → `import` → `set-catalog` → `validate` on a single-tile object and the result shows up correctly in both games. The MCP server exposes the same operations.

Read first: [PLAN.md](../PLAN.md) §4–8, [docs/formats.md](../docs/formats.md) §3, 7, 8, 9, [docs/testing.md](../docs/testing.md) §4, 6, 7.

Depends on: M1 complete (M1-T13 passed).

## Task graph

```
T01 ─┐
T02 ─┼─ T05 ─┬─ T06 ── T07 ─────────────┐
T03 ─┤       ├─ T08 ── T09 ── T10 ──────┤
T04 ─┘       ├─ T11                     ├─ T14 ── T15
             ├─ T12                     │
             └─ T13 ────────────────────┘
```

T01–T04 are independent of each other (four parallel lanes). After T05: T06→T07, T08→T09→T10, T11, T12, T13 are parallel lanes. T14 needs all commands; T15 is the gate.

"Tests pass" as defined in M1. Each chunk task extends `cargo xtask corpus` level L1 with its type.

---

### M2-T01 — OBJD read/write

- **Goal:** Typed, lossless OBJD for all TS1 versions.
- **Inputs:** docs/formats.md §3; `docs/survey-findings.md` (versions present).
- **Outputs:** `chunks::objd::Objd { version, words: Vec<u16>, trailing }` with named accessors/setters for the fields in formats.md §3 (`guid()`, `set_guid()`, `master_id()`, `sub_index()`, `price()`, `catalog_strings_id()`, `base_graphic_id()`, `num_graphics()`, `function_flags()`, `build_mode_type()`, …), `is_master()`, `is_multi_tile()`, `catalog_visible()` (rule of §3.2). A field accessor beyond the chunk's word count returns `None`; setting it returns `E_VALIDATION`.
- **Acceptance:**
  - Tests pass; synthetic round trips for word counts 78, 93, 94, 95, 106 and an odd payload length (trailing byte kept).
  - `objd::guid_offset`: GUID is read from payload bytes 28–31 little-endian.
  - Corpus: L1 OBJD = 100%; version 142 counted as `unsupported` if present.
  - docs/formats.md §3 tags updated.
- **Depends on:** M1.
- **Parallel with:** M2-T02, M2-T03, M2-T04.

### M2-T02 — STR# / CTSS

- **Goal:** Typed, lossless string tables with safe editing of single strings.
- **Inputs:** docs/formats.md §9.
- **Outputs:** `chunks::strs::StringTable` supporting formats 0, −1, −2, −3; raw byte strings with lossy-free accessors (`get(lang, index) -> &[u8]`, `get_text` decoding Windows-1252), `set(lang, index, text)` that re-encodes only that entry, preserves comments, entry order and `trailing`. Format −4 → `E_UNSUPPORTED`.
- **Acceptance:**
  - Tests pass; synthetic round trips for each format, with trailing `0xA3` filler and with non-ASCII bytes.
  - `strs::edit_is_local`: after `set`, every other entry's bytes and the trailing bytes are unchanged.
  - Text not representable in Windows-1252 → `E_VALIDATION`.
  - Corpus: L1 STR# = 100% and CTSS = 100%. docs/formats.md §9 updated (language grouping, trailing bytes).
- **Depends on:** M1.
- **Parallel with:** M2-T01, M2-T03, M2-T04.

### M2-T03 — DGRP

- **Goal:** Typed, lossless draw groups.
- **Inputs:** docs/formats.md §7.
- **Outputs:** `chunks::dgrp::DrawGroup { version, images: Vec<Image { direction, zoom, sprites: Vec<SpriteRef { sprite_id, frame, x, y, z, flags, .. }> }> }` for versions 20000, 20001, 20003, 20004; writes the version it read; enums `Direction {Rf, Lf, Lb, Rb}` and `Zoom {Near, Med, Far}` with raw-value preservation for unknown values.
- **Acceptance:**
  - Tests pass; synthetic round trip per version.
  - Corpus: L1 DGRP = 100%. docs/formats.md §7 updated with: versions found, zoom numbering, image-count histogram.
- **Depends on:** M1.
- **Parallel with:** M2-T01, M2-T02, M2-T04.

### M2-T04 — BMP_ and thumbnails

- **Goal:** Read/write catalog thumbnails.
- **Inputs:** docs/formats.md §3.3, §8.
- **Outputs:** `chunks::bmp`: `Bmp::info()` (width, height, bit depth, palette), `to_png()`, `from_png(png, like: &BmpInfo)` producing a BMP with the same dimensions/bit depth as the one it replaces (resizing with a box filter and quantizing when the original is indexed). Survey addition: histogram of BMP_ ids relative to OBJD CatalogStringsID, sizes and depths → resolves the thumbnail-id question; result written to docs/formats.md §3.3/§8 and encoded in `object::thumbnail_chunk()`.
- **Acceptance:**
  - Tests pass; `bmp::png_roundtrip` (BMP → PNG → BMP pixel-equal for 8-bit and 24-bit synthetic images).
  - Corpus: every BMP_ payload parses as a BMP or is listed; L1 BMP_ (verbatim) = 100%.
  - `grep -n "TODO-M2-T04" docs/formats.md` returns nothing.
- **Depends on:** M1.
- **Parallel with:** M2-T01, M2-T02, M2-T03.

### M2-T05 — Object model, reference graph, install index

- **Goal:** Turn an IFF into objects with their dependent chunks, and index all objects of an install.
- **Inputs:** docs/formats.md §3.1–3.3; PLAN.md §4 (`<source>` grammar), §10 R10.
- **Outputs:**
  - `object::ObjectFile` (an `IffFile` + decoded views): `objects() -> Vec<ObjectDef>`; for each: OBJD, name/description (CTSS), thumbnail chunk, DGRP list, the set of sprite chunks and frames reachable from them, PALTs used; `shared_palettes()`; unreferenced sprite/PALT chunks.
  - `install::Index`: all OBJDs across FARs, loose IFFs and Downloads → `{guid, name, source, objd_id, master_id, sub_index, price, category, visible}`; cached in the platform cache dir keyed by path + size + mtime; `--no-cache` flag. (M6-T08 later replaces this cache with the persistent, searchable index; keep the record type and the lookup API small and stable so that swap is internal.)
  - `ops::list` object mode with filters `--name`, `--guid`, `--category`, `--multi-tile`; `ops::inspect` object mode; `<source>` resolution for `guid:` and `name:`.
- **Acceptance:**
  - Tests pass; on `testkit::object` fixtures the graph lists exactly the expected chunk ids; a dangling DGRP→sprite reference produces `W_DANGLING_REF`, not an error.
  - CLI snapshots for `list --json` and `inspect --json` in object mode; ambiguous `name:` → exit 6 with candidates.
  - Index cache test: second run does not re-read unchanged files (assert via a read counter in the test harness).
  - Corpus: `objectizer list --json` over the real install exits 0 and reports ≥ 1 object per FAR that contains OBJDs; count of dangling references is reported in the gate record.
- **Depends on:** M2-T01, M2-T02, M2-T03, M2-T04.
- **Parallel with:** nothing (hub task).

### M2-T06 — GUID allocation and ledger

- **Goal:** Unique GUIDs, PLAN.md §8.
- **Inputs:** `install::Index`; docs/formats.md §13 (cookie).
- **Outputs:** `guid::allocate(index, ledger, opts)`, `guid::check(guid)`, ledger file read/write (JSON, atomic); `objectizer guid new|check|ledger`. `--cookie` is implemented only after its bit placement is confirmed from at least one user-supplied TMog-made object with a known cookie; until then `--cookie` returns `E_UNSUPPORTED` with an explanatory message, and the finding is recorded in docs/formats.md §13.
- **Acceptance:**
  - Tests pass; `guid::no_collisions`: with an injected RNG that first yields GUIDs present in the index/ledger, the allocator skips them; 0 and 0xFFFFFFFF are never issued.
  - Ledger survives concurrent writers (test: two processes allocate 100 each; 200 distinct entries).
  - `guid check` exits 6 for a GUID present in the index, 0 otherwise (CLI test).
- **Depends on:** M2-T05.
- **Parallel with:** M2-T08, M2-T11, M2-T12, M2-T13.

### M2-T07 — `clone` (single-tile)

- **Goal:** Copy an object's IFF with new GUID(s) and a new name.
- **Inputs:** PLAN.md §4 (`clone`), §8.
- **Outputs:** `ops::clone`: resolves `<source>` (loose IFF, FAR entry, `guid:`, `name:`), copies the whole IFF, assigns a new GUID to **every** OBJD in it (single-tile files usually have one; if the file contains several independent single-tile objects, all are re-GUIDed and listed), sets the catalog name (CTSS string 0, default language; other languages left alone unless `--all-langs`), writes to `--out` or `<Downloads>/objectizer/<name>.iff`, records GUIDs in the ledger. Multi-tile sources are rejected with `E_UNSUPPORTED` until M3-T02. Response: old→new GUID map, output path, warnings.
- **Acceptance:**
  - Tests pass; `clone::only_expected_bytes_change`: byte diff between source and clone is confined to the GUID fields (and OriginalGUID if the survey shows it must follow — decide and document), the edited CTSS chunk, and rsmp/header offset if sizes changed.
  - `clone --dry-run` writes nothing and no ledger entry; output-exists → exit 6 unless `--force`.
  - Cloning from `<far>::<entry>` works on a synthetic FAR.
  - Corpus: clone every single-tile catalog-visible object of the install into a temp dir; each result passes `objectizer validate` and L0 re-parse; none of the new GUIDs collide.
- **Depends on:** M2-T05, M2-T06, M1-T12.
- **Parallel with:** M2-T08…T13.

### M2-T08 — `export` (sprite bundle)

- **Goal:** Write the bundle of PLAN.md §6.
- **Inputs:** PLAN.md §6.
- **Outputs:** `image::png` (indexed, grayscale and RGBA PNG read/write via the `png` crate), `bundle::Manifest` (serde + JSON schema file `docs/bundle.schema.json` generated by a test), `ops::export` with `--channels`, `--zooms`, `--composite`. Frame naming from the DGRP graph as specified. Deterministic output (same input → identical files).
- **Acceptance:**
  - Tests pass; on `testkit::object`: expected file list snapshot; `color` PNGs are colour-type 3 with PLTE equal to the PALT and tRNS on the transparent index; `z`/`alpha` are 8-bit grayscale; manifest validates against `docs/bundle.schema.json`.
  - `export --dry-run` writes nothing.
  - Exported composite of a synthetic frame equals a reference RGBA computed independently in the test.
  - Corpus: export every single-tile object to a temp dir without error (bundles deleted afterwards).
- **Depends on:** M2-T05.
- **Parallel with:** M2-T06, M2-T11, M2-T12, M2-T13.

### M2-T09 — `import`: unchanged and indexed paths

- **Goal:** Apply a bundle back; lossless when nothing or only palette-consistent pixels changed.
- **Inputs:** PLAN.md §6 import rules 1, 2, 4, 5.
- **Outputs:** `ops::import` (`--into`, `--out`/`--in-place`, hidden test flag `--reencode-all`): hash comparison against the manifest, indexed-PNG path, manifest `x`/`y`/size changes, z/alpha validation (dimensions must match colour; alpha quantized to 5 bits), catalog block applied via the same code as `set-catalog` once M2-T12 lands. Bundle validation errors → `E_VALIDATION` with the offending file.
- **Acceptance:**
  - Tests pass.
  - `import::untouched_is_identity`: export → import → output IFF byte-identical (L4) on synthetic fixtures.
  - `import::reencode_all_is_pixel_identical`: with `--reencode-all`, result is L3-equal and re-export yields identical PNG files (L5).
  - `import::single_frame_edit`: change one pixel index in one PNG → only that frame's bytes (plus offsets/rsmp) differ in the output.
  - Wrong-size z PNG → exit 5.
  - Corpus: `cargo xtask corpus` gains L4 and L5 over all single-tile objects: both 100%.
- **Depends on:** M2-T08, M1-T09, M1-T12.
- **Parallel with:** M2-T11, M2-T12, M2-T13.

### M2-T10 — `import`: RGBA path, quantization, new palettes

- **Goal:** Accept images from normal editors.
- **Inputs:** PLAN.md §6 import rule 3; formats.md §4.5 (Volcanic behaviour as reference).
- **Outputs:** `image::quantize`: nearest-colour mapping to an existing palette (`--palette keep`), and palette generation to ≤ 255 colours + transparent index (`--palette new`, Wu or median-cut; deterministic), optional dithering off by default. Transparency from the PNG alpha channel: alpha 0 → transparent index; if the frame has an alpha plane and no `alpha` file was changed, partial PNG alpha updates the alpha plane. New PALT chunk gets a free id; frames of the same sprite chunk imported together share one new palette (SPR2 v1000 has one palette per chunk — enforce and explain in the error if a bundle edit would need two).
- **Acceptance:**
  - Tests pass.
  - `quantize::exact_when_possible`: an RGBA image whose colours are all in the palette imports to exactly the original indices.
  - `quantize::new_palette_bounds`: ≤ 256 entries, transparent index reserved, deterministic across runs and platforms (hash of output asserted).
  - `import::shared_palette_untouched`: with `--palette new`, other sprite chunks that used the old PALT still decode to identical RGBA.
  - Round trip: RGBA export (`--composite` images used as input) → import `--palette keep` → RGBA of every frame equals the original.
- **Depends on:** M2-T09.
- **Parallel with:** M2-T11, M2-T12, M2-T13.

### M2-T11 — `recolor` (palette only)

- **Goal:** Recolour without touching sprite data.
- **Inputs:** PLAN.md §4 (`recolor`).
- **Outputs:** `ops::recolor`: `--hue/--sat/--light` (HSL shift), `--map <json>` (explicit `[[r,g,b] → [r,g,b]]` or index → colour), `--palette-png <png>` (replace from a 16×16 swatch as written by `export`), `--palt <id>` to target one palette (default: all palettes used by the object's sprites), `--protect <indices>`; reports affected sprite chunks; transparent index is never changed.
- **Acceptance:**
  - Tests pass.
  - `recolor::only_palt_changes`: every non-PALT chunk is byte-identical after recolor.
  - `recolor::hue_360_is_identity`; `recolor::swatch_roundtrip` (export swatch → `--palette-png` same swatch → identical file).
  - `--dry-run` reports the changed entries count and writes nothing.
- **Depends on:** M2-T05.
- **Parallel with:** M2-T06…T10, T12, T13.

### M2-T12 — `set-catalog`

- **Goal:** Edit name, description, price and thumbnail.
- **Inputs:** PLAN.md §4; formats.md §3.3, §8, §9.
- **Outputs:** `ops::set_catalog`: name/description → CTSS strings 0/1 for `--lang` (default: language 1) or `--all-langs`; price → OBJD Price (for multi-tile later: master; document which OBJDs carry price per survey); thumbnail → the chunk identified by M2-T04, created if absent with a free/conventional id. Selecting which object in a multi-object file: `--guid`.
- **Acceptance:**
  - Tests pass.
  - `set_catalog::minimal_diff`: only the CTSS chunk, the 2 price bytes and/or the BMP_ chunk (plus rsmp/offsets) differ.
  - `inspect --json` after `set-catalog` shows the new values (CLI test).
  - Price > 65535 or negative → exit 2 or 5 with a clear message (test).
- **Depends on:** M2-T05 (uses T01, T02, T04).
- **Parallel with:** M2-T06…T11, T13.

### M2-T13 — `validate`

- **Goal:** Catch broken files before the game does.
- **Inputs:** formats.md §3.2, §3.3; PLAN.md §10 R4.
- **Outputs:** `ops::validate` for IFF objects and bundles. Checks (each with a stable code): container parses, rsmp consistent with chunks; every OBJD's DGRP range exists; every DGRP sprite reference resolves to an existing chunk and frame; every sprite's PALT exists and has ≤ 256 colours; SPR2 frames decode; flags/planes consistent; CTSS exists for CatalogStringsID with non-empty name; GUIDs non-zero, unique within the file, and (if an install is available) not colliding with other files; catalog-visibility rule; thumbnail present. Severity `error` / `warning`. Exit 5 if any error.
- **Acceptance:**
  - Tests pass; one test per check code using a `testkit::object` fixture broken in exactly that way.
  - Corpus: every Maxis object file validates with 0 errors, or the failing check is relaxed/documented (the game obviously accepts these files); summary recorded in the gate record.
  - `docs/validate-codes.md` lists every check code (test asserts the doc and the enum are in sync).
- **Depends on:** M2-T05.
- **Parallel with:** M2-T06…T12.

### M2-T14 — MCP server

- **Goal:** `objectizer-mcp` per PLAN.md §5.
- **Inputs:** PLAN.md §5; all `ops` request/response structs.
- **Outputs:** stdio MCP server using `rmcp`; tools `locate`, `list`, `inspect`, `extract`, `clone`, `export`, `import`, `recolor`, `set_catalog`, `validate`, `roundtrip`, `guid_new`, `guid_check`; input schemas generated from the request structs; `dry_run` on mutating tools; path-safety rule (writes only to request-provided paths or `Downloads/objectizer/`); `docs/mcp.md` with a config snippet for Claude Code / Claude Desktop and one worked example session.
- **Acceptance:**
  - Tests pass; `cargo test -p objectizer-mcp`: child-process test doing `initialize`, `tools/list` (names equal the list above, snapshot), and one `tools/call` per tool on synthetic fixtures; each result's payload equals the CLI `--json` `data` for the same request.
  - A mutating call with `dry_run: true` leaves the temp dir unchanged.
  - Error results carry `error.code` from PLAN.md §3.4.
- **Depends on:** M2-T07, M2-T09, M2-T10, M2-T11, M2-T12, M2-T13.
- **Parallel with:** M2-T15 preparation.

### M2-T15 — M2 gate: end-to-end and in-game **[human gate]**

- **Goal:** Prove the single-tile workflow.
- **Inputs:** a real install of each game kind available.
- **Outputs:**
  - Automated E2E test `e2e::single_tile` (synthetic) and `cargo xtask corpus` E2E pass: for N=50 pseudo-randomly chosen single-tile objects (seeded), run clone → export → programmatic edit (invert colours of every `color` PNG via palette, shift one frame by a pixel) → import → set-catalog → validate; all exit 0.
  - `docs/checklists/M2-ingame.md` (shape in docs/testing.md §6) with exact commands producing three test objects: a palette recolor, an RGBA-repainted object with new palette, and one with new name/price/thumbnail.
  - Human runs the checklist on Legacy and on Complete Collection (whichever are available) and in Simitone; results recorded below. Also answers: is `<install>/Downloads` the folder Legacy reads (formats/PLAN §7 unverified item); does the object appear with the chosen thumbnail.
  - "Gate record" filled in; formats.md open items for M2 closed.
- **Acceptance:**
  - `cargo xtask corpus` exits 0 with L0–L5 hard gates at 100% and the E2E pass green.
  - `grep -n "TODO-M2" docs/formats.md` returns nothing.
  - Gate record contains at least one in-game pass on a real game (Legacy or Complete) — entered by the human; an agent must not fill this in.
- **Depends on:** M2-T01 … M2-T14.
- **Parallel with:** nothing.

## Gate record

_To be filled by M2-T15._
