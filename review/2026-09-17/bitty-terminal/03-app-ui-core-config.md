# Bitty Terminal — Read-Only Review: App / UI / Core / Config (03)

- **Date:** 2026-09-17
- **Scope:** `bitty/crates/bitty-app`, `bitty/crates/bitty-ui`, `bitty/crates/bitty-core`, `bitty/crates/bitty-config`; runtime interaction points in `bitty-runtime`/`bitty-render` where config and chrome actions propagate. Intended contracts checked against `bitty/docs/specifications/configuration-model-rfc.md`, `bitty/docs/configuration/lua-and-xdg.md`, `bitty/docs/specifications/workspace-compositor.md`.
- **Mode:** Strictly read-only, static analysis. No source changes, no builds, no tests executed, no commits, no nested agents. Existing targeted tests were consulted as evidence, not run.
- **Report series:** sibling `01-terminal-engine.md` covers the terminal-engine crates; this report covers the application/UI/config surface only.

## 1. Baseline

- **bitty repo:** HEAD `06bc1f45995fd297a3f7324688bc81b0598ecf67`, branch `main`. Working tree clean except untracked `.targets/` (verified via `git status --porcelain`; no staged/unstaged diffs). Umbrella directory `bitty-terminal/` is not a git repo.
- **research repo:** HEAD `d70152a079c75204ec37e99a7bf3f54b8190e423`; untracked `review/2026-09-17/` contains sibling `01-terminal-engine.md` (untouched).
- All findings below are pinned to `06bc1f4`.

## 2. Coverage Ledger

| Area                                                                                                                                                                             | Coverage                                                                                                                                                          |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `bitty-core/src/lib.rs`                                                                                                                                                          | Full (1-line facade: "Compilation target for the pre-implementation Bitty workspace", no executable logic; dependency metadata not rechecked in the second pass). |
| `bitty-app`: `main.rs`, `terminal_app.rs`, `chrome_keys.rs`, `config_cli.rs`, `spawn.rs`, `run.rs`, `cli.rs`, `logging.rs`, `layout_cmd.rs`, `ipc_serve.rs`, `plugin_runtime.rs` | Full or near-full read.                                                                                                                                           |
| `bitty-app/src/ctl/` (apply, dispatch, tests)                                                                                                                                    | Full read of apply/dispatch paths; tests sampled (incl. `tests.rs:1497-1498` reload assertion).                                                                   |
| `bitty-ui` (layout, view, focus, geometry, decoration, scrollbar, selection, search, panel, presentation)                                                                        | Sampled reads; render-only code skimmed via outlines.                                                                                                             |
| `bitty-config` (file, merge, types, reload, trust, keymap, theme, migration, plan, validation)                                                                                   | Sampled reads; `error.rs`/`lib.rs` outline-only; large bodies of `types.rs`/`merge.rs`/`file.rs` unread.                                                          |
| `bitty-runtime` / `bitty-render`                                                                                                                                                 | Targeted reads only (resize, workspaces, layout_focus, search, grid, hidpi) to validate cross-crate claims.                                                       |
| Test bodies (`bitty-app`, `bitty-config`)                                                                                                                                        | Unread except where cited as defect evidence.                                                                                                                     |
| `bitty-core`                                                                                                                                                                     | N/A beyond the facade — no logic to review.                                                                                                                       |

Method: `ctxctl outline` per file, `ctxctl read` ranges for suspect regions, `grep` for cross-references. No dynamic execution.

## 3. Defects

### TERM-APP-001 (reclassified P2, static response-contract defect) — reload probe reports success without applying config

