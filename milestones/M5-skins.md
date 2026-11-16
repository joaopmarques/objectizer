# M5 — Skins and clothing

Outcome: list, inspect, validate, retexture and clone skins (heads, bodies, clothing categories) — the texture-level workflow first, mesh files handled losslessly.

Read first: [PLAN.md](../PLAN.md) §4, [docs/formats.md](../docs/formats.md) §12.

Depends on: M1 complete (FAR, IO, install discovery, CLI). Independent of M3 and M4.

Scope note: M5 does not edit meshes or animations. Meshes (SKN/BMF) and descriptors (CMX/BCF) are parsed so they can be validated, renamed and cloned consistently; geometry editing stays with Blender + ts1-blender-io.

## Task graph

```
T01 ─┬─ T02 ─┐
     ├─ T03 ─┼─ T06 ── T07
     ├─ T04 ─┤
     └─ T05 ─┘
```

T02–T05 are parallel after T01.

---

### M5-T01 — Format documentation spike

- **Goal:** Write down the skin formats before anyone writes a parser.
- **Inputs:** `mixiate/ts1-blender-io` source (MPL-2.0; may be ported later), FreeSO/Simitone vitaboy code (MPL-2.0), community guides cited in formats.md §12. Survey addition: file-type histogram under every `Skins` location and FAR, text vs binary counts, texture sizes/bit depths, naming-pattern conformance rate.
- **Outputs:** formats.md §12 rewritten with: CMX text grammar; BCF binary layout; SKN text grammar; BMF binary layout; how a CMX references skeleton, suits, skins, bones and textures; how the game picks a texture for a mesh (naming rule); every item tagged with its source; a list of what remains unverified. No code besides survey additions.
- **Acceptance:**
  - `grep -n "TODO-M5-T01" docs/formats.md` returns nothing; §12 contains subsections titled `CMX`, `BCF`, `SKN`, `BMF`, `Naming`, `Textures`, `Locations` (grep check).
  - Tests pass (survey additions unit-tested on synthetic files).
- **Depends on:** M1.

### M5-T02 — CMX and BCF

- **Goal:** Parse and write skin descriptors.
- **Inputs:** formats.md §12 (CMX, BCF).
- **Outputs:** `skin::cmx`: one model `Cmx { skeletons, suits, skills }` with text reader/writer and binary (BCF) reader/writer; lossless for both forms (text: preserve line endings and formatting of untouched lines; binary: raw-preserving); rename helper that updates internal self-references.
- **Acceptance:**
  - Tests pass; `testkit::skin` synthetic CMX and BCF round-trip byte-identically; text↔binary conversion preserves the model (`cmx::text_binary_equivalence`).
  - Corpus: L0/L1 = 100% over all `.cmx` and `.bcf` files (loose and in FARs), or failures classified with `unexplained` = 0.
- **Depends on:** M5-T01.
- **Parallel with:** M5-T03, M5-T04, M5-T05.

### M5-T03 — SKN and BMF

- **Goal:** Parse and write meshes losslessly.
- **Inputs:** formats.md §12 (SKN, BMF).
- **Outputs:** `skin::mesh`: `Mesh { name, texture_name, bones, faces, bindings, vertices, … }` with SKN text and BMF binary reader/writer; rename and retarget-texture helpers; summary info (vertex/face counts, bones used).
- **Acceptance:**
  - Tests pass; synthetic SKN and BMF round-trip byte-identically; `mesh::text_binary_equivalence`.
  - Corpus: L0/L1 = 100% over all `.skn` and `.bmf`, or failures classified with `unexplained` = 0.
- **Depends on:** M5-T01.
- **Parallel with:** M5-T02, M5-T04, M5-T05.

### M5-T04 — Skin naming and set resolution

