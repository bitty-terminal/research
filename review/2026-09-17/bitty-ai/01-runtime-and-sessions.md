# Runtime and session review — 2026-09-17

## Scope and verdict

Read-only source review of `bitty-ai/crates/bitty-ai-runtime`, concentrating on session lifecycle, sequential scheduling, tool execution, permission seams, provider contracts, cancellation, reconciliation, and recovery declarations. No source edits, commits, installations, network fetches, subagents, CarryCtx operations, exploit execution, or reproduction files were used. The first pass authored only this report; the second pass edits both reports and creates their campaign product index.

**Second-pass disposition: 1 P1 and 5 P2 correctness/resource defects; 2 P2 contract/integration gaps; 1 P3 evidence-description issue.** AI-RUN-002 and AI-RUN-006 are reclassified below, not silently counted as proven execution failures. All IDs are preserved. These are static findings, not executed reproductions. Existing assertions show current expectations, not passing-test evidence.

The original method/coverage ledger below records the first pass only. The independent verification section records the second pass's narrower reads, current baselines, corrections, and limitations; it does not inherit the first pass's full-file coverage.

The runtime is explicitly an experimental, synchronous, std-only skeleton. Absent network adapters, durable persistence, multi-agent scheduling, and real capability enforcement are not themselves defects against its current scope. `bitty-ai-slice` and the context/cache pipeline are left to the subsequent reviewer.

Paths beginning `crates/` below are relative to the `bitty-ai` repository. Documentation and historical-review paths name their separate repositories explicitly.

## Baseline and method

| Repository      | Initial HEAD                               | Initial dirty state            | Pre-report recheck                                                                                                                             |
| --------------- | ------------------------------------------ | ------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `bitty-ai`      | `e7cbe69f0fac8a785657b5ff0bab850b66e9f690` | Clean                          | Review HEAD advanced to `97d312585c20a5b17063084cf66ba1bb7438667c` (AI-0097); see below; clean; `git diff --check` passed at the baseline HEAD |
| `bitty-ai-docs` | `bb66faf88d0a9cff4ea753671e0dbefb5fadea80` | Clean                          | Same HEAD; clean                                                                                                                               |
| `research`      | `26a46ed8c15aa4e714ca9add4ff4c07bea02ff22` | Untracked `review/2026-09-16/` | HEAD became `d70152a079c75204ec37e99a7bf3f54b8190e423`, clean before this report                                                               |

The umbrella directory is not a Git repository. The research HEAD transition occurred independently during this review; this reviewer performed no commit or cleanup. Historical reports were read from the working tree, including the initially untracked September 16 files. The docs submodule pin was not refreshed or compared with the standalone docs HEAD; intended-behavior citations use the standalone `bitty-ai-docs` checkout.

Read workspace, implementation-repository, documentation-repository, and research `AGENTS.md` guidance. No nested `AGENTS.md` was found under runtime crates or research. Read the ctxctl skill; its MCP outline endpoint rejected the external workspace path, so installed `ctxctl outline` was used inside `bitty-ai`, followed by line-numbered source reads. Folded outlines count only as discovery, not full coverage.

The requested `bitty-ai-docs/research/summary` directory does not exist in this checkout. The actual shared summaries are under `research/summary`; relevant records there and canonical AI specifications were read instead. No remote material was fetched.