- **Where:** `bitty/crates/bitty-app/src/ctl/apply.rs:502-513`
- **Cause:** The `METHOD_RELOAD_CONFIG` handler only calls `bitty_config::file::probe_config_path(None)` and immediately returns `{"reloaded": true, ..., "hot_swap": "follow-up"}`. It never loads, merges, or validates the configuration, and never applies anything to the live runtime.
- **Impact / severity correction:** The `reloaded: true` field misleadingly suggests application, but the same response explicitly says `hot_swap: follow-up`, and the code documents a deferred implementation. Retain P2 response-contract/config-path correctness, not P1 outage, data loss, or a demonstrated security-policy reload failure. `probe_config_path(None)` reads only XDG/HOME (`bitty/crates/bitty-config/src/file.rs:224-270`), unlike non-safe startup's CLI/environment resolution (`bitty/crates/bitty-app/src/config_cli.rs:136-153`). A wrong reported path is established; client interpretation and operational impact were not measured. The original canonical-doc classification comparison was not independently rechecked in the second pass.
- **Enabling test:** `bitty/crates/bitty-app/src/ctl/tests.rs:1497-1498` (`control_elevated_close_reports_not_found_not_denied`) asserts the elevated reload "must succeed", locking the false-success semantics into the suite.
- **Fix:** Until reload is implemented, return an explicit unsupported/probe-only result without `reloaded: true`. For implementation, retain the effective startup context (including safe mode, explicit path, profiles, and overrides), validate/classify changes, and report applied/deferred/rejected outcomes. Do not newly read external config in a safe-mode session.
- **Regression test idea:** Headless ctl test that changes a `Hot` field in a temp config, invokes `reload-config` with `BITTY_CONFIG` pointing at it, and asserts the runtime value changed and the response reports the applied field; plus a negative test that a malformed config yields `rejected`, not success.

### TERM-APP-002 (P2) — Zoom backup is not workspace-scoped; stale restore clobbers another workspace's layout

- **Where:** `bitty/crates/bitty-app/src/chrome_keys.rs:834-862` (set site: `ToggleZoom` backs up the whole layout and swaps in a single leaf), `chrome_keys.rs:484-492` (`restore_zoom` unconditionally `runtime.set_layout(backup)`), `chrome_keys.rs:641-644` (close-view consults the backup).
- **Cause:** `zoom_backup: Option<LayoutNode>` (`terminal_app.rs:107`) is a single slot on `TerminalApp`, not keyed by workspace id. Workspace switching (`WorkspaceFocus`/`WorkspacePrev`/`WorkspaceNext`/`WorkspaceLast`, `chrome_keys.rs:777-812`; verified against `bitty-runtime` workspaces.rs) does not invalidate or re-scope the backup.
- **Impact:** A backup taken in one workspace can replace the active layout in another. Correction: the next ToggleZoom with an existing backup takes the restore branch; it does not first capture `layout-B`. `bitty/crates/bitty-runtime/src/runtime/layout_focus.rs:766-828` unconditionally installs the supplied layout, and `runtime/workspaces.rs:295-318,400-414` switches the active layout independently. This establishes wrong-workspace layout replacement, not process termination or durable data loss.
- **Fix:** Key the backup per workspace (`HashMap<WorkspaceId, LayoutNode>` on the runtime or app), clear it on workspace switch/close, and make `restore_zoom` a no-op (with a warning) when the backup's workspace is not active. Alternatively store the zoom backup inside the runtime's workspace state so `set_layout` ownership rules (CTX-0359) apply.
- **Regression test idea:** Headless test: create two workspaces, zoom in ws1, switch to ws2, invoke `ToggleZoom`/a tree mutation, assert ws2's leaf set is unchanged and the restore is refused or scoped.

### TERM-APP-003 (reclassified P3 maintenance, not a confirmed functional defect) — duplicated fallback assembly

