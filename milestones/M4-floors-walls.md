# M4 — Floors and walls

Outcome: create, inspect and export custom floors (`.flr`) and walls (`.wll`) from PNG images, and extract built-in ones as editable sources.

Read first: [PLAN.md](../PLAN.md) §4, [docs/formats.md](../docs/formats.md) §10, 11, 5.

Depends on: M1 complete; M2-T02 (STR#); M2-T08/T10 (PNG IO, quantization); M3-T04 (SPR#, walls only); M3-T05 (downscale). Floors (T01, T02) can start before M3-T04 is done.

## Task graph

```
T01 ── T02 ─┐
T03 ── T04 ─┼─ T05 ── T06
```

Lanes {T01→T02} (floors) and {T03→T04} (walls) are parallel.

---

### M4-T01 — Floor model

- **Goal:** Read/write `.flr` and understand built-in floors.
- **Inputs:** formats.md §10. Survey addition: for every `.flr` and `floors.iff`: SPR2 ids, frame flags, pixel opcodes used, dimensions per zoom, PALT usage, label prefixes of the far chunk, STR# layout.
- **Outputs:** `floor::Floor { name, price, description, sound: Soft|Medium|Hard|Unknown(raw), near, med, far, palette }` parsed from a `.flr` IFF; lossless write; `floor::builtin(install)` enumerating floors in `floors.iff` with strings from `Build.iff`; `objectizer floor list`, `floor inspect`; `validate` support for `.flr` (ids 1/257/513 present, flags and opcodes within what the survey found, dimensions, STR# 0 with 3 strings, numeric price); formats.md §10 updated.
- **Acceptance:**
  - Tests pass; `testkit::floor` synthetic `.flr` parses and round-trips byte-identically.
  - Corpus: L0/L1 = 100% over all `.flr` files and `floors.iff`; `floor list --json` exits 0 on the install.
  - `grep -n "TODO-M4-T01" docs/formats.md` returns nothing.
- **Depends on:** M1, M2-T02.
- **Parallel with:** M4-T03.

### M4-T02 — Floor export and create

- **Goal:** PNG ↔ floor.
- **Inputs:** M4-T01 findings; PLAN.md §6 conventions for PNGs.
- **Outputs:** `objectizer floor export <flr|builtin:<id>> --out <dir>` (writes `near.png`, `med.png`, `far.png` as indexed PNGs + `floor.json` with name/price/description/sound); `objectizer floor create --image <png> | --from <dir> --name --price --description [--sound soft|medium|hard] --out <flr>`: accepts either a flat square texture (projected to the isometric diamond — projection defined and tested) or ready-made diamond tiles; generates the three zooms; quantizes to one palette; encodes with the floor-safe opcode subset; default output `<install>/GameData/Floors/objectizer/<name>.flr`. MCP tools `floor_list`, `floor_inspect`, `floor_export`, `floor_create`.
- **Acceptance:**
  - Tests pass; `floor::create_then_inspect` round trip on synthetic images; created file passes `validate`; encoder output contains only the opcodes allowed by the survey (asserted by scanning the command stream).
  - `floor::export_create_identity`: export a synthetic floor then `create --from` it → pixel-identical sprites (L3) and equal strings.
  - Corpus: export every floor in the install, re-create from the export, L3-equal to the original = 100%.
  - `--dry-run` writes nothing.
- **Depends on:** M4-T01, M2-T10, M3-T05.
- **Parallel with:** M4-T03, M4-T04.

### M4-T03 — Wall model

- **Goal:** Read/write `.wll` and understand built-in walls.
- **Inputs:** formats.md §11. Survey addition: for every `.wll` and `walls.iff`: SPR# ids, frame counts, per-frame dimensions per zoom, which frame is which wall orientation, thumbnail frame, PALT usage, STR# layout.
- **Outputs:** `wall::Wall { name, price, description, zooms: [frames…], palette }`; lossless write; `wall::builtin(install)`; `objectizer wall list`, `wall inspect`; `validate` support for `.wll`; formats.md §11 updated with a frame table.
- **Acceptance:**
  - Tests pass; `testkit::wall` synthetic `.wll` round-trips byte-identically.
  - Corpus: L0/L1 = 100% over all `.wll` files and `walls.iff`; `wall list --json` exits 0.
  - `grep -n "TODO-M4-T03" docs/formats.md` returns nothing.
- **Depends on:** M1, M2-T02, M3-T04.
- **Parallel with:** M4-T01, M4-T02.

### M4-T04 — Wall export and create

- **Goal:** PNG ↔ wall.
- **Inputs:** M4-T03 findings.
- **Outputs:** `objectizer wall export <wll|builtin:<id>> --out <dir>` (one indexed PNG per frame per zoom + `wall.json`); `objectizer wall create --image <png> | --from <dir> --name --price --description --out <wll>`: from a flat front-facing wall texture, generates all orientation frames (shear/skew per orientation as found in M4-T03, defined and tested) and the three zooms; default output `<install>/GameData/Walls/objectizer/<name>.wll`. MCP tools `wall_list`, `wall_inspect`, `wall_export`, `wall_create`.
- **Acceptance:**
  - Tests pass; `wall::export_create_identity` on synthetic walls (L3-equal); created file passes `validate`.
  - Corpus: export every wall, re-create from the export, L3-equal = 100%.
  - `--dry-run` writes nothing.
- **Depends on:** M4-T03, M2-T10, M3-T05.
- **Parallel with:** M4-T02.

### M4-T05 — Integration: locate, list, validate, MCP

- **Goal:** Floors and walls are first-class in the shared commands.
- **Inputs:** PLAN.md §4 and §5; outputs of M4-T02 and M4-T04.
- **Outputs:** `locate` reports `floors_dir` and `walls_dir`; `objectizer list --kind floor|wall|object`; `validate` dispatches on file kind; `roundtrip` includes `.flr`/`.wll` in L0–L3; MCP tool list updated (PLAN.md §5 `floor_*`, `wall_*`); `docs/mcp.md` updated.
- **Acceptance:**
  - Tests pass; MCP `tools/list` snapshot includes the eight new tools; CLI snapshots for `list --kind`.
  - `cargo xtask corpus` exits 0.
- **Depends on:** M4-T02, M4-T04.

### M4-T06 — M4 gate **[human gate]**

- **Goal:** Prove floors and walls in the real game.
- **Inputs:** docs/testing.md §6; a real install of each game kind available.
- **Outputs:** `docs/checklists/M4-ingame.md`: one floor created from a flat texture, one wall created from a flat texture, one exported-and-recoloured built-in of each; checks: appears in Build mode with name/price, tiles seamlessly at all three zooms and four rotations, walls show correctly cut-away/up, footstep sound as chosen. Gate record filled in.
- **Acceptance:**
  - `cargo xtask corpus` exits 0; `grep -n "TODO-M4" docs/formats.md` returns nothing; human in-game pass recorded.
- **Depends on:** M4-T01 … M4-T05.

## Gate record

_To be filled by M4-T06._
