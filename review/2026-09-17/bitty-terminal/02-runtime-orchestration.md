# Runtime orchestration review — 2026-09-17

## Independent second-pass verification

- Read-only source check at unchanged `bitty` HEAD `06bc1f45995fd297a3f7324688bc81b0598ecf67`; only pre-existing untracked `.targets/`. Original coverage and checks below describe the first pass.
- **Independently verified:** TERM-RUN-001/002/003, P1 static mechanisms. `bitty/crates/bitty-runtime/src/plugin_runtime/spawn.rs:672-831` shows unbounded reader joins outside the wait deadline, empty timeout output, and post-spawn error returns without cleanup ownership. `plugin_runtime/store.rs:50-123,143-189` shows mutation before persistence and destination deletion on any rename error. These citations were reread, not merely checked for existence.
- Qualification: no real process hang, leak, lost file, platform replacement failure, or production incident was reproduced. P1 prioritizes ownership and committed-state integrity under the stated error conditions; it does not assert frequency. A byte retention cap does not bound drain completion. The runner's ignored kill/wait errors also do not prove cleanup succeeded.
- **Not independently reverified:** TERM-RUN-004/005/006/007/008/009/010/011. A partial read of store loading is not full validation of TERM-RUN-005's depth/numeric/round-trip claims. In particular, experimental panel APIs are not automatically live UI defects.
- Additional context sampled: `plugin_runtime/spawn.rs:1008-1052`, `plugin_runtime/services.rs:197-252`, and the layout/workspace slices recorded in report 03. No product build, test, fault injection, CarryCtx, network, installation, or source edit. Markdown checks and exact cross-report scope are in [the product index](README.md).

## Scope, baseline, and method

Read-only static review of `bitty/crates/bitty-runtime`, prioritizing lifecycle, routing, persistence, and panels. Only this report was authorized for writing. No CarryCtx, network, installation, nested agents, commits, builds, runtime tests, exploit execution, or reproduction code was used. Safe test ideas below are proposed defensive checks, not executed evidence.

Paths in findings are relative to the umbrella workspace. Coverage paths are relative to `bitty/crates/bitty-runtime/`.

| Repository            | HEAD                                       | Initial dirty baseline                                  |
| --------------------- | ------------------------------------------ | ------------------------------------------------------- |
| `bitty`               | `06bc1f45995fd297a3f7324688bc81b0598ecf67` | `?? .targets/`; no tracked or staged diff               |
| `research`            | `d70152a079c75204ec37e99a7bf3f54b8190e423` | Seven existing untracked campaign reports, listed below |
| `bitty-terminal-docs` | `7947fb39acf77d308c2eb8bc57b24c481a841a17` | Clean                                                   |

The umbrella is not a Git repository. Initial Git queries there failed; baselines were then taken in the independent repositories. `bitty` pins its `docs` submodule to `56f70bc9b98bf7585d97ccafde33c81c5d69ac9d`; this review consulted the separate terminal-docs checkout above, not an updated submodule.

Existing untracked files in `research/review/2026-09-17/`:

- `bitty-ai/01-runtime-and-sessions.md`
- `bitty-ai/02-context-slices-and-providers.md`
- `bitty-plugins/01-sdk-and-template.md`
- `bitty-plugins/02-registry-and-distribution.md`
- `bitty-plugins/03-independent-plugins.md`
- `bitty-terminal/01-terminal-engine.md`
- `bitty-terminal/03-app-ui-core-config.md`

Read the umbrella, `bitty`, `research`, and `bitty-terminal-docs` AGENTS in full, plus the ctxctl-core skill. No narrower AGENTS appeared under the runtime or report scope. Explicit parent-path checks found no additional ancestor AGENTS. CarryCtx instructions in repository guidance were not executed, honoring the task-specific prohibition.

Used source outlines followed by selected source ranges, test-body reads, and targeted searches. The ctxctl MCP rejected the checkout as outside its configured root; the installed ctxctl CLI worked from `bitty`. An outline or search hit is not counted as a full source read. Comments, prior reviews, and test names were treated as claims to check, not proof.

## Summary

