# IPC, Agent, Performance, and Test-Support Review

## Independent second-pass verification

- `bitty` HEAD rechecked at `06bc1f45995fd297a3f7324688bc81b0598ecf67`, only pre-existing untracked `.targets/`. No product tests were rerun. The 284-test receipt below is a first-pass record, not independent evidence for these findings or their fixes.
- **Independently verified static mechanisms:** TERM-IPC-001 and TERM-IPC-004 (P1); TERM-IPC-008, TERM-IPC-010 and TERM-IPC-011 (P2). No runtime reproduction or exploitability conclusion follows.
- TERM-IPC-001: `crates/bitty-ipc/src/auth.rs:125-168`, `devtools/serve.rs:803-826`, and `crates/bitty-app/src/ipc_serve.rs:295-354` establish the marker-only live seam. This is a deliberately disclosed filesystem-attestation substitute, not proof of absent access control or a cross-user compromise. The code itself claims peer-credential verification where it only supplies a marker. P1 is qualified control-assurance work; the original normative-document comparison was not freshly audited.
- TERM-IPC-004: `crates/bitty-ipc/src/ctl.rs:1033-1054,1070-1087,1175-1217` and `crates/bitty-app/src/ctl/apply.rs:27-41` show queued work has no deadline/cancellation token and is applied before reply-send failure. Scope checks still apply. Generic MCP/channel extensions were not independently reverified.
- TERM-IPC-008: `crates/bitty-agent/src/observation.rs:140-166` confirms unchecked UTF-8 truncation and absent truncation marker. No panic case was generated or executed.
- TERM-IPC-010/011: `crates/bitty-perf/src/latency.rs:282-299,336-361,382-396` confirms synthetic fallback, including after a successful PTY spawn; `idle.rs:324-355,384-394` maps missing CPU data to PASS. Synthetic fallback is intentional, but mixed provenance cannot prove real echo latency. Missing CPU evidence is inconclusive, not a pass. Bench threshold behavior, statistical validity, and real compositor latency were not rerun or independently rechecked.
- **Not independently reverified:** TERM-IPC-002/003/005/006/007/009/012/013/014/015. Their first-pass qualifications remain essential: unused provenance helpers, explicitly synthetic transport, API-only lifecycle models, and measurement scaffolds are not demonstrated production failures.
- Outlines preceded the above slices. All other source ranges, integration paths, dependency/platform implementations, and `bitty-compat-lab` remain outside this sample. Markdown-only checks: [product index](README.md).

## Scope and baseline

- Review date: 2026-09-17. Read-only source review; no CarryCtx, delegation, commits, installs, fetches, or exploit reproduction code.
- Source repository: `bitty`, HEAD `06bc1f45995fd297a3f7324688bc81b0598ecf67` (`v0.0.20-85-g06bc1f4`). Initial and final status: only untracked `.targets/`; tracked and staged diffs empty. That pre-existing directory was not used or changed by this review.
- Report repository: `research`, initial HEAD `d70152a079c75204ec37e99a7bf3f54b8190e423`; initial status already contained untracked `review/2026-09-17/`. Existing reports were preserved.
- Scope: `crates/bitty-ipc`, `crates/bitty-agent`, `crates/bitty-perf`, and `crates/bitty-test-support`. Selected app/runtime callers and three performance benches were sampled to establish reachability and measurement semantics. `bitty-compat-lab` is excluded.
- All `crates/...`, `benches/...`, and `justfile:...` citations below are **repository-relative to `bitty`**. Other repositories are named explicitly.
- Guidance read: workspace, `bitty`, and `research` AGENTS files; no nested crate AGENTS files were found. CtxCtl skill loaded. Its MCP adapter rejected the workspace path; the installed CLI worked. TOML and shell outlines are unsupported, so TOML used ordinary reads. No CarryCtx state or commands were used.
- Historical input: `research` repository, `review/2026-09-15/05-ipc-agent-perf-test.md`, read in full. Historical severity and implementation claims were not assumed correct.

## Executive assessment

First-pass inventory: **2 P1, 10 P2, 3 P3**. The independent second pass verified TERM-IPC-001/004 and the sampled P2 mechanisms below; other IDs keep first-pass status, not renewed confirmation.

The code has useful defensive foundations: bounded frame allocation before dispatch, bounded request and response queues, explicit scope checks, consent checks before tool/execution provider invocation, provider target-echo checks, and immutable untrusted labels on snapshot/tool result validation. Those controls do not establish live peer identity, cancellation, real compositor timing, or complete harness coverage.

The most important corrections to the earlier review are:

- Do not claim that repeated connection credential queries identify the current holder of a transferred descriptor. The confirmed issue is absent live peer verification, not a demonstrated descriptor-transfer exploit.
- Do not claim unknown or expired token errors necessarily disclose a currently usable token. They do disclose supplied token text; derived Debug additionally exposes retained token data.
- `latency_real` does **not** enforce the advertised 8/15 ms budgets; exceeding them only prints a note.
- PTY backend detection explicitly distinguishes backend availability from program availability. Windows ConPTY support is acknowledged in the harness; the earlier suggestion that it conflates the two is incorrect.
- Documentation describing newer fixes is not evidence that those fixes exist at this checkout's HEAD.

## Coverage ledger

Definitions: **full** means every line was read; **sampled** means selected bodies, ranges, outlines, or search matches were reviewed; **unread** means no substantive implementation review. Running a test does not promote its source to full coverage.

| Area                          | Coverage | Files and limits                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| ----------------------------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Crate metadata                | Full     | All four scoped `Cargo.toml` files. IPC, Agent, and test-support have no dependencies; perf pulls runtime/render/platform dependencies.                                                                                                                                                                                                                                                                                                                                                                 |
| Agent queue                   | Full     | `crates/bitty-agent/src/queue.rs`, including tests.                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| PTY harness                   | Full     | `crates/bitty-test-support/src/lib.rs`, including macro and tests.                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| Perf root                     | Full     | `crates/bitty-perf/src/lib.rs`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| IPC auth and transport        | Sampled  | `auth.rs`: credential/marker constructors, endpoint verifier, token construction/store/verification; `channel.rs`: bounded queue and pending/timeout operations; `frame.rs`: incremental framing; `limits.rs`: limiter; `wire.rs`: depth, ambient-field and envelope validation; `transport.rs`: close/send/raw-wire/forward contract; `mcp.rs`: request send, response correlation, expiry, synthetic serialization; `bridge.rs`: call/answer/expiry. Unit-test outlines sampled, not every test body. |
| IPC service surfaces          | Sampled  | `ctl.rs`: outline and queue/waker/elevation code; `execution.rs`: outline, default provider, dispatch/reconcile/resolve; `snapshot.rs`: outline, validation, bounding, dispatch; `tool_dispatch.rs`: outline and dispatch; `host_bridge.rs`: store, live readers/tools, caller binding; `scope.rs`: symbol outline only; `lib.rs`: export/test outline only.                                                                                                                                            |
| Devtools                      | Sampled  | `devtools.rs`: constants/export outline; `devtools/serve.rs`: discovery, context, directory/socket attestation, entire connection loop; `devtools/json.rs`: outline and params/version-adjacent parsing slice; `devtools/profiling.rs`: ring publication, scope check, bounded drains and polling handlers; `devtools/automation.rs`: outline only.                                                                                                                                                     |
| IPC integration tests         | Sampled  | Search matches in `tests/devtools_profiling.rs`, especially sequence/truncation assertions; other integration-test references appeared in reachability searches only. No integration suite was executed.                                                                                                                                                                                                                                                                                                |
| Agent messages/session/tools  | Sampled  | `message.rs`: message validation/size/trust helper; `session.rs`: observation insertion, message state transitions, completion/failure; `observation.rs`: outline plus sizing/truncation/trust helper; `tool.rs`: outline, key matching, scrub entry points, ToolCall views; `lib.rs`: outline only.                                                                                                                                                                                                    |
| Perf implementation           | Sampled  | `latency.rs`: report/percentiles, entire PTY-echo measurement, budget/summary helpers, smoke assertions; `idle.rs`: report, sampling, verdict/summary; `startup.rs`: probes, threshold helpers and real-window wrapper. Runtime construction and much of headless measurement setup were not read.                                                                                                                                                                                                      |
| External code context         | Sampled  | `crates/bitty-app/src/ipc_serve.rs:295-354`; `crates/bitty-app/src/ctl/client.rs:189-239`; `crates/bitty-app/src/ctl/apply.rs:27-78`; `crates/bitty-runtime/src/host_bridge.rs:101-158,182-206`; main bodies of `benches/latency_real.rs`, `benches/startup_real.rs`, `benches/idle_real.rs`; full `justfile`.                                                                                                                                                                                          |
| IPC unread implementation     | Unread   | `src/error.rs`, `src/frame_digest.rs`, `src/rich_fragment.rs`, `src/devtools/handlers.rs`, `src/devtools/automation_ops.rs`, `src/devtools/tests.rs`, `examples/introspect_probe.rs`. Test execution covered some of these, but this review does not certify their implementation.                                                                                                                                                                                                                      |
| IPC unread integration bodies | Unread   | `tests/bridge_client.rs`, `tests/devtools_automation.rs`, `tests/devtools_frame_digest.rs`, `tests/devtools_roundtrip.rs`, `tests/devtools_verify.rs`, `tests/execution_service.rs`, `tests/snapshot_service.rs`, `tests/tool_dispatch_service.rs`. Incidental search hits are not substantive coverage.                                                                                                                                                                                                |
| Agent unread implementation   | Unread   | `src/id.rs`, `src/error.rs`; most lower-level redaction scanner bodies and tests in `tool.rs`.                                                                                                                                                                                                                                                                                                                                                                                                          |
| Other unread material         | Unread   | All four crate READMEs; unlisted ranges within sampled files; other benches, fuzz targets, platform implementations, CI workflows, and compatibility-lab code.                                                                                                                                                                                                                                                                                                                                          |