During report writing, `bitty-ai` advanced from `e7cbe69` to `97d3125` ([AI-0097], PR #185), adding two new files and updating exports: `crates/bitty-ai-runtime/src/fallback.rs` (+364), `src/lib.rs` (+5), and `tests/fallback_envelope.rs` (+385). All findings and coverage claims were made against `e7cbe69`; the new fallback-envelope code was not reviewed and is not covered by any finding in this report.

## Intended behavior

- `research/summary/023.md:11-18` selects deterministic fake-provider verification first, one cohesive runtime, and a frozen multi-agent scope. `bitty-ai-docs/specifications/implementation-profile-v0.1.md:14-17` explicitly makes that profile experimental, not accepted architecture.
- `research/summary/032.md:11-16` and `bitty-ai-docs/specifications/provider-plugin-boundary.md:26-34` place vendor integrations and credentials outside Core. The std-only runtime is consistent with that direction.
- `research/summary/012.md:11-14` separates tool counts from model round trips. Current batch execution is sequential, not concurrent; that narrower implementation is stated in `tests/batch_evidence.rs:3-8`.
- `bitty-ai-docs/specifications/persistence-profile-r6.md:224-235` requires reconciliation before retry and preservation of actual/Unknown outcomes, without reversal claims. Lines 237-264 explicitly defer background scheduling and durable storage.
- `research/summary/037.md:11-17` describes future session-sharded storage, not implemented persistence. `adoption.rs` and `fencing.rs` are declarative checks; they do not perform durable adoption or cross-process fencing.
- `research/summary/040.md:14-20` keeps extension contributions separate from permission grants. `extension.rs:12-20` correctly describes declaration-only traits, not an enforcement system.

## Findings and second-pass dispositions

### AI-RUN-001 — P1: Starting another turn discards unresolved effects

**Evidence:** `crates/bitty-ai-runtime/src/agent.rs:467-478`, `:505-511`, `:833-840`, `:906-911`; `crates/bitty-ai-runtime/tests/agent_turn_semantics.rs:169-226`; `crates/bitty-ai-runtime/tests/turn_lifecycle.rs:101-149`.

**Cause:** An Unknown outcome leaves the session Active. The next `run_turn` checks lifecycle state but not outstanding Unknown records, then clears all execution history before starting new work. The existing multi-turn test explicitly expects the earlier unresolved ID to become `NoUnknown` after the second turn.

**Independent verification — retained P1, narrowed impact:** At current source HEAD `97d312585c20a5b17063084cf66ba1bb7438667c`, `agent.rs:509` executes `self.executions.clear();`; `:906-911` searches only that vector and returns `ReconcileOutcome::NoUnknown` when the ID is absent. This proves loss of the agent's reconciliation lookup after a subsequent admitted turn, not loss of every possible host-owned copy of the evidence. Per-turn clearing is explicitly intentional (`agent.rs:389-391`), but no separate pending-effect lookup survives it.

**Contract distinction:** `bitty-ai-docs/specifications/persistence-profile-r6.md:227-229` says “Reconcile by state inspection or user direction before retry” and “Retry eligibility is decided per effect against current state.” That supports preserving unresolved evidence; it does **not** require blocking every unrelated turn. A blanket session gate is one possible remediation, not a proven contract requirement. No automatic retry, deployed effect duplication, or durable-record loss was established.

**Remediation:** Keep a separately bounded pending-effect set, or refuse turn admission while unresolved effects remain. Clearing ordinary turn counters must not remove pending reconciliation state. Any user-directed abandonment should be explicit and preserve attribution.

**Regression suggestion:** Assert pending IDs remain queryable across admitted unrelated turns and cannot authorize a blind retry of the same effect. If a blanket gate is selected instead, assert its refusal explicitly. Replace the lost-ID expectation while retaining independent per-turn cost/count reset assertions.

### AI-RUN-002 — P2 integration gap: No dispatch identity in the executor seam

**Reclassified by independent verification:** The missing parameter is proven; an inability to reconcile correctly in every host is not. `agent.rs:393-395` exposes the attributed records after a synchronous call, and a host can associate its own ordered ledger with those records or carry an application identity in arguments. Neither workaround is a first-class runtime dispatch-identity contract. Retain P2 as adapter-design work before durable/effectful integration, not as a demonstrated wrong-effect lookup. No production reconciler adapter was traced.

**Evidence:** `crates/bitty-ai-runtime/src/agent.rs:605-613`; `crates/bitty-ai-runtime/src/tool.rs:495-508`, `:712-727`, `:742-763`; `crates/bitty-ai-runtime/src/reconcile.rs:106-115`.

**Cause:** The runtime issues an `ExecutionId` before dispatch, but `ToolExecutor::execute` receives only tool name, arguments, and timestamp. `UnknownReconciler::reconcile` later requires that runtime-issued ID. The bus attaches the ID only to its returned record, after the executor call.

**Impact, narrowed:** The executor lacks the runtime-issued key at dispatch time. Repeated calls may have distinct arguments or host sequence numbers; the original blanket indistinguishability claim is withdrawn. Correlation across acknowledgement loss and future persistence remains an explicit adapter obligation, not a currently proven runtime misattribution.

**Remediation:** Pass a typed execution context containing the dispatch identity into the executor, and use the same identity for status queries. Specify identity ownership and lifetime before adding durable adapters; current process-local IDs must not become persistent external IDs accidentally.

**Regression suggestion:** Use a benign recording executor/reconciler pair to assert exact identity propagation for multiple same-tool calls and that status lookup addresses the corresponding dispatch, without re-execution.

### AI-RUN-003 — P2: Configured tool cap and batch admission disagree across rounds

**Evidence:** `crates/bitty-ai-runtime/src/agent.rs:64-65`, `:505`, `:569-575`, `:585-614`; `crates/bitty-ai-runtime/src/tool.rs:30-39`, `:685-694`, `:720-727`; `crates/bitty-ai-runtime/tests/batch_evidence.rs:310-315`.

**Cause:** The configurable cap is checked against each provider response's vector length, while the hard bus counter is cumulative for the whole `run_turn`. `precheck` checks only batch length and ignores calls already dispatched. The documented effective per-turn minimum is therefore not enforced consistently.

**Impact:** A configuration tighter than eight can be exceeded cumulatively by several smaller rounds. Conversely, a later otherwise-valid batch can pass precheck, execute only its prefix, and fail when the cumulative hard counter reaches eight. That contradicts predictable all-or-nothing admission for a known budget limit. The test comment that this per-call counter failure is unreachable through `run_turn` is only true for a first oversized batch, not later rounds.

**Remediation:** Define the accounting scope once. For a logical-turn cap, compare the entire batch with both the configured remaining allowance and the bus's remaining hard allowance before dispatch. Keep the per-call check as defense in depth.

**Regression suggestion:** Cover several provider rounds under a tight configured cap and a later batch that does not fit the remaining hard allowance. Assert zero dispatch from the rejected batch and preservation of earlier effects.

### AI-RUN-004 — P2: Post-effect result rejection is recorded as execution failure

**Evidence:** `crates/bitty-ai-runtime/src/tool.rs:726-740`; `crates/bitty-ai-runtime/src/agent.rs:649-657`; `crates/bitty-ai-runtime/tests/runtime_fail_closed.rs:516-549`. Compare the correctly preserved success status on artifact-storage failure at `tests/runtime_fail_closed.rs:604-608`.

**Cause:** The executor can acknowledge success, after which the bus rejects an oversized summary or body with an ordinary `ToolError`. The agent maps every dispatch error to `ToolStatus::Failed`, whose definition says the host executed and reported failure (`tool.rs:422-427`). This also affects a just-in-time authorization refusal after batch admission, as asserted at `tests/runtime_fail_closed.rs:471-480`. Initial precheck refusals do not create such execution records; executor-returned Denied is correctly preserved at `tool.rs:765-772`.

**Impact:** The evidence model conflates whether an effect ran with whether its result could be accepted. A successful effect is represented as Failed; a call never sent to the executor can also appear as a failed execution. Consumers of records, cancellation counts, and adoption history cannot reliably distinguish these cases. No actual automatic re-execution is claimed here.

**Remediation:** Separate effect status, dispatch admission, and result-validation/storage status. Preserve acknowledged success when payload acceptance fails; represent pre-dispatch rejection as a refusal rather than an executed failure.

**Regression suggestion:** Check result-bound failures after a benign acknowledged effect and authorization failures before executor contact. Assert distinct effect states and accurate attempted/admitted/completed counts.

### AI-RUN-005 — P2: Cancellation during the last text emission can return Completed

**Evidence:** `crates/bitty-ai-runtime/src/stream.rs:307-318`; `crates/bitty-ai-runtime/src/agent.rs:560-567`, `:725-733`; `crates/bitty-ai-runtime/src/session.rs:384-399`.

**Cause:** Fragment emission checks cancellation before each sink callback but not after the last callback. A sink can retain a cloned session handle and request cancellation while accepting that final chunk. Emission still returns success; a text-only provider round then returns Completed. `finish(false)` preserves the already-Canceled state.

**Impact:** The method outcome says Completed while the shared session says Canceled. Downstream completion handling and cancellation acknowledgement disagree at a supported synchronous callback boundary. Existing stream tests cover cancellation before emission, not cancellation from the final sink callback.

**Remediation:** Recheck cancellation after emission and immediately before declaring completion, using the same reconciliation path as other cancellation boundaries.

**Regression suggestion:** Have a benign sink request cancellation when accepting the final text chunk. Assert a cancellation-consistent outcome/state while preserving already accepted bytes; also cover a multi-chunk final batch.

### AI-RUN-006 — P2 contract gap: Invocation budget is described as execution-wide

**Reclassified by independent verification:** Frozen timestamps, reported rather than awaited delays, re-invocation after escalation, and terminal-session non-resumption are intentional (`agent.rs:873-898`; `session.rs:392-400`). They are not independently proven scheduling or lifecycle defects. The definite inconsistency is between those invocation semantics and the execution-wide/no-further-retry wording at `reconcile.rs:21-24,36-48,59-65`; `agent.rs:862-864` also overstates that resolution always leaves the session Active. The original implication that later resolution must reactivate a Failed session is withdrawn.

**Evidence:** `crates/bitty-ai-runtime/src/reconcile.rs:36-48`, `:59-65`; `crates/bitty-ai-runtime/src/agent.rs:873-898`, `:906-968`; `crates/bitty-ai-runtime/tests/unknown_reconcile.rs:300-355`.

**Cause:** Attempts are a local counter initialized on every call. All attempts execute immediately at the same timestamp; exhaustion fails the session but leaves the Unknown record unchanged. Re-invocation is explicitly documented and accepted even after escalation, resetting the budget. The time-advancement test demonstrates this and checks resolution, but not the resulting still-Failed session state.

**Impact, narrowed:** Callers cannot rely on an execution-wide query maximum from these declarations; only each invocation is capped. The clock-advance test demonstrates later status resolution, not session resumption. The contradiction is in contract wording, not evidence that frozen-clock evaluation or retaining a Failed state is itself incorrect.

**Remediation, revised:** First reconcile the public contract: document a per-invocation budget, post-escalation inspection, and unchanged terminal session state if these are the intended semantics. Only if an execution-wide budget is selected should per-effect counters/deadlines and Pending scheduling be introduced. Later resolution must not silently reopen terminal sessions.

**Regression suggestion:** Assert cumulative query counts and exact state transitions across caller-advanced timestamps, including repeated pre-deadline calls, final exhaustion, and manual inspection after escalation.

### AI-RUN-007 — P2: Adoption allocates from an unchecked auxiliary list

**Evidence:** `crates/bitty-ai-runtime/src/adoption.rs:104-118`, `:314-319`, `:335-352`; `crates/bitty-ai-runtime/src/adoption.rs:499-503`.

**Cause:** The initial bound checks only survivor IDs and evidence length. `unknown_effects` is omitted, yet its length is passed directly to `Vec::with_capacity` before membership/duplicate rejection.

**Impact:** The advertised 32-survivor bound does not bound the check's allocation. An invalid declaration can cause allocation proportional to an unrelated, oversized list before being refused. This is a defensive resource-bound issue in the public seam, not a claim of a deployed remote entry point.

**Remediation:** Validate all collection lengths before allocating or inspecting content. The disposition list cannot legitimately exceed the survivor set or the hard survivor cap; use that bound for scratch capacity as well.

**Regression suggestion:** Add ordinary boundary-value tests for disposition count independent of survivor/evidence count. Assert typed size refusal before content validation; no stress workload is needed.

### AI-RUN-008 — P2: Outbound error/status normalization is incomplete

**Evidence:** `crates/bitty-ai-runtime/src/tool.rs:183-195`, `:665-669`, `:765-768`; `crates/bitty-ai-runtime/src/agent.rs:273-276`, `:636-645`, `:939-945`; `crates/bitty-ai-runtime/src/reconcile.rs:190-204`. The stated shared outbound policy is at `crates/bitty-ai-runtime/src/bridge.rs:97-118`.

**Cause:** Provider transport errors and tool Unknown reasons use the scrub-aware helper, but malformed tool names are copied into errors verbatim; authorizer/executor denial reasons and reconciled terminal-status reasons are retained without that bound. Pending reconciliation reasons use a separate truncate-only helper and then reach an outbound escalation report.

**Impact:** These runtime-owned error/record surfaces can retain unbounded host text or display control characters despite the documented bounded, single-line boundary. A failed name check does not make its echoed value safe. This finding concerns diagnostic integrity and resource discipline, not a permission bypass.

**Remediation:** Apply one bounded diagnostic normalization policy at every host/model-to-runtime error/status boundary, including resolved statuses. Keep raw observation data separate from display-safe diagnostic strings where fidelity is required.

**Regression suggestion:** Add non-operational table-driven assertions for length, UTF-8 validity, and display-safe characters on all error/status conversion branches. Include denial and reconciliation outcomes rather than only Unknown/provider transport errors.

### AI-RUN-009 — P3: MCP version-source tests overstate what the registry proves

**Evidence:** `crates/bitty-ai-runtime/tests/mcp_fail_closed.rs:3-10`, `:17-30`, `:100-130`, `:192-223`; `crates/bitty-ai-runtime/src/tool.rs:201-213`, `:263-270`, `:301-324`.

**Cause:** The test equates a locally computed schema digest with an externally supplied version source. Every byte string, including an empty schema, has a digest. The same test accepts initial registration of a foreign spec expressly described as arriving without a version source. Production registration contains no provenance/version-source field or check.

**Impact:** The suite proves immutable same-name registration, local schema-byte identity, unknown-name refusal, and default denial. It does not prove rejection when an external version source is unavailable. Its NO-GAP language can incorrectly close an integration evidence requirement. No live MCP transport vulnerability is asserted: there is no MCP transport in this crate.

**Remediation:** Narrow the test/evidence description to the actual invariants. Keep external version-source availability an open adapter admission requirement; if later implemented, represent that evidence separately from a content digest.

**Regression suggestion:** At the future importer boundary, distinguish absent version/provenance evidence from changed schema bytes. Assert refusal for the former and version-bound admission for the latter without adding transport code to this skeleton merely to satisfy a test name.

## Validation of earlier reviews

### September 15: `research/review/2026-09-15/07-bitty-ai.md`

| Earlier claim                                    | Current-source disposition                                                                                                                                                                                                                         |
| ------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| P0-1 stale per-turn authorization level          | Original scenario fixed: fresh base at `agent.rs:588` and `:610`; regression at `runtime_fail_closed.rs:388-483`. This does not establish a general reentrant-callback proof.                                                                      |
| P0-2 silent empty result on artifact failure     | Fixed at caller: `agent.rs:620-621` and `:753-760` propagate failure while retaining effect attribution; regression `runtime_fail_closed.rs:553-616`.                                                                                              |
| P1-1 unbounded execution history                 | Original accumulation fixed by clearing/cap; however, the new loss of unresolved evidence is AI-RUN-001.                                                                                                                                           |
| P1-2 terminal-session re-entry                   | Fixed at entry, `agent.rs:473-478`; completed/failed cases in `turn_lifecycle.rs:153-263`. Mid-turn lifecycle changes are a separate concern.                                                                                                      |
| P1-5 repeated sequence zero within one turn      | Fixed: `agent.rs:720-735`, `stream.rs:293-318`, tests `stream.rs:353-398`. Cross-turn transport identity is not proven; see concerns below.                                                                                                        |
| P1-6 per-agent identity collisions               | Fixed for IDs issued by `IdIssuer`: process-global atomic at `session.rs:108-145`; interleaved-issuer test at `:469-483`. Public raw numeric constructors remain host-owned.                                                                       |
| P1-7 Rc/Cell incompatible with all async use     | Reclassified as declared scope, not a runtime defect: `session.rs:284-320` documents a single-thread driver and `:551-567` asserts !Send. !Send does not universally prohibit await on a local executor.                                           |
| P2-1 unreachable oversized-chunk variant         | Removed and layer ownership documented at `stream.rs:19-28`, `:109-117`; fragment check at `:179-183`. Transport encoding was not reviewed.                                                                                                        |
| P2-5 undocumented hard/config cap minimum        | Minimum now documented in `tool.rs:30-39`, with raised-config test. Cross-round enforcement still disagrees with that statement: AI-RUN-003.                                                                                                       |
| P2-6 alias fall-through and zero-weight split    | Both fixed: alias-exclusive branch at `selection.rs:575-591`; weights at `:747-764` agree with `agent.rs:996-998`. Relevant selection assertions read through `:1562`.                                                                             |
| P2-7 outbound reason strings                     | Partially addressed in bridge/provider conversion; not complete across tool/reconciler seams: AI-RUN-008.                                                                                                                                          |
| P2-9 frozen reconcile timestamps                 | Now explicitly documented and tested, not an undocumented assumption. Per-execution budget/lifecycle mismatch remains AI-RUN-006. Full-history input charging alone is not confirmed as double counting: each round actually resends that history. |
| Slice claims; context assembly and prompt claims | Deferred to the other reviewer; not repeated as current findings.                                                                                                                                                                                  |

### September 16 claims

- The scoped runtime remains synchronous and dependency-free: `provider.rs:767-790`, `session.rs:284-320`, and the empty dependency section in runtime `Cargo.toml:16-17` support the narrower claims in `research/review/2026-09-16/02-code-vs-docs-progress-audit.md:118-124`. The reviewed runtime provider implementation is FakeProvider. This is not proof about every provider in the whole workspace.
- The old zero-weight fix claim is supported by current arithmetic and selection tests. Claims about the broader accounting test suite were not independently verified by executing it.
- Eight extension traits exist, but they prove Rust interface separation, not deployed capability isolation. The broad wording at that review's `:143-144` should be narrowed to `extension.rs:12-20`.
- No durable persistence is implemented in the reviewed lifecycle code; absence is consistent with the v0.1 profile, not automatically a defect. The assertion that every state transition can be serialized/replayed at `research/review/2026-09-16/04-architectural-drift-and-subjective-analysis.md:116` is not supported by a serialization/replay API in the reviewed files.
- The same report's `:126-132` inference that network-free Core is an operational dead end is subjective, not a confirmed source defect. External adapters are the intended boundary. Slice/host claims, cache-marker claims, cross-repository submodule lag counts, and context/prompt claims are not validated here and must not be treated as current findings from this report.

## Optimizations, separate from defects

1. `agent.rs:522-526` clones the full conversation and references each round; `stream.rs:253-271` materializes all fragments before `emit_fragments` clones them again. With B bytes, fragmentation is O(B) time and O(B) temporary space; a borrowing iterator could reduce fragment scratch space to O(fragment size). Measure allocations before changing the API; no benchmark was run.
2. Registry lookups and bounded adoption set checks are linear scans. Adoption is O(S² + U·S) time and O(S + U) scratch space for S survivors and U dispositions, excluding cloned string bytes. After fixing the unchecked U bound, the cap of 32 makes a hash-based rewrite unnecessary without measurements.
3. The zero-weight helper exists in both agent and selection modules. A single internal helper would reduce semantic drift; current results agree, so this is maintenance only.

## Unverified concerns and integration limitations

- A host callback can mutate a cloned session while elevation or authorization is being evaluated. Current code rechecks some boundaries but does not carry an immutable generation/lifecycle guard through all callbacks (`session.rs:414-422`, `tool.rs:659-665`, `agent.rs:594-613`). Host-side reentrancy assumptions were not verified; no operational scenario is provided or claimed as a separate security finding.
- `StreamChunk` has no turn identity; `next_seq` restarts on re-entry. `agent_turn_semantics.rs:716-775` explicitly documents the need for turn-scoped transport deduplication. The assertion in `stream.rs:10-13` must not be read as cross-turn safety. Transport remapping is left to the slice reviewer.
- Provider request validation is delegated to adapter implementations. FakeProvider validates budget, timeout, and sampling; the generic agent does not independently validate every returned field or enforce a whole-response byte cap. Real adapter conformance, pre-I/O authorization, cancellation during a blocking call, and latency guarantees remain unverified here.
- Writer leases and adoption claims are caller-attested declarations, not authenticated cross-process capabilities. Writer identity is explicitly not checked by `fencing.rs:63-65`. Session binding, epoch persistence/election, and transport enforcement must be examined in the future host adapter, not inferred from equality checks.
- `UnknownDisposition::Reconciled` is currently refused for both Unknown and terminal evidence (`adoption.rs:378-395`). The declared workaround omits dispositions on terminal evidence; the sampled recovery test adopts later-turn records rather than the previously reconciled record. This is an awkward/provisional API, not additional proof of unsafe adoption.
- Tool result bodies are retained internally while subsequent provider messages carry summaries (`agent.rs:758-775`, `:784-795`). Artifact accessibility and context-projection semantics are handed off to the context reviewer, rather than investigated here.

## Exact coverage ledger

Full means every source line was returned and read, including inline tests. Sample means only the listed ranges were read; outlines do not upgrade coverage. Blank separators between sampled ranges are not implicitly counted. No coverage percentage or exhaustive-test claim is made.

### Runtime source

All paths in this table are under `bitty-ai/crates/bitty-ai-runtime/`.

| File                 | Coverage                            | Exact lines                                                |
| -------------------- | ----------------------------------- | ---------------------------------------------------------- |
| `src/agent.rs`       | Full                                | 1-1061                                                     |
| `src/session.rs`     | Full                                | 1-568                                                      |
| `src/tool.rs`        | Full                                | 1-1107                                                     |
| `src/provider.rs`    | Full                                | 1-1051                                                     |
| `src/stream.rs`      | Full                                | 1-451                                                      |
| `src/reconcile.rs`   | Full                                | 1-332                                                      |
| `src/bridge.rs`      | Full                                | 1-1337                                                     |
| `src/adoption.rs`    | Full                                | 1-609                                                      |
| `src/fencing.rs`     | Full                                | 1-228                                                      |
| `src/lib.rs`         | Full                                | 1-145                                                      |
| `src/selection.rs`   | Sample; all production definitions  | 1-817, 1038-1562                                           |
| `src/extension.rs`   | Sample; all production declarations | 1-182                                                      |
| `src/context.rs`     | Deferred                            | Not read; initial line-count inventory only                |
| `src/compression.rs` | Deferred                            | Not read; initial line-count inventory only                |
| `src/prompt.rs`      | Deferred                            | Not read; initial line-count inventory only                |
| `src/cache_key.rs`   | Deferred                            | Not read; initial line-count inventory only                |
| `src/fingerprint.rs` | Deferred                            | Not read; initial line-count inventory only                |
| `README.md`          | Full                                | 1-46                                                       |
| `Cargo.toml`         | Full                                | 1-17                                                       |
| `src/fallback.rs`    | Not read                            | Added by AI-0097 after review baseline; follow-up reviewer |

**Source coverage: 10 complete Rust files, 2 sampled Rust files, 5 explicitly deferred Rust files.** All production definitions in the 12 reviewed modules were inspected; the unreviewed unit-test portions of selection and extension remain excluded from full-file counts.

### Runtime integration tests

All paths are under `bitty-ai/crates/bitty-ai-runtime/tests/`.

| File                          | Coverage     | Exact lines                                                |
| ----------------------------- | ------------ | ---------------------------------------------------------- |
| `agent_turn_semantics.rs`     | Full         | 1-775                                                      |
| `turn_lifecycle.rs`           | Full         | 1-338                                                      |
| `unknown_reconcile.rs`        | Full         | 1-436                                                      |
| `runtime_fail_closed.rs`      | Full         | 1-898                                                      |
| `cost_ceiling.rs`             | Full         | 1-348                                                      |
| `batch_evidence.rs`           | Full         | 1-343                                                      |
| `writer_fencing.rs`           | Full         | 1-255                                                      |
| `mcp_fail_closed.rs`          | Full         | 1-291                                                      |
| `recovery_adoption.rs`        | Sample       | 367-532; outline for discovery                             |
| `result_schema_disclosure.rs` | Outline only | No body-read coverage                                      |
| `sampling.rs`                 | Outline only | No body-read coverage                                      |
| `extension_points.rs`         | Outline only | No body-read coverage                                      |
| `accounting_bounds.rs`        | Not read     | Deferred supplemental coverage                             |
| `cache_invalidation.rs`       | Not read     | Context/cache reviewer                                     |
| `cache_key.rs`                | Not read     | Context/cache reviewer                                     |
| `granularity_denial.rs`       | Not read     | Context/cache reviewer                                     |
| `input_fingerprint.rs`        | Not read     | Context/cache reviewer                                     |
| `schema_invalidation.rs`      | Not read     | Supplemental schema/cache coverage remains open            |
| `subscription_bounds.rs`      | Not read     | Supplemental accounting coverage remains open              |
| `fallback_envelope.rs`        | Not read     | Added by AI-0097 after review baseline; follow-up reviewer |

**Integration-test coverage: 8 complete files, 1 sampled file, 3 outline-only files, 7 unread files.** No Rust tests were executed.

### Guidance, contracts, and historical material

| Repository-relative path                                                       | Coverage                                                 |
| ------------------------------------------------------------------------------ | -------------------------------------------------------- |
| Workspace `AGENTS.md`                                                          | Full, 1-276                                              |
| `bitty-ai/AGENTS.md`                                                           | Full, 1-152                                              |
| `bitty-ai-docs/AGENTS.md`                                                      | Full, 1-147                                              |
| `research/AGENTS.md`                                                           | Full, 1-47                                               |
| Installed `ctxctl-core/SKILL.md`                                               | Full, 1-132; initial CLI read plus remaining skill lines |
| `bitty-ai/Cargo.toml`                                                          | Full, 1-32                                               |
| `research/.markdownlint-cli2.jsonc`                                            | Full, 1-11                                               |
| `bitty-ai-docs/specifications/implementation-profile-v0.1.md`                  | Full, 1-110                                              |
| `bitty-ai-docs/specifications/provider-plugin-boundary.md`                     | Sample, 25-114; additional search matches only           |
| `bitty-ai-docs/specifications/persistence-profile-r6.md`                       | Sample, 219-306; additional search matches only          |
| `bitty-ai-docs/specifications/task-lifecycle-r5.md`                            | Search matches only; no full-section coverage            |
| `research/summary/012.md`                                                      | Full, 1-18                                               |
| `research/summary/023.md`                                                      | Full, 1-22                                               |
| `research/summary/032.md`                                                      | Full, 1-18                                               |
| `research/summary/037.md`                                                      | Full, 1-19                                               |
| `research/summary/040.md`                                                      | Full, 1-20                                               |
| `research/review/2026-09-15/07-bitty-ai.md`                                    | Full, 1-185                                              |
| `research/review/2026-09-16/02-code-vs-docs-progress-audit.md`                 | Sample, 114-160; additional search matches only          |
| `research/review/2026-09-16/04-architectural-drift-and-subjective-analysis.md` | Sample, 112-133; additional search matches only          |
| `research/review/2026-09-16/01-docs-hierarchy-and-submodule-sync.md`           | Search matches only                                      |
| `research/review/2026-09-16/03-research-records-traceability-audit.md`         | Search matches only                                      |
| `research/review/2026-09-16/README.md`                                         | Search matches only                                      |

No slice source, live host adapters, unrelated repository implementations, research originals, full security corpus, or full canonical specification corpus was reviewed. Therefore this is a focused runtime review, not an end-to-end security or product-conformance approval.

## Independent second-pass verification

This section supersedes first-pass severity/causality claims where explicitly marked above. Verification was source-to-contract inspection, not execution. Workspace, research, implementation, standalone AI-docs, and (for the pinned IPC check in report 02) terminal-repository guides were read. The ctxctl skill was loaded; its MCP path guard refused the external workspace, so installed CLI outlines and targeted reads were used. No CarryCtx or nested agents were used.

### Source baseline and campaign movement

- Second-pass source HEAD: `97d312585c20a5b17063084cf66ba1bb7438667c`, clean at entry. Report 01 began at `e7cbe69f0fac8a785657b5ff0bab850b66e9f690`; the locally inspected diff contains only fallback, its integration tests, and five export lines. All RUN finding source ranges remain at their original line numbers. Report 02 already used this source HEAD.
- Standalone docs HEAD: `57c2d7d49c83f8094c5586ddf7338f686d77209e`, clean at entry, versus both reports' `bb66faf88d0a9cff4ea753671e0dbefb5fadea80`. Changes add host-boundary/promotion documents and update their index/triage. The cited context-management, persistence R6, and implementation-profile documents are unchanged by that diff.
- Source's recorded and checked-out docs pin remains `07169bd9108875179899ac77c3af3c0073a8506a`; it was not refreshed. Standalone-docs citations are not claims about that pinned snapshot.
- Research HEAD: `d70152a079c75204ec37e99a7bf3f54b8190e423`; the campaign directory was already untracked. No commit was created. Working-tree reads were not an atomic snapshot: entry/recheck agreement cannot exclude transient concurrent edits.

### Verified dispositions

| ID         | Independent result                                                                                                                                                                                                                                                                                                           |
| ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| AI-RUN-001 | Retained P1: loss of the agent's pending lookup is proven. Blanket unrelated-turn prohibition and loss of every host copy are withdrawn. Intentional per-turn accounting does not preserve unresolved lookup.                                                                                                                |
| AI-RUN-002 | Reclassified P2 integration gap: missing executor identity parameter is proven; unavoidable host miscorrelation is not.                                                                                                                                                                                                      |
| AI-RUN-003 | Retained P2: `begin_turn` runs once (`agent.rs:505`), config checks each round (`:569-575`), and bus count persists (`tool.rs:720-727`). The hard ceiling still holds; the defect is inconsistent cumulative configured allowance and later-batch admission. Transactional authorization is not rollback of earlier effects. |
| AI-RUN-004 | Retained P2: acknowledged-success result rejection becomes Failed; JIT refusal can also become Failed. Initial precheck refusal and executor Denied were distinguished.                                                                                                                                                      |
| AI-RUN-005 | Retained P2: final sink callback can cancel while returning success; emission returns Some, and the agent returns Completed while `finish` preserves Canceled. No thread race is required.                                                                                                                                   |
| AI-RUN-006 | Reclassified P2 contract/test gap: per-invocation scheduling and terminal non-resumption are intentional; execution-wide budget and always-Active prose are inconsistent.                                                                                                                                                    |
| AI-RUN-007 | Retained P2: after valid epoch/Failed-state checks, scratch capacity uses unchecked disposition length (`adoption.rs:351`); this is extra allocation from an already-owned input, not a remote-ingress claim.                                                                                                                |
| AI-RUN-008 | Retained P2: raw malformed names, denial reasons, terminal reconciliation reasons, and truncate-only pending reasons have caller-visible paths. No permission bypass is claimed.                                                                                                                                             |
| AI-RUN-009 | Retained P3 evidence issue: first registration succeeds without external version evidence; digest availability alone does not prove importer refusal. No MCP implementation defect is inferred.                                                                                                                              |

### Second-pass coverage and gaps

Targeted reads covered admission, reset, provider rounds, dispatch/error attribution, emission, cancellation and reconciliation in `agent.rs:455-662,671-681,720-735,748-781,799-818,820-975`; supporting definitions at `:60-78,389-395`. The original conversion citations were not independently body-read. Supporting reads included `tool.rs:30-39,183-197,263-270,295-325,421-509,633-781`, `session.rs:377-400`, `stream.rs:280-319`, `reconcile.rs:1-65,106-116,190-204`, `adoption.rs:104-119,296-357`, and runtime `bridge.rs:94-118`. These are sampled files, not full-module second-pass coverage.

Test bodies sampled: `agent_turn_semantics.rs:169-226`, `unknown_reconcile.rs:284-355`, `runtime_fail_closed.rs:439-480,486-549,593-616`, `batch_evidence.rs:287-343`, and `mcp_fail_closed.rs:1-30,100-130,192-223`. Other tests and historical-review closure tables were not independently revalidated. Docs reads included R6 `:219-265`, context-management `:72-85`, implementation-profile `:12-91`, host-boundary design `:87-129`, and full research summaries 023/037. Search results and outlines do not expand this ledger.

No Rust build, lint, typecheck, tests, benchmark, provider I/O, exploit, or reproduction was run. No source was changed. The [product index](README.md) records final owned-file validation and residual coverage gaps for both reports.

## Checks

- Static inspection only for Rust: no builds, cargo tests, clippy, formatting, executable reproductions, or network/provider calls. Existing test success is not asserted.
- Pre-report `bitty-ai` HEAD/status recheck and `git diff --check`: passed, source unchanged.
- Second-pass Markdownlint: installed CLI 0.23.1 / markdownlint 0.41.1, `--no-globs`, exactly the two reports and `README.md`: 3 files, zero issues. Edited sections and index were read back; no fix flag was used.
- Closing observation: source remains `97d312585c20a5b17063084cf66ba1bb7438667c`, clean; docs advanced during verification to `fb4cda1c5e9792c848d036903c7db322ee6d1743`, clean. The new diff adds git-wrapper design and updates its index only; cited contracts are unchanged. Research remains `d70152a079c75204ec37e99a7bf3f54b8190e423`; all three owned reports/index are untracked. `git diff --check` passed, but it does not inspect those untracked Markdown files; Markdownlint did.
