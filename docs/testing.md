# Testing strategy

Names of commands, crates, error codes and tasks come from [PLAN.md](../PLAN.md).

## 1. Principles

1. **The user's game files are the oracle.** Formats are only as documented as the files prove them to be.
2. **No Maxis/EA asset ever enters the repository, CI, an issue, a report or a log.** Tests that need real files read them from outside the repo and are skipped when they are absent. Reports contain counts, hashes, offsets and at most 32 bytes of hex context around a mismatch.
3. **Everything an agent must check is a command with an exit code.** Anything that needs a human (running the game) is an explicit, short **[human gate]** task.
4. **Unedited data is byte-identical.** This is tested at every level, on every file the user owns.

## 2. Test tiers

| Tier | Needs game files | Runs in CI | Command |
|---|---|---|---|
| T-unit | no | yes | `cargo test --workspace` |
| T-synthetic (integration on generated fixtures) | no | yes | `cargo test --workspace` (same run) |
| T-property / fuzz-lite | no | yes | `cargo test --workspace` (proptest, bounded cases) |
| T-corpus | **yes** | no (local only) | `cargo xtask corpus` |
| T-survey | **yes** | no | `cargo xtask survey` |
| T-cli (CLI contract) | no | yes | `cargo test -p objectizer-cli` |
| T-mcp (MCP contract) | no | yes | `cargo test -p objectizer-mcp` |
| T-ingame | yes + the game | no | human checklists, §6 |

Static checks, always: `cargo fmt --all --check`, `cargo clippy --workspace --all-targets -- -D warnings`, `cargo xtask guard-assets`.

## 3. Synthetic fixtures

Built in code by `objectizer-testkit`; never stored as binary files in git.

- `testkit::iff::IffBuilder` — builds an IFF byte buffer from chunks; options to add/omit `rsmp`, add duplicate ids, a chunk whose size overruns EOF, non-zero bytes after the label NUL.
- `testkit::far::FarBuilder` — builds FAR1 archives.
- `testkit::spr2` — builds frames from pixel grids, plus hand-assembled command streams covering every row/pixel opcode, odd counts with padding, zero-sized frames, flags 1/3/7.
- `testkit::object` — a complete minimal single-tile object (OBJD + CTSS + BMP_ + DGRP ×1 with 12 images + SPR2 + PALT) and a 2×1 multi-tile object; all pixel art is procedurally generated (gradients, checkerboards).
- `testkit::floor`, `testkit::wall`, `testkit::skin` — added with M4/M5.

Every bug found on real files gets a synthetic reproduction (same structure, generated content) so CI covers it.

## 4. The round-trip harness (key gate)

Implemented by `objectizer_core::ops::roundtrip`, exposed as `objectizer roundtrip` and driven over the whole install by `cargo xtask corpus`.

Input set: every `*.far` (recursively, including IFFs inside), every loose `*.iff`, `*.flr`, `*.wll` under the game dir, plus `Downloads`.

| Level | What is checked | Requirement |
|---|---|---|
| **L0 — container** | `write(parse(file)) == file` for every IFF and every FAR, with no edits | 100% of files, hard gate (M1) |
| **L1 — typed chunks** | For every chunk type objectizer has a typed model for: `serialize(decode(payload)) == payload` | 100% of chunks of supported versions, hard gate (per type, in the task that adds the type). Unsupported versions must be reported as `E_UNSUPPORTED`, not mis-parsed |
| **L2 — SPR2 frame re-encode, bytes** | For every SPR2 frame: `encode(decode(frame_bytes)) == frame_bytes` | Tracked target ≥ 99% of frames. Every mismatch is assigned to a named class in the report; the M1 gate (M1-T13) requires that no class is "unexplained" |
| **L3 — SPR2 frame re-encode, pixels** | `decode(encode(decode(frame))) == decode(frame)` for colour index, z and alpha planes plus header fields | 100%, hard gate (M1) |
| **L4 — bundle** | `export` then `import` with untouched files leaves the IFF byte-identical | 100% of objects, hard gate (M2); extended to multi-tile (M3), floors/walls (M4) |
| **L5 — forced re-encode** | Like L4, but import is told to treat every frame as changed (`--reencode-all`, test-only flag): result must be L3-equal, and re-exporting gives identical PNGs | 100%, hard gate (M2) |

Why L2 is a target and not a hard gate: Maxis may have used more than one encoder across seven expansion packs, and some frames may contain garbage. The lossless data model (PLAN.md §3.3) means L2 misses never reach users, because unedited frames are copied, not re-encoded. L2 still matters: it is the evidence that the encoder produces data in the shape the game has always been given.

Report format (`--report <path>`, JSON, safe to share):

```json
{
  "schema": 1,
  "game": { "kind": "legacy", "root_sha": "…" },
  "totals": { "files": 0, "chunks": 0, "spr2_frames": 0 },
  "levels": {
    "L0": { "pass": 0, "fail": 0 },
    "L1": { "by_type": { "OBJD": { "pass": 0, "fail": 0, "unsupported": 0 } } },
    "L2": { "pass": 0, "fail": 0, "classes": { "trailing-transparent-run": 0, "pad-byte-value": 0, "unexplained": 0 } },
    "L3": { "pass": 0, "fail": 0 }
  },
  "failures": [ { "level": "L3", "file": "ExpansionPack3/…::chair.iff", "chunk": "SPR2#256", "frame": 4, "offset": 1234, "class": "…" } ]
}
```