First-pass inventory: **3 P1, 7 P2, 1 P3**. The independent second pass verified TERM-RUN-001/002/003 only; remaining findings retain first-pass status, not renewed confirmation. Priorities distinguish high-impact process/persistence failures (P1), correctness and contract failures (P2), and telemetry defects (P3). These are not severity claims about a demonstrated external attack surface. Experimental registry findings are explicitly separated from the live `Runtime` path.

The runtime uses synchronous orchestration and standard-library threads/channels. `Runtime` owns the live primary PTY and pane sessions; the terminal/panel registries are separate domain APIs. Plugin key/value persistence is implemented, whereas terminal restart restoration must not be inferred from `PersistentId` allocation tests. Real PTY tests do exist; headless rendering does not mean synthetic PTY input only.

## Confirmed findings

First-pass records below; second-pass status is stated above.

### TERM-RUN-001 — P1: Process timeout does not cover output-drain completion

- **Evidence:** `bitty/crates/bitty-runtime/src/plugin_runtime/spawn.rs:743`, `:755`, `:776`, `:809`, `:829`.
- **Cause:** The deadline governs only the child wait loop. Both normal exit and timeout synchronously join readers that wait for pipe EOF. A child exit is not equivalent to EOF when another process retains a pipe endpoint. Timeout also discards the readers' retained bytes and returns empty streams.
- **Impact:** A permitted execution can keep its synchronous caller waiting beyond the declared timeout; partial diagnostic output is lost on the timeout branch. This is a lifecycle defect in the runner reached through the authorized bridge, not evidence that authorization is absent.
- **Fix:** Apply one deadline/cancellation policy to child supervision and output collection. Use owned process-tree supervision where supported, cancellable drain completion, and bounded partial results. Report cleanup uncertainty honestly instead of unconditionally claiming the child was killed and reaped after ignored errors.
- **Safe test idea:** Use a fake child/reader supervisor with controlled completion signals and a virtual deadline; assert bounded return, retained partial output, and explicit incomplete-cleanup status. No real descendant-process scenario was executed.

### TERM-RUN-002 — P1: Post-spawn errors can abandon a running child

- **Evidence:** `bitty/crates/bitty-runtime/src/plugin_runtime/spawn.rs:695`, `:715`, `:726`, `:747`.
- **Cause:** After successful `Command::spawn`, reader-thread creation and `try_wait` propagate errors with `?`. Those returns bypass kill/wait cleanup. The later missing-pipe cleanup branch cannot handle thread creation errors because they have already returned. Dropping `std::process::Child` is not a kill-and-reap operation; an already-created reader handle is also detached on return.
- **Impact:** Resource-pressure or wait errors can leave an execution running after dispatch reports failure, with no retained supervisor handle. On Unix, an unreaped exited direct child is also possible.
- **Fix:** Establish an owned cleanup guard immediately after spawn; commit ownership only after all reader setup succeeds. Every later error must enter the same cancellation/reaping path, subject to the bounded completion design in TERM-RUN-001.
- **Safe test idea:** Inject failure independently at each reader-start and wait operation into a fake process backend. Assert cleanup ownership is retained and cancellation/reaping is requested exactly once.

### TERM-RUN-003 — P1: Store errors are not transactional and replacement can delete the last good file

- **Evidence:** `bitty/crates/bitty-runtime/src/plugin_runtime/store.rs:92`, `:95`, `:121`, `:166`, `:174`.
- **Cause:** Both set and delete mutate `entries` before persistence succeeds. After any rename error, not just a platform-specific replacement conflict, the fallback removes the destination and retries rename. A failure or interruption between removal and replacement loses the prior committed file.
- **Impact:** `E_STORE_IO` can leave memory showing a write that failed durably, and the replacement fallback can destroy previously committed plugin settings. Temp-then-rename wording overstates the actual failure atomicity.
- **Fix:** Persist a candidate snapshot before publishing it in memory. Use a platform-correct atomic replacement primitive; never delete the previous committed file as a generic recovery step. Use uniquely owned temporary files and an explicit durability policy.
- **Safe test idea:** A fault-injected filesystem adapter should fail write, replacement, and cleanup separately; verify the previous in-memory and persisted snapshots remain intact on each rejected transaction. Include deletion transactions.

