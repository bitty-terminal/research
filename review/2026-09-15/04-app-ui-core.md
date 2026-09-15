# Bitty app-shell review: bitty-app / bitty-ui / bitty-core

- Scope:
  - `bitty/crates/bitty-app/` (~35 rs files, app entry and assembly)
  - `bitty/crates/bitty-ui/` (11 rs files, UI layer)
  - `bitty/crates/bitty-core/` (1 rs file, core facade)
- Method: Read-only review, no source changes.
  - Read `Cargo.toml` (each crate's duties and dependency declarations) and the entries `bitty/crates/bitty-app/src/main.rs::main`, `bitty/crates/bitty-ui/src/lib.rs`, `bitty/crates/bitty-core/src/lib.rs`.
  - Traced the startup flow: `cli::parse_args` → subcommand dispatch (`run`/`config`/`init`/`doctor`/`ctl`/`list`/`inspect`/`dev`/`plugin` ahead of config loading) → `config_cli::load_merged_config`/`load_app_config` → `resolve_keymaps` → `runtime_config_from_effective` → `Runtime::new` → `layout_cmd::build_layout`/`set_layout`/`apply_focus` → `plugin_runtime::discover_and_activate` → `spawn::resolve_spawn_program`/`spawn_default_shell`/`spawn_startup_pane_shells` → `--headless` chimney vs `platform::App::run(TerminalApp)` → `DisplayUnavailable` fallback rebuild.
  - UI side: read `layout.rs::LayoutNode`, `view.rs::View`, `focus.rs::Focus`, `geometry.rs::Rect`, `decoration.rs::Decoration`, `scrollbar.rs`, `selection.rs`, `search.rs::SearchState`, `panel.rs::OverlayManager`/`route_input`, `presentation.rs::PresentationMode`, cross-checked against input routing in `terminal_app.rs::TerminalApp::handle_event`/`intercept_chrome_key` (in `chrome_keys.rs`).
  - Focus: whether startup failure paths are fail-closed with usable messages, component duty separation, whether state should move up or down, whether the facade leaks internals.

## Defect list

### APP-01 (P1) Fallback path copies the whole startup assembly, easy to fork

- Location: `bitty/crates/bitty-app/src/main.rs::main` (`DisplayUnavailable` fallback block, ~60 lines)
- Symptom: Layout construction, `apply_focus`, main-shell spawn, and per-leaf shell spawn from the normal path are written out again in the fallback block; the fallback re-reads `SHELL`/config, rebuilds `Runtime`, and re-runs `build_layout`.
- Trigger: No display server → `App::run` → `DisplayUnavailable` → fallback; whenever either path later changes spawn rules, focus semantics, or pane sizing, the other path is easily missed.
- Evidence: Main path `build_layout(&args...)` + `apply_focus` + `resolve_spawn_program` + `spawn_default_shell` + `spawn_startup_pane_shells`; the fallback block repeats the same five steps with only `fallback_`-prefixed variable names. `_plugin_runtime` remains the product activated under the old `Runtime`'s dimensions; the fallback's new `Runtime` never re-discovers/reactivates plugins, and the comment admits event delivery is a later slice.

### APP-02 (P1) CLI error policy is split: same user typo exits 2 on one half, warning-ignored on the other

- Location: `bitty/crates/bitty-app/src/cli.rs::parse_args`, `bitty/crates/bitty-app/src/layout_cmd.rs::build_layout`, `bitty/crates/bitty-app/src/layout_cmd.rs::apply_focus`
- Symptom: Unknown `--flag` (`unknown_flag`) and illegal `--font-size`/`--opacity` values fail closed (exit 2 with usage); but `--split-ratio abc`, `--split garbage`, `--layout typo`, `--focus typo`, `--log-level typo` only `eprintln!("warning: ... — ignoring")` and continue with defaults.
- Trigger: `bitty --split-ratio abc` silently becomes 0.5; `bitty --layout split:zzz:NaN` silently becomes horizontal 0.5; `bitty --focus foo` silently ignored; users cannot tell "took effect" from "was discarded" via the exit code.
- Evidence: In `cli.rs` the `--split-ratio=`/`--split=` illegal branches are warnings; `layout_cmd.rs::build_layout` only warns `warning: unknown --layout spec ... — falling back` on a fully unknown spec; `apply_focus` returns `false` + warning on unknown specs; contrast with `unknown_flag → exit 2` in `main.rs`.

### APP-03 (P1) `parse_layout_spec` silently normalizes; error messages unusable

- Location: `bitty/crates/bitty-app/src/layout_cmd.rs::parse_layout_spec`
- Symptom: `split:` axis parse failure defaults to horizontal, ratio parse failure defaults to 0.5; `stack:` count parse failure defaults to 2 with `clamp(1,8)`; `overlay:` quadruple parse failure falls back to the default overlay; the whole function returns `Some`, so callers cannot tell users which segment was illegal.
- Trigger: `--layout split:foo:xyz`, `--layout stack:9999`, `--layout overlay:1,2` never error — the first two silently take defaults, `stack:9999` silently becomes 8.
- Evidence: Multiple `unwrap_or(0.5)`, `unwrap_or(SplitAxis::Horizontal)`, `unwrap_or(2).clamp(1,8)` in the function, and the overlay-parse-failure comment `fallback to default overlay on parse failure`.

### APP-04 (P2) Split ratio is never clamped or echoed at the assembly layer, relying on downstream fallback

- Location: `bitty/crates/bitty-app/src/layout_cmd.rs::build_layout`, `bitty/crates/bitty-app/src/cli.rs::parse_args`
- Symptom: `Args::split_ratio: Option<f32>` may be NaN/negative/>1/inf and is passed straight into `LayoutNode::split`, relying on `bitty-ui::clamp_ratio` (`[0.10,0.90]`, non-finite → 0.5) as the backstop; a user passing `--split-ratio 99` is silently clamped to 0.9 with no hint.
- Trigger: Any out-of-range/NaN ratio; same for `--split h:NaN`.
- Evidence: In `build_layout`, `let ratio = args.split_ratio.unwrap_or(0.5)` directly constructs the split; `cli.rs` only warns on parse failure with no check when parsing succeeds but the value is out of range; the true clamp point is `MIN_RATIO`/`MAX_RATIO` on the `bitty-ui` side.

### APP-05 (P1) Spawn failure is never fatal; `--headless` green can mask a fully broken shell chain

- Location: `bitty/crates/bitty-app/src/main.rs::main`, `bitty/crates/bitty-app/src/spawn.rs::spawn_default_shell`/`spawn_with_fallback`
- Symptom: Main spawn failure only does `eprintln!("bitty: PTY spawn failed: ... — continuing without child")`, and then `run_headless_smoke` still returns 0 on synthetic bytes; CI only runs this path, so a misconfigured `terminal.shell`, missing `$SHELL`, or missing `/bin/sh` never turns red.
- Trigger: `terminal.shell` points at a non-executable file + fallback `/bin/sh` also unavailable; `--headless` still prints `ok — tick presented` and exits 0.
- Evidence: Comment in `spawn_default_shell`: `Callers log and continue without a child on error so headless smoke still ticks`; two `spawn_startup_pane_shells` comments in `main.rs`: `Best-effort with loud warnings (spawn failures stay non-fatal, startup parity)`.

### APP-06 (P2) Startup `eprintln` bypasses the verbosity gate; quiet-by-default does not hold

- Location: `bitty/crates/bitty-app/src/main.rs::main`, `bitty/crates/bitty-app/src/terminal_app.rs::TerminalApp::set_log_level`, `bitty/crates/bitty-app/src/logging.rs::effective_log_level`
- Symptom: Startup summaries for `theme/keymaps/layout/spawn` unconditionally `eprintln` before `set_log_level`, so `--log-level error` still floods; only the per-frame `bitty tick` is gated by `LogLevel::tick_enabled`.
- Trigger: Any normal startup + `--log-level error`; only errors expected, yet several `bitty: theme ...`/`bitty: layout installed ...` lines still appear.
- Evidence: Three consecutive `eprintln!` blocks after `load_app_config` in `main.rs`, while `app.set_log_level(effective_log_level(&args))` is not called until just before `App::run`; `logging.rs` comments claim the default `Warn` is quiet, but the startup summaries sit outside the gate.

### APP-07 (P2) `RUST_LOG`/`BITTY_LOG` substring matching misfires

- Location: `bitty/crates/bitty-app/src/logging.rs::log_level_from_env_value`
- Symptom: Scans the whole filter string with `contains("trace"/"debug"/...)`, most-verbose wins; `RUST_LOG=mydebugger=info` contains the `debug` substring and is judged `Debug`; `RUST_LOG=off` matches nothing and falls back to default instead of honoring `off`.
- Trigger: Filter string containing one of the above substrings as part of another word, or using `off`/`none` expecting silence.
- Evidence: Five chained `if lower.contains(...)` in the function, with no word boundaries and no `off` handling; `effective_log_level` priority comments claim to reuse `RUST_LOG` conventions, but the implementation is a substring heuristic.

### APP-08 (P1) `TerminalApp` state bloat; layout surgery lives on the wrong layer

- Location: `bitty/crates/bitty-app/src/terminal_app.rs::TerminalApp`, `bitty/crates/bitty-app/src/chrome_keys.rs::split_focused_leaf`/`close_focused_leaf`/`resize_focused_pane`/`apply_chrome_action`
- Symptom: `TerminalApp` has 14 fields spanning the composition root, input routing, layout backup, log gate, title dedup, and IME sync; the ~2800-line `chrome_keys.rs` re-implements split/close/resize/workspace tree surgery at the app layer, while `bitty-ui::LayoutNode` should own those invariants.
- Trigger: Any new layout gesture must touch the app's backup/focus/spawn trio together; `zoom_backup` coupled with `spawn_spec` and `chrome_held` forces unit tests to construct a full `TerminalApp`.
- Evidence: `TerminalApp` fields `runtime/window/pty_rx/keymaps/app_mods/chrome_held/zoom_backup/spawn_spec/log_level/presented_frames/last_applied_title/title_applies/ime_cursor_area/window_opacity`; `chrome_keys.rs` header comment states `Pure-move extraction from main.rs ... No behavior change`, i.e. the surgery logic was only relocated, not sunk into ui/runtime.

### APP-09 (P1) Single-slot `zoom_backup` races external layout changes; restore can drop panes

- Location: `bitty/crates/bitty-app/src/chrome_keys.rs::TerminalApp::restore_zoom`, `bitty/crates/bitty-app/src/terminal_app.rs::TerminalApp::drive_tick`
- Symptom: Zoom backs up the whole tree into `Option<LayoutNode>`; `drive_tick` first `drain_global_control_queue` every frame (`ctl` may change layout), then ticks. If `ctl split/close` lands during zoom, `toggle_zoom off` overwrites the new tree with the stale backup, dropping `ctl`'s panes.
- Trigger: `toggle_zoom on` → `bitty ctl split/...` → `toggle_zoom off`; some chrome actions (`NewSplit`/`CloseView`/`Resize`/`WorkspaceMove`) `restore_zoom` first, but `WorkspaceNew/Prev/Next` and the `ctl` path do not restore — inconsistent window.
- Evidence: `restore_zoom` is an unconditional `set_layout(backup)`; only four branches of `apply_chrome_action` call it; the `drive_tick` comment admits `ctl` applies changes on the main thread's sole `Runtime` owner but says nothing about version-checking against `zoom_backup`.

### APP-10 (P1) Panel modality and chrome dispatch disagree; overlay capture missing

- Location: `bitty/crates/bitty-ui/src/panel.rs::OverlayManager::modal_active`/`route_input`, `bitty/crates/bitty-app/src/chrome_keys.rs::TerminalApp::modal_capture_active`/`resolve_dispatch`
- Symptom: The UI side has a 4+1 overlay manager plus `route_input(PanelFocus, ViewFocus)`, while the app-side modal predicate only checks `has_pending_paste/has_pending_ws_close/has_pending_close_confirm`; the comment plainly says panel overlays feed the same predicate only once wired into key dispatch — i.e. today panel modality captures no bound chord at all.
- Trigger: While a panel overlay declares modal and is shown, user chrome shortcuts still execute behind it (layout/workspace changes), contradicting the `Modal → swallowed` design doc.
- Evidence: Routing comment of `panel.rs::modal_active` and `route_input`: `Platform -> Router -> focused Panel/View -> keymap`; the `chrome_keys.rs::modal_capture_active` docs list only the two pending-confirm gates, so the `Modal` arm of `resolve_priority_for` is unreachable for panel modality.

### UI-01 (P2) Two overlay/presentation systems coexist with unclear ownership

- Location: `bitty/crates/bitty-ui/src/layout.rs::LayoutNode::Overlay`/`overlay_stack`/`OverlayTier`, `bitty/crates/bitty-ui/src/panel.rs::Overlay`/`OverlayManager`, `bitty/crates/bitty-ui/src/presentation.rs::PresentationMode`
- Symptom: `LayoutNode::Overlay{base,overlay,bounds}` participates in the solver and affects allocation; `OverlayManager` declares presentation-only, never touching the grid; `PresentationMode::{Floating,Fullscreen,Scratchpad}` parses but `can_transition` rejects everything (only identity allowed) and the solver ignores the field (unit test locks byte-identical). All three define "float/fullscreen/scratchpad" with overlapping meanings but no connection.
- Trigger: Implementing either float or fullscreen semantics later requires touching all three, or else a half-wired state of "parseable but not enterable" or "entered but layout-agnostic" appears.
- Evidence: `presentation.rs` module docs: `Only Tiled is live ... transition-gated`; overlay solving in `layout.rs` and 4+1 management in `panel.rs` each have independent tests; `View::presentation` defaults to `Tiled` and `layout_allocations_ignore_presentation_mode` explicitly asserts it is ignored.

### CORE-01 (P2) `bitty-core` is an empty shell under a facade name; declared dependency graph differs from reality

- Location: `bitty/crates/bitty-core/src/lib.rs`, `bitty/crates/bitty-core/Cargo.toml`, `bitty/crates/bitty-app/Cargo.toml`, `bitty/crates/bitty-app/src/main.rs`
- Symptom: `bitty-core` is a single doc line, self-described as `Bootstrap seed (to be retired)` with no re-exports; `bitty-app` calls itself a thin composition root and its docs claim it `depends on bitty-runtime only`, while actually wiring 8 libraries directly (config/ipc/perf/platform/plugin-host/runtime/render/term-state) through no facade at all.
- Trigger: Any cross-crate refactor: callers reach past the facade straight into internals, so the facade can neither converge the API nor isolate changes.
- Evidence: `bitty-core/src/lib.rs` is one line in full; `bitty-app/Cargo.toml` lists 8 workspace path dependencies; the `main.rs` header doc `depends on bitty-runtime only` vs actual `use bitty_platform/...; use bitty_runtime::Runtime` plus `bitty_config`/`bitty_ipc` references.

### CFG-01 (P2) `runtime_config_from_effective` mixes clamping with fail-closed silently

- Location: `bitty/crates/bitty-app/src/config_cli.rs::runtime_config_from_effective`
- Symptom: `gaps_in/out`, `padding`, `radius`, `scrollbar.width`, focus delay, outline width, and `decoration` out-of-range silently clamp; `terminal.scrollback` out-of-range fails closed; unknown theme warns and falls back to default; `background_image` `~`-expansion failure fails closed. Three policies coexist in one function with no unified echo, so users cannot tell which layer rewrote what.
- Trigger: `gaps_in` 9999 silently becomes max; oversize `scrollback` exits directly; misspelled `theme` warns and continues.
- Evidence: Consecutive defensive `.min(MAX_...)` clamps in the function with the comment `defense-in-depth so a future bound drift can never wrap`, alongside the scrollback branch `return Err(... exceeds the supported maximum ...)` and the theme branch `unknown theme ... falling back`.

### PLUGIN-01 (P2) Plugin snapshot is frozen and the runtime is held by `_`; startup pays with no integration

- Location: `bitty/crates/bitty-app/src/plugin_runtime.rs::CommittedSnapshot::snapshot`/`discover_and_activate`, `bitty/crates/bitty-app/src/main.rs::main`
- Symptom: Snapshot source hard-codes `generation=1`, empty `rows`, and startup-instant width/height; the `PluginRuntime` returned by `discover_and_activate` is held as `_plugin_runtime` and never driven (the comment says command/event delivery is a later slice); plugin activation failure only `eprintln`s.
- Trigger: Any plugin depending on live snapshots/`on_event`/command delivery: startup logs show `active` but behavior never fires; the `--safe` skip logic is correct but there is no delivery off-safe either.
- Evidence: `CommittedSnapshot::snapshot` builds a fixed table (`rows: array([])`); `discover_and_activate` comment: `command/event delivery from the event loop is a follow-up slice`; binding in `main.rs` named `_plugin_runtime`.

### IPC-01 (P2) IPC fail-soft has no exit-code distinction; silent downgrade

- Location: `bitty/crates/bitty-app/src/ipc_serve.rs::serve_in_background`/`unix_serve`
- Symptom: Socket bind failure only does `eprintln!("bitty: ipc disabled (fail-soft): ...")`, and non-unix is always disabled; the terminal keeps running and `ctl` later times out, forcing users to correlate two log sites themselves.
- Trigger: Socket path occupied/insufficient permissions/non-unix platform + `ctl`/`devtools` handshake use.
- Evidence: The `Err(reason)` branch of `unix_serve` builds an `enabled:false` guard; same in the non-unix branch of `serve_in_background`; `main.rs` prints `ipc serving ...` only when `is_enabled`, otherwise nothing.

## Suggestions

1. Extract a `startup_assembler`, killing the fallback copy
   - Plan: Promote "build `Runtime` from `Args`+`EffectiveConfig`+env (including layout/focus/spawn/pane shells)" to a pure assembly function shared by the main and fallback paths; the fallback only passes a `DisplayUnavailable` context. Rebuild the plugin runtime with the new `Runtime` or explicitly declare headless never activates.
   - Benefit: Both paths behave identically; regression surface halved.
   - Effort: M.

2. Unify the illegal-CLI-value policy with effective-value echo
   - Plan: Make illegal `--split`/`--split-ratio`/`--layout`/`--focus` uniformly fail closed (exit 2 + usage), or grade explicitly: layout-class warnings must print "what was discarded, what was used"; add one `effective layout=... focus=... ratio=...` line to the startup summary. Return `Result<LayoutNode, SpecError>` from `parse_layout_spec` instead of `Option`, and surface errors from `build_layout`.
   - Benefit: Typos no longer silently become defaults; tickets stay reproducible.
   - Effort: S (parsing) + S (echo).

3. Clamp ratio explicitly with a warning at the assembly layer
   - Plan: Normalize with `clamp_ratio` before `build_layout` and `eprintln` when clamping fires; treat NaN/inf as illegal under suggestion 2's fail-closed path.
   - Benefit: UI solver keeps its total; app-layer semantics stay observable.
   - Effort: S.

4. Add a spawn-health gate to `--headless`
   - Plan: Under `--headless`, still run the synthetic chimney on main-spawn failure but distinguish "pipeline ok" from "shell usable" via a different exit code or `--strict-spawn`, or add a machine-readable `spawn=ok|failed` line to the output for CI to assert.
   - Benefit: Keeps the existing green while making shell-chain breakage discoverable.
   - Effort: S.

5. Bring startup logs under the verbosity gate with levels
   - Plan: Move `set_log_level` to right after arg parsing; route `theme/layout/spawn` summaries through `Info`, keeping warnings/errors unconditional; only errors under `--log-level error`.
   - Benefit: Quiet-by-default becomes trustworthy; log-gate docs match behavior.
   - Effort: S.

6. Tighten env level parsing
   - Plan: Tokenize on `RUST_LOG` commas/`=` and match level tokens exactly, adding `off/none` support; keep the `BITTY_LOG` precedence rule.
   - Benefit: Eliminates `mydebugger`-style misfires.
   - Effort: S.

7. Sink layout surgery into `bitty-ui`; slim `TerminalApp` to assembly+routing
   - Plan: Move `split_focused_leaf`/`close_focused_leaf`/`resize_focused_pane`/`find_resize_target` into `LayoutNode` methods (or a new `edit` module in `bitty-ui`); keep only keymap→action mapping plus spawn linkage in app; split `TerminalApp` into small structs by "composition root / event routing / logging".
   - Benefit: Layout invariants maintained in one place; unit tests no longer need a full app.
   - Effort: M.

8. Add a version/generation check to `zoom_backup`
   - Plan: Record a `generation` or layout hash alongside the backup; before restoring, if `ctl`/other actions have advanced the current tree, abandon the restore or rebase-then-restore with a warning; make the `WorkspaceNew/Prev/Next` vs `restore_zoom` relation explicit (restore or clear, either with a comment).
   - Benefit: Eliminates silent zoom+ctl pane loss.
   - Effort: M.

9. Wire panel modality into the dispatch predicate
   - Plan: Feed `OverlayManager::modal_active`/`PanelFocus` input (or a unified `ModalKind` enum) into `modal_capture_active`, using the `route_input` result as a front input to `resolve_dispatch`; add end-to-end unit tests that "bound chords are swallowed under modal, unbound pass through".
   - Benefit: The overlay docs' capture promise actually holds; background state stops mutating behind dialogs.
   - Effort: M.

10. Decide ownership of the three overlay systems
    - Plan: Docs first: a mapping table for `LayoutNode::Overlay` (in-solver), `OverlayManager` (pure presentation), `PresentationMode` (requested state) plus a single admission point; before opening `can_transition`, make the solver mode-aware (drop the byte-identical assertion or explicitly keep it).
    - Benefit: Later float/fullscreen work no longer half-wires.
    - Effort: M (docs S + implementation M).

11. Settle the `bitty-core` question
    - Plan, either: short-term, fix the `bitty-app` doc "depends only on runtime" to the real dependency table with rationale; long-term, put the stable facade (config/runtime/platform re-exports plus assembly traits) truly into `bitty-core` so app depends only on it. Before retirement, stop new code from wiring internal crates directly.
    - Benefit: Architecture statements match code; change-isolation point is explicit.
    - Effort: S (docs) / L (facade landing).

12. Unify out-of-range echo in `runtime_config_from_effective`
    - Plan: Emit uniform `eprintln!("warning: <field> <raw> out of range — clamped to <max>")` at clamp sites, or converge on a written rule of "geometry clamps silently, semantics fail closed" with effective values shown in `config check`.
    - Benefit: Config drift becomes observable; triage stops guessing.
    - Effort: S.

13. Make plugin startup honest
    - Plan: Document the current snapshot as a frozen stub (or name it `FrozenStartupSnapshot` outright), distinguish "loaded but undelivered" in activation logs; avoid advertising `active` before event delivery lands; count failures into `doctor`.
    - Benefit: Plugin authors are not misled; startup cost stays visible.
    - Effort: S.

14. Make IPC downgrade discoverable
    - Plan: On IPC disabled, add one warning each to the startup summary and `doctor` (with socket path + cause); point `ctl` timeout errors back at that line.
    - Benefit: Two-process issues become correlatable without digging through two stderrs.
    - Effort: S.

## Test gaps

- Fallback assembly consistency: no "same `Args` yields identical layout/focus/spawn on main vs `DisplayUnavailable` fallback" comparison test; today only eyeballing keeps the two 60-line blocks in sync.
- Illegal CLI combos: no fail-closed vs warning matrix test for `--split-ratio NaN/inf/out-of-range`, `--layout split:foo:xyz/stack:0/stack:9999/overlay:1,2`, `--focus empty/oversize id`; existing coverage is mostly happy-path.
- Headless masking spawn: no assertion that "when the whole shell chain is broken, `--headless` output contains `spawn=failed` with an optional nonzero exit"; today synthetic bytes are always green.
- Zoom vs external-change race: no pane-survival test for `zoom on → ctl split/close → zoom off`; no backup-semantics test for `WorkspaceNew/Prev/Next` under zoom.
- Panel modal capture: no routing test that bound chords are swallowed and unbound pass through to the PTY when `OverlayManager::modal_active=true`; `route_input` has only pure-function unit tests, never wired into `intercept_chrome_key`.
- Three layout systems: no test of non-Tiled `PresentationMode` entering the solver (today only ignored-assertion); no paint-order test stacking `LayoutNode::Overlay` with `OverlayManager`.
- Frozen plugin snapshot: no failing test of "snapshot rows/generation seen by plugins advance with the terminal" (adding one today would correctly go red, proving the stub nature); no test counting activation failures into `doctor`.
- GPU/platform downgrade: three `try_attach_gpu` failures (init/surface/configure) rely only on env-gated manual verification; no headless unit test for zero-size surface skip or unsupported-opacity warning. No budget test for `pollster::block_on` blocking time on the event-loop thread.
- IPC downgrade: no integration test for socket-occupied/permission-denied → `enabled=false` + startup warning + `doctor` linkage; no coverage of the non-unix path.
- Log gate: no assertion that the startup summary stays silent under `--log-level error`; no case that `RUST_LOG=mydebugger=info` is not misjudged as Debug; no `off` semantics.
- Config convergence: no clamp-echo test (`gaps_in 9999 → max + warning`); no control table of scrollback-out-of-range fail-closed vs unknown-theme fallback; no `~`-expansion fail-closed case when `HOME` is missing threaded through the multi-value `background_image_roots` path (`expand_home_path` has unit tests but they never reach it).
- Facade: `bitty-core` has zero tests and zero re-exports; the architecture assertion "bitty-app really depends on 8 internal crates" is missing, so nothing stops further facade bypasses.

Date: 2026-09-15
Covered crates: `bitty-app`, `bitty-ui`, `bitty-core`