### Documentation evidence and revision mismatch

Consulted local documentation only; no network fetch was needed:

- `bitty-terminal-docs` HEAD `7947fb39acf77d308c2eb8bc57b24c481a841a17`, clean: `specifications/devtools-rfc.md:759-792`.
- `bitty-docs` HEAD `fc7ed9750491875bea9986be35a4df5074ad84ba`, clean: `docs/security/p0-acceptance-criteria.md:330-385`; security threat/risk/evidence matches were sampled.
- Performance RFC was located, not substantively read. Budget numbers below are corroborated directly by `crates/bitty-perf/src/lib.rs:32-59`, not by a claimed full RFC audit.

The devtools RFC's implementation note at lines 781-792 claims endpoint re-verification, client endpoint checking, and token-free child-token failure reasons under CTX-0463/#773. This checkout still has the older marker-only serving path and token-interpolating errors. Treat the docs as ahead of this source baseline; no remote history or later fix was fetched. The pinned `bitty/docs` submodule was not initialized or updated.

## Confirmed findings

First-pass records below; second-pass status is stated above.

### TERM-IPC-001 — P1: Live serving substitutes filesystem attestation for peer verification

- **Evidence:** `crates/bitty-ipc/src/auth.rs:147-149`, `crates/bitty-ipc/src/devtools/serve.rs:803-826`; live caller `crates/bitty-app/src/ipc_serve.rs:320-327`.
- **Cause:** The production caller constructs a `VerifiedPeer` through `transport_attested_peer(runtime_uid)`. The marker constructor ignores its UID argument and never examines the accepted stream. The serving path then marks the connection locally attested. The primitive that checks supplied credentials exists, but is not used by this live caller.
- **Impact:** The accepted peer-credential requirement is not implemented at this boundary. Owner-only filesystem modes are a useful access control, but are not evidence that connection credentials were checked. This is a confirmed assurance/control gap, not a demonstrated cross-user compromise.
- **Fix:** Obtain peer identity through a reviewed platform seam, compare it with the runtime identity, and create the marker only on success. Keep endpoint ownership/type/mode checks separate. Remove comments claiming that filesystem attestation proves a credential check. Define descriptor delegation semantics explicitly; do not promise that repeating a connect-time credential query detects later descriptor transfer.
- **Safe test idea:** Use injected credential-provider results to assert that denied or unavailable verification prevents any read/dispatch. Add platform CI tests of the identity extraction seam using ordinary locally created connections; no descriptor-transfer reproduction is needed.

### TERM-IPC-002 — P2: Client discovery hints are not followed by server identity checks

- **Evidence:** `crates/bitty-ipc/src/devtools/serve.rs:44-63`; client `crates/bitty-app/src/ctl/client.rs:197-219`.
- **Cause:** The resolver accepts a nonempty socket override after shape checks; `ctl_roundtrip` connects and writes the request without endpoint ownership/type validation or connected-server identity validation.
- **Impact:** Socket selection is treated as sufficient trust. Requests and returned control outcomes have no verified server identity at this seam. This requires an inappropriate endpoint selection; merely supporting a configurable socket path is not itself a defect.
- **Fix:** Treat discovery as a hint. Validate the endpoint and its trusted ancestry, then verify the connected peer through the platform identity seam before sending request bytes. Preserve explicit multi-instance selection without equating selection with authentication.
- **Safe test idea:** An injected connector/attestor should demonstrate that a mismatched or unavailable server identity causes zero writes. Do not run a deceptive listener or send real terminal content.

### TERM-IPC-003 — P2: Sustained request limit is stored but never enforced