### TERM-RUN-004 — P2: Focused input uses another terminal's mode state

- **Evidence:** `bitty/crates/bitty-runtime/src/runtime/selection.rs:550`; `bitty/crates/bitty-runtime/src/runtime/input.rs:117`, `:369`, `:957`; `bitty/crates/bitty-runtime/src/runtime/panes.rs:480`, `:550`; `bitty/crates/bitty-runtime/src/runtime/layout_focus.rs:860`.
- **Cause:** Confirmed paste framing and window-focus reporting consult the primary `self.state`, while delivery correctly selects the focused pane's writer. Kitty/mouse mode caches are repaired after pane pumping, but `set_focus` and workspace loading do not refresh them at the focus transition. The pane processing swap restores the primary state before subsequent input.
- **Impact:** A pane can receive paste framing or focus notifications based on a different terminal's modes; keyboard encoding can retain the previously focused pane's protocol until a later pump. Correct destination routing alone does not establish correct per-session encoding.
- **Fix:** Resolve the focused terminal state and writer as one routing decision. Query that state's paste/focus/keyboard/mouse modes directly, or invalidate and recompute caches on every focus, workspace, close, and session replacement transition. A session-less non-owner should not inherit primary modes.
- **Safe test idea:** Use two in-memory terminal states and recording writers with different mode configurations; check focus transitions and confirmed paste without an intervening PTY pump. Existing `tests/pane_sessions.rs:112` verifies destination isolation, not encoding isolation; `tests/suspicious_paste.rs:283` covers only the primary mode state.

### TERM-RUN-005 — P2: Store loading does not enforce the store's mutation invariants

- **Evidence:** `bitty/crates/bitty-runtime/src/plugin_runtime/store.rs:50`, `:63`, `:92`, `:192`, `:231`, `:358`; `bitty/crates/bitty-runtime/src/plugin_runtime/mod.rs:944`.
- **Cause:** Loading checks a file-size ceiling and JSON syntax, then inserts entries without validating keys, entry count, per-value size, aggregate quota, or finite numeric values. Conversely, recursive set validation has no matching JSON depth ceiling, so the write and read domains differ.
- **Impact:** Persisted state can be accepted even though the same values would be rejected by set; a value accepted for writing may make the store fail to reopen at the parser depth limit. Activation then fails through `open_store`, despite a previously successful store operation.
- **Fix:** Share one bounded validation routine between candidate writes and decoded loads, accounting for the root object in depth limits. Read through a bounded reader rather than relying solely on a prior metadata size check. Validate the complete candidate before publication.
- **Safe test idea:** Property-test that every accepted bounded value round-trips through encode/load, and that invalid decoded candidates are rejected without changing a last-good store. The sampled activation test at `tests/plugin_runtime.rs:378` checks an oversized file, not decoded quotas or round-trip depth.

### TERM-RUN-006 — P2: Panel bus admission does not guarantee aggregate limits

- **Evidence:** `bitty/crates/bitty-runtime/src/registry/panel.rs:578`, `:609`, `:629`, `:675`, `:699`, `:235`, `:539`, `:551`.
- **Cause:** DropOldest admission evicts at most one event and does not recheck either aggregate byte limit. Eviction selects the first hash-map queue, which can be empty or contain an insufficiently sized event. Configured topic/subscription limits are validated but the bus uses hard-coded maxima instead of the configured lower values.
- **Impact:** The experimental panel API can exceed its stated per-panel/global envelope and ignore a user's tighter topic/subscription budgets. Arbitrary queue selection also does not implement the documented oldest-entry selection. This does not establish an unbounded-memory claim: other hard limits still exist.
- **Fix:** Compute the actual admission delta, including coalescing, then evict eligible nonempty entries until all budgets hold or reject. Maintain explicit eviction ordering if oldest-across-queues is intended. Pass validated configuration into the bus and enforce it at admission.
- **Safe test idea:** Test an abstract small-budget queue model with mixed event sizes, empty subscriptions, coalescing, and configured limits below maxima; assert every successful transition preserves all counters and limits. Existing `src/registry/panel_tests.rs:307` uses small payloads and its global branch has only ten 64-entry queues, so its asserted 8192-event ceiling is never approached.

