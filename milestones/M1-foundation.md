# M1 — Foundation: IFF/FAR read-write, SPR2 decode/encode, round-trip harness

Outcome: objectizer can read and write every IFF and FAR1 file in the user's install without changing a byte, decode and encode SPR2, and prove it with `cargo xtask corpus`. A CLI skeleton exposes `locate`, `list`, `inspect`, `extract`, `roundtrip`.

Read first: [PLAN.md](../PLAN.md) §3–4, [docs/formats.md](../docs/formats.md) §1, 2, 4, 6, [docs/testing.md](../docs/testing.md), [AGENTS.md](../AGENTS.md).

## Task graph

```
T01 ─┬─ T02 ─┬─ T03 ─┬─ T06 ── T07 ─┬─ T08 ── T09 ─┐
     │       │       ├─ T12 (needs T08)             ├─ T10 ── T13
     │       └─ T04 ─┘                              │
     ├─ T05 ────────────────────────(T08, T11)      │
     └────────────── T11 (needs T03, T04, T05) ─────┘
```

Parallel lanes once T02 is done: {T03 → T06 → T07}, {T04}, {T05}. T11 can start as soon as T03, T04, T05 are done and grows as later tasks land. T08 needs T04, T05, T07. T09 and T12 need T08's findings.

Conventions for every task below: "tests pass" means `cargo fmt --all --check && cargo clippy --workspace --all-targets -- -D warnings && cargo test --workspace && cargo xtask guard-assets` all exit 0. Corpus criteria use `OBJECTIZER_GAME_DIR`.

---

### M1-T01 — Workspace scaffold, CI, asset guard, error model

- **Goal:** Create the repository skeleton every other task builds on.
- **Inputs:** PLAN.md §3.1, §3.4, §9; docs/testing.md §9.
- **Outputs:**
  - `git init`; `LICENSE` (MPL-2.0 text); `README.md` (what it is, status, legal note: needs your own game files); `.gitignore` (target, corpus/survey outputs, common asset extensions).
  - Cargo workspace with crates `objectizer-core`, `objectizer-cli`, `objectizer-mcp` (stub `main` that exits 8), `objectizer-testkit`, `xtask`; `rust-toolchain.toml`.
  - `objectizer_core::error`: `Error`, `ErrorCode` (the 8 codes of PLAN.md §3.4), `Warning`, `ErrorCode::exit_code()`.
  - `cargo xtask guard-assets` as specified in docs/testing.md §9.
  - `.github/workflows/ci.yml` with the 3-OS matrix and the four steps.
  - `STATUS.md`: task status board (every task ID from `milestones/`, all `todo`), format per AGENTS.md §2.
  - Record Niotso's license in docs/formats.md §0 (replace the **[U]**).
- **Acceptance:**
  - Tests pass (see conventions).
  - `cargo test -p objectizer-core error::` includes a test asserting each `ErrorCode` maps to the exit code in PLAN.md §3.4.
  - `cargo xtask guard-assets` exits non-zero when a file named `x.iff` or a file starting with the IFF or FAR magic is `git add`-ed (covered by a test that runs the guard on a temp git repo), and 0 on the clean tree.
  - `grep -c "Mozilla Public License" LICENSE` ≥ 1.
  - Every task heading in `milestones/*.md` has a row in `STATUS.md` (scripted check: the two sorted ID lists are equal).
- **Depends on:** nothing.
- **Parallel with:** nothing (first task).

### M1-T02 — Binary IO helpers and testkit builders