- **Evidence:** `crates/bitty-ipc/src/limits.rs:101-123`; limiter use `crates/bitty-ipc/src/devtools/serve.rs:750-773`.
- **Cause:** `check` enforces only the one-second burst count and explicitly discards `limit_per_sec`. The claimed token bucket is actually a burst-sized sliding window.
- **Impact:** The default does not deliver its documented 100 requests/second sustained policy; the higher burst ceiling can persist. Resource planning based on the lower rate is incorrect.
- **Fix:** Implement a monotonic-clock token bucket with separate refill rate and capacity, or a precisely specified equivalent. Keep denial transactional and constructor limits bounded.
- **Safe test idea:** Deterministic simulated-clock tests should cover initial burst, sustained refill over several windows, recovery after idleness, and unchanged state on denial. No live request flood is necessary.

### TERM-IPC-004 — P1: Timed-out control work remains executable

- **Evidence:** `crates/bitty-ipc/src/ctl.rs:1040-1050,1078-1083,1175-1217`; consumer `crates/bitty-app/src/ctl/apply.rs:33-38`. Related generic behavior: `crates/bitty-ipc/src/channel.rs:533-535,566-581` and `crates/bitty-ipc/src/mcp.rs:504-520`.
- **Cause:** `recv_timeout` abandons the reply receiver without removing or marking the queued control. `PendingControl` carries neither deadline nor cancellation state. The consumer applies the operation before discovering that reply delivery failed. Generic endpoint/MCP expiry similarly removes pending correlation records without withdrawing queued requests.
- **Impact:** A client can receive an unavailable/timeout result while a not-yet-started input or management operation executes later. Retrying can duplicate effects. The generic endpoint documentation promises pending expiry, not universal cancellation; the concrete live control queue is the stronger correctness issue.
- **Fix:** Carry a server deadline and dispatch-state token with queued work. Reject expired, not-yet-started work before applying it. For already-started work, return an explicit unknown outcome and provide reconciliation rather than implying no effect. Remove withdrawn generic queue entries or make transport handover deadline-aware.
- **Safe test idea:** Use a fake clock, a paused in-memory consumer, and a harmless counter operation. Confirm expired queued work does not increment the counter, while already-started work reports an explicit ambiguous outcome. Avoid real terminal input or process management.

### TERM-IPC-005 — P2: Bridge answers occupy the queue before correlation

- **Evidence:** `crates/bitty-ipc/src/bridge.rs:242-267`; `crates/bitty-ipc/src/channel.rs:538-550`.
- **Cause:** `answer` enqueues the response and only then calls `complete`. Unknown IDs return false but still leave a response queued, contrary to the bridge's “without insertion” contract.
- **Impact:** Uncorrelated or duplicate answers consume the bounded response capacity and are observable through `take_response`; legitimate answers can be refused despite the unknown-answer return value implying no insertion. No external response injection path was demonstrated.
- **Fix:** Check correlation before enqueueing. Preserve the pending entry if a correlated response cannot be enqueued; do not complete first and lose retryability on a full queue.
- **Safe test idea:** A tiny in-memory queue should remain empty after an unknown answer. A known answer rejected for capacity should retain its pending record and succeed after space is freed.

### TERM-IPC-006 — P2: Child-token diagnostics retain and expose token text

- **Evidence:** `crates/bitty-ipc/src/auth.rs:382-389` interpolates the supplied token into failure reasons; `crates/bitty-ipc/src/auth.rs:331-334` derives Debug for a map whose keys are raw token strings.
- **Cause:** Authentication errors use credential-shaped input as diagnostic text, and the token store has an unsanitized diagnostic representation.
- **Impact:** Logging a verification error discloses the submitted token text; debugging a populated store exposes retained token keys. The former does not prove replay of a valid live token: unknown and expired inputs are not valid for the tested authorization attempt. No production log sink was verified.
- **Fix:** Use static failure categories and an explicitly redacted Debug implementation. Avoid printing token-bearing map keys or values, including expired-entry diagnostics.
- **Safe test idea:** Use an unmistakably synthetic test marker and assert it is absent from error Display/Debug and store Debug for missing, expired, and retained entries. No real token or replay is needed.

### TERM-IPC-007 — P2: Ambient-field scan rejects ordinary JSON values

- **Evidence:** `crates/bitty-ipc/src/wire.rs:235-267,304-306`; bridge invocation `crates/bitty-ipc/src/bridge.rs:223-224`.
- **Cause:** Every depth-one quoted string is treated as a possible key, without checking key/value position. The scan operates on the params object supplied to `validate_request_envelope`.
- **Impact:** Valid method parameters containing ordinary values equal to reserved field names can be rejected as ambient authority. The raw-string scan also does not establish decoded-key semantics. Server scope checks remain essential; this finding does not claim scope escalation.
- **Fix:** Validate object structure and decoded keys with a bounded parser, and apply the reserved-field rule at its intended envelope level rather than indiscriminately to method parameters.
- **Safe test idea:** Table-driven ordinary-value, nested-object, and decoded-key equivalence cases should separate valid data from reserved envelope fields. Validate parser depth and payload limits independently.

