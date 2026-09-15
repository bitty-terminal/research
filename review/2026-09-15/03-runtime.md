# Bitty Runtime review report (read-only)

- Scope: `bitty/crates/bitty-runtime/` (49 `rs` source files, 116 `rs` files including tests, ~2.6M; the review target is the runtime core)
- Method: Read-only, no code changes. First read `bitty/crates/bitty-runtime/Cargo.toml`, `bitty/crates/bitty-runtime/src/lib.rs` and the module tree (`lib.rs` module declarations, per-directory `mod.rs`), then captured the backbone with `ctxctl outline` (event loop `tick`/`tick_at`, task scheduling `poll_pty`/`pump_pane_sessions`, session management `spawn_shell*`/`close_pane_session`/`TerminalRegistry`), and finally sampled concurrency primitives (`Arc`/`Mutex`/`mpsc::sync_channel`/`Atomic*`/`JoinHandle`), executor usage (confirmed no `tokio`/`async` — all `std::thread`), and shutdown/cancel paths (`PanelWorker::shutdown`/`Drop`, forwarder `JoinHandle`, `spawn_process` timeout kill, `Esc` cancel chain).
- Sampling strategy (too many files to read fully): prioritize core paths (`src/runtime.rs`, `src/runtime/pty.rs`, `src/runtime/panes.rs`, `src/runtime/present.rs`, `src/queue.rs`, `src/panels_async.rs`, `src/registry/terminal.rs`, `src/error.rs`, `src/plugin_runtime/spawn.rs`, `src/plugin_runtime/mod.rs`, `src/host_bridge.rs`, `src/runtime/input.rs`, `src/runtime/selection.rs`), overlaid with "recently modified" (`git -C bitty log` shows `plugin_runtime/spawn.rs`, `host_bridge.rs`, `panels_async.rs`, `runtime/workspaces.rs`, `runtime/panes.rs` as recent high-churn files) and "most complex" (top rows: `src/config.rs` ~2219 lines, `src/runtime/present.rs` ~2031 lines, `src/plugin_runtime/spawn.rs` ~1717 lines, `src/runtime.rs` ~1307 lines). Remaining panel integrations (`ai_panel.rs`, `mail_panel.rs`, `browser_panel.rs`, etc.) were outline-read and sampled only, not audited line by line.

Key background conclusion: the crate has no `async` executor dependency (no `tokio`/`futures` in `Cargo.toml`, no `async fn` in source); the concurrency model is "synchronous non-blocking drain + bounded `sync_channel` + one `std::thread` per source". The headless gaps and GPU absence are honestly declared in `lib.rs` and are not recorded as defects in this report.

## Defect list

### P1 — `Runtime` has no `Drop`; forwarder `JoinHandle`s detach on respawn/close/drop paths (task leak)

- Path+symbol: `bitty/crates/bitty-runtime/src/runtime.rs::Runtime` (fields `pty_forward_handle`, `pty_forward_rx`, `pane_sessions`), `bitty/crates/bitty-runtime/src/runtime/panes.rs::Runtime::spawn_shell_with_args`, `bitty/crates/bitty-runtime/src/runtime/panes.rs::Runtime::spawn_shell_for_view`, `bitty/crates/bitty-runtime/src/runtime/panes.rs::Runtime::close_pane_session`, `bitty/crates/bitty-runtime/src/runtime/pty.rs::Runtime::promote_pty_reader_to_forwarder`
- Symptom: `spawn_shell_with_args` replaces the old child via bare `self.pty_forward_rx = None; self.pty_forward_handle = None;` (the old `JoinHandle` is discarded, i.e. detached); `close_pane_session` directly `pane_sessions.remove(view)`, likewise discarding `PaneSession.forward_handle`; in the whole crate only `bitty/crates/bitty-runtime/src/panels_async.rs::PanelWorker` implements `Drop` — `Runtime` itself has no `Drop`, so forwarder threads have no join point when the process/test ends.
- Trigger: Respawning the main shell, replacing/closing a split pane, dropping `Runtime` after closing a workspace; always reproduces once `set_pty_waker` has promoted to forwarder.
- Evidence: The comment in `src/runtime/panes.rs` admits "detached: it owns the old reader and exits on EOF/disconnect once the old PTY is dropped"; the field comment in `src/runtime.rs` says "detached on respawn"; `rg "impl Drop"` over `src` hits only `panels_async.rs`. A detached thread can only exit if "the old PTY being dropped makes the reader return EOF", so it lingers whenever the platform-layer reader does not return `None` before the child fully exits; and the detached thread still holds a clone of `PtyWaker` (`Arc<dyn Fn>`), so it may keep invoking the embedder callback after `Runtime` is destroyed.