- **Goal:** Bounds-checked reader/writer and the synthetic fixture builders for containers.
- **Inputs:** docs/formats.md §1, §2; docs/testing.md §3.
- **Outputs:**
  - `objectizer_core::io::{Reader, Writer}`: u8/u16/u32/i16/i32/f32 in both endiannesses, fixed-length bytes, C strings, Pascal strings, `remaining()`, absolute seek; every read returns `Result` with offset in the error.
  - `objectizer_testkit::iff::IffBuilder`, `objectizer_testkit::far::FarBuilder` (independent byte-level builders, **not** using core's writers, so they can cross-check them).
- **Acceptance:**
  - Tests pass.
  - Unit tests: reading past the end returns `E_PARSE` with the correct offset; round trip of each primitive in both endiannesses.
  - `IffBuilder` output for a fixed chunk list matches a hex literal in the test (header 64 bytes, chunk header 76 bytes, big-endian size including header).
- **Depends on:** M1-T01.
- **Parallel with:** M1-T05.

### M1-T03 — IFF container: lossless read/write

- **Goal:** `IffFile` per PLAN.md §3.3.
- **Inputs:** docs/formats.md §1.
- **Outputs:** `objectizer_core::iff`: `IffFile::parse(&[u8])`, `IffFile::to_bytes()`, `Chunk { type, id, flags, label_raw, data, dirty }`, lookup by `(type, id)` (first wins, duplicates produce `W_DUPLICATE_CHUNK`), `add_chunk`, `remove_chunk`, `replace_data`, free-id finder. `rsmp` is held as an opaque chunk in this task. Tolerates: `2.0` signature, chunk size overrunning EOF (`W_CHUNK_OVERRUN`, bytes preserved), trailing bytes after the last chunk (preserved).
- **Acceptance:**
  - Tests pass.
  - Test `iff::roundtrip_synthetic`: for every `IffBuilder` variant (with/without rsmp, duplicates, overrun, label garbage, trailing bytes) `to_bytes(parse(x)) == x`.
  - Proptest `iff::roundtrip_random`: random chunk lists round-trip.
  - Truncation/bit-flip test: no panic, only `Ok` or `E_PARSE`.
  - Corpus (run after T10 exists, recorded in T13): L0 = 100% for IFF.
- **Depends on:** M1-T02.
- **Parallel with:** M1-T04, M1-T05.

### M1-T04 — FAR1 archive read/write

- **Goal:** Read FAR1 v1a, list and extract entries, and write archives; lossless rewrite of an unmodified archive.
- **Inputs:** docs/formats.md §2.
- **Outputs:** `objectizer_core::far::FarArchive`: `parse`, `entries()`, `read(name)`, `to_bytes()`, `build(entries)`. Preserves entry order, both length fields, and any gap bytes between data and manifest. Rejects v1b / FAR3 with `E_UNSUPPORTED`.
- **Acceptance:**
  - Tests pass.
  - `far::roundtrip_synthetic` and a proptest: `to_bytes(parse(x)) == x`; `parse(build(entries))` returns the same names and bytes.
  - A FAR3-magic or version ≠ 1 buffer returns `E_UNSUPPORTED`.
  - Corpus (recorded in T13): L0 = 100% for FAR; update docs/formats.md §2 with whether data is contiguous.
- **Depends on:** M1-T02.
- **Parallel with:** M1-T03, M1-T05.

### M1-T05 — Game install discovery

- **Goal:** Find installs and enumerate content sources without the Windows registry.
- **Inputs:** PLAN.md §7.
- **Outputs:** `objectizer_core::install`: `discover(opts) -> Vec<Install>`, `Install { kind: Legacy|Complete|Unknown, root, downloads_dir, saves_dir, sources() }` where `sources()` yields every `*.far`, loose `*.iff`, `*.flr`, `*.wll` under the root with a stable relative path. Steam `libraryfolders.vdf` parser; CrossOver/Wine/Proton prefix scan; `OBJECTIZER_GAME_DIR` and `--game-dir` overrides. `ops::locate`.
- **Acceptance:**
  - Tests pass.
  - Tests build fake install trees in a temp dir (empty files with the right names; a hand-written `libraryfolders.vdf` and `appmanifest_3314060.acf`) and assert: Steam library on a second drive is found; kind is `legacy` for the Steam layout and `complete` for the Maxis layout; env override wins; a non-existent path returns `E_NOT_FOUND`.
  - No registry or network access (grep: no `winreg`, no HTTP crate in `Cargo.lock` for core).
- **Depends on:** M1-T01.
- **Parallel with:** M1-T02, M1-T03, M1-T04.

### M1-T06 — PALT

- **Goal:** Typed palette with lossless round trip.
- **Inputs:** docs/formats.md §6.
- **Outputs:** `objectizer_core::chunks::palt::Palette { version, reserved: [u8; 8], colors: Vec<[u8; 3]>, trailing }`, `decode`, `encode`.
- **Acceptance:**
  - Tests pass; synthetic round trip incl. non-zero version/reserved and trailing bytes.
  - `encode` of a palette with > 256 colours returns `E_VALIDATION`.
  - Corpus (recorded in T13): L1 PALT = 100%; docs/formats.md §6 updated with observed version/reserved values.
- **Depends on:** M1-T03.
- **Parallel with:** M1-T04, M1-T05.

### M1-T07 — SPR2 decoder

- **Goal:** Decode SPR2 v1000 chunks into frames with colour-index, z and alpha planes, keeping raw frame bytes.
- **Inputs:** docs/formats.md §4.1–4.3. May port from FreeSO `SPR2.cs` (MPL-2.0, add attribution line).
- **Outputs:** `objectizer_core::chunks::spr2`: `Spr2 { version, default_palette, frames: Vec<Spr2Frame>, layout }` where `layout` records offset order and any gap bytes; `Spr2Frame { raw: Vec<u8>, header, .. }`; `Spr2Frame::decode() -> Pixels { width, height, color: Vec<u8>, z: Option<Vec<u8>>, alpha: Option<Vec<u8>> (5-bit values) }`; `Spr2::to_bytes()` that reassembles from raw frames (no encoder yet). Unknown row/pixel opcodes → `E_PARSE` with offset. Version 1001 → `E_UNSUPPORTED`.
- **Acceptance:**
  - Tests pass.
  - `spr2::decode_handmade`: hand-assembled command streams from `testkit::spr2` (every opcode, odd counts with pad, skip rows, zero-size frame, flags 1/3/7) decode to the expected planes.
  - `spr2::container_roundtrip`: `to_bytes(parse(x)) == x` for synthetic chunks incl. out-of-order offsets and gaps.
  - Corpus (recorded in T13): every SPR2 frame in the install decodes without error or is listed with a reason; L1 SPR2 (container level) = 100%.
- **Depends on:** M1-T03, M1-T06.
- **Parallel with:** M1-T04, M1-T05.

### M1-T08 — Corpus survey

- **Goal:** Measure, on the user's real files, everything docs/formats.md marks as unknown for M1, especially how Maxis encoded SPR2.
- **Inputs:** docs/formats.md §1.1, §2, §4.4, §6, §14; docs/testing.md §5. Also retry fetching the design documents (formats.md source S11) and, if reachable, note relevant facts with citation.
- **Outputs:**
  - `cargo xtask survey` → `target/survey/survey.json` + printed summary.
  - Statistics at minimum: chunk-type histogram and versions (OBJD, DGRP, SPR2, SPR#, STR# formats, PALT); IFF header variants; rsmp presence, version, type/entry ordering, self-listing; FAR layout facts; SPR2: flags histogram, per-flag pixel-opcode histogram, opcode used for opaque z = 0 pixels, trailing opcode-3 runs (count of rows ending with one vs rows ending early), leading-run shapes, max run lengths, pad-byte value histogram, skip-row usage at frame start/end, empty-row encodings, bytes after end marker, offset ordering/gaps, per-frame palette id values, zero-size frames; all split by source (base game / each expansion / Downloads).
  - `docs/survey-findings.md` (numbers and conclusions only) and updated tags in docs/formats.md (**[U]** → **[V-corpus]** or corrected text) for §1.1, §2, §4.1, §4.4, §6.
  - A written "Maxis SPR2 encoder rules" section in `docs/survey-findings.md`: one explicit rule per question 1–8 of formats.md §4.4, or "varies: …" with the variants.
- **Acceptance:**
  - Tests pass (the survey's statistics code is unit-tested on synthetic chunks with known answers: `xtask::survey::tests`).
  - `cargo xtask survey` exits 0 on a real install and `target/survey/survey.json` validates against the schema checked in at `xtask/survey.schema.json`.
  - `docs/survey-findings.md` exists and contains a heading for each of the 8 questions of formats.md §4.4 (`grep -c '^### Q[1-8]' docs/survey-findings.md` = 8).
  - `cargo xtask guard-assets` passes (no asset bytes in the findings).
- **Depends on:** M1-T04, M1-T05, M1-T07.
- **Parallel with:** M1-T11.
- **Needs game files:** yes. If the agent has none, deliver the tool and tests, mark the findings criteria open, and stop; M1-T09 must not start without findings.

### M1-T09 — SPR2 encoder

- **Goal:** Encode `Pixels` + header into frame bytes that follow the Maxis rules found in M1-T08.
- **Inputs:** `docs/survey-findings.md` encoder rules; docs/formats.md §4. Read `mixiate/ts1-compiler` (MPL-2.0) and FreeSO `SPR2FrameEncoder.cs` (MPL-2.0) first; port where they agree with the findings, with attribution.
- **Outputs:** `Spr2Frame::encode(pixels, header, &EncodeOptions) -> Vec<u8>`; `Spr2::replace_frame(index, pixels, header)`; `Spr2::push_frame`; `Spr2::to_bytes()` now lays out re-encoded frames and rebuilds the offset table. If the survey found more than one Maxis variant, `EncodeOptions` selects one and the default is the most common.
- **Acceptance:**
  - Tests pass.
  - Proptest `spr2::encode_decode_random`: random grids for flags 1, 3, 7, sizes 0–300, → `decode(encode(p)) == p`.
  - `spr2::encode_rules`: one unit test per encoder rule in `docs/survey-findings.md`, asserting the exact bytes for a small grid.
  - Runs longer than 8191 pixels and rows longer than 8191 bytes are handled or rejected with `E_VALIDATION` (test).
  - Corpus (via T10, recorded in T13): L3 = 100%; L2 ≥ 99% with zero `unexplained`.
- **Depends on:** M1-T07, M1-T08.
- **Parallel with:** M1-T12.

### M1-T10 — Round-trip harness

- **Goal:** The gate tool of docs/testing.md §4, levels L0–L3.
- **Inputs:** docs/testing.md §4.
- **Outputs:** `ops::roundtrip` (request: paths, levels; response: the report); `objectizer roundtrip`; `cargo xtask corpus`; mismatch classifier for L2 (classes named after the survey questions, plus `unexplained`); report JSON with relative paths only.
- **Acceptance:**
  - Tests pass.
  - `roundtrip::detects_l0_break`, `…l1…`, `…l2…`, `…l3…`: on synthetic inputs, a deliberately wrong writer/encoder (test-only injection) makes each level fail and the command exit 5; clean inputs exit 0.
  - With `OBJECTIZER_GAME_DIR` unset and no install found, `cargo xtask corpus` exits 3.
  - Snapshot test of the report JSON shape.
- **Depends on:** M1-T03, M1-T04, M1-T09 (L0/L1 parts can be built after T03/T04; L2/L3 after T09).
- **Parallel with:** M1-T11, M1-T12.

### M1-T11 — CLI skeleton

- **Goal:** The `objectizer` binary with the envelope, exit codes and the M1 commands.
- **Inputs:** PLAN.md §4.
- **Outputs:** `objectizer locate`, `objectizer list <path>` (FAR entries or IFF chunks; `--type` filter), `objectizer inspect <iff|far::entry> [--chunk TYPE#ID]` (chunk table; typed dump for PALT and SPR2 headers/frame table), `objectizer extract`, `objectizer roundtrip`; global flags `--json`, `--dry-run`, `--game-dir`, `--quiet`, `-v`, `--force`; `<far>::<entry>` source syntax; human-readable tables when not `--json`.
- **Acceptance:**
  - Tests pass.
  - `cargo test -p objectizer-cli`: `insta` snapshots of `--json` output for each command on synthetic fixtures; a test per error code proving the exit code; `extract --dry-run` leaves the temp dir unchanged and lists `data.writes`.
  - `objectizer --help` lists exactly the M1 commands (snapshot).
  - With `--json`, stdout is one parseable JSON document for both success and failure (test pipes through `serde_json`).
- **Depends on:** M1-T03, M1-T04, M1-T05 (and T07, T10 for their commands).
- **Parallel with:** M1-T08, M1-T09, M1-T12.

### M1-T12 — rsmp parse and rebuild

- **Goal:** Keep IFFs valid when chunks are added, removed or resized.
- **Inputs:** docs/formats.md §1.1 as corrected by M1-T08.
- **Outputs:** `objectizer_core::iff::rsmp`: typed parse, `rebuild(&IffFile)`, and the write-path rule of PLAN.md §3.3 (verbatim if layout unchanged, rebuilt otherwise, absent stays absent). Header rsmp offset patched.
- **Acceptance:**
  - Tests pass; synthetic tests for both rsmp versions.
  - Corpus check (part of `cargo xtask corpus` from this task on, level name `L1` type `rsmp`): for every IFF that has an rsmp, `rebuild(file)` equals the stored rsmp byte-for-byte. Target 100%; mismatches classified. If Maxis rsmp cannot be reproduced exactly, document why in docs/formats.md §1.1 and require instead: parsed-equal (same entries) = 100%.
  - Test: add a chunk to a synthetic IFF with rsmp → written file parses, rsmp lists the new chunk with the right offset.
- **Depends on:** M1-T03, M1-T08.
- **Parallel with:** M1-T09, M1-T10, M1-T11.

### M1-T13 — M1 gate

- **Goal:** Prove M1 on real files and freeze the findings.
- **Inputs:** everything above; a real install (ideally both game kinds).
- **Outputs:** "Gate record" section below filled in: date, commit, game kind(s), totals and per-level results copied from `target/corpus/report.json`; list of L2 mismatch classes with counts; any files excluded and why. docs/formats.md open-item index updated for all M1 items.
- **Acceptance:**
  - `cargo xtask corpus` exits 0 on at least one real install: L0 100%, L1 100% for PALT/SPR2/rsmp, L3 100%, L2 ≥ 99% and `unexplained` = 0.
  - `grep -n "TODO-M1" docs/formats.md` returns nothing.
  - Tests pass on the gate commit.
- **Depends on:** M1-T01 … M1-T12.
- **Parallel with:** nothing.

## Gate record

_To be filled by M1-T13._