- **Where:** `bitty/crates/bitty-app/src/main.rs:608-679` (fallback block), duplicating `main.rs:506-546` (primary spawn path).
- **Cause:** On `PlatformError::DisplayUnavailable` the fallback re-runs `runtime_config_from_effective` → `Runtime::new` → `build_layout`/`apply_focus` → a re-created `SpawnSpec` (`main.rs:653-658`) → primary spawn → `spawn_startup_pane_shells` → `run_headless_smoke`. This is a hand-copied fork of the startup sequence rather than a shared function; it already drifted in shape (its own `fallback_spec`, separate `is_ok()` handling) and must be edited in lockstep with the primary path for every future startup change (env handling, arg0 policy, pane spawn rules).
- **Impact:** A future fix could be applied inconsistently to these duplicate paths. The duplication remains, but no current behavioral divergence or failing CI contract was established; the prior APP-01 P1 priority is not retained.
- **Fix:** Extract one `bootstrap_session(&mut runtime, &args, &app_config) -> BootstrapOutcome` used by both the real-mode path and the fallback, parameterized only by the runtime instance.
- **Regression test idea:** A source-level unit test is fragile; instead, add a headless integration test that forces `DisplayUnavailable` and asserts the spawned leaf/pane set matches the real-mode startup fixture (same shell resolution, same pane count).

### TERM-APP-004 (reclassified P3 test gap, not an independent product defect) — reload assertion checks status only

- **Where:** `bitty/crates/bitty-app/src/ctl/tests.rs:1487-1499`.
- The authorization-focused test checks only `done.ok`, not effective configuration. It can remain green after a correct successful reload implementation; the earlier claim that it necessarily blocks such a fix is withdrawn. An explicit unsupported response would require updating this expectation.
- Add effective-state and failure-outcome coverage when reload is implemented, while preserving the surrounding authorization test. This is a companion coverage follow-up to TERM-APP-001, not a second functional defect.

### TERM-APP-005 (reclassified maintenance/scaffold, no defect priority) — empty core facade

- **Where:** `bitty/crates/bitty-core/src/lib.rs:1` contains a single pre-implementation doc line.
- No executable logic or defect exists in that file. The "to be retired" wording is not present there; publication and external consumer usage were not independently established.
- Evaluate removal through dependency and compatibility review, not an unconditional deletion or `compile_error!` recommendation.

## 4. Reconciliation With Prior Reviews (2026-09-15)

| Prior finding                                                                  | Status at HEAD `06bc1f4`                                                                                                                                                      |
| ------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| APP-01 (P1) fallback path forks the startup assembly                           | **Still present** — see TERM-APP-003 (`main.rs:622-679`).                                                                                                                     |
| CTX-0187 focus-latch                                                           | **Implemented** — `resolve_priority_for` (`chrome_keys.rs:436-479`) shows the modal/priority table with dedicated close-confirm arms.                                         |
| CTX-0382 title sanitize                                                        | **Implemented** — title sanitization present on the presentation path (verified via runtime title handlers; no raw-escape passthrough found in sampled code).                 |
| CTX-0370 close-confirm gate                                                    | **Implemented** — `ViewCloseRequest::Pending` flow at `chrome_keys.rs:654-660`, modal confirm logic at `chrome_keys.rs:453-467`.                                              |
| PTY/config-layer P1-1 (config Drop blocking, from `02-pty-platform-config.md`) | **Out of scope here** (bitty-runtime/bitty-config Drop path); not re-verified in this pass beyond confirming `bitty-config` reload-classification code is unchanged in shape. |

## 5. Optimization Candidates

1. **`restore_zoom` + `CloseView` double layout clone** (`chrome_keys.rs:661-667`): `restore_zoom()` does `set_layout(backup)`, then the close path clones the layout again and calls `set_layout_closing`. One fused operation on the runtime (restore-and-mutate) would avoid two full `LayoutNode` traversals per close. Complexity: current path is O(n) twice per close, fused is O(n) once (n = leafs).
2. **`spawn_startup_pane_shells` spec cloning** (`main.rs:506-511` vs `653-658`): `SpawnSpec` re-created with `String` clones of `$SHELL` and config shell; passing references into the resolver avoids the clones and, per TERM-APP-003, deduplicates the logic itself.
3. **Per-tick help-row regeneration** (`chrome_keys.rs:864-874`): `help_rows_from_keymaps` recomputes on every overlay show; caching keyed on keymap generation would make popup open O(1). Minor — popup frequency is low.
4. **`zoom_backup` as `Option<LayoutNode>` full-tree clone** (`chrome_keys.rs:851`): zooming a 50-pane workspace clones the entire tree. A structural share (e.g. `Rc`/`Arc` interior or persistent tree) would reduce the clone to O(1) with O(n) only on restore — same asymptotics, better constant and allocation profile.