### P1 — Forwarder promotion path fails thread creation with a bare `expect` panic (library code must not panic)

- Path+symbol: `bitty/crates/bitty-runtime/src/runtime/pty.rs::Runtime::promote_pty_reader_to_forwarder`, `bitty/crates/bitty-runtime/src/runtime/panes.rs::Runtime::promote_pane_reader_to_forwarder`
- Symptom: Both sites use `.expect("std thread spawn cannot fail with default builder options")`. This contradicts the same crate's `PanelWorker::try_spawn`, which maps spawn failure fail-closed to `RuntimeError::InvalidConfig`.
- Trigger: `Builder::spawn` returning `Err` under FD exhaustion, memory pressure, or frequent split/spawn.
- Evidence: The `expect` lines near L228–249 in `src/runtime/pty.rs` and L277–301 in `src/runtime/panes.rs`; compare the `map_err` line in `src/panels_async.rs::PanelWorker::try_spawn`.

### P1 — `tick_at` is a ~1245-line mega-function (god function)

- Path+symbol: `bitty/crates/bitty-runtime/src/runtime/present.rs::Runtime::tick_at` (L685–1930)
- Symptom: Hover dwell, paste-banner folding, sync-update deferral, layout reflow, Kitty origin binding, alt latch, multi-origin generation comparison, layout/focus change detection, animation advance, frame-on-demand short-circuit, empty-layout handling, full-frame decision, etc. are all squeezed into one `&mut self` function. Any branch missing a `mark_frame_presented` / `last_presented_*` update yields a stale frame or overdraw; the review already found at least 5 independent `pending_full = true` sources plus multiple early `return None`s — very high mental load.
- Trigger: Any change adding a new present condition (new overlay, new animation, new defer semantics) must pass through this function; regression risk grows linearly with branch count.
- Evidence: `outline` shows `tick_at` alone owns ~1245 of the file's 42 symbols/lines; at 2031 lines the file is the second largest in `src`.

### P1 — Input/reply write failures are silently swallowed; the error-propagation chain breaks

- Path+symbol: `bitty/crates/bitty-runtime/src/runtime/input.rs::Runtime::push_input_bytes` (`writer.write_all` + `flush` both `let _`), `bitty/crates/bitty-runtime/src/runtime/pty.rs::Runtime::write_replies`, `bitty/crates/bitty-runtime/src/runtime/panes.rs::Runtime::write_pane_replies`
- Symptom: With a live writer, input is directly `let _ = writer.write_all(bytes)` and returns — failures get no counter and no error return; `write_replies` `break`s on write failure and drops the remaining replies, while `replies_overflowed` only counts the 4 KiB cap on the `State` side, not writer-side loss. Callers cannot distinguish "delivered" from "discarded".
- Trigger: Child exits and the PTY master returns EIO/EBADF, writer half-close, dead pane writer.
- Evidence: The double `let _` near L395–405 in `src/runtime/input.rs`; the `if writer.write_all(&chunk).is_ok() { total += ... } else { break; }` branch in `src/runtime/pty.rs::write_replies`; the `take_replies`/`flush_pty_replies` aliases share the same path.

### P1 — Library code calls `eprintln!` on hot paths (attacker-controllable log spray)

