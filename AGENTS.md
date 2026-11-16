# AGENTS.md — conventions for implementing agents

You are implementing objectizer, an open-source Rust tool for cloning and editing The Sims 1 objects, floors, walls and skins. Read this file, then [PLAN.md](PLAN.md), then the milestone file of your task.

## 1. Hard rules

1. **Never add game assets to the repository.** No `.iff`, `.far`, `.flr`, `.wll`, `.bmp`, `.cmx`, `.skn`, `.bcf`, `.bmf`, sprites, textures, sounds, or byte dumps of them, and no reference sprites, contact sheets, renders (`.exr`), `.blend` files or imported Sim meshes — not in tests, docs, issues, commit messages or logs. Fixtures are generated in code (`objectizer-testkit`). Reports may contain counts, hashes, offsets and at most 32 bytes of hex context. `cargo xtask guard-assets` must pass.
2. **Never copy from closed or unlicensed sources.** Transmogrifier, IFF Pencil 2 and other closed tools: workflow knowledge only, no code, no decompilation. SneakyLib (no license) and SimObliterator (unclear license): read to understand, never copy or translate.
3. **Porting is allowed only from MPL-2.0 sources** (FreeSO, Simitone, mixiate's ts1-compiler / ts1-renderer / ts1-blender-io). A file with ported logic gets, under the MPL header: `// Portions derived from <project> <path> (MPL-2.0), <url>`. Python files under `blender/` use the same line with `#`. **No GPL dependencies**: in particular never add `imagequant`, even though ts1-compiler uses it.
4. **Do not rename public things.** Crate names, command names, flags, MCP tool names, error codes, exit codes, bundle file naming and task IDs are defined in PLAN.md. If one is wrong, change PLAN.md in the same commit and say so in the hand-over; never let code and plan diverge.
5. **Never claim a criterion you did not run.** If you have no game files, corpus criteria stay open and you say so. Human gates are filled in by the human only.
6. **Unedited data stays byte-identical.** Don't "clean up" files: no reordering chunks, normalizing flags, trimming labels, dropping unknown chunks or trailing bytes.

## 2. Picking up a task

1. Tasks live in `milestones/M*-*.md`, IDs `M<n>-T<nn>`. Pick one whose **Depends on** tasks are all done (see `STATUS.md`, created by M1-T01: a table of task ID → `todo | in-progress | done | blocked`, with commit hash and notes).
2. Set it to `in-progress` in `STATUS.md` in your first commit, on a branch named `m<n>-t<nn>-<slug>`.
3. Read the task's **Inputs**. Read the cited sections of `docs/formats.md`; anything tagged **[U]** is a hypothesis — your code must preserve the bytes and your tests must not bake the hypothesis in as fact unless your task is the one that verifies it.
4. Stay inside the task's **Outputs**. Unrelated problems you notice go into `STATUS.md` under "Follow-ups", not into your diff.
5. If the task is blocked (needs game files, Blender, a user-supplied sample, a human decision), mark it `blocked` with the reason and stop; do not improvise around the block.
6. Finish by meeting the Definition of Done (§5) and writing the hand-over (§6).

Tasks listed as "Parallel with" each other touch different modules; if you must edit a shared file (`ops/mod.rs`, CLI command table, `docs/formats.md`), keep the edit minimal and append-only to reduce merge conflicts.

## 3. Coding standards

- Rust stable (pinned in `rust-toolchain.toml`), edition 2021. `cargo fmt` defaults. `cargo clippy --workspace --all-targets -- -D warnings` clean; no blanket `#[allow]` at crate level.
- `#![forbid(unsafe_code)]` in `objectizer-core`, `objectizer-cli`, `objectizer-mcp`.
- **Library code never panics on input.** No `unwrap`/`expect`/indexing on data derived from files; use `Reader` and checked arithmetic. `unwrap` is fine in tests.
- Errors: return `objectizer_core::Error` with the right `ErrorCode` and structured details (path, chunk type/id, offset). Non-fatal findings are `Warning`s with stable `W_*` codes.
- `objectizer-core` does no printing, no `std::process::exit`, no reading of env vars except in `install::discover`, no global mutable state. All file writes go through one helper that does temp-file + atomic rename and honours `dry_run`.
- Every operation = `ops::<name>(Request) -> Result<Response, Error>`; request/response derive `Serialize`, `Deserialize`, `JsonSchema`. CLI and MCP only translate and render. No format logic outside `objectizer-core`.
- Typed chunk models keep unknown data (`trailing`, `unknown_words`, raw enum values) and re-serialize to identical bytes when unmodified.
- Determinism: identical inputs → identical outputs on all three OSes (no `HashMap` iteration order in output, no timestamps in files, fixed quantizer tie-breaking, `\n` line endings in generated text).
- Paths: use `Path`/`PathBuf`, never string concatenation; game file names are matched case-insensitively.
- Dependencies: prefer small, well-known crates (`clap`, `serde`, `serde_json`, `schemars`, `thiserror`, `png`, `directories`, `sha2`, `proptest`, `insta`, `rmcp`). New dependencies need a one-line justification in the hand-over. No network access anywhere in the tool.
- Comments explain why, not what. Format facts cite `docs/formats.md` sections (`// formats.md §4.3`).
- Public items in `objectizer-core` have doc comments.

## 3a. Blender scripts and presets (M7)

- Python under `blender/` runs **inside Blender only** (`blender -b … -P`). It contains no format logic and no camera math: it applies the scene spec JSON computed by `objectizer-core` and the look preset. Target the Blender version in `blender/BLENDER_VERSION`; use only `bpy` and the standard library (no pip installs, no add-ons at runtime).
- The driver opens the user's model read-only, never saves it, and writes only under the output directory it was given.
- Models used in tests and examples are built by script at run time. Never commit `.blend`, `.exr` or rendered images.
- `presets/` holds JSON only. A frozen look preset (`maxis-look-v<N>.json`) is immutable: changing the look means running the calibration again and adding `v<N+1>` with its report.
- Reference sprites from the user's install go to the platform cache dir or a user-named directory outside the repository, never into the working tree. Committed calibration data is limited to GUID lists, aggregate statistics and scores.
- Calibration is an agent loop: change preset parameters within the documented bounds → `objectizer calibrate run` → read the JSON report → repeat. Do not hand-tune by eye and do not edit thresholds to make a run pass.
- Vision review: follow the rubric in `docs/authoring.md`, look at every contact sheet, and record the model and verdict in the `vision` block of `review.json`. A vision pass never substitutes for the metric verdict or the in-game gate.

## 4. Running things

```bash
cargo fmt --all --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace                 # unit, synthetic, property, CLI and MCP contract tests — no game files needed
cargo xtask guard-assets               # must pass before every commit
```

With game files (never in CI):

```bash
export OBJECTIZER_GAME_DIR="/path/to/The Sims install"
cargo xtask survey                     # statistics → target/survey/
cargo xtask corpus                     # round-trip gate → target/corpus/report.json, exit 0 = hard gates pass
```

With Blender (never required in CI; M7 only):

```bash
export OBJECTIZER_BLENDER="/path/to/blender"   # optional if blender is on PATH
cargo xtask blender-tests              # T-blender tier; prints a notice and skips if Blender is missing
```

The CLI during development: `cargo run -p objectizer-cli -- <command> --json`.

Snapshot tests use `insta`: review with `cargo insta review`; never accept a snapshot change you can't explain in the hand-over.

## 5. Definition of Done

A task is done when all of these hold at one commit:

1. Every bullet under the task's **Acceptance** is satisfied, or explicitly marked open with the reason (only allowed for criteria needing game files, Blender, or a human).
2. The four commands in §4 (first block) exit 0.
3. New behaviour has tests; every bug fixed has a regression test built from synthetic data.
4. `docs/formats.md` is updated if you verified or corrected a format fact (tag changed, figure cited, open-item index updated).
5. Any change to a public name or JSON shape is reflected in PLAN.md and snapshot tests.
6. `STATUS.md` is updated (`done` + commit hash, or `blocked` + reason).
7. No game assets, no secrets, no absolute paths from your machine in the diff.

## 6. Hand-over note

End your work with a short note (commit message body or PR description):

- Task ID and one-line result.
- Acceptance checklist: each criterion → passed (command + result) / open (why).
- Corpus results if run: game kind, totals, report summary (no file contents).
- Format facts confirmed or corrected.
- New dependencies and why.
- Follow-ups you noticed but did not do.

## 7. Where things are

| Path | What |
|---|---|
| `PLAN.md` | Architecture, names, CLI/MCP surface, bundle format, license, risks |
| `milestones/` | Tasks |
| `docs/formats.md` | File-format reference and open-item index |
| `docs/testing.md` | Test tiers, round-trip levels, gates |
| `docs/survey-findings.md` | Created by M1-T08: measured facts about real files |
| `docs/checklists/` | Created by gate tasks: human in-game checklists |
| `docs/authoring.md`, `docs/look-metrics.md` | Created in M7: modelling rules, new-object recipe, calibration procedure, review rubric, metric definitions |
| `blender/` | Created in M7: Python driver, stand-in and material builders (run inside Blender) |
| `presets/` | Created in M7: JSON look presets, material/stand-in definitions, reference-set list, aggregate reference metrics |
| `examples/` | Created in M7: script-built example models with command lists |
| `STATUS.md` | Created by M1-T01: task status board |
| `crates/` | `objectizer-core`, `objectizer-cli`, `objectizer-mcp`, `objectizer-testkit` (and `objectizer-gui` in M6) |
| `xtask/` | Dev commands |
