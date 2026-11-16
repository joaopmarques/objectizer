# The Sims 1 file formats — reference for objectizer

This is a restatement, in our own words, of what public sources say about the formats objectizer handles. It is the starting point for implementation, not ground truth: **the user's game files are ground truth**, and task M1-T08 (corpus survey) exists to confirm or correct everything marked Unverified.

Status tags used below:

- **[V-src]** — read directly in open source code that parses real files (source cited).
- **[V-2src]** — two independent implementations agree.
- **[U]** — unverified: inferred, single weak source, or known to be uncertain. Must be confirmed on real files before code relies on it; parsers must preserve the bytes rather than assume.
- **[TODO-Mx-Tyy]** — the task that resolves an open item.

When a task confirms or corrects an item, it updates this file (change the tag to **[V-corpus]** and add the survey figure).

## 0. Sources

| ID | Source | License | Use | Access status at planning time |
|---|---|---|---|---|
| S1 | FreeSO, `riperiperi/FreeSO` (active upstream; `RHY3756547/FreeSO` is identical for format code) — `TSOClient/tso.files/Formats/IFF/`, `tso.files/FAR1/`, `tso.content/`, `FSO.IDE/` (Volcanic) — https://github.com/riperiperi/FreeSO | MPL-2.0 | may port | read |
| S2 | Niotso `Libraries/FileHandler/iff/spr2.c` — https://github.com/daddesio/Niotso | not checked **[U]** | read only until license recorded | read |
| S3 | niotso wiki (wiki.niotso.org) and IFFSnooper docs | — | reference | **not reachable** (SSL error, archive blocked) |
| S4 | SneakyLib — https://github.com/SneakySims/SneakyLib | none (all rights reserved) | read only, never copy | README read |
| S5 | SimObliterator Suite — https://github.com/DnfJeff/SimObliterator_Suite | unclear (README says MIT, GitHub shows none) | read only | README read |
| S6 | trolando/Sims1Tools (IFF Pencil 2, FARx…) — https://github.com/trolando/Sims1Tools | Apache-2.0, binaries only | workflow reference | README read |
| S7 | mixiate ts1-compiler (Rust) — https://github.com/mixiate/ts1-compiler | MPL-2.0 | may port | README read, **source not yet read** [TODO-M1-T09] |
| S8 | mixiate ts1-blender-io — https://github.com/mixiate/ts1-blender-io ; ts1-renderer — https://github.com/mixiate/ts1-renderer | MPL-2.0 | may port | README/wiki read, source not yet read [TODO-M5-T01] |
| S9 | Cyndersanity, Sims Legacy mod guide — https://cyndersanity.github.io/sims-legacy-mod-guide | — | workflow, paths | read |
| S10 | riperiperi, Legacy Collection technical overview — https://gist.github.com/riperiperi/78c843b7fc3b11a92e3d5585a7815fee | — | Legacy differences | read (summarized) |
| S11 | Don Hopkins / Maxis design documents — https://linkedbyair.net/bin/The%20Sims%20Design%20Documents/ (e.g. `117492995-The-Sims-Object-IFF-File-Format.pdf`, `…Transmogrifer-Renovation-Plan.pdf`) | — | reference | **404 at planning time**; retry in M1-T08 |
| S12 | Community TMog tutorials — https://bwsost.woobsha.com/ , https://simfreaks.com/tutorials/objects/ | — | TMog workflow only | read |
| S13 | Simitone — https://github.com/riperiperi/Simitone | MPL-2.0 | reference renderer | not read |
| S14 | Legacy iffinator in FreeSO, `Other/tools/Iffinator/Iffinator/Flash/Iff.cs` | MPL-2.0 | rsmp layout | read |

Transmogrifier itself is closed-source freeware. Nothing here comes from its code; only its user-visible workflow is described (§13).

## 1. IFF container

Source: S1 `IffFile.cs`.

- **File header, 64 bytes.** [V-src]
  - Bytes 0–59: ASCII signature `IFF FILE 2.5:TYPE FOLLOWED BY SIZE` + one `0x00` + ` JAMIE DOORNBOS & MAXIS 1` (34 + 1 + 25 = 60). A `2.0` variant exists (seen by FreeSO in `house11.iff`).
  - Bytes 60–63: offset of the `rsmp` chunk, u32 **big-endian**; 0 if none.
  - objectizer stores all 64 bytes verbatim and patches only the rsmp offset.