### TERM-RUN-007 — P2: An unmounted panel can resume to Mounted without a view

- **Evidence:** `bitty/crates/bitty-runtime/src/registry/panel.rs:1055`, `:1099`, `:1106`, `:1173`, `:1186`.
- **Cause:** Unmount clears both attachment maps and `rec.view`, but leaves state `Suspended`. Resume accepts any Suspended record and changes it to Mounted without checking retained attachment. Suspension-with-attachment and unmount-without-attachment are conflated.
- **Impact:** The registry can report Mounted, and subsequently allow focus, for a panel with no host view; mounting it afterwards is rejected because Mounted is not an allowed source state. This violates the experimental API's own lifecycle invariants, independently of live app integration.
- **Fix:** Distinguish unmounted and suspended states, or require a retained consistent attachment for resume. Reconcile focus and state changes atomically with attachment maps.
- **Safe test idea:** State-machine tests should assert Mounted/Focused implies exactly one host mapping after every transition, with rejected transitions preserving the old state. Existing `src/registry/panel_tests.rs:41` tests suspend/resume with attachment; `:140` tests unmount/remount but not the invalid resume boundary.

### TERM-RUN-008 — P2: Live view allocation forgets retired identity

- **Evidence:** `bitty/crates/bitty-runtime/src/runtime/workspaces.rs:265`, `:286`; `bitty/crates/bitty-runtime/src/runtime/panes.rs:195`, `:366`.
- **Cause:** `next_view_id_global` derives the next number from current layout/stashed live IDs, not a monotonic allocation history. Once an ID disappears from both live and stashed trees, it can be returned again. Live pane APIs use bare `ViewId` without a generation check.
- **Impact:** A retained view reference cannot distinguish a retired leaf from a newly allocated leaf with the same number. Delayed operations can target the wrong incarnation. The known WS-INV-4/F-1 gap remains; live uniqueness tests do not prove retirement safety.
- **Fix:** Use a per-runtime monotonic allocator with fail-closed exhaustion and generation-bearing handles wherever reuse is possible. Include hidden/zoom-retained leaves in allocation ownership, not just currently presented trees.
- **Safe test idea:** A pure layout state-machine test should retain retired handles across deletion, stash refresh, and later allocation, then assert rejection of stale operations. No live-process reproduction was performed.

### TERM-RUN-009 — P2: Forwarder thread creation turns a recoverable OS error into panic

- **Evidence:** `bitty/crates/bitty-runtime/src/runtime/pty.rs:243`; `bitty/crates/bitty-runtime/src/runtime/panes.rs:294`.
- **Cause:** Both `Builder::spawn` results use `expect` after taking ownership of the reader. Default thread options do not make OS resource allocation infallible.
- **Impact:** Installing a waker or spawning a pane with a waker can panic instead of returning a runtime error under resource pressure. Recovery also needs to account for the reader moved into the failed closure.
- **Fix:** Make promotion fallible and preserve a usable reader/session on failure, using an ownership handoff design rather than only replacing `expect` with `?`. Propagate or explicitly record a degraded direct-poll mode.
- **Safe test idea:** Inject a failing thread factory with fake readers; assert no panic and no half-published forwarder/session. This confirms the September 15 mechanism but reduces its default priority from P1 to P2 absent frequency evidence.

### TERM-RUN-010 — P2: Panel worker teardown has no bound for an in-flight probe

- **Evidence:** `bitty/crates/bitty-runtime/src/panels_async.rs:133`, `:209`, `:212`, `:265`, `:316`.
- **Cause:** Probe callbacks receive no cancellation/deadline context. Shutdown only checks a flag between probes and joins synchronously; Drop invokes shutdown.
- **Impact:** A slow or nonterminating host probe can block owner teardown indefinitely. Queue capacity and polling interval do not bound callback duration. This is a confirmed API/lifecycle gap; no specific production UI freeze was observed or timed.
- **Fix:** Define a cancellable, bounded probe contract and isolate blocking I/O accordingly. Keep deterministic shutdown ownership without adding an unconditional UI-thread join to other lifecycle paths.
- **Safe test idea:** Use a fake cooperatively cancellable probe and completion barrier to assert the shutdown budget. Existing tests at `src/panels_async.rs:439` time snapshot reads, while teardown tests at `:470` and `:496` use immediately completing probes.