- Path+symbol: `bitty/crates/bitty-runtime/src/runtime/pty.rs::Runtime::handle_pty_bytes` (illegal-base64 OSC 52 branch, Kitty display-failure branch), the spawn-failure branch at the `bitty/crates/bitty-runtime/src/runtime/workspaces.rs::Runtime::spawn_shell_for_view` call site
- Symptom: Each untrusted PTY byte triggering illegal OSC 52 / illegal Kitty payload produces one `eprintln!`; high-frequency output drives high-frequency stderr writes, with no rate limit and no switch. The embedder cannot intercept it (a library should never write stderr directly).
- Trigger: Malicious or corrupt remote output continuously sending illegal OSC 52; `cat` of a large file hitting the Kitty rejection branch.
- Evidence: Near L478 (`bitty: rejecting invalid OSC 52 ...`) and L554 (`bitty: rejecting kitty image ...`) in `src/runtime/pty.rs`; the `osc52_rejected_writes` counter exists, but the warn itself does not converge.

### P1 — `Esc` swallows all pending arms at once (input hijack against fullscreen apps)

- Path+symbol: `bitty/crates/bitty-runtime/src/runtime/selection.rs::Runtime::cancel_pending_on_escape` (called by `bitty/crates/bitty-runtime/src/runtime/input.rs::Runtime::handle_key_event`, which hits `return None` on match)
- Symptom: When several of `pending_close_confirm`, `pending_ws_close`, `help_visible`, `pending_paste` coexist, one `Esc` cancels/closes all of them and that `Esc` never reaches the PTY. The comment says "loud, no partial state", but for vim/fullscreen terminal apps `Esc` is a mode key — a user's `Esc` is intercepted by the runtime exactly when a banner/pending exists, desyncing the app state machine.
- Trigger: User presses `Esc` in vim while a paste confirmation is pending or help is visible; the workspace-close arm survives switching (see "switching keeps the pending arm" in `src/runtime/workspaces.rs`), so pressing `Esc` in another workspace also cancels it by mistake.
- Evidence: The four consecutive cancel blocks and trailing `return cancelled`/`true` near L495–540 in `src/runtime/selection.rs`; `if self.cancel_pending_on_escape(&event) { return None; }` in `src/runtime/input.rs::handle_key_event`.

### P1 — `PanelWorker` shutdown cannot cancel a running probe (unstoppable worker thread)

- Path+symbol: `bitty/crates/bitty-runtime/src/panels_async.rs::PanelWorker::run`, `bitty/crates/bitty-runtime/src/panels_async.rs::PanelWorker::shutdown`, `bitty/crates/bitty-runtime/src/panels_async.rs::PanelWorker::drop`
- Symptom: `shutdown` means "set flag + try_send wakeup + join", but `run` never checks the flag while `probe()` executes; `poll_interval` only affects idle waits. `probe: impl Fn() -> T` has no cancellation contract, so `shutdown`/`Drop` blocks the calling thread for as long as a slow probe (`git status`, large directory scan) runs. Calling `shutdown` from `Drop` means dropping the worker on the UI thread can block it for a long time.
- Trigger: Closing a panel/workspace or quitting the app exactly while a slow probe is in flight.
- Evidence: The `run` loop at L186–219 in `src/panels_async.rs` (flag checked once before `probe()`, never during); `handle.join()` in `shutdown` at L265–274; `Drop` calling `shutdown` at L312–318. The module docs' "at most one in-flight probe plus join" admits the bound shape but never bounds the bound itself.

### P1 — `spawn_process` timeout path discards already-drained output, and join can be held by grandchildren (no cancel point)

- Path+symbol: `bitty/crates/bitty-runtime/src/plugin_runtime/spawn.rs::spawn_process` (timeout branch), `bitty/crates/bitty-runtime/src/plugin_runtime/spawn.rs::drain_bounded`, `bitty/crates/bitty-runtime/src/plugin_runtime/spawn.rs::join_reader`
- Symptom: After timeout the code does `child.kill + child.wait`, then only `let _ = handle.join()` on both reader threads, discarding their retained output and finally returning `Unknown` with empty stdout/stderr. Callers cannot see the partial output produced before the timeout; if a grandchild forked by the child inherits the pipe write end and never exits, `drain_bounded` never reads EOF and the `join` blocks the whole `dispatch` calling thread (synchronous `sleep(SPAWN_POLL_INTERVAL)` polling, with no cancellation/overall-timeout applied to the join itself).
- Trigger: Any `process.spawn` call with a timeout that times out; children spawning grandchildren or ignoring SIGKILL in extreme cases.
- Evidence: The timeout branch near L760–800 in `src/plugin_runtime/spawn.rs` (both `take()`s followed by discarded join results, returning empty `stdout`/`stderr`); the `loop { pipe.read }` in `drain_bounded` has no timeout; `join_reader` is a blocking `join`.