## 6. Gaps (what this review could not establish)

- **No dynamic verification.** All defect claims are static; none were reproduced by running binaries or tests (read-only mandate). TERM-APP-002 in particular is argued from code paths (workspace switch does not touch `zoom_backup` — confirmed by grep: only set sites are `chrome_keys.rs:853` and take sites `485`/`834`), not by an executing repro.
- **Unread bodies.** `bitty-config` `types.rs`/`merge.rs`/`file.rs` large bodies, `bitty-ui` search/panel internals beyond sampling, and nearly all test bodies were not fully read; defects could exist there, especially in merge precedence corner cases.
- **`bitty-core`** contains no logic; "review" reduces to confirming it is an empty facade (TERM-APP-005).
- **UI rendering paths** (bitty-ui presentation, geometry, hidpi in bitty-render) were skimmed, not line-audited; visual/interaction bugs are out of reach of static sampling.
- **Plugin runtime** (`plugin_runtime.rs` read fully, but host-service behavior in `bitty-plugin-host` is outside this slice's crates) — cross-crate plugin lifecycle defects may exist unexamined.
- **Docs submodule commit** for `bitty/docs` was not pinned in this session's notes; doc quotes are cited by path/line at review time, not by submodule SHA.
- **Prior CTX-### fix verification** (focus-latch, title sanitize) is spot-check based, not exhaustive re-derivation.

## 7. Checks Performed

- Both repo baselines captured before and re-verified after analysis (`bitty` HEAD `06bc1f4…`, clean except untracked `.targets/`; research HEAD `d70152a…`) — no source modifications made or observed.
- All cited `path:line` references re-read at HEAD before inclusion (notably `main.rs:608-679`, `chrome_keys.rs:484-492/834-862`, `ctl/apply.rs:502-513`, `ctl/tests.rs:1497-1498`).
- `zoom_backup` lifecycle exhaustively grepped (7 hits, all accounted for above); workspace-switch handlers confirmed not to clear it.
- `bitty-config` reload classification (`reload.rs`) cross-checked against the documented per-field inventory (`lua-and-xdg.md:741-770`) — they match; the mismatch is the app-side handler (TERM-APP-001), not the schema.
- Markdownlint: report linted with `markdownlint-cli2` against `research/.markdownlint-cli2.jsonc`; second-pass validation is recorded in [the product index](README.md).

## 8. Independent second-pass disposition

- Source HEAD remains `06bc1f45995fd297a3f7324688bc81b0598ecf67`; no product tests or source edits in this pass. Original baseline/coverage/check statements describe the first pass, not newly repeated exhaustive work.
- Independently verified product mechanisms: TERM-APP-001 (P2, misleading probe receipt, explicit hot-swap scaffold) and TERM-APP-002 (P2, cross-workspace restore). TERM-APP-003/004/005 were checked and reclassified as maintenance/test/scaffold, not additional functional defects.
- Exact body sample: app `ctl/apply.rs:502-520`, `ctl/tests.rs:1487-1502`, `chrome_keys.rs:484-492,777-862`, `config_cli.rs:117-153`, `main.rs:506-546,608-679`; config `file.rs:224-270`; core `lib.rs:1`; runtime `layout_focus.rs:766-828` and `workspaces.rs:295-318,400-414`. Outlines preceded reads; `zoom_backup` references were searched. Other original citations and canonical-doc comparisons were not independently renewed.
- Startup duplication is established, but a present semantic mismatch from it was not. Prefer parity tests before refactoring. Clearing a zoom backup without restoring its owning workspace can lose the original layout; any fix must preserve workspace ownership rather than merely discard the backup.
- `bitty-ui`, most configuration merge/validation bodies, prior CTX closure claims, real GUI behavior, and non-Linux behavior remain outside this second-pass sample.