### TERM-RUN-011 — P3: Worker pending telemetry can drift beyond queue capacity

- **Evidence:** `bitty/crates/bitty-runtime/src/panels_async.rs:199`, `:224`, `:239`, `:322`.
- **Cause:** The producer increments `queued` after publishing the token. The consumer can receive and saturating-decrement before that increment. This loses a decrement; repeated handoffs can accumulate stale positive counts. Stronger atomic ordering alone does not fix the operation ordering.
- **Impact:** `pending()` can overstate outstanding work beyond its documented capacity-plus-handover bound, misleading diagnostics. The actual channel remains bounded; this is not queue-memory growth.
- **Fix:** Reserve accounting before token publication and roll it back on send failure, or derive telemetry from a synchronized queue abstraction. Define post-shutdown counter semantics.
- **Safe test idea:** Deterministic scheduling/model checks should interleave publish, receive, and accounting, asserting a drained queue eventually reports zero. Existing `src/panels_async.rs:410` checks one timed sample, not this interleaving.

## Recheck of September 15 `03-runtime.md`

Every defect heading in that report is dispositioned below. Confirmation here means the cited current mechanism was inspected, not that its old severity or reproduction language is accepted.

| September 15 claim                                                      | Current disposition                                                                                                                                                                                                                                                                                                         |
| ----------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| No Runtime Drop / detached forwarders always leak                       | Detachment remains at `runtime/panes.rs:95` and `:366`; only PanelWorker implements Drop in the source search. **Permanent leak not established.** `runtime/pty.rs:16-40` exits on EOF/send failure and joins its reader. PTY backend teardown was not audited here; absence of a Runtime Drop alone is insufficient proof. |
| Forwarder creation `expect`                                             | Confirmed; TERM-RUN-009, P2.                                                                                                                                                                                                                                                                                                |
| `tick_at` mega-function is P1                                           | Outline confirms `runtime/present.rs:685-1930`; size is maintainability risk, not a confirmed P1 defect. Body not fully reviewed.                                                                                                                                                                                           |
| Input/reply write errors swallowed                                      | Confirmed best-effort behavior at `runtime/input.rs:377-400`, `runtime/pty.rs:665-689`, and `runtime/panes.rs:523-544`. Better error counters are warranted; routine child-exit write failure is not itself evidence of a P1 outage. No new ID assigned.                                                                    |
| Unbounded diagnostic spray                                              | Per-rejection stderr calls remain at `runtime/pty.rs:480` and `:556`. OSC52 logging is behind the explicit clipboard-write gate. Rate/caller blocking impact was not measured; retain as diagnostic hardening, not demonstrated P1 severity.                                                                                |
| Esc cancellation is input hijacking                                     | **Rejected as stated.** `runtime/selection.rs:495-544` deliberately consumes dismissal; `tests/suspicious_paste.rs:509-531` explicitly requires no Esc leakage. Forwarding dismissal to the terminal would change the safety/UX contract.                                                                                   |
| PanelWorker cannot cancel an active probe                               | Confirmed, scoped as TERM-RUN-010, P2.                                                                                                                                                                                                                                                                                      |
| Spawn timeout discards output / unbounded reader join                   | Confirmed and refined as TERM-RUN-001. Normal completion joins share the deadline gap.                                                                                                                                                                                                                                      |
| 1024-chunk drains mean unbounded work / waker storm                     | The 1024 caps and per-chunk wakes remain. The normal pump chunks are bounded and tests describe 8 KiB chunks, not the prior report's 4 KiB assumption. Work can be expensive but the cap is not mathematically unbounded; latency/wake coalescing is an optimization pending measurements.                                  |
| Damage displaces meaningful cold events                                 | DropOldest behavior remains (`queue.rs:95`, `runtime/pty.rs:562`), intentionally bounded and counted. Class coalescing/prioritization needs a consumer contract; do not call every dropped observation a correctness defect.                                                                                                |
| `poll_pty_timeout` starves panes                                        | `runtime/pty.rs:365-391` waits on primary before pane drain. Pane latency can reach the caller's timeout; it does eventually drain. Scope is this blocking helper, not all app polling; no infinite starvation established.                                                                                                 |
| Workspace confirmation can silently target wrong workspace after switch | **Not confirmed.** Repeat-close checks the current index; explicit confirm targets the named armed workspace. Removal remaps/clears arms (`workspaces.rs:495-604`, `:753-764`); `:1002-1038` tests switch retention and close invalidation. Paste target binding remains a separate hypothesis below.                       |
| Runtime god object / duplicate constructors                             | Sampled duplicated initialization confirmed, optimization only; no constructor inconsistency established.                                                                                                                                                                                                                   |
| ColdQueue zero capacity inconsistent with PanelWorker                   | Both behaviors remain, but `queue.rs:56-62` documents its panic precondition and Runtime validates before construction. Not a demonstrated production defect.                                                                                                                                                               |
| Relaxed atomics cause shutdown visibility failure                       | **Not established.** No memory-model proof was provided. TERM-RUN-011 is a concrete accounting race independent of the old weak-memory conjecture.                                                                                                                                                                          |