Exit code: 0 if all hard-gate levels requested pass, else 5 (`E_VALIDATION`). File paths in reports are relative to the game dir.

`cargo xtask corpus` = `objectizer roundtrip <game-dir> --level L0,L1,L2,L3 --report target/corpus/report.json` plus, from M2 on, the L4/L5 passes; it prints a one-line summary per level and exits non-zero on any hard-gate failure. If `OBJECTIZER_GAME_DIR` is unset and discovery finds nothing, it exits 3 with a message; it never silently passes.

## 5. Corpus survey

`cargo xtask survey` (M1-T08) walks the same input set and writes `target/survey/survey.json` and a human summary, answering the open questions in docs/formats.md §14: chunk-type histogram, versions per chunk type, SPR2 opcode statistics, pad-byte values, row-ending behaviour, rsmp presence/ordering, FAR layout, palette sharing, etc. The summary (numbers only) is committed as `docs/survey-findings.md`; the raw JSON is not.

Run it on **both** game versions where available; differences between Legacy and Complete Collection are a finding in themselves.

## 6. Testing per game version

| | Complete Collection | Legacy Collection |
|---|---|---|
| Automated corpus tests on macOS | Point `OBJECTIZER_GAME_DIR` at a copy of the install folder (from a Windows machine, a Wine/CrossOver bottle, or the original discs' install) | Point it at the Steam install folder inside a CrossOver/Whisky bottle or a copy from a Windows machine. The game does not need to run for this. |
| In-game check | Wine/CrossOver on macOS, or Windows VM/PC | Steam in CrossOver (Vulkan via MoltenVK — may or may not work on a given Mac) or a Windows VM/PC; Steam Deck also works |
| Secondary renderer | Simitone pointed at the install (macOS/Linux via its own build) | same |

In-game checks cannot be automated and are kept short. Each **[human gate]** task has a checklist file under `docs/checklists/` created by that task, with this shape:

1. Produce the artefact with exact commands (given in the task).
2. Copy to the path printed by `objectizer locate`.
3. Start the game, open Buy/Build mode, category X.
4. Tick: appears in catalog with the new name/price/thumbnail; places on a lot; all 4 rotations and 3 zooms look right; no purple/garbled pixels; original object still present and unchanged; save, reload, object persists.
5. Record game kind, version/patch, platform, pass/fail and screenshots (screenshots stay out of git).

Simitone is used the same way but is only supporting evidence: it is more forgiving than the real game (it ignores `rsmp`, tolerates more encodings), so a pass in Simitone never replaces a pass in the game.

## 7. CLI and MCP contract tests

- Snapshot tests (`insta`) of `--json` output for every command on synthetic fixtures; snapshots are the contract. Changing one requires bumping `schema` or proving the change is additive.
- Exit-code table tests: one test per `ErrorCode` proving the mapping in PLAN.md §3.4.
- `--dry-run` tests: run each mutating command with `--dry-run` in a temp dir and assert that no file was created or modified (directory hash before == after) and that `data.writes` lists the intended outputs.
- MCP: start `objectizer-mcp` as a child process, perform `initialize`, `tools/list`, and one `tools/call` per tool; assert the tool list equals PLAN.md §5 and that each result equals the CLI's `--json` `data` for the same request.

## 8. Robustness

- Property tests (proptest): random pixel grids → `encode` → `decode` equals input (colour, z, alpha) for all flag combinations; random chunk lists → IFF write → parse equals input; random string tables; random OBJD word arrays.
- Mutation tests on parsers: truncate and bit-flip synthetic files; parsers must return `E_PARSE` (or succeed), never panic, never allocate more than a bounded multiple of the input size. One `cargo fuzz` target per parser is optional and not a gate.
- All length/offset arithmetic is checked; `#![forbid(unsafe_code)]` in `objectizer-core`.

## 9. CI

GitHub Actions, matrix `ubuntu-latest`, `macos-latest`, `windows-latest`, stable Rust:

1. `cargo fmt --all --check`
2. `cargo clippy --workspace --all-targets -- -D warnings`
3. `cargo test --workspace`
4. `cargo xtask guard-assets`

`guard-assets` fails if any tracked file (a) has an extension in `.iff .far .flr .wll .bmp .cmx .skn .bcf .bmf .cfp .tga .spf .hot .xa .wav`, or (b) contains the byte strings `IFF FILE 2.` or `FAR!byAZ` at offset 0, or (c) is a binary file larger than 256 KiB outside an allow-list. Test code builds those magic strings from parts at runtime so the source does not trip the guard.

Corpus tiers never run in CI. A milestone gate task records the local corpus result (the JSON report's totals and tool commit hash) in the milestone file's "Gate record" section.

## 10. Definition of "verified" for a task

A task's acceptance criteria are met only when, at the task's commit: all static checks pass; `cargo test --workspace` passes; and, if the task lists corpus criteria, `cargo xtask corpus` (or the stated narrower command) passes on at least one real install, with the result recorded. If the implementing agent has no access to game files, it must say so in the hand-over and leave the corpus criterion **open** — never claim it.
