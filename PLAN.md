# objectizer — master plan

Status: planning complete, no code written yet. This document is the source of truth for names (crates, commands, error codes, file names, task IDs). If another document disagrees with this one, this one wins and the other is a bug.

Related documents:

- [AGENTS.md](AGENTS.md) — conventions for implementing agents
- [docs/formats.md](docs/formats.md) — file-format reference with sources and verification status
- [docs/testing.md](docs/testing.md) — testing strategy and gates
- Milestones: [M1](milestones/M1-foundation.md), [M2](milestones/M2-single-tile-objects.md), [M3](milestones/M3-multi-tile-parity.md), [M4](milestones/M4-floors-walls.md), [M5](milestones/M5-skins.md), [M6](milestones/M6-gui.md), [M7](milestones/M7-blender-pipeline.md)

## 1. Goals

1. Replace the Transmogrifier (TMog) workflow for The Sims 1 with one open-source tool that runs natively on macOS, Linux and Windows: clone an object with new GUIDs, export its sprites, edit them, re-import, and get a working `.iff`.
2. Go beyond TMog: catalog thumbnail editing, palette-only recolor, validation, multi-tile objects, floors, walls, skins, a searchable index of every object in the game, authoring of new objects from Blender renders with a calibrated Maxis look, and later a GUI.
3. Be drivable by coding agents: a library, a CLI with stable JSON output and exit codes, and an MCP server that is a thin layer over the library.
4. Work from the user's own game files, for both The Sims Legacy Collection (2025) and The Sims Complete Collection.
5. Never corrupt what it doesn't edit: content the user did not change is written back byte-identical.

## 2. Non-goals