### TERM-IPC-008 — P2: Terminal observation truncation can panic on valid Unicode

- **Evidence:** `crates/bitty-agent/src/observation.rs:152-162`.
- **Cause:** The helper calls `String::truncate(MAX_OBSERVATION_BYTES)` without moving the cut to a UTF-8 character boundary. It also omits the “loud marker” promised by its documentation.
- **Impact:** Valid multilingual terminal text crossing the byte budget can panic instead of yielding a bounded observation. Successful truncation is indistinguishable from complete content. This is statically confirmed; no panic reproduction was executed.
- **Fix:** Backtrack to a character boundary before truncation, and carry explicit truncation metadata or a reserved-budget marker. Ensure the final observation still fits its byte limit.
- **Safe test idea:** Ordinary multilingual fixtures at nearby budget boundaries should never panic, remain valid UTF-8, remain bounded, and clearly indicate whether content was shortened.

### TERM-IPC-009 — P2: Message-level trust query always reports trusted content

- **Evidence:** `crates/bitty-agent/src/message.rs:201-213`; observation-level distinction `crates/bitty-agent/src/observation.rs:181-186`.
- **Cause:** `is_untrusted_content` is a public boolean query implemented as constant false; the message type has no provenance field supporting the advertised classification.
- **Impact:** It cannot represent terminal-origin content once converted into a message. A consumer using the helper would receive a false assurance. The crate-wide caller search found the definition but no production use, so an active confused-deputy path is not established.
- **Fix:** Remove/deprecate the misleading query or represent provenance explicitly and preserve it through observation-to-message conversion. Do not replace it with content sniffing.
- **Safe test idea:** Convert an ordinary terminal observation into a message through the intended host adapter and assert that its untrusted provenance survives without changing its content.

### TERM-IPC-010 — P2: PTY latency report substitutes synthetic work for delayed echo

- **Evidence:** `crates/bitty-perf/src/latency.rs:289-296,342-359,382-395`; report fields `crates/bitty-perf/src/latency.rs:121-136`; bench `benches/latency_real.rs:33-41,52-72`.
- **Cause:** Failure to start `cat` silently switches measurements. Even after a successful spawn, a single empty poll causes synthetic terminal bytes to be inserted immediately. Samples are not correlated with the particular key's actual PTY return. The report only identifies the rendering seam as headless, not the echo source. The bench prints a note, rather than failing, when the true 8/15 ms budget is exceeded.
- **Impact:** The nominal PTY measurement can omit actual scheduling/echo latency and mix incomparable paths. Green smoke tests and this bench do not demonstrate PB-4 key-to-screen compliance.
- **Fix:** Distinguish synthetic, PTY, and real-present measurements in the result type. Use bounded key/echo correlation for a real PTY measurement; mark timeout/missing echo as incomplete, not synthetic success. Provide a separately selected reference-machine budget gate with nonzero exit on violation.
- **Safe test idea:** A fake delayed-echo source should produce a delayed or incomplete sample rather than a fast synthetic pass. Test provenance fields and threshold verdicts using fixed sample arrays; do not run a display or PTY benchmark in this review.

### TERM-IPC-011 — P2: Missing CPU evidence is reported as a passing CPU budget

- **Evidence:** `crates/bitty-perf/src/idle.rs:324-355,384-394`; `benches/idle_real.rs:49-54`.
- **Cause:** `sample_self_cpu_pct` takes a single external `ps` sample after 200 ms; `meets_cpu_budget` maps `None` to true. The formatter then prints a CPU PASS even when the sample is unavailable. A process CPU percentage is not an interval-isolated ten-minute idle measurement.
- **Impact:** Missing measurements, especially where `ps` is absent, become positive budget evidence. The bench treats over-budget samples as notes. Frame-on-demand checks remain useful but prove a different property from actual wakeups and average CPU.
- **Fix:** Return pass/fail/inconclusive separately, retain the sampling method/window, and use interval CPU deltas for the reference measurement. Bound the external sampler's execution. Keep frame-on-demand and CPU verdicts independent.
- **Safe test idea:** Construct reports with absent, below-budget, and above-budget samples and assert distinct summaries. Use an injected sampler for unavailable/timeout cases; no long idle run is needed.

### TERM-IPC-012 — P2: Live snapshot store lacks per-terminal retirement