The earlier blanket test-gap claim is inaccurate: `tests/pty_wakeup.rs:38`, `tests/real_pty.rs:30`, and `tests/pane_sessions.rs:69` spawn real shells under Unix gates. Those tests still do not prove bounded wall-clock drain/teardown or production GPU behavior. `tests/real_pty.rs:95` contains an always-true present assertion; its flood test checks output/damage, not peak memory. Registry resize loss attribution is partially tested at `src/registry/tests.rs:401`; it should not be described as wholly absent.

Also, `src/lib.rs:75-80` describes GPU attachment as future work, but `src/runtime.rs:1009-1031` already exposes attach/detach. Neither stale prose nor the existence of that API proves live GPU correctness.

## Hypotheses and optimizations — not confirmed findings

- **Pending-paste target identity:** Pending text is global, and delivery follows current focus (`runtime/selection.rs:421-474`). Decide whether focus/session replacement must invalidate consent or whether confirmation intentionally targets current focus. Do not assume workspace-close semantics apply to paste. Prefer target/session-bound confirmation for defensive clarity.
- **Primary close ownership:** Workspace live-count/kill helpers count only pane sessions (`runtime/workspaces.rs:454-486`); the primary is deliberately rehomed on close (`runtime/layout_focus.rs:807-811`). This can leave the primary hidden behind a pane-owned survivor. Clarify lifecycle policy before changing tested rehome behavior; current tests check owner liveness, not that every surviving process is interactively reachable.
- **Origin-less cold observations and global search/selection:** Pane processing swaps parser/state/overlap but invokes shared observation, search, and selection logic (`runtime/panes.rs:480-512`, `runtime/pty.rs:405-637`). Cross-pane presentation and observer attribution need a focused integration review; no complete caller/consumer proof was obtained.
- **Registry completeness:** Bus events currently use `Generation::INITIAL` (`registry/panel.rs:602`); some bus APIs omit generation and mount cannot validate terminal-registry occupancy. Separate trusted host primitives from cross-component public surfaces before claiming the accepted generation/capability contract is fully wired. No external authorization bypass is claimed here.
- **Panel focus state:** `focus_panel` sets the new record to Focused without demoting the old record. The separate MRU answers can therefore disagree with record state. Full UI consumer effects were not traced; include state/focus consistency in the lifecycle test model.
- **Performance:** Coalesce wakes and Damage notifications, budget aggregate pane drains, consolidate duplicated constructors, and split presentation phases only with behavior-preserving tests and measured workloads. No benchmark was run.

## Documentation and persistence interpretation

Consulted local sources:

- `bitty-terminal-docs/specifications/workspace-panel-invariants.md`, all 216 lines: draft status, live versus experimental layers, existing tests, and F-1/F-2 gaps.
- `bitty-terminal-docs/specifications/terminal-registry-view-lifecycle-rfc.md:120-279`: identity retirement, lifecycle, focus, resource bounds, and persistence. Remaining lines unread except targeted search context.
- `bitty-terminal-docs/specifications/panel-runtime-rfc.md:141-177` and `:383-409`: accepted mount/suspend distinction and queue envelope; other focus/persistence headings sampled by search only.
- `research/summary/017.md` and `research/summary/039.md`, full: invariant-hardening priority and separation of panel/activity/terminal ownership.
- `research/review/2026-09-15/03-runtime.md`, all 152 lines, rechecked above.
- Runtime `Cargo.toml` and `README.md`, full, and selected `src/lib.rs` docs. They are maps, not proof that deferred or implemented claims remain current.

The terminal registry creates a new empty `State` at `src/registry/terminal.rs:164`; its recreation tests at `src/registry/tests.rs:454-488` assert fresh IDs, not restored scrollback. No workspace serialization implementation was established by the selected reads and persistence searches. Canonical WS-INV F-2 already calls out that gap. Do not mistake in-memory scrollback/selection persistence or plugin JSON storage for session restart survival. Missing post-v1 daemon/session survival is not logged as a new defect.

## Exact coverage ledger

Tracked inventory: **49 Rust files under `src`, 67 Rust integration-test files, 3 fixture files, Cargo.toml, and README.md**. There are 116 Rust files total. The 49 source files include two dedicated test modules; production files also contain inline tests. Classification is by source-body coverage, not test execution. Full means every source line was read; sampled means only the listed inclusive ranges were read. Outline-only and search-only bodies are not audited coverage.

### Full source-body reads

- `src/plugin_runtime/store.rs:1-554`
- `src/registry/panel.rs:1-1490`
- `src/registry/panel_tests.rs:1-419`
- `src/queue.rs:1-154`

### Sampled source-body reads

| File                             | Exact ranges                                    |
| -------------------------------- | ----------------------------------------------- |
| `src/ai_panel.rs`                | 1-55, 140-239                                   |
| `src/browser_panel.rs`           | 267-426                                         |
| `src/host_bridge.rs`             | 1-206                                           |
| `src/lib.rs`                     | 1-182                                           |
| `src/mail_panel.rs`              | 1-58, 310-383                                   |
| `src/panels_async.rs`            | 1-335, 338-514                                  |
| `src/plugin_runtime/mod.rs`      | 526-714, 734-850, 944-969                       |
| `src/plugin_runtime/services.rs` | 198-215                                         |
| `src/plugin_runtime/spawn.rs`    | 672-831                                         |
| `src/registry/terminal.rs`       | 126-185, 272-322, 511-810, 1050-1143, 1182-1209 |
| `src/registry/tests.rs`          | 109-211, 239-317, 401-488                       |
| `src/runtime.rs`                 | 262-340, 698-740, 790-875, 930-991, 1002-1037   |
| `src/runtime/input.rs`           | 117-161, 184-428, 941-1043                      |
| `src/runtime/layout_focus.rs`    | 784-898                                         |
| `src/runtime/panes.rs`           | 1-567                                           |
| `src/runtime/plugin.rs`          | 1-28                                            |
| `src/runtime/pty.rs`             | 1-40, 199-249, 314-391, 405-637, 665-701        |
| `src/runtime/selection.rs`       | 398-554                                         |
| `src/runtime/workspaces.rs`      | 250-414, 454-792, 936-1038                      |

Outline-only source files: `src/workspace.rs`, `src/runtime/close_confirm.rs`, `src/runtime/present.rs`. Outlines can be folded by ctxctl; only displayed definitions were inspected.

Unread source bodies (23 files; incidental search hits do not upgrade coverage):

- `src/config.rs`, `src/error.rs`, `src/inspect.rs`, `src/palette.rs`, `src/paste.rs`
- `src/project.rs`, `src/project_scope.rs`, `src/queries.rs`, `src/registry.rs`
- `src/shell_integration.rs`, `src/statusline.rs`, `src/tabs.rs`
- `src/plugin_runtime/manifest_toml.rs`, `src/plugin_runtime/package.rs`, `src/plugin_runtime/resolution.rs`
- `src/runtime/animations.rs`, `src/runtime/background_images.rs`, `src/runtime/help.rs`, `src/runtime/kitty_images.rs`
- `src/runtime/mouse_chrome.rs`, `src/runtime/resize.rs`, `src/runtime/scrollbar.rs`, `src/runtime/search.rs`