- **Goal:** Understand which files form one skin and what each name means.
- **Inputs:** formats.md §12 (Naming, Locations).
- **Outputs:** `skin::name::SkinName` parser/formatter (prefix, number, gender, age, body type, skin tone, label) tolerant of real-world variants found by the survey; `skin::SkinSet` resolver: given any file, find its CMX, meshes and all textures (all skin tones); `skin::index(install)` over `GameData/Skins`, `ExpansionShared/SkinsBuy`, expansion FARs; free-number finder for new skins; collision detection.
- **Acceptance:**
  - Tests pass; table-driven `name::parse_format` (≥ 40 cases incl. malformed); `SkinSet` resolution on a synthetic directory.
  - Corpus: ≥ 99% of texture files in skin locations resolve to a set or are listed by class in the gate record.
- **Depends on:** M5-T01.
- **Parallel with:** M5-T02, M5-T03, M5-T05.

### M5-T05 — Skin textures

- **Goal:** Convert between editor images and game-ready textures.
- **Inputs:** formats.md §12 (Textures); `image::quantize` from M2-T10.
- **Outputs:** `skin::texture`: read 8-bit BMP (and other encodings the survey found), `to_png`, `from_png` (resize policy: must match expected size for the part unless `--resize`; quantize to 256 colours; write 8-bit indexed BMP with the header fields Maxis files use).
- **Acceptance:**
  - Tests pass; `texture::png_roundtrip` (indexed BMP → PNG → BMP byte-identical when untouched); RGBA input produces an 8-bit BMP with ≤ 256 colours; wrong size → exit 5 without `--resize`.
  - Corpus: every skin texture reads; untouched round trip byte-identical = 100% or classified.
- **Depends on:** M5-T01, M2-T10.
- **Parallel with:** M5-T02, M5-T03, M5-T04.

### M5-T06 — `skin` commands and MCP tools

- **Goal:** The user-facing skin workflow.
- **Inputs:** PLAN.md §4 (`skin` group).
- **Outputs:** `objectizer skin list [--category --gender --age]`, `skin inspect <name|file>`, `skin export <name> --out <dir>` (PNGs for all tones + `skin.json`), `skin retexture <name> --from <dir> [--out-dir]` (replace textures of an existing custom skin), `skin clone <name> --new-label <label> [--number auto|N] [--from <dir>]` (new set: copies CMX/mesh files with consistent renames and internal references, new textures), `skin validate <dir|name>` (naming, internal names match file names, textures indexed 8-bit and right size, CMX references resolve, no subfolders in `GameData/Skins`, no name collisions). Default output dirs: `GameData/Skins` or `ExpansionShared/SkinsBuy` according to category. MCP tools `skin_list`, `skin_inspect`, `skin_export`, `skin_retexture`, `skin_clone`, `skin_validate`.
- **Acceptance:**
  - Tests pass; CLI snapshots; `skin::clone_consistency`: cloned synthetic set passes `skin validate`, and a byte diff vs the source shows only names/references changed in CMX/mesh files.
  - `skin::export_retexture_identity`: export then retexture from the untouched export → byte-identical textures.
  - `--dry-run` writes nothing; MCP parity tests.
  - Corpus: `skin validate` over every Maxis skin set yields 0 errors or the rule is relaxed and documented.
- **Depends on:** M5-T02, M5-T03, M5-T04, M5-T05.

### M5-T07 — M5 gate **[human gate]**

- **Goal:** Prove skins in the real game.
- **Inputs:** docs/testing.md §6; a real install of each game kind available.
- **Outputs:** `docs/checklists/M5-ingame.md`: a retextured body, a retextured head, a cloned clothing item in a buyable category (if the expansion is present); checks in Create-a-Sim and in a wardrobe/dresser interaction, three skin tones, both game kinds where available. Gate record filled in.
- **Acceptance:**
  - `cargo xtask corpus` exits 0 including skin levels; `grep -n "TODO-M5" docs/formats.md` returns nothing; human in-game pass recorded.
- **Depends on:** M5-T01 … M5-T06.

## Gate record

_To be filled by M5-T07._