- **Evidence:** `crates/bitty-ipc/src/host_bridge.rs:104-171`.
- **Cause:** A process-global map only inserts or overwrites entries; new IDs are refused at the count cap. There is no production per-terminal removal operation. The only clear operation is described as a test helper, and readers return retained entries without lifecycle checks.
- **Impact:** Integrators cannot retire one closed terminal. After enough distinct IDs, subsequent publications can be refused even if the corresponding terminals have closed; stale snapshots remain available by ID. Production UI publication was not established by the sampled caller search, so this is a confirmed service lifecycle defect rather than a demonstrated current UI failure.
- **Fix:** Bind store ownership to terminal lifecycle and add explicit retirement, preserving active entries. Readers should distinguish closed, never-published, and current terminals. Add per-entry byte bounds before storage rather than relying only on output truncation.
- **Safe test idea:** Publish and retire a sequence of ordinary terminal fixtures beyond the entry cap; active entries should remain readable, retired IDs should not serve stale state, and capacity should recover.

### TERM-IPC-013 — P3: Raw-wire transport helper contradicts its framing contract

- **Evidence:** `crates/bitty-ipc/src/transport.rs:195-240`.
- **Cause:** The helper claims wire bytes will be split for incoming consumption, but stores the entire encoded wire as one outgoing payload. Its first limit allows the payload cap plus header, then `Frame::new` applies the smaller payload cap to the encoded wire.
- **Impact:** A maximum-sized valid encoded frame cannot pass this helper, and tests using it do not model actual frame delivery. The method is documented internally as a nonfaithful stub and is not shown on a production transport path.
- **Fix:** Remove/rename the helper to match its opaque-chunk semantics, or decode through the real framer and enqueue atomically with the appropriate queue limit.
- **Safe test idea:** Normal single-frame, concatenated-frame, and exact-cap fixtures should agree with `encode_frame`/`decode_frame` without invoking sockets.

### TERM-IPC-014 — P3: First assistant tool call does not enter waiting state

- **Evidence:** `crates/bitty-agent/src/session.rs:268-280`.
- **Cause:** The broad `Created` transition is evaluated before the assistant-with-tool-calls case. A first assistant tool-call turn enters Running, whereas the same turn from Running enters WaitingToolResult.
- **Impact:** Session state depends on an unrelated preceding turn and fails to represent outstanding tool work in a valid public-API sequence. This crate does not execute those calls, so no external effect is claimed.
- **Fix:** Derive the next state from the accepted message as well as the previous state, or explicitly reject unsupported starting roles before mutation. Define interleaving and pending-result semantics separately from structural message validation.
- **Safe test idea:** Compare a first assistant tool-call turn with the same turn following a user turn; both should have the documented waiting behavior or an explicit validation error.

### TERM-IPC-015 — P3: Real-window startup entry point never creates a window

- **Evidence:** `crates/bitty-perf/src/startup.rs:503-550`; bench labeling `benches/startup_real.rs:38-49`.
- **Cause:** The real-window flag runs headless startup and then validates window configuration and repeats an event-loop availability probe. It neither creates a live Window nor presents through a compositor in the reviewed path. The repeated EventLoop attempt is process-order-sensitive and is classified as unavailable at `startup.rs:331-345`.
- **Impact:** This API is a capability/configuration probe, not real-window cold-start evidence. Comparing one measured total against helpers named p50/p99 is also not a measured startup distribution. Existing headless flags prevent a fully silent substitution, but the entry-point name and phase labels overstate its coverage.
- **Fix:** Rename it as a probe until a process-isolated real-window harness exists. Measure repeated fresh-process startup through actual first presentation, and compute distribution statistics over those runs.
- **Safe test idea:** Assert that probe reports never claim a real presentation. Test report aggregation with fixed timing fixtures; reserve compositor runs for a separate reference environment.

## September 15 findings rechecked

Every prior defect is accounted for below. “Retained” means the current code supports a narrower confirmed finding, not that the prior severity or exploit narrative is endorsed.