- No dependency on Windows, Wine, FreeSO or Simitone at runtime. (They are used only as test renderers.)
- No shipping, committing or downloading of Maxis/EA assets, ever. All fixtures are synthetic or generated locally from the user's install.
- No behaviour (BHAV) editing, no TTAB/SLOT editing, no save-game editing. These chunks are carried through untouched.
- No patching of `Sims.exe`, no DRM interaction.
- No The Sims Online / FAR3 / TSO chunk versions (SPR2 v1001, STR# format −4, OBJD v142). The parsers may reject them with `E_UNSUPPORTED`.
- No renderer of its own and no modelling: new objects are modelled by the user or an agent in Blender, and objectizer drives the user's Blender install headless (§6a). Blender is needed only for `render` and `calibrate run`; every other command works without it.
- No GPL-licensed dependencies (e.g. imagequant).
- No copying from Transmogrifier, IFF Pencil 2, SneakyLib or SimObliterator (see §9).

## 3. Architecture

### 3.1 Workspace

Rust, one Cargo workspace, edition 2021, MSRV pinned in `rust-toolchain.toml` (stable).

| Crate | Kind | Purpose |
|---|---|---|
| `objectizer-core` | lib | All format parsing/writing and all operations. No printing, no process exit, no global state. |
| `objectizer-cli` | bin `objectizer` | Argument parsing (clap), JSON/human rendering, exit codes. Calls `objectizer-core` only. |
| `objectizer-mcp` | bin `objectizer-mcp` | MCP server over stdio. One tool per CLI command, same request/response structs as the CLI's JSON. |
| `objectizer-testkit` | lib (dev) | Synthetic fixture builders, corpus walker, round-trip comparison helpers. |
| `xtask` | bin (dev) | `cargo xtask corpus`, `cargo xtask survey`, `cargo xtask guard-assets`. |
| `objectizer-gui` | bin (M6) | Desktop GUI over `objectizer-core`. |

Non-crate directories: `blender/` (Python scripts executed inside Blender, M7), `presets/` (JSON presets, M7), `examples/` (M7).

Rule: every user-visible operation is a function in `objectizer-core::ops` that takes a request struct and returns `Result<Response, Error>`, where request and response derive `serde::{Serialize, Deserialize}` and `schemars::JsonSchema`. The CLI and MCP server only translate arguments to requests and render responses. This is what keeps the three front ends consistent.

### 3.2 Modules of `objectizer-core`

| Module | Contents | First task |
|---|---|---|
| `error` | `Error`, `ErrorCode`, `Warning` | M1-T01 |
| `io` | Endian-aware `Reader`/`Writer` over byte slices, bounds-checked | M1-T02 |
| `iff` | `IffFile`, `Chunk`, `ChunkKey`, `rsmp` parse/rebuild | M1-T03, M1-T12 |
| `far` | `FarArchive` (FAR1 v1a read/write) | M1-T04 |
| `install` | Game install discovery, file enumeration, version detection | M1-T05 |
| `chunks::palt` | `Palette` | M1-T06 |
| `chunks::spr2` | `Spr2`, `Spr2Frame`, decoder, encoder | M1-T07, M1-T09 |
| `chunks::objd` | `Objd` (all TS1 versions) | M2-T01 |
| `chunks::strs` | `StringTable` for `STR#` and `CTSS` | M2-T02 |
| `chunks::dgrp` | `DrawGroup` | M2-T03 |
| `chunks::bmp` | `BMP_` wrapper, thumbnail conversion | M2-T04 |
| `chunks::spr` | `SPR#` decode/encode | M3-T04 |
| `object` | `ObjectFile`, `ObjectDef`, reference graph (OBJD → DGRP → SPR2 → PALT, CTSS, BMP_) | M2-T05, M3-T01 |
| `guid` | Allocation, collision index, ledger | M2-T06 |
| `image` | PNG channel IO, quantization, downscaling | M2-T08, M2-T10, M3-T05 |
| `bundle` | Bundle manifest, export, import | M2-T08, M2-T09 |
| `ops` | `locate`, `list`, `inspect`, `extract`, `clone`, `export`, `import`, `recolor`, `set_catalog`, `validate`, `roundtrip`, `guid`, `index_build`, `index_status`, `preview`, `render`, `compile`, `calibrate_refs`, `calibrate_measure`, `calibrate_run`, `review` | per task |
| `floor`, `wall` | `.flr` / `.wll` models | M4 |
| `skin` | CMX/BCF/SKN/BMF, naming, textures | M5 |
| `index` | Persistent object index, query engine | M6-T08 |
| `preview` | Object compositing at any zoom/rotation, thumbnail cache | M6-T09 |
| `render::scene` | Camera, resolution and rotation math; scene spec | M7-T02 |
| `render::blender` | Blender discovery and headless process runner | M7-T03 |
| `render::compile` | Render set → planes: downsample, alpha, depth mapping, tile split, cropping | M7-T04 |
| `calibrate` | Reference set, look metrics, scoring, review | M7-T07, M7-T09, M7-T10 |

### 3.3 Data model: lossless by default

This is the main mitigation for the byte-exact risk (§10, R1).

- `IffFile` stores the original 64 header bytes, the chunks **in file order**, and for each chunk its type, id, flags, 64-byte label field (raw bytes, not a trimmed string) and raw payload bytes.
- A chunk is parsed into a typed value on demand (`chunk.decode::<Objd>()`). Typed values keep any bytes the parser does not understand (trailing garbage, filler such as `0xA3` runs, unknown fields) in explicit `trailing: Vec<u8>` / `unknown_*` fields.
- A chunk is re-serialized **only if it was mutated**. An unmodified chunk is written from its raw bytes.
- `Spr2` keeps raw bytes **per frame**. Replacing one frame re-encodes only that frame; the others are copied.
- `rsmp` is preserved verbatim when the set, order and sizes of chunks are unchanged, and rebuilt (M1-T12) when they changed. A file that had no `rsmp` gets none.
- All writes go to a temp file in the destination directory followed by an atomic rename. Nothing is ever edited in place; commands that modify take `--out` or an explicit `--in-place`.

Consequence: `parse → write` with no edits reproduces the input byte-for-byte by construction, and the SPR2 encoder is exercised only on frames the user actually changed.

### 3.4 Error model

`objectizer-core` returns `Result<T, Error>`; it never panics on malformed input (fuzz-tested, see docs/testing.md). `Error` carries an `ErrorCode`, a message, and structured `details` (file path, chunk type/id, byte offset when known). Non-fatal findings are `Warning { code, message, details }` values returned inside the response.

| `ErrorCode` | Exit code | Meaning |
|---|---|---|
| — | 0 | Success |
| `E_INTERNAL` | 1 | Bug or unexpected state |
| `E_USAGE` | 2 | Bad arguments, mutually exclusive flags |
| `E_NOT_FOUND` | 3 | File, object, chunk or game install not found |
| `E_PARSE` | 4 | Malformed input file |
| `E_VALIDATION` | 5 | `validate` or `roundtrip` found failures; bundle fails checks on `import` |
| `E_CONFLICT` | 6 | GUID collision, output path exists without `--force` |
| `E_IO` | 7 | Filesystem error |
| `E_UNSUPPORTED` | 8 | Recognized but unsupported format/version |

## 4. CLI surface

Binary name: `objectizer`.

Global flags (all commands):

| Flag | Meaning |
|---|---|
| `--json` | Print exactly one JSON document (the envelope below) on stdout; logs go to stderr |
| `--dry-run` | Do everything except write files; the response lists what would be written (`data.writes[]`) |
| `--game-dir <path>` | Game install root. Falls back to env `OBJECTIZER_GAME_DIR`, then to `locate` discovery |
| `--quiet` / `-v` | Log level on stderr |
| `--force` | Allow overwriting existing outputs |

JSON envelope (stable, versioned by `schema`):

```json
{ "schema": 1, "ok": true, "command": "clone", "data": { }, "warnings": [ { "code": "W_...", "message": "..." } ] }
{ "schema": 1, "ok": false, "command": "clone", "error": { "code": "E_CONFLICT", "message": "...", "details": { } }, "warnings": [] }
```

Commands:

| Command | Milestone | Purpose |
|---|---|---|
| `objectizer locate` | M1 | Find game installs; report kind (`legacy` / `complete`), root, Downloads path, object sources |
| `objectizer list [<path>]` | M1 (chunks/files), M2 (objects) | List FAR entries, IFF chunks, or objects. With no path: all objects in the install + Downloads. Filters: `--type`, `--name`, `--guid`, `--category`, `--multi-tile`, `--kind object\|floor\|wall` (M4); `--no-cache`. From M6-T08 (index-backed): `--query <text>`, `--source`, `--origin maxis\|custom`, `--single-tile`, `--price-min`, `--price-max`, `--visible`/`--hidden`, `--lang`, `--sort`, `--limit`, `--offset` |
| `objectizer inspect <source>` | M1 (chunk level), M2 (object level) | Full structured dump of one file, chunk or object: OBJD fields, graph of DGRP/SPR2/PALT/CTSS/BMP_, frame table |
| `objectizer extract <far> [--entry <name>] --out <dir>` | M1 | Extract files from a FAR1 archive |
| `objectizer clone <source> --name <name> [--out <path>] [--cookie <u16>] [--guid <hex>] [--all-langs] [--recolor-hue <deg>]` | M2, M3 | Copy an object's IFF (from a loose file or FAR entry) with fresh GUIDs. Default output: `<Downloads>/objectizer/<name>.iff`. `--recolor-hue` (M3) clones and recolours in one step |
| `objectizer export <source> --out <bundle-dir> [--channels color,z,alpha] [--zooms near,med,far] [--composite] [--tiles all\|<x,y>] [--rotations all\|drawn]` | M2, M3 | Write a sprite bundle (§6). `--tiles` and `--rotations` arrive in M3 |
| `objectizer import <bundle-dir> --into <iff> [--out <iff> \| --in-place] [--palette keep\|new] [--auto-zoom] [--format bundle\|tmog]` | M2, M3 | Apply an edited bundle to an IFF. `--auto-zoom` and `--format tmog` arrive in M3. Hidden test-only flag: `--reencode-all` |
| `objectizer recolor <iff> (--map <json> \| --hue <deg> --sat <f> --light <f> \| --palette-png <png>) [--palt <id>] [--protect <indices>] [--out <iff>]` | M2 | Palette-only recolor: rewrite PALT entries, sprites untouched |
| `objectizer set-catalog <iff> [--guid <hex>] [--name] [--description] [--price] [--thumbnail <png>] [--lang <code> \| --all-langs] [--out <iff>]` | M2 | Edit catalog name, description, price, thumbnail |
| `objectizer validate <path>` | M2 | Structural and referential checks on an IFF, bundle, `.flr`, `.wll` or skin set |
| `objectizer roundtrip <path>... [--level L0,L1,L2,L3] [--report <json>]` | M1 | Round-trip harness over files, FARs or a whole install |
| `objectizer guid new [--cookie <u16>]` / `guid check <hex>` / `guid ledger` | M2 | GUID allocation and collision checks |
| `objectizer floor list\|inspect\|export\|create` | M4 | Floors (`.flr`) |
| `objectizer wall list\|inspect\|export\|create` | M4 | Walls (`.wll`) |
| `objectizer skin list\|inspect\|export\|retexture\|clone\|validate` | M5 | Skins and clothing |
| `objectizer index build\|status\|clear` | M6-T08 | Build, inspect or clear the local object index (§4a). `list` builds it on first use |
| `objectizer preview <source> [--zoom] [--rot] [--state n] [--scale 1\|2\|4] [--sheet] --out <png>` | M6-T09 | Composite an object at any zoom/rotation into a PNG |
| `objectizer render <model.blend> --out <render-dir> [--tiles XxY] [--preset <name\|path>] [--blender <path>] [--zooms] [--views] [--samples N] [--collection <name>] [--with-scale-figure] [--inject-materials] [--list-materials] [--write-template <file.blend>]` | M7 | Render a model headless in Blender into a render set (§6a) |
| `objectizer compile <render-dir> --base <source> --out <bundle-dir> [--name] [--state n] [--palette per-object\|per-sprite] [--alpha binary\|soft] [--no-dither] [--thumbnail auto\|none]` | M7 | Turn a render set into a sprite bundle for a cloned base object |
| `objectizer calibrate refs\|measure\|run\|compare` | M7 | Reference export, look metrics, calibration run and comparison (§6a) |
| `objectizer review <bundle-or-iff> --refs <cache-dir> --out <review-dir> [--class <material>]` / `review --check <review.json>` | M7 | Metric and vision review of an authored object against Maxis references |

Batch use (M3-T07): `clone`, `export`, `recolor`, `validate` accept several sources or `--from-list <json>`; results come back in `data.items[]`; `--keep-going` continues after a failed item.

Dev commands (not part of the user CLI): `cargo xtask corpus`, `cargo xtask survey`, `cargo xtask guard-assets`, `cargo xtask parity-check` (M3-T08), `cargo xtask bench-index` (M6-T08), `cargo xtask blender-tests` and `cargo xtask doc-commands` (M7).

`<source>` grammar (used by `inspect`, `clone`, `export`): a path to an `.iff`; or `<far-path>::<entry-name>`; or `guid:<hex>`; or `name:<catalog or file name>` resolved through the install index. Ambiguous `name:` matches fail with `E_CONFLICT` and list candidates in `details`.

### 4a. Object index and query model

The index (M6-T08) is a local cache of one record per logical object across every FAR, loose IFF and Downloads file. It lives in the platform cache dir, is rebuilt incrementally, and is never committed or shared.

Record fields: `guid`, `tile_guids[]`, `name` and `description` per language present, `internal_name` (OBJD label), `source { path, far_entry }`, `source_group` (`base`, `expansion:<n>`, `shared`, `downloads`, `other`), `origin` (`maxis` or `custom`, decided by location), `price`, `kind` (`object`, later `floor`, `wall`), `category` and sub-sorts, `visible`, `tile_count`, `footprint`, `has_thumbnail`, `content_hash`.

`objectizer list` (object mode), the MCP `list` tool and the GUI catalog view (M6-T10) all run the same query: free-text `--query` (case- and accent-insensitive, over names, description, file name and GUID hex) combined with the filters in the command table, plus `--sort`, `--limit`, `--offset`. The response always carries `total`. Whatever the GUI can find, an agent can find with the same arguments.

Previews (M6-T09): `objectizer preview` composites an object at any zoom and rotation into a PNG; a size-capped thumbnail cache sits beside the index.

## 5. MCP tool surface

Binary `objectizer-mcp`, stdio transport, built with the official Rust MCP SDK (`rmcp`). One tool per CLI command; tool input schema = the op's request struct (via `schemars`), tool result = the same JSON envelope's `data` / `error`.

| MCP tool | CLI equivalent | Mutates files |
|---|---|---|
| `locate` | `locate` | no |
| `list` | `list` | no |
| `inspect` | `inspect` | no |
| `extract` | `extract` | yes |
| `clone` | `clone` | yes |
| `export` | `export` | yes |
| `import` | `import` | yes |
| `recolor` | `recolor` | yes |
| `set_catalog` | `set-catalog` | yes |
| `validate` | `validate` | no |
| `roundtrip` | `roundtrip` | no |
| `guid_new`, `guid_check` | `guid new`, `guid check` | ledger only |
| `floor_*`, `wall_*`, `skin_*` | M4/M5 groups | varies |
| `index_build`, `index_status` | `index build`, `index status` | cache only |
| `preview` | `preview` | yes (one PNG) |
| `render` | `render` | yes; runs Blender |
| `compile` | `compile` | yes |
| `calibrate_refs`, `calibrate_measure`, `calibrate_run` | `calibrate refs`, `calibrate measure`, `calibrate run` | yes (cache / report dirs); `calibrate_run` runs Blender |
| `review` | `review` | yes (review dir) |

Every mutating tool accepts `dry_run: bool`. Tools never write outside paths given in the request, the install's `Downloads/objectizer/` directory, or objectizer's own platform cache dir (index, thumbnails, reference exports). Tools that start Blender (`render`, `calibrate_run`) are long-running: they report progress and honour cancellation. Read-only tools are annotated `readOnlyHint`.

## 6. Sprite bundle format

A bundle is a directory. It is the only interchange format between objectizer and image editors or agents.

```
<bundle>/
  objectizer.bundle.json            manifest
  sprites/
    s00100/                         one directory per SPR2 (or SPR#) chunk id, 5 digits
      f000_near_rf_color.png
      f000_near_rf_z.png
      f000_near_rf_alpha.png
      f001_near_lf_color.png
      ...
  palettes/
    p00100.png                      16x16 swatch of the PALT (informational)
  catalog/
    thumbnail.png                   catalog icon, if the object has one
  composite/                        only with --composite: RGBA previews, never read on import
    s00100_f000_near_rf.png
```

File name: `f<frame:03>_<zoom>_<rot>_<channel>.png`

- `frame` — frame index inside the sprite chunk. This is the identity; zoom and rotation are descriptive.
- `zoom` — `near`, `med`, `far` (near is the largest).
- `rot` — `rf`, `lf`, `lb`, `rb` (right-front, left-front, left-back, right-back: FreeSO's DGRP direction names). A frame used by more than one draw-group image is named after its first reference in (DGRP id, image index) order; a frame used by none is named `f<frame>_unref_xx_<channel>.png`.
- `channel`:
  - `color` — 8-bit **indexed** PNG. Its PLTE is the frame's PALT, pixel values are the palette indices, the frame's transparent index is marked fully transparent via tRNS. Exact indices survive a round trip.
  - `z` — 8-bit grayscale PNG, the raw z byte (0 nearest, 255 farthest/transparent). Present only if the frame has the z flag.
  - `alpha` — 8-bit grayscale PNG. Stored 5-bit value `a5` is written as `round(a5 * 255 / 31)`; import computes `round(a8 * 31 / 255)`. Present only if the frame has the alpha flag.

All channel images of a frame have the frame's width × height (the cropped sprite rectangle, not the full tile canvas).

Manifest `objectizer.bundle.json` (schema versioned):

```json
{
  "schema": 1,
  "tool": "objectizer 0.1.0",
  "source": { "file": "chair.iff", "sha256": "…" },
  "objects": [ { "guid": "0x12345678", "objd_id": 16807, "name": "Chair", "master_id": 0, "sub_index": 0 } ],
  "palettes": [ { "id": 256, "colors": 256, "swatch": "palettes/p00256.png" } ],
  "sprites": [
    {
      "chunk": "SPR2", "id": 256, "label": "chair",
      "frames": [
        {
          "index": 0, "zoom": "near", "rot": "rf",
          "width": 62, "height": 97, "x": -31, "y": -80,
          "flags": 7, "palette": 256, "transparent_index": 0,
          "files": { "color": "sprites/s00256/f000_near_rf_color.png", "z": "…_z.png", "alpha": "…_alpha.png" },
          "sha256": { "color": "…", "z": "…", "alpha": "…" },
          "used_by": [ { "dgrp": 100, "image": 0, "flip": false } ]
        }
      ]
    }
  ],
  "draw_groups": [ { "id": 100, "images": [ { "direction": "rf", "zoom": "near", "sprites": [ { "sprite": 256, "frame": 0, "x": 0, "y": 0, "flags": 0 } ] } ] } ],
  "catalog": { "name": "Chair", "description": "…", "price": 80, "thumbnail": "catalog/thumbnail.png" }
}
```

Import rules:

1. A frame whose channel files all still hash to the manifest's `sha256` is **not touched** (its raw bytes stay).
2. A changed `color` PNG that is indexed with a PLTE equal to the frame's PALT is imported by index, with no quantization (lossless path, M2-T09).
3. A changed `color` PNG that is RGB/RGBA or has a different PLTE goes through quantization (M2-T10): `--palette keep` maps to the nearest existing PALT colour; `--palette new` builds a new PALT (≤ 255 colours + transparent index) and writes it as a new chunk so other sprites sharing the old PALT are unaffected.
4. Changed dimensions are allowed; `x`/`y` in the manifest are honoured, so an agent can resize a sprite by editing the manifest.
5. Missing `z` for a frame that needs one: keep the old z if dimensions are unchanged, otherwise fail with `E_VALIDATION` (z cannot be invented). `--auto-zoom` (M3-T05) derives `med`/`far` frames for all channels from `near`.
6. `draw_groups` and `catalog` edits in the manifest are applied in M3 (`draw_groups`) and M2 (`catalog`).

## 6a. Authoring new objects: the Blender pipeline

Milestone [M7](milestones/M7-blender-pipeline.md). Conventions and their sources are in docs/formats.md §15.

```
model.blend ──render──▶ render set ──compile──▶ sprite bundle ──import──▶ cloned base .iff ──validate / review──▶ game
              (Blender,              (Rust: split, depth,       (§6)
               headless)              downsample, quantize)
```

- **`render`** runs the user's Blender headless (`blender -b model.blend -P blender/objectizer_render.py -- scene-spec.json out/`). The scene (orthographic camera at 30° elevation and 45° yaw, rotation origin, resolution, passes, lights, world) is created by the script from a scene spec computed in the Rust core, and a versioned look preset (`presets/maxis-look-v<N>.json`). The model file contains geometry and materials only.
- **`compile`** is pure Rust and deterministic: linear-light downsample from the supersampled render, alpha threshold, depth mapped into the per-tile 8-bit z window, multi-tile split, cropping and offsets, palette quantization with dithering. Output is a normal sprite bundle (§6).
- **New object recipe:** `clone <base>` (a Maxis object with the wanted behaviour and footprint) → `render` → `compile --base <clone>` → `import <bundle> --into <clone>` → `set-catalog` → `validate` → `review`.
- **`calibrate`** measures how close rendered stand-in models are to a reference set of Maxis sprites exported from the user's install into the local cache, so an agent can tune the look preset until the metrics converge; the result is frozen as `maxis-look-v1`.
- **`review`** compares an authored object with its nearest Maxis reference (metrics + contact sheets for a vision model) and gates on both.

Design decisions (justified in the M7 file): objectizer **ports** the camera math and compile logic from mixiate's ts1-renderer and ts1-compiler (MPL-2.0) instead of depending on the add-on at runtime; the scene is generated from a JSON spec, not a committed `.blend`; no GPL dependency (ts1-compiler's imagequant is replaced by the project's own quantizer); Blender is an external tool found via `--blender`, `OBJECTIZER_BLENDER`, or `PATH`, with one pinned LTS version in `blender/BLENDER_VERSION`.

Render set layout (`render` output, `compile` input):

```
<render-dir>/
  objectizer.render.json                 manifest: schema, tiles, preset name + hash, Blender version, scene spec, file list
  renders/
    <view>_<zoom>_color.exr              RGBA, linear, at the supersample factor (default 2×)
    <view>_<zoom>_depth.exr              single channel, camera-space Z in Blender units, at 1×
```

`<view>` is `rf|lf|lb|rb` and `<zoom>` is `near|med|far`, the same names as bundles: 4 × 3 × 2 = 24 image files. The mapping between Blender object rotations (0°, −90°, −180°, −270°) and view names is fixed by M7-T01. File formats may be revised by M7-T01 after reading the upstream sources; the names stay.

Repository directories added by M7: `blender/` (Python driver, stand-in and material builders — MPL-2.0, run only inside Blender), `presets/` (JSON only: look presets, material and stand-in definitions, the reference-set GUID list, aggregate reference metrics), `examples/` (script-built example models). No `.blend`, `.exr` or rendered image is committed.

## 7. Game versions and paths

Both games use the same IFF/FAR formats (no evidence of new chunk types or versions in Legacy; verified on the user's files by M1-T08). Differences that matter:

| | Complete Collection | Legacy Collection |
|---|---|---|
| Install root (Windows) | `C:\Program Files (x86)\Maxis\The Sims` | Steam: `…\steamapps\common\The Sims Legacy Collection`; EA App: `C:\Program Files\EA Games\The Sims Legacy` |
| Registry install key | yes (`HKLM\SOFTWARE\WOW6432Node\Maxis\The Sims`) | **none** — why old tools can't find it |
| Custom objects | `<install>\Downloads` | `<install>\Downloads` (per community guides; whether a Saved Games Downloads is also read is **unverified**, see M2-T15) |
| Saves | `<install>\UserData*` | `%USERPROFILE%\Saved Games\Electronic Arts\The Sims 25\UserData*` |
| Maxis objects | FARs under `GameData`, `ExpansionPack*`, `ExpansionShared` | same layout |
| Renderer | software / D3D, 16-bit | Vulkan, 24-bit; still consumes SPR2 colour + z + alpha |

Discovery order (`install` module, M1-T05), no registry access:

1. `--game-dir`, then `OBJECTIZER_GAME_DIR`.
2. Steam: parse `libraryfolders.vdf` in the platform's Steam root, look for app `3314060`.
3. Default paths for EA App and Complete Collection on Windows.
4. Wine/CrossOver/Proton prefixes on macOS/Linux: scan `~/Library/Application Support/CrossOver/Bottles/*/drive_c`, `~/.wine/drive_c`, Steam `compatdata/3314060/pfx/drive_c` for the same default paths.
5. Kind detection: `legacy` if the Steam/EA layout or app manifest matches, else `complete`; reported as `unknown` if neither can be decided.

Testing per version is described in [docs/testing.md](docs/testing.md) §6.

## 8. GUIDs

- A GUID is the 32-bit value at OBJD offset 28.
- `clone` gives every OBJD in the cloned file a new GUID (M3 for multi-tile; all tiles and the master).
- Allocation: random 32 bits from the OS RNG; with `--cookie <u16>` the community "magic cookie" occupies the middle 16 bits (bit placement is **unverified** — M2-T06 confirms against TMog-made objects before enabling the flag).
- A candidate is rejected if it collides with any GUID found in the install's FARs, in `Downloads`, or in the local ledger (`<config-dir>/objectizer/guids.json`, platform config dir via the `directories` crate).
- There is no known list of Maxis-reserved ranges; the install scan is the authority.

## 9. License

**Decision: MPL-2.0 for all crates.**

| Option | Port FreeSO / ts1-compiler / ts1-blender-io code | Downstream reuse | Verdict |
|---|---|---|---|
| MPL-2.0 | Yes, file by file, keeping notices | Any project may link/embed; only modified objectizer files must stay MPL | **Chosen** |
| MIT / Apache-2.0 | No — clean-room only; reading for understanding is allowed but translating is not | Maximum | Rejected: the three best references for the hardest parts (SPR2 encoder, Volcanic importer, BCF/BMF/SKN) are all MPL-2.0 |
| GPL-3.0 | Yes (MPL-2.0 → GPL is allowed one way) | Forces GPL on anything linking the library, including a future GUI plugin ecosystem or other tools | Rejected: restricts reuse and buys nothing here |

What this implies for implementing agents (repeated in AGENTS.md):

- **May port** (translate into Rust) from MPL-2.0 sources: FreeSO (`riperiperi/FreeSO`), Simitone, `mixiate/ts1-compiler`, `mixiate/ts1-renderer`, `mixiate/ts1-blender-io`. A file containing ported logic carries the MPL-2.0 header plus a line `// Portions derived from <project> <path> (MPL-2.0), <url>`.
- **May read, must not copy**: Niotso (check its license before porting; treat as read-only until an agent records the license in docs/formats.md), SneakyLib (no license = all rights reserved), SimObliterator (license unclear), Cyndersanity's guide, community tutorials.
- **Never**: Transmogrifier, IFF Pencil 2 or any other closed tool's code or resources; decompilation of any of them; any Maxis asset.
- File formats themselves are facts and are not copyrightable; `docs/formats.md` restates them in our own words with citations.

## 10. Risks and mitigations

| # | Risk | Mitigation |
|---|---|---|
| R1 | SPR2 encoder is not byte-identical to Maxis's (pixel-command choice, trailing transparent runs, pad bytes, more than one historical encoder) | Lossless data model (§3.3) means unedited frames never pass through the encoder. Corpus survey (M1-T08) derives rules from real data before the encoder is written (M1-T09). Gate: L3 pixel-identical 100% required, L2 byte-identical tracked with target ≥ 99% and every mismatch class documented (M1-T13). |
| R2 | Format details taken from FreeSO are wrong or TSO-specific for TS1 files (OBJD versions, DGRP versions, `rsmp`, STR# filler) | Every such item is tagged Unverified in docs/formats.md; M1-T08 survey and L1 round-trip prove or correct them on the user's files. Parsers preserve unknown bytes instead of assuming. |
| R3 | Malformed Maxis files (chunk sizes past EOF, duplicate type+id, zero-size frames) | Parser is tolerant and records `Warning`s; such files are still L0 round-trippable because raw bytes are kept. Known cases become synthetic fixtures. |
| R4 | Cloned objects don't appear or crash in game (GUID collision, missing catalog flags, thumbnail id) | `validate` (M2-T13) encodes the catalog-visibility rules; GUID collision index (M2-T06); human in-game gates per version (M2-T15, M3-T08). |
| R5 | Legacy Collection can't be run on the user's macOS machine easily | Automated gates never need the game. In-game gates are short human checklists runnable in CrossOver, a Windows VM or on any Windows PC; Simitone is the secondary renderer on macOS. |
| R6 | Accidentally committing Maxis assets | `cargo xtask guard-assets` in CI and as a pre-commit check (M1-T01); corpus tests read from outside the repo; reports contain only statistics and hashes. |
| R7 | TMog XML/BMP layout and magic-cookie bit placement are known only from tutorials | Treated as interop extras behind explicit tasks (M3-T06, M2-T06) that require a user-supplied sample before implementation. |
| R8 | Skin formats (BCF/BMF/CFP) are documented only as source code | M5 starts with a documentation spike (M5-T01) that writes the layouts into docs/formats.md before any parser task starts. |
| R9 | Agents drift on names and interfaces across parallel tasks | This file fixes command names, error codes and module names; AGENTS.md makes changing them a plan change, not a code change. |
| R10 | Large Downloads folders / many FARs make `list` slow | Install index cached under the platform cache dir, keyed by path + size + mtime (M2-T05), replaced by the persistent, incrementally rebuilt index with a benchmarked query engine (M6-T08). |
| R11 | The original Maxis light rig and materials are undocumented; authored objects look out of place | Calibration against the user's own Maxis sprites with measurable look metrics and a frozen, hash-checked preset (M7-T07, M7-T09); metric + vision review per object (M7-T10); human side-by-side check in game (M7-T12). |
| R12 | Rendering facts were relayed from upstream sources, not read in this planning session (camera constants, depth window, posterize bits) | M7-T01 verifies each against the ts1-renderer / ts1-compiler source before code depends on it; the tile-diamond and depth tests (M7-T02, M7-T03) would fail on a wrong constant. |
| R13 | Wrong depth mapping → authored objects sort wrongly against Sims and neighbours | Exact analytic depth tests (M7-T04); review checks the depth plane (M7-T10); explicit seated-Sim and behind-counter checks in game (M7-T12). |
| R14 | Blender version drift or Cycles non-determinism breaks renders or tests | One pinned Blender LTS, version warning, tolerance-based render tests, deterministic post-render stages; Blender tests are a local tier, never a CI requirement (docs/testing.md §11). |
| R15 | Reference sprites or contact sheets containing Maxis pixels leak into the repo | They are written only to the platform cache or a user-given directory outside the repository (path check in `calibrate` and `review`), and `guard-assets` rejects images and `.exr`/`.blend` files. Only aggregate statistics and GUID lists are committed. |

## 11. Milestones

| Milestone | Outcome | Depends on |
|---|---|---|
| [M1 — Foundation](milestones/M1-foundation.md) | IFF + FAR1 read/write, PALT, SPR2 decode/encode, round-trip harness passing on the user's install, CLI skeleton | — |
| [M2 — Single-tile objects](milestones/M2-single-tile-objects.md) | `clone`, `export`, `import`, `recolor`, `set-catalog`, `validate`, MCP server; a recoloured single-tile clone shows up in both games | M1 |
| [M3 — Multi-tile and TMog parity](milestones/M3-multi-tile-parity.md) | Multi-tile clone/export/import, SPR#, auto-zoom, mirrored rotations, TMog folder import, parity matrix | M2 |
| [M4 — Floors and walls](milestones/M4-floors-walls.md) | Create/export `.flr` and `.wll` | M1, M2-T02 (STR#), M2-T08 and M2-T10 (PNG IO, quantization), M3-T04 (SPR#, walls only), M3-T05 (downscale) |
| [M5 — Skins and clothing](milestones/M5-skins.md) | CMX/BCF/SKN/BMF parsing, skin retexture and clone | M1, M2-T10 (quantization, for M5-T05) |
| [M6 — GUI](milestones/M6-gui.md) | Persistent object index, search and previews in core/CLI/MCP (M6-T08, M6-T09), then a cross-platform desktop app whose main screen is a searchable catalog of every object in the game | M2 (minimum; M6-T09 needs M3-T03), M3–M5 for full coverage |
| [M7 — Blender rendering pipeline](milestones/M7-blender-pipeline.md) | `render`, `compile`, `calibrate`, `review`: author new objects in Blender with a calibrated Maxis look and get a playable `.iff` | M2, M3-T01, M3-T03 |

Ordering notes (changes from the original brief and why):

- The **MCP server is the last implementation task of M2** rather than a separate milestone: it is a thin layer, and having it early lets agents run the end-to-end flow themselves.
- **SPR# lives in M3** (M3-T04) because walls need it and the M1 survey will tell whether any objects still use it.
- **M4 and M5 do not depend on M3's multi-tile work** (M4 needs only M3-T04 and M3-T05 from it), so they can be staffed in parallel with M3 once their listed dependencies are done.
- The **corpus survey precedes the encoder** inside M1, because the only inspectable encoder (FreeSO's) is not known to match Maxis output.
- **M7 (Blender pipeline) is numbered last but runs right after M2, alongside M3.** It was added after the first version of this plan; numbering it M7 keeps every existing task ID stable. It needs only M2 plus M3-T01 and M3-T03.
- **The object index and previews (M6-T08, M6-T09) are core/CLI/MCP work inside the GUI milestone** and should be scheduled right after M2 / M3-T03, so agents get search long before the GUI exists.

Task ID format: `M<milestone>-T<two digits>`, e.g. `M2-T07`. Human-only checks are tasks too and are marked **[human gate]**.