### Integration tests

Full reads: `tests/pane_sessions.rs:1-275`, `tests/pty_wakeup.rs:1-122`, `tests/real_pty.rs:1-131`.

Sampled reads:

- `tests/panel_session_invariants.rs:107-214,270-319,344-429`
- `tests/plugin_runtime.rs:118-195,267-404`
- `tests/suspicious_paste.rs:283-321,509-553`

Outline-only: `tests/plugin_store.rs`. Despite its name, this file primarily covers package installation/index resolution, not the key/value store's failure atomicity.

Unread integration-test bodies (60 files; some had targeted search hits):

- `background_images_present.rs`, `browser_panel.rs`, `bundled_dogfood_runtime.rs`, `close_confirm.rs`, `cwd_inherit.rs`, `decoration_present.rs`, `dogfooding.rs`, `file_manager_panel.rs`, `final_integration.rs`, `font_zoom.rs`
- `git_panel.rs`, `help_popup.rs`, `ime_input.rs`, `input_snap_to_live.rs`, `keyboard_input.rs`, `kitty_apc_wiring.rs`, `kitty_images_present.rs`, `kitty_origin_binding.rs`, `mail_panel.rs`, `mouse_chrome.rs`
- `nested_tmux_present.rs`, `palette_statusline_project_panel.rs`, `pane_da_wakeup.rs`, `pane_damage.rs`, `panel_animations.rs`, `panel_gaps.rs`, `panel_invariants.rs`, `panel_live_framehash.rs`, `plugin_package.rs`, `project_scope.rs`
- `pty_reply.rs`, `resize_scrollback.rs`, `runtime_input.rs`, `runtime_layout.rs`, `runtime_panes.rs`, `runtime_plugin.rs`, `runtime_present.rs`, `runtime_resize.rs`, `runtime_soft_present.rs`, `scroll_follow.rs`
- `scrollback_cap.rs`, `scrollback_search_selection_persistence.rs`, `scrollback_search_ui_integration.rs`, `scrollbar.rs`, `selection_clear.rs`, `selection_clipboard.rs`, `shell_integration_panel.rs`, `soak.rs`, `spawn_surface.rs`, `split_live_reflow.rs`
- `split_zoom_repaint.rs`, `tabs_compat.rs`, `terminal_queries.rs`, `underline_thickness_mirror.rs`, `v01_minimal_terminal.rs`, `window_padding_opacity.rs`, `window_radius_noop.rs`, `window_winit.rs`, `workspace_decoration.rs`, `workspace_panel.rs`

All three fixtures unread: `tests/fixtures/bundled-sample/bitty-plugin.toml`, `tests/fixtures/bundled-sample/lua/helper.lua`, `tests/fixtures/bundled-sample/lua/init.lua`. No other crate source bodies or reference-repository implementations were audited.

## Checks and remaining gaps

- Static review only: no Rust tests, clippy, rustfmt, typecheck, GPU/PTY activity, fault injection, or cross-platform execution. No test-pass or exhaustive-review claim is made.
- Existing tests were evaluated by their assertions and gates. Presence of a test does not establish it ran on this checkout, and randomized tests cover their selected sequences only.
- Report lint command: `markdownlint-cli2 --no-globs review/2026-09-17/bitty-terminal/02-runtime-orchestration.md`, from `research`, using its existing `.markdownlint-cli2.jsonc`; no fix mode or network fetch.
- Final verification must read the report back and compare Git HEAD/status against the baseline. Only this report is intended to differ; existing `.targets/` and sibling reports must remain untouched.
- Highest-value follow-up: deterministic fault-injection seams for process ownership and store commits, followed by per-session input-mode and panel lifecycle invariant tests. End-to-end application exposure, backend process-tree cleanup, Windows replacement semantics, and complete rendering behavior remain outside this sampled review.