| Prior ID | Current disposition                               | Current evidence / correction                                                                                                                                                                                                                                                                  |
| -------- | ------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| P1-1     | Retained, narrowed: TERM-IPC-001                  | Live marker attestation remains. Descriptor-transfer narrative and suggested repeated credential-query cure are not established.                                                                                                                                                               |
| P1-2     | Retained, narrowed: TERM-IPC-002                  | Client connects then writes without identity verification. Environment control is a prerequisite, not proof of an unconditional leak.                                                                                                                                                          |
| P1-3     | Retained, P2: TERM-IPC-003                        | Sustained rate remains unused.                                                                                                                                                                                                                                                                 |
| P1-4     | Retained, strengthened at live ctl: TERM-IPC-004  | Pending expiry is not cancellation; live queue has no expiry before apply.                                                                                                                                                                                                                     |
| P1-5     | Retained, P2: TERM-IPC-005                        | Unknown answers still enqueue. Production hostile-response reachability not shown.                                                                                                                                                                                                             |
| P1-6     | Retained, P2: TERM-IPC-006                        | Token text still appears in errors. Replay of a usable token does not follow from unknown/expired errors alone.                                                                                                                                                                                |
| P1-7     | Retained, P2: TERM-IPC-009                        | Constant false remains; no production caller established.                                                                                                                                                                                                                                      |
| P1-8     | Retained as measurement defects: TERM-IPC-010/011 | Loose smoke thresholds are explicit, not inherently a defect. More importantly, `latency_real` also does not hard-gate 8/15 ms.                                                                                                                                                                |
| P1-9     | Out of scope                                      | Compatibility-lab findings not revalidated. No implied closure.                                                                                                                                                                                                                                |
| P1-10    | Not retained as a confirmed defect                | `wire.rs:75-84` intentionally rejects unsupported versions; distinct protocols can have distinct versions. Future negotiation/capability advertisement is an evolution concern, not proof of a present breakage.                                                                               |
| P2-1     | Partly correct; not retained at prior severity    | `frame.rs:191-247` permits a larger transient append, then drains complete frames. Persistent residual bytes remain bounded by a partial frame/header; absence of time in a pure framer is not itself a defect. Total request time belongs to the connection layer.                            |
| P2-2     | Retained as API uncertainty                       | `auth.rs:88-94` returns zero UID/GID on all platforms, including Unix. No privileged production caller of this helper was demonstrated. Rename/remove the placeholder before production use.                                                                                                   |
| P2-3     | Retained, narrowed: TERM-IPC-007                  | False-positive value handling confirmed. No authorization bypass claimed.                                                                                                                                                                                                                      |
| P2-4     | Not retained as an independent defect             | `verify_unix_endpoint` is explicitly a pure supplied-metadata verifier; filesystem checks reside in `serve.rs:516-543,581-617`. Layer separation is not itself an exploitable symlink bug.                                                                                                     |
| P2-5     | Retained, P3: TERM-IPC-013                        | Raw-wire helper contract and double bounds still disagree.                                                                                                                                                                                                                                     |
| P2-6     | Mostly intentional stub limitations               | MCP's 32 pending/10-second default and generic channel's different limits do not require equality. Close is explicitly terminal. `transport.rs:281-289` documents retryable residue and a moved-count result, contrary to the claim of no signal. Real reconnect/heartbeat remains unverified. |
| P2-7     | Open reliability gap; no live reproduction        | `serve.rs:710-741` has per-read but no total-frame deadline. `bitty-app/src/ipc_serve.rs:329-333` actually supplies wall time to the limiter; backward adjustment can prolong refusal until time catches up, not necessarily forever. See uncertainty below.                                   |
| P2-8     | Retained through TERM-IPC-004                     | Priority is an optimization; expiry before effect is the correctness requirement. A wake hook exists and should not be overlooked.                                                                                                                                                             |
| P2-9     | Mixed                                             | Permissive role/tool placement is explicit in `message.rs:159-164`; loss counters are explicit in `queue.rs:63-75`. Neither alone proves a production defect. TERM-IPC-014 identifies a concrete state transition error.                                                                       |
| P2-10    | Partly retained as optimization                   | `tool.rs:165` still matches any key containing token; `tool.rs:988-1003` retains scrubbed views. No evidence showed original arguments being mutated or business logic broken. Raw heap retention is a hardening concern, not a demonstrated new leak path.                                    |
| P2-11    | Retained in TERM-IPC-011/015, narrowed            | Process-global EventLoop behavior and non-interval CPU sampling remain. Driver latency comments are not a measured current timeout guarantee.                                                                                                                                                  |
| P2-12    | Out of scope                                      | Compatibility JSON parser and geometry assumptions not revalidated.                                                                                                                                                                                                                            |

The historical test-gap list also overstates absence in places: this HEAD's existing unit suite includes `serve_connection_oversize_frame_closes`, incremental concatenated-frame tests, and peer-full forwarding tests. Their execution passed below; they do not establish slow-stream deadlines or OS-level authentication. Test-support explicitly documents Windows ConPTY and POSIX-program exclusions at `crates/bitty-test-support/src/lib.rs:3-10,52-60`.

## Optimizations, limitations, and uncertainty

These items are not counted as additional confirmed P1/P2/P3 findings:

- **Request deadlines:** `serve_connection` depends on caller-configured read/write timeouts and has no total frame deadline. Add an absolute monotonic deadline and account for time spent in partial reads. Validate with a deterministic bounded stream double, not a slow-client campaign. Clock rollback is concrete in the current app caller, but the duration of practical impact was not measured.
- **Service execution lifecycle:** `execution.rs:1102-1140` invokes a provider before validating/storing its output. Provider failure after an effect or invalid output can leave no tracked reconciliation record. Confirm the provider error contract and host reachability before promoting this to a live duplicate-execution finding; fake-provider counters can safely validate the state model.
- **Snapshot semantics:** `snapshot.rs:435-443` delegates text-zone filtering to the provider; `host_bridge.rs:142-154` simply clones by terminal ID. Current runtime snapshots use placeholder line spans at `crates/bitty-runtime/src/host_bridge.rs:111-115`. Zone narrowing must not be advertised as text isolation without a verified provider contract. No sensitive-content test was run.
- **Store memory and cloning:** Snapshot entry count is bounded, but `publish_live_snapshot` validates only terminal ID before retaining arbitrary DTO fields; readers clone the entry before output bounding. Validate byte caps at publication and clone only required bounded fields. Map lookup is O(log n); cloning is O(b) in retained entry bytes, so count bounds alone do not establish a byte-memory bound.
- **Profiling cursors:** `devtools/profiling.rs:487-554,575-605` emits oldest matching records but returns the store head sequence, even when the response is truncated. Consumers must advance from the last emitted sample, not blindly from the response head. Existing test matches exercise truncation; downstream cursor interpretation was not inspected. Separate `headSeq` from `nextCursor` to remove ambiguity. Ring scans are O(n), bounded at 32 entries; serialization is bounded by the drain byte cap.
- **Redaction precision:** Replace broad token-substring matching with a documented key policy where practical. Preserve safe values without weakening secret-shaped value detection. Zeroization requires a defined ownership/crash-dump threat model and should not be presented as a substitute for boundary redaction.
- **Queue observability:** SideQueue drops are counted and deliberately nonblocking. Returning a loss delta with each drained batch would make gaps easier for consumers to notice. Push/pop are O(1); draining k elements is O(k). Do not introduce blocking into the terminal observation path.
- **Harness skips:** `require_pty!` returns successfully after a stderr notice, so the standard test count does not distinguish skipped live work. A separate machine-readable live-coverage receipt would improve evidence. This is documented behavior, not an unexpected backend-detection failure.
- **Protocol surfaces:** Generic wire v1, devtools 1.0, and synthetic MCP framing are different contracts. `mcp.rs:537-570` is explicitly not normative JSON-RPC. Do not certify real MCP interoperability, reconnect, Windows named-pipe security, or minor-version compatibility from these stubs.
- **Automation and digest gaps:** Only automation outlines and execution results were observed. Bearer generation, family separation, capture masking, digest implementation, and automation dispatch were not substantively audited. Existing passing tests are not a substitute for that review.

## Checks and execution limits

Executed a bounded, offline library-test run from `bitty`:

```sh
RUSTUP_AUTO_INSTALL=0 CARGO_NET_OFFLINE=true \
CARGO_TARGET_DIR="$REVIEW_TARGET" timeout 120 \
cargo test --offline --locked \
-p bitty-ipc -p bitty-agent -p bitty-test-support --lib
```

`REVIEW_TARGET` denotes a fresh review-specific temporary build directory, not the checkout's existing `.targets/`. Result: exit 0; build completed in 4.56 seconds; **42 Agent + 239 IPC + 3 test-support = 284 tests passed**, zero failed/ignored. Tests used existing fixtures and local test-owned socket seams; no new reproduction program or live service campaign was created. These tests confirm baseline health, not regressions for each finding above.

The targeted Cargo command intentionally bypassed the repository's workspace-wide `just test` recipe to respect the user's no-heavy-build restriction. No Rust source was changed. Full `just check`, workspace clippy/typecheck, perf compilation/execution, integration tests, fuzzing, Windows/macOS runs, actual desktop/PTY measurements, and ten-minute CPU sampling were not run.

Report quality gate: installed `markdownlint-cli2` v0.23.1, with the research repository's configuration and `--no-globs`, targeting only this report: exit 0, zero issues. No formatter was fetched or installed. The review-specific temporary build directory was removed and its absence verified. Final tracked/staged source diffs and HEAD were rechecked; source baseline remained unchanged.

Only durable authored artifact: `research/review/2026-09-17/bitty-terminal/04-ipc-agent-perf-testsupport.md`. No index, source, configuration, or other review file was edited.