- **Chunk header, 76 bytes, big-endian.** [V-src]

  | Offset | Size | Field |
  |---|---|---|
  | 0 | 4 | type, ASCII (`OBJD`, `SPR2`, `PALT`, `STR#`, `BMP_`, …) |
  | 4 | 4 | u32 size of the chunk **including** this 76-byte header |
  | 8 | 2 | u16 id |
  | 10 | 2 | u16 flags |
  | 12 | 64 | label, NUL-padded (bytes after the first NUL may be garbage **[U]** — keep raw) |

- Chunks follow each other with **no alignment or padding**. [V-src]
- **Chunk payloads are little-endian** in general (each FreeSO chunk class opens a little-endian reader); exceptions are noted per chunk (SPR#, possibly SPR2 can be big-endian). [V-src]
- Known irregularities objectizer must tolerate (warn, keep bytes): a chunk whose size runs past end of file (FreeSO comment about `walls2.iff`); duplicate (type, id) pairs — FreeSO uses the first. [V-src] How the game resolves duplicates: **[U]**.
- FreeSO's own writer is **not** a model for byte-exact output: it regroups chunks by type, drops `rsmp` and unknown chunk types, and rewrites flags 0 as `0x10`. Do not port its save path.

### 1.1 `rsmp` (resource map)

Source: S14 only; low confidence. **[U] [TODO-M1-T08, M1-T12]**

After the normal 76-byte chunk header:

```
u32 reserved
u32 version          (0 or 1)
u32 magic 'pmsr'     ('rsmp' byte-reversed → body is little-endian)
u32 size
u32 typeCount
per type:
  4 bytes type code  (probably byte-reversed, e.g. 'DJBO')
  u32 entryCount
  per entry:
    u32 offset       (file offset of the chunk)
    u16 chunkId
    [version 1: extra u16]
    u16 flags
    label            (version 0: NUL-terminated, padded to even length? ; version 1: u8-length Pascal string)
```

Open questions for the survey: exact label padding, ordering of types and entries (file order vs sorted), whether `rsmp` lists itself, whether the game works without it (FreeSO-written files have none and community tools often omit it — suggests yes).

## 2. FAR1 archive

Source: S1 `FAR1Archive.cs`, `FarEntry.cs`. All little-endian. [V-src]

```
header (16 bytes):  "FAR!byAZ"  u32 version = 1   u32 manifestOffset
file data …
manifest at manifestOffset:
  u32 fileCount
  per file:
    u32 length
    u32 length2      (same value; no compression in TS1)
    u32 dataOffset
    u32 nameLength   (TS1 "1a"; The Sims Online "1b" uses u16 — out of scope)
    bytes name       (no terminator)
```

Open: whether file data is always contiguous in manifest order with no gaps (needed for byte-exact FAR rewrite) **[U] [TODO-M1-T04/T08]**. Exact FAR file names per expansion are not listed here on purpose; `install` enumerates `*.far` recursively.

## 3. OBJD (object definition)

Source: S1 `OBJD.cs`, `TS1ObjectProvider.cs`, `VMEntity.cs`, `UICatalog.cs`. Little-endian. [V-src] unless noted.

Layout: `u32 version`, then an array of u16 words. Number of words by version:

| Version | u16 words after the version | Notes |
|---|---|---|
| 136 | 78 | ends after ShadowBrightness (offset 158) |
| 138 | 93 ("138a") or 106 ("138b") | FreeSO reads the extras under try/catch — length must be taken from the chunk size, not the version |
| 139 | 94 | |
| 140, 141 | 95 | |
| 142 | 103 | The Sims Online; out of scope |

objectizer's rule: field count = `(payload_len − 4) / 2`; fields are addressed by index; anything beyond the known table is kept as `unknown_words`.

Byte offsets within the payload (u16 unless noted):

| Off | Field | Off | Field |
|---|---|---|---|
| 0 | version (u32) | 4 | StackSize |
| 6 | **BaseGraphicID** (first DGRP id) | 8 | **NumGraphics** |
| 10 | BHAV_Main | 12 | BHAV_Gardening |
| 14 | **TreeTableID** (TTAB/TTAs) | 16 | InteractionGroupID (i16) |
| 18 | ObjectType | 20 | **MasterID** |
| 22 | **SubIndex** (i16) | 24 | BHAV_WashHands |
| 26 | AnimationTableID | **28** | **GUID** (u32 LE) |
| 32 | Disabled | 34 | BHAV_Portal |
| **36** | **Price** | 38 | BodyStringID |
| 40 | **SlotID** | 42 | BHAV_AllowIntersection |
| 44 | UsesFnTable | 46 | BitField1 |
| 48–66 | BHAV ids: PrepareFood, CookFood, PlaceSurface, Dispose, Eat, PickupFromSlot, WashDish, EatSurface, Sit, Stand | 68 | SalePrice |
| 70 | InitialDepreciation | 72 | DailyDepreciation |
| 74 | SelfDepreciating | 76 | DepreciationLimit |
| 78 | RoomFlags | 80 | **FunctionFlags** |
| **82** | **CatalogStringsID** | 84 | Global |
| 86 | BHAV_Init | 88 | BHAV_Place |
| 90 | BHAV_UserPickup | 92 | WallStyle |
| 94 | BHAV_Load | 96 | BHAV_UserPlace |
| 98 | ObjectVersion | 100 | BHAV_RoomChange |
| 102 | MotiveEffectsID | 104 | BHAV_Cleanup |
| 106 | BHAV_LevelInfo | 108 | CatalogID |
| 110 | BHAV_ServingSurface | 112 | LevelOffset |
| 114 | Shadow | 116 | NumAttributes |
| 118 | BHAV_Clean | 120 | BHAV_QueueSkipped |
| 122 | FrontDirection | 124 | BHAV_WallAdjacencyChanged |
| 126 | MyLeadObject | 128 | DynamicSpriteBaseId |
| 130 | NumDynamicSprites | 132 | ChairEntryFlags |
| 134 | TileWidth | 136 | LotCategories |
| 138 | **BuildModeType** | 140 | OriginalGUID (u32) |
| 144 | SuitGUID (u32) | 148 | BHAV_Pickup |
| 150 | ThumbnailGraphic | 152 | ShadowFlags |
| 154 | FootprintMask | 156 | BHAV_DynamicMultiTileUpdate |
| 158 | ShadowBrightness | 160 | BHAV_Repair |
| 162 | WallStyleSpriteID | 164–176 | Ratings (i16): Hunger, Comfort, Hygiene, Bladder, Energy, Fun, Room |
| 178 | RatingSkillFlags | 180 | NumTypeAttributes |
| 182 | MiscFlags | 184 | TypeAttrGUID (u32) |
| 188+ | 138b extras: FunctionSubsort, DTSubsort, KeepBuying, VacationSubsort, ResetLotAction, CommunitySubsort, DreamFlags, RenderFlags, VitaboyFlags, STSubsort, MTSubsort | | |

ObjectType values: 2 Person, 4 Normal, 7 SimType, 8 Portal, 9 Cursor, 10 Prize, 11 Internal, 12 Gift, 34 Food.

### 3.1 Multi-tile objects

- One OBJD per tile plus one master OBJD, all in the same IFF. [V-src] + S9
- Tiles of one object share **MasterID** (non-zero). The master has **SubIndex = −1** (0xFFFF). [V-src]
- A tile's SubIndex encodes its position (commonly `(y << 8) | x`) **[U] [TODO-M3-T01]**.
- Single-tile: MasterID = 0.
- Each OBJD (master and tiles) has its own GUID; all must be unique. [V-src] (registered individually by FreeSO)

### 3.2 Catalog visibility (FreeSO's rule for TS1) [V-src]; the real game's rule is **[U] [TODO-M2-T15]**

An OBJD appears in the catalog when all hold: `FunctionFlags > 0 || BuildModeType > 0`; `Disabled == 0`; `MasterID != 0 || NumGraphics > 0`; `MasterID == 0 || SubIndex == −1`. Buy category = bit position in FunctionFlags (0–7); build category from BuildModeType. Expansion sub-sorts are the 138b extras.

### 3.3 References from OBJD to other chunks

| OBJD field | Target | Status |
|---|---|---|
| BaseGraphicID … +NumGraphics−1 | `DGRP` ids | [V-src] |
| CatalogStringsID | `CTSS` id: string 0 = name, string 1 = description | [V-src] |
| CatalogStringsID | `BMP_` with the same id = catalog thumbnail (FreeSO `UICatalog.cs`) | [V-src]; S9 instead says "`BMP_` id 2000" for a retexture workflow — reconcile **[U] [TODO-M2-T04]** |
| TreeTableID | `TTAB` + `TTAs` | [V-src] |
| SlotID | `SLOT` | [V-src] |
| BHAV_* | `BHAV` ids (local < 4096 semantics etc. not needed; carried untouched) | [V-src] |
| BodyStringID, AnimationTableID | probably `STR#` | **[U]** |
| DynamicSpriteBaseId … +NumDynamicSprites−1 | extra `DGRP`/sprite ids | **[U] [TODO-M2-T05]** |
| chunk label of the OBJD | object's internal name | [V-src] |

`GLOB` chunk: name of the semi-global IFF (Pascal string if first byte < 48, else C string). `FWAV`: NUL-terminated sound name. Both carried untouched.

## 4. SPR2 (sprites with z-buffer and alpha)

Sources: S1 `SPR2.cs`, `SPR2FrameEncoder.cs`; S2 `spr2.c`. [V-2src] unless noted.

### 4.1 Chunk header (version 1000, the TS1 form)

```
u32 version = 1000
u32 frameCount
u32 defaultPaletteId      (PALT chunk id)
u32 offsets[frameCount]   (from start of chunk payload)
frames …
```

- Frame length is not stored: it is the next offset minus this one (or payload end for the last frame). Whether offsets are always ascending and frames contiguous: **[U] [TODO-M1-T08]** — the decoder must not assume it; the lossless model slices frames by sorted offset and keeps any gap bytes.
- Niotso treats the header as big-endian if its first two bytes are zero; row data is always little-endian. Whether TS1 ships big-endian SPR2: **[U] [TODO-M1-T08]**.
- Version 1001 (The Sims Online; palette id and count swapped, per-frame `u32 version, u32 size` wrappers): out of scope → `E_UNSUPPORTED`, unless the survey finds it in TS1 files.

### 4.2 Frame header (16 bytes, little-endian)

| Off | Type | Field |
|---|---|---|
| 0 | u16 | width |
| 2 | u16 | height |
| 4 | u32 | flags: `0x1` colour, `0x2` z-buffer, `0x4` alpha. Observed valid values: 1, 3, 7 |
| 8 | u16 | paletteId (FreeSO: ignored for v1000 — the chunk default is used; 0 or `0xA3A3` = default) |
| 10 | u16 | transparent colour index (< 256) |
| 12 | i16 | y offset (**y before x**) |
| 14 | i16 | x offset |

Zero-sized frames (width or height 0) exist. [V-src S2]

### 4.3 Row and pixel commands

Every command is a u16 LE word: `opcode = word >> 13`, `count = word & 0x1FFF`.

Row level:

| Opcode | Meaning |
|---|---|
| 0 | Pixel row. `count` = byte length of this row **including the 2-byte word itself** (even, ≥ 2). Pixel commands follow until the bytes are consumed. Pixels not covered are transparent. |
| 4 | Skip `count` rows (fully transparent). |
| 5 | End of frame (`0xA000`). |

Pixel level (inside a row):

| Opcode | Payload per pixel | Needs flags | Padding |
|---|---|---|---|
| 1 | z, colour index (2 bytes) | z | none |
| 2 | z, colour index, alpha (3 bytes; alpha 0–31) | z + alpha | 1 pad byte if `3·count` is odd |
| 3 | none — `count` transparent pixels | — | — |
| 6 | colour index only (1 byte; decoder sets z = 0) | — | 1 pad byte if `count` is odd |

Decoded transparent pixels: colour = transparent index, z = 255, alpha = 0.
Alpha is 5-bit. FreeSO decodes as `trunc(a5 × 8.2258)` and encodes as `ceil(a8 / 8.2258)`; objectizer keeps the 5-bit value as the model and uses the rounding in PLAN.md §6 only for PNG files.

### 4.4 What is not known about Maxis-encoded data **[U] — all [TODO-M1-T08], consumed by M1-T09**

1. Opcode choice for opaque pixels in frames that have a z-buffer: always 1, or 6 when z = 0 (FreeSO's encoder uses 6 whenever z == 0)?
2. Whether a row ends right after its last non-transparent pixel or carries a trailing opcode-3 run (FreeSO emits the trailing run).
3. Whether leading transparent pixels are always one opcode-3 run; maximum run lengths; whether runs are split at any boundary.
4. Value of pad bytes (FreeSO writes 0; `0xA3` appears as Maxis filler elsewhere).
5. Whether empty rows are always merged into one opcode-4 command, including at the start and end of the frame, and whether a fully transparent row is ever written as opcode 0 with count 2.
6. Whether frames with flags 1 or 3 exist in object IFFs and what they look like (floors are reported by S4 to work only with opcodes 6 and 3).
7. Whether any bytes follow the `0xA000` end marker inside a frame slice.
8. Whether different expansion packs show different encoder behaviour.

### 4.5 Volcanic's importer (FreeSO IDE) — behaviour reference [V-src S1 `FSO.IDE/Common/SpriteEncoderUtils.cs`, `SPR2ResourceControl.cs`]

- Input files `name_sprid_rN_{near|med|far}_{color|alpha|depth}.bmp`; frame index = `rot + zoomIndex × (frames / 3)` (assumes frames ordered near, med, far).
- Background key colour is yellow (255, 255, 0); the bounding box of non-key pixels becomes the frame rectangle.
- Alpha and depth are taken from the blue channel of their bitmaps; depth is used as-is.
- Colour is quantized with Xiaolin Wu's algorithm to 255 colours; index 255 is the transparent index; flags = 7.
- Palette: reuse the frame's PALT if nothing else references it, else first matching/unreferenced PALT, else a new PALT chunk.
- Can generate med/far by downscaling ×2 and ×4.

## 5. SPR# (older sprites; used by walls)

Source: S1 `SPR.cs` (reader only). [V-src]; no second source, no writer anywhere open → treat details as **[U] until M3-T04's survey**.

```
u32 version       if the first u16 is 0 the chunk is big-endian
u32 frameCount
u32 paletteId
version != 1001:  u32 offsets[frameCount]
version == 1001:  frames sequential, each prefixed by u32 version, u32 size
frame:  u32 reserved, u16 height, u16 width      (height first)
```

Commands are byte pairs `(cmd, count)`:

| cmd | Meaning |
|---|---|
| 0x00, 0x10 | start marker, ignored |
| 0x04 | row; `count` = bytes in the row including this pair; contains pixel commands |
| 0x05 | end of frame |
| 0x09 | skip `count` rows |

Pixel commands inside a row, also `(cmd, n)` pairs: 1 = skip `n` transparent pixels; 2 = fill `n` pixels with one colour (index byte + pad byte); 3 = `n` literal index bytes, padded to even.

Known version numbers in the wild (502–505, 1000, 1001) are **[U]**. FreeSO's big-endian handling is buggy, so BE behaviour is **[U]**.

## 6. PALT (palette)

Source: S1 `PALT.cs`. Little-endian. [V-src]

```
u32 version        (value in Maxis files [U]; FreeSO writes 0... see note)
u32 colorCount     (256 in practice; more than 256 breaks the game per S4)
8 bytes reserved
colorCount × (r, g, b)
```

Note: the value of `version` and of the reserved bytes in Maxis files is **[U] [TODO-M1-T06/T08]**; keep them verbatim.
Palettes are shared: several SPR2 chunks may reference one PALT. A recolor that edits a PALT in place affects all of them; `recolor` reports which sprites are affected.

## 7. DGRP (draw group)

Source: S1 `DGRP.cs`. Endianness assumed little **[U]**. [V-src] for layout.

```
u16 version
version < 20003 (20000, 20001):
  u16 imageCount
  per image:  u16 spriteCount, u8 direction, u8 zoom
    per sprite: u16 type (ignored), u16 spriteId, u16 frameIndex, u16 flags, i16 x, i16 y  [+ f32 z if 20001]
version >= 20003 (20003, 20004):
  u32 imageCount
  per image:  u32 direction, u32 zoom, u32 spriteCount
    per sprite: u32 spriteId, u32 frameIndex, i32 x, i32 y, f32 z, u32 flags  [+ f32 objX, f32 objY if 20004]
```

- direction bit values: `0x01` right-back (`rb`), `0x04` right-front (`rf`), `0x10` left-front (`lf`), `0x40` left-back (`lb`). [V-src]
- zoom values: 1 = far, 2 = medium, 3 = near **[U] [TODO-M2-T03]**.
- A normal object DGRP has 12 images (4 directions × 3 zooms) **[U] [TODO-M2-T03]**.
- sprite flags: `0x1` flip horizontally (this is how two drawn rotations cover four views), `0x4` luminous; `0x2`, `0x8`, `0x10` unknown — preserve.
- `spriteId` is resolved as SPR2 first, then SPR#. [V-src]
- Which versions occur in TS1 files: **[U] [TODO-M2-T03]**. objectizer writes back the version it read; it never upgrades.

## 8. BMP_

Source: S1 `BMP.cs`. The payload is a complete Windows `.bmp` file. [V-src]
Catalog thumbnail: see §3.3. Required dimensions and bit depth for thumbnails are **[U] [TODO-M2-T04]** (the survey records what Maxis files use; `set-catalog` matches the original's size and depth).

## 9. STR# and CTSS (string tables)

Source: S1 `STR.cs`; `CTSS` uses the same class. Little-endian. [V-src]

`i16 format`, then:

| Format | Layout |
|---|---|
| 0 | `u16 n`, n Pascal strings (u8 length) |
| −1 (0xFFFF) | `u16 n`, n C strings |
| −2 (0xFFFE) | `u16 n`, n × (value C string, comment C string) |
| −3 (0xFFFD) | `u16 n`, n × (u8 language code, value C string, comment C string) — the usual TS1 form |
| −4 (0xFFFC) | The Sims Online only — out of scope |

- Language codes 1–20, 1 = US English (0 is treated as 1). In format −3 the entries for all languages are interleaved in one list; string index `i` of language `L` is the i-th entry carrying code `L` **[U] [TODO-M2-T02]** (FreeSO's exact grouping must be confirmed against files).
- Text encoding: single-byte, probably Windows-1252 for Western languages **[U]**; objectizer keeps raw bytes and only re-encodes strings it edits.
- Trailing bytes after the last string exist in Maxis files (S4: "trailing garbage the game ignores"; FreeSO appends `0xA3` bytes when writing for TS1). Preserve as `trailing`. Whether the game needs them: **[U]**; `set-catalog` keeps the original trailing bytes.

## 10. Floors

Source: S1 `WorldFloorProvider.cs`; S4 README; S9. [V-src] for ids; S4 notes are single-source.

- Custom floor = `.flr` file (an IFF) in `GameData/Floors` (subfolders allowed).
  - `SPR2` id **1** = far, **257** = medium, **513** = near.
  - `STR#` id 0: string 0 = name, 1 = price (decimal text), 2 = description.
  - A PALT for the sprites.
- Built-in floors: `GameData/floors.iff` — SPR2 ids `i`, `i+256`, `i+512`; id 2048 is water. Strings in `Build.iff` `STR#` 0x82 (three per floor).
- S4: floor sprites only work with pixel opcodes 6 and 3 (no z, no alpha — consistent with frame flags = 1) **[U] [TODO-M4-T01]**.
- S4: the far-zoom chunk's label prefix selects the footstep sound (`SR…` soft, `MR…` medium, `HR…` hard) **[U] [TODO-M4-T01]**.
- Tile sprite dimensions per zoom (expected 127×64-ish diamonds: near 128×64, med 64×32, far 32×16) **[U] [TODO-M4-T01]**.

## 11. Walls

Source: S1 `WorldWallProvider.cs` only. [V-src], single source.

- Custom wall = `.wll` file (an IFF) in `GameData/Walls`.
  - `SPR#` id **1** = far, **1793** = medium, **2049** = near. Each chunk has several frames (wall segment orientations); the catalog thumbnail is frame 2 of chunk 1793.
  - `STR#` id 0: name, price, description.
- Built-in: `GameData/walls.iff` (styles: ids `i`, `i+512`, `i+1024` for even `i` in 2–510; patterns at `i+1536`, `i+1792`, `i+2048`), names in `Build.iff` `STR#` 0x81 / 0x83.
- Frame count, frame order and dimensions per zoom: **[U] [TODO-M4-T03]**.
- `WALm` / `FLRm` chunks (wall/floor maps inside house files): partly understood, out of scope.

## 12. Skins and clothing

Sources: S8 (wiki only), S9, http://www.simfreaks.com/jerome/guides/uk_guide4.htm. Binary layouts are **not documented anywhere readable at planning time** → **[U] [TODO-M5-T01]** writes this section properly from the ts1-blender-io source.

What is known:

- A skin is a set of files tied together by name: a `.cmx` (text descriptor: which meshes on which bones with which texture) or `.cmx.bcf` (binary form), one or more `.skn` (text mesh) or `.bmf` (binary mesh), and `.bmp` textures.
- File naming: `<prefix><number><gender F|M|U><age A|C><body Fat|Fit|Skn|Chd>[<tone lgt|med|drk>]_<Name>`, e.g. `B001MAFatLgt_Extreme.bmp`. Prefixes: `C` head, `B` body, `L` sleepwear, `F` formal, `S` swimwear, `W` winter, `H` high fashion / hands, `N`/`U` nude/underwear.
- The name inside a `.cmx` and on line 1 of a `.skn` must match the file name.
- Textures must be 8-bit indexed BMP (otherwise rendered white); typical sizes body 256², head 128², hands 64².
- Locations: `GameData/Skins` (no subfolders; loose files override FAR contents), `ExpansionShared/SkinsBuy`, expansion FARs under `Skins/`. Skeletons: `GameData/Animation/Animation.far` (`adult-skeleton.cmx.bcf`, `child-skeleton.cmx.bcf`).
- CFP (compressed animation data) is out of scope for M5.

## 13. Transmogrifier workflow (behaviour reference only)

From S9 and S12; nothing from TMog's code or documents (S11 unreachable).

1. Clone: pick an object, name it, enter a magic cookie; TMog writes a copy with new GUIDs to `Downloads/Transmogrified/`.
2. Export: writes an XML map plus 8-bit BMPs, in folders per sprite and per tile. Per view up to three files: `<name>_<large|medium|small>_<front|back>_p.bmp` (colour, indexed), `_z.bmp` (depth), `_a.bmp` (alpha). Only two drawn rotations are exported; the others are mirrored via the draw-group flip flag. Options: just colours / one zoom / all zooms / all channels.
3. Edit the BMPs, keeping the background key colour and file names.
4. Import by selecting the XML; TMog can derive the smaller zooms from the large one.
5. Set name, description, price. Thumbnails need another tool.

XML elements seen in tutorials: `<drawgroups>` / `<drawgroupitem spritenum flags shadowflags>`, `<sprites>` / `<sprite framecount>` / `<spriteframe index filename zoom proxy>`. Full schema **[U] [TODO-M3-T06]**.
GUID composition (cookie in the middle 16 bits, random elsewhere) is from a tutorial: **[U] [TODO-M2-T06]**.

## 14. Open-item index

| Item | Section | Resolved by |
|---|---|---|
| rsmp layout and ordering | 1.1 | M1-T08, M1-T12 |
| FAR data contiguity | 2 | M1-T04, M1-T08 |
| PALT version/reserved values | 6 | M1-T06, M1-T08 |
| Maxis SPR2 encoder rules (8 questions) | 4.4 | M1-T08 → M1-T09 |
| SPR2 offset ordering, big-endian SPR2 | 4.1 | M1-T08 |
| OBJD versions present, extras | 3 | M1-T08, M2-T01 |
| STR# −3 grouping, trailing bytes, encoding | 9 | M2-T02 |
| DGRP versions, zoom values, image count | 7 | M2-T03 |
| Thumbnail chunk id and bitmap properties | 3.3, 8 | M2-T04 |
| Dynamic sprites | 3.3 | M2-T05 |
| Magic-cookie bit placement | 13 | M2-T06 |
| Real game's catalog-visibility rule; Legacy Downloads path | 3.2 | M2-T15 |
| Multi-tile SubIndex encoding | 3.1 | M3-T01 |
| SPR# details | 5 | M3-T04 |
| TMog XML schema | 13 | M3-T06 |
| Floor sprite rules | 10 | M4-T01 |
| Wall frames | 11 | M4-T03 |
| Skin binary formats | 12 | M5-T01 |
| Niotso license | 0 | M1-T01 |