### P1 — `poll_pty` collects up to 1024 chunks per call with no merging — waker storm (the upstream side of backpressure in name only)

- Path+symbol: `bitty/crates/bitty-runtime/src/runtime/pty.rs::Runtime::poll_pty`, `bitty/crates/bitty-runtime/src/runtime/panes.rs::Runtime::pump_pane_sessions`, `bitty/crates/bitty-runtime/src/runtime/pty.rs::pty_forward_loop`
- Symptom: The docs promise end-to-end backpressure of "`CHANNEL_CAPACITY_CHUNKS × READ_CHUNK_SIZE` bounded + kernel PTY buffer + child blocking", but one `poll_pty` can collect 1024 chunks (~4 MiB at the default chunk cap) and run each through `handle_pty_bytes` (each chunk's parser actions accumulate into an unbounded `Vec<TerminalAction>`, and each action may additionally push `ColdEvent::Damage`). A single `poll_pty(&mut self)` monopolizes the runtime for an unbounded time, starving ticks. In the reverse direction, the forwarder-side `pty_forward_loop` calls `waker()` once per chunk, so high-frequency output means high-frequency cross-thread wakeups with no merge throttling.
- Trigger: `cat` of a large file, fast screen-refresh tests, malicious PTY flood; headless `poll_pty_timeout` blocks on primary first while primary is quiet (see P2), further amplifying latency.
- Evidence: The `while out.len() < 1024` loop in `src/runtime/pty.rs::poll_pty`; the uncapped `let mut actions: Vec<TerminalAction> = Vec::new()` inside `handle_pty_bytes`; the per-chunk `(waker)()` in `pty_forward_loop`.

### P2 — `ColdQueue`/`SideQueue` DropOldest has no priority; high-frequency Damage drowns meaningful events

- Path+symbol: `bitty/crates/bitty-runtime/src/queue.rs::ColdQueue::push`, `bitty/crates/bitty-runtime/src/runtime/pty.rs::Runtime::handle_pty_bytes` (pushes one `ColdEvent` per action), `bitty/crates/bitty-runtime/src/runtime/plugin.rs::cold_to_observation`
- Symptom: Every damage-carrying action pushes one `ColdEvent::Damage { generation }`, so under high throughput the low-frequency critical events (`TitleChanged`/`CwdChanged`/`ZoneMarked`/`Bell`) are squeezed out by the same oldest-drop policy. The `dropped` counter exists but has no per-class attribution, so `bitty plugin doctor` only sees a total.
- Trigger: Sustained screen refresh plus plugin logic depending on title/cwd/zone.
- Evidence: `pop_front + wrapping_add(1)` in `src/queue.rs::push`; unconditional `cold_queue.push(Damage)` + `push_observation(Damage)` on the damage branch in `src/runtime/pty.rs`.

### P2 — `poll_pty_timeout` starves panes while primary is quiet (opposite of the comment's promise)

- Path+symbol: `bitty/crates/bitty-runtime/src/runtime/pty.rs::Runtime::poll_pty_timeout`
- Symptom: It first waits for the primary's first chunk via `rx.recv_timeout(timeout)` and drains no pane until then; the comment says "quiet primary must not starve panes", but the implementation is exactly "panes are delayed by up to one timeout while primary is quiet". The embedder cannot drain concurrently either while `&mut self` is held.
- Trigger: Idle primary + high-frequency split-pane output + a caller using `poll_pty_timeout` to wait for echo.
- Evidence: The `first`-collection block first matches `pty_forward_rx.recv_timeout` / `reader.recv_timeout`, and only the `None` arm reaches `pump_pane_sessions`.

### P2 — Close/cancel semantics: three arm sets each on their own (paste/ws_close/close_confirm have no unified invalidation scope)

- Path+symbol: `bitty/crates/bitty-runtime/src/runtime/selection.rs::Runtime::confirm_pending_paste`, `bitty/crates/bitty-runtime/src/runtime/workspaces.rs::Runtime::cancel_pending_ws_close`, `bitty/crates/bitty-runtime/src/runtime/close_confirm.rs::Runtime::cancel_pending_close_confirm`
- Symptom: Three pendings each have their own `Option` + standalone cancel fn, only temporarily stitched together by `cancel_pending_on_escape`; there is no unified rule for "arm scope/timeout/invalidation". `pending_ws_close` survives workspace switches (explicitly documented), so a confirm gesture in another workspace may kill the wrong workspace; `close_pane_session` clears the close arm of the matching view (`clear_pending_close_for_view`), but cross-invalidation between ws-level arms and pane lifetimes is uncovered.
- Trigger: Confirming with the same gesture after switching workspaces post-arm / after closing the matching pane.
- Evidence: Module comment "switching workspaces keeps the pending arm" in `src/runtime/workspaces.rs`; `close_pane_session` in `src/runtime/panes.rs` only clears the view-level arm.

### P2 — `Runtime` god object + duplicated constructors (a hotbed of inconsistency)

- Path+symbol: `bitty/crates/bitty-runtime/src/runtime.rs::Runtime` (~60 fields), `bitty/crates/bitty-runtime/src/runtime.rs::Runtime::with_plugin_host_capacity`, `bitty/crates/bitty-runtime/src/runtime.rs::Runtime::with_plugin_host`
- Symptom: `Runtime` spans every responsibility — PTY/parsing/state/rendering/layout/focus/workspace/selection/clipboard/IME/wheel/animation/Kitty/backgrounds/inspection-loop/plugin-host, etc.; the two constructors each repeat ~150 lines of field initialization (`pending_input`, `last_presented_*`, `kitty_*`, `backgrounds`, `workspaces`, etc. verbatim), so every new field must be edited in two places and easily drifts. Submodules read/write parent fields directly via `use super::*` (`panes.rs`/`pty.rs` both do `use super::*`), so module boundaries exist in name only.
- Trigger: Any change adding new runtime state.
- Evidence: Struct definition at L262–583 in `src/runtime.rs`; the two constructor bodies at L698–850 and L856–991; L8 `use super::*` in `src/runtime/panes.rs`, L7 `use super::*` in `src/runtime/pty.rs`. `ctxctl deps` shows `runtime.rs` with 110 imports — the largest coupling surface.

### P2 — Zero-capacity semantics inconsistent within one crate (panic vs fail-closed)

- Path+symbol: `bitty/crates/bitty-runtime/src/queue.rs::ColdQueue::new`, `bitty/crates/bitty-runtime/src/panels_async.rs::PanelWorker::try_spawn`
- Symptom: `ColdQueue::new(0)` directly `assert!(capacity > 0)` panics, while `PanelWorker::try_spawn(..., 0, ...)` returns `Err(RuntimeError::InvalidQueueCapacity)`. Callers constructing `ColdQueue` directly can be panicked; `RuntimeConfig` validation is the only barrier.
- Trigger: Unit tests / new call sites constructing the queue directly, bypassing `RuntimeConfig`.
- Evidence: The `assert` at L61–68 in `src/queue.rs`; `if queue_cap == 0 ... return Err` at L133–142 in `src/panels_async.rs`.

### P2 — All-Relaxed atomic orderings + `Mutex` granularity documented but with no deadlock regression net

- Path+symbol: `bitty/crates/bitty-runtime/src/panels_async.rs::PanelWorker` (fields `slot: Arc<Mutex<SnapshotSlot>>`, `shutdown_flag/queued/dropped` all `Ordering::Relaxed`)
- Symptom: The current design — "probe runs outside the lock, only an `Arc` swap happens inside" — is correct; no path holding two locks or calling probe under lock was found. But the `Relaxed` store of `shutdown_flag` relies on the subsequent `try_send` wakeup to carry visibility; if the wakeup token is shed while the worker sits in a long probe, flag visibility has no happens-before guarantee (works on x86 in practice, theoretically delayable on weak memory models). `latest`/`generation`/`completed` each `lock()` on every clone, contending with the worker completion path on the same lock under high-frequency ticks — the critical section is tiny but contention is unmeasured.
- Trigger: ARM-weak-memory + shutdown landing exactly during a long probe; extreme tick-high-frequency + worker-high-frequency-completion combos.
- Evidence: `SnapshotSlot` at L97–103, `run` at L186–219, `shutdown` at L265–274, and `slot.lock()` in `latest/generation/completed` at L287–309 in `src/panels_async.rs`.

## Suggestions

- Add `Drop`/explicit `close` for `Runtime` and `PaneSession` (expected benefit: no more forwarder-thread leaks or post-destruction waker callbacks; effort M)
  - Plan: implement `Drop` (or a named `shutdown` + `Drop` fallback) for `Runtime` in `bitty/crates/bitty-runtime/src/runtime.rs`, joining `pty_forward_handle` and each pane `forward_handle`; on the replace path in `spawn_shell_with_args`/`spawn_shell_for_view`, join the old handle before publishing the new session (or explicitly hand over EOF-waiting on the old reader with a timeout); same join-before-return in `close_pane_session`. Do not join while holding other big locks, to avoid importing P1-probe-style blocking onto the main thread — signal first, then bounded-join where needed.
- Return an error instead of panicking on forwarder-creation failure (expected benefit: fail-closed under resource exhaustion, consistent with `PanelWorker`; effort S)
  - Plan: change both `expect`s in `bitty/crates/bitty-runtime/src/runtime/pty.rs` and `bitty/crates/bitty-runtime/src/runtime/panes.rs` to `map_err(|_| RuntimeError::InvalidConfig(...))` and roll the taken reader back into place; add a unit test "reader/writer restored on spawn failure".
- Split `tick_at` (expected benefit: testability and regression safety; effort L)
  - Plan: extract per-phase pure functions: `apply_time_gates(now)` (hover/paste/sync defer), `collect_frame_basis()` (frames/allocations/focus/kitty origins), `decide_full_frame(...)`, `present_full(...)`/`present_incremental(...)`/`mark_idle(...)`. Make each phase's inputs/outputs explicit with `tick_at` as orchestration only; add a unit test per early-return branch.
- Make input/reply write failures explicit (expected benefit: callers can tell delivered from dropped; effort S)
  - Plan: change writer failures in `push_input_bytes`/`write_replies`/`write_pane_replies` to counters (e.g. `pty_write_dropped: u64` + `last_pty_write_error` slot, reusing the `osc52_rejected_writes` pattern) or return `Result<usize, RuntimeError>`; extend `replies_overflowed` semantics or add `replies_write_dropped` for `bitty plugin doctor` attribution.
- Converge `eprintln!` into an injectable diagnostic event (expected benefit: no attacker-controllable stderr spray; effort S)
  - Plan: add a bounded diagnostic ring (reuse the `ColdQueue` pattern or the `inspect` ring); OSC 52/Kitty rejections only count + enter the ring, no per-event `eprintln!` by default; keep a debug switch for explicit loud mode.
- Shard `poll_pty` by time + merge wakers (expected benefit: heavy output no longer starves ticks; wakeup storms converge; effort M)
  - Plan: shard `poll_pty` by the minimum of "chunk cap + byte cap + time budget", leaving the remainder for the next poll; throttle `pty_forward_loop` to "wake immediately on first chunk + one merged wakeup per following window"; change `poll_pty_timeout` to wait on primary and panes together (or non-blocking-drain panes first, then block on primary), honoring the "quiet primary must not starve panes" comment.
- Give the `Esc` cancel chain an explicit scope (expected benefit: fullscreen apps stop losing `Esc`; effort M)
  - Plan: change `cancel_pending_on_escape` to act "only when the event is unclaimed by the terminal app", or handle arms one by one by focus/overlay priority while recording which arm was consumed; split informational overlays such as help (`Esc`-consumable) from confirm gates such as paste/ws_close (explicit gesture required); bind workspace arms to workspace ids so switching auto-invalidates or explicitly migrates them.
- Make `PanelWorker` probes cancellable or time-bounded (expected benefit: bounded shutdown path; effort M)
  - Plan: either change the probe signature to `Fn(AtomicBool /*cancel*/) -> T` with polling in long loops, or change `shutdown` to "signal + bounded join, detach with a leak count on timeout" (must link with the P1-Drop fix to avoid silent leaks). Either way, add a "shutdown duration bound under slow probe" unit test.
- Keep partial output on `spawn_process` timeout + guard joins (expected benefit: timeouts stay reviewable, no permanent blocking; effort M)
  - Plan: collect already-drained partial output on the timeout branch too, marking `truncated` consistently with the normal path; add an overall timeout to reader joins, detaching with `evidence_refs` on timeout so inherited pipes from grandchildren cannot block forever.
- Deduplicate constructors and decouple modules (expected benefit: no more new-field drift; effort M)
  - Plan: extract a single `Runtime::from_validated_parts(config, plugin_host, ...)` initialization point so both public constructors only validate and assemble the host before calling it; gradually narrow `use super::*` to explicit imports + small interfaces (e.g. `pane_geometry()`, `kitty_layer_mut()`), reducing direct parent-field writes from submodules.
- Unify zero-capacity semantics (expected benefit: the library never panics; effort S)
  - Plan: change `ColdQueue::new` to return `Result` (or a `const`-assert alternative), at minimum returning `RuntimeError::InvalidQueueCapacity` consistently with `PanelWorker`; add a `forbid`-level panic audit across the crate (`expect`/`unwrap` allowed in test modules only).

## Test gaps

- No coverage of true-PTY and true-GPU paths (known honest gap): `src/lib.rs` already declares CI runs headless only; `GpuContext::initialize`/`create_surface`/real present, `ConPTY` (Windows), and real wakeup timing of `poll_pty_timeout` have no assertions. `tests/soak.rs`, `tests/pty_wakeup.rs`, `tests/pty_reply.rs` are headless/synthetic bytes and cannot prove backpressure and waker-merge behavior of cross-thread forwarders under a real child flood.
- Missing close/leak regression net: no tests for "old forwarder joined after respawn", "`close_pane_session` returns thread count to baseline", "no leftover threads / no waker callbacks after `Runtime` drop"; `PanelWorker` has `teardown_joins_worker_promptly`/`drop_joins_worker_without_hang`, but they use millisecond fake probes — no shutdown-bound test under long probes, no spawn-failure injection test.
- No assertions on error propagation: input/reply loss counts on writer EIO/EBADF, rate convergence of OSC 52/Kitty rejections, distinguishing `replies_overflowed` from writer-side loss — all untested; `eprintln!` output itself has no capture assertion either.
- No fullscreen control for cancel semantics: no test that "`Esc` reaches the PTY when vim holds focus + pending paste/help"; no cross-scenario tests for workspace-arm keep-vs-invalidate across switches or whether a ws arm mis-kills after its pane closed.
- No pressure test for high-throughput backpressure: no tests for "per-`poll_pty` duration bound at the 1024-chunk cap", "`ColdQueue` critical-event survival under screen flood", "per-second waker count bound"; `tests/scroll_follow.rs`, `tests/scrollback_cap.rs` cover capacity caps, not scheduling fairness.
- Registry exhaustion and concurrency: `TerminalRegistry::create_terminal` has unit tests for `GenerationExhausted`/`TooManyTerminals`/`PersistentIdInUse`, but cross-sequences of `attach/detach/move/replace` with post-`dispose` operations, and the `RESIZE_DEBOUNCE_CAP` merge cap's loss attribution under resize bursts, have no integration tests.
- Timeout and partial output: retaining partial output after `spawn_process` timeout, guaranteed reachability of `Unknown` `reconcile`, non-permanent blocking of join when grandchildren inherit pipes — all untested; existing `src/plugin_runtime/spawn.rs` tests focus on auth/quota, not executor-cancel semantics.
- Mega-function branches: the sync-defer window, alt-latch flip, animation-expiry frames, multi-origin generation comparison, and empty-layout idle five-way combos in `tick_at` have no decision-table tests; new branches can regress silently.

Date: 2026-09-15
