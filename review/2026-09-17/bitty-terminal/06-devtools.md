# DevTools review — 2026-09-17

## Independent second-pass verification

- Source HEAD rechecked at `77ca09c951fb3ac390f561a6e9a6afbc7a2ac64e`; pre-existing `AGENTS.md` modification preserved. The first-pass tests/typecheck below were not rerun and are not per-finding regressions.
- **Independently verified:** TERM-DEV-001/002/005 (P1 static mechanisms) and TERM-DEV-003/004 (P2 static mechanisms). No runtime symptom, panic input, socket campaign, or exploit was executed.
- TERM-DEV-001: installed-entry source calls the synchronous stub runner, while the async live runner is separate. TERM-DEV-002: live connection construction leaves typed inspection/automation on the stub and controls on a local receipt model. Intentional headless models are not inherently bugs; the defect is failing to separate them from an executable/live-connected public surface. Do not infer a server authorization bypass or real reclamation from those local receipts.
- TERM-DEV-003: the decoder rejects incomplete payloads, and the data callback clears its buffer on that rejection; segmentation handling is the verified mechanism. TERM-DEV-004: the catch marks settled before invoking a callback that returns when settled, leaving the promise without settlement or a timer on synchronous write failure. Actual Bun exception frequency is unmeasured.
- TERM-DEV-005: Rust fixed-byte string end offsets are unchecked against UTF-8 scalar boundaries. The source establishes a panic-capable library path; use of that Rust path by the installed TypeScript executable was not traced. P1 is a library robustness priority, not a demonstrated whole-app crash.
- Exact body sample (relative to `bitty-devtools`): `bin/bitty-devtools.ts:1-30`; `src/cli.ts:570-631,655-715`; `src/client.ts:85-114,199-263,574-630,654-660`; `src/control.ts:118-149,187-251`; `src/transport.ts:100-125,606-628`; `src/ipc-socket.ts:280-302,357-404`; `crates/devtools-client/src/redaction.rs:56-82,185-201,224-242`. Outlines preceded slices.
- **Not independently reverified:** TERM-DEV-006/007/008/009/010/011/012/013/014/015. TERM-DEV-015 remains a test-coverage finding, not proof of a missing product guard. Remaining protocol, tracing, campaign, script, typed redaction, and platform behavior retain first-pass qualifications only.
- No product edits or execution, network, installs, commits, or CarryCtx. Markdown checks and wider omissions are in [the product index](README.md).

## Scope and evidence standard

Read-only review of the entire `bitty-devtools` repository as an inventory, with full or sampled body coverage explicitly recorded below. The repository implements consumer-side protocol helpers, a headless diagnostics model, Unix socket adapters, automation bindings, campaign tooling, and delivery scripts. It does not own the server protocol. Current code takes precedence over comments, historical reviews, and implementation-status prose.

Only this report was written. No CarryCtx commands, nested agents, network fetches, installs, commits, source edits, custom reproductions, exploit execution, or live campaigns were performed. Security observations are defensive. Existing selected headless tests were run; they do not reproduce or prove the findings below.

**Confirmed** means the mechanism follows from inspected current code, not that a production incident was observed. P1 denotes major functional failure or process reliability risk; P2 denotes narrower correctness, privacy, lifecycle, or safety failures; P3 denotes a specific verification defect. Priorities are not exploitability ratings. This is not a completeness, interoperability, or release-readiness claim.

Unless another repository is named, all evidence paths are relative to `bitty-devtools/`. Documentation and research paths explicitly identify their repository.

## Baseline

| Repository            | HEAD                                       | Initial dirty state                                                                                           |
| --------------------- | ------------------------------------------ | ------------------------------------------------------------------------------------------------------------- |
| `bitty-devtools`      | `77ca09c951fb3ac390f561a6e9a6afbc7a2ac64e` | Unstaged `AGENTS.md` only; scratch-location wording changed, two insertions and two deletions; no staged diff |
| `research`            | `d70152a079c75204ec37e99a7bf3f54b8190e423` | Eight existing untracked September 17 reports, listed below                                                   |
| `bitty-terminal-docs` | `7947fb39acf77d308c2eb8bc57b24c481a841a17` | Clean                                                                                                         |

The umbrella is not a Git repository. Initial Git queries there failed; the subsequent independent-repository baselines above are authoritative. No core `bitty` source revision is claimed: this pass did not audit the serving core implementation.

Existing untracked reports under `research/review/2026-09-17/`:

- `bitty-ai/01-runtime-and-sessions.md`
- `bitty-ai/02-context-slices-and-providers.md`
- `bitty-plugins/01-sdk-and-template.md`
- `bitty-plugins/02-registry-and-distribution.md`
- `bitty-plugins/03-independent-plugins.md`
- `bitty-terminal/01-terminal-engine.md`
- `bitty-terminal/02-runtime-orchestration.md`
- `bitty-terminal/03-app-ui-core-config.md`

The devtools HEAD and dirty diff were rechecked after the selected tests and report creation and were unchanged. Sibling reports and the pre-existing AGENTS edit were preserved. At the final research status check, two additional sibling reports (`04-ipc-agent-perf-testsupport.md` and `05-pluginhost-lua-package-rich.md`) had appeared alongside this report; they were not read or modified by this review.

Read the umbrella, devtools, research, and terminal-docs AGENTS files in full, and loaded `ctxctl-core`. Explicit ancestor checks found no AGENTS at the higher workspace/mount ancestors. Discovery found no narrower AGENTS in the reviewed source or report paths. CarryCtx workflow requirements were not executed, honoring the task-specific prohibition.

The ctxctl MCP rejected this checkout as outside its root; the installed CLI worked. Outlines were followed by source slices. Folded outlines count only as orientation. Shell outlines were unsupported, so both scripts were read with the file reader. One CLI read omitted the required `--lines` argument and failed; it was corrected. Early discovery output was truncated in places and is not treated as audited body coverage.

## Summary

**15 first-pass static findings: 3 P1, 11 P2, 1 P3.** The independent second pass verified TERM-DEV-001/002/003/004/005; the rest keep first-pass status, not renewed confirmation. Priorities emphasize the disconnected executable/live-client paths and UTF-8 slicing robustness in the new Rust redactor. Several September 15 claims remain, but scope-matrix divergence and compatibility-JSON divergence have been addressed. The value heuristic and default keystroke gate now exist; neither should be reported as wholly absent.

Selected existing checks passed: 38 tests across four files, the TypeScript no-emit check, and syntax checks for both shell scripts. These passing checks leave the specific branches and integration gaps below unverified dynamically.

## Confirmed findings

First-pass records below; second-pass status is stated above.

### TERM-DEV-001 — P1: Installed CLI entry never selects the live socket runner

- **Evidence:** `bin/bitty-devtools.ts:9`, `bin/bitty-devtools.ts:14`; `src/cli.ts:582`, `src/cli.ts:619`, `src/cli.ts:655`, `src/cli.ts:694`; `src/transport.ts:612-621`; `package.json:14-16`.
- **Cause:** The executable calls synchronous `runCli`, which constructs the in-memory `IpcTransport`. The separate asynchronous `runCliLive` actually dials, but the executable does not call it. The parsed options at `src/cli.ts:159-194` provide no live-runner selector.
- **Impact:** Selecting a real socket through the advertised executable still reaches an empty stub response queue, rather than inspecting the running instance. The new socket adapter does not close the executable integration gap.
- **Fix:** Wire the production entry to an awaited live runner, keeping the injected synchronous runner explicitly test-only. Preserve the no-instance failure and cleanup behavior.
- **Safe test idea:** Inject a fake connection factory into the executable-level runner and assert a normal inspect invocation reaches it, while help and invalid arguments do not. No real socket is needed.

### TERM-DEV-002 — P1: Live composition still routes typed operations to headless implementations

- **Evidence:** `src/client.ts:85-93`, `src/client.ts:199-229`, `src/client.ts:237-263`, `src/client.ts:574-630`, `src/client.ts:654-660`; `src/control.ts:118-149`, `src/control.ts:187-251`.
- **Cause:** `connectLiveSocket` installs both a real socket and a stub, but `inspectionTransport.request` always calls the stub. `automationClient` binds that same synchronous adapter. Only explicit `requestLive` uses the socket. Control methods delegate to a local object that constructs fixed reclamation/disposal receipts without a server request or owned runtime state.
- **Impact:** Typed inspection and automation cannot use the real connection through this composition. More seriously, a live-connected client can report local control success without the runtime having performed the operation. This is a client truthfulness defect, not evidence of a server authorization bypass.
- **Fix:** Separate simulated and live clients in types and construction. Route live typed operations through an asynchronous validated RPC adapter; unsupported live methods must report unavailable rather than manufacture success. Retain local models only under explicit simulation APIs.
- **Safe test idea:** A recording fake RPC adapter should see each typed live call. A control receipt must come from the adapter, and a disconnected or unsupported operation must not return a success receipt.

### TERM-DEV-003 — P2: Unix socket reader rejects ordinary stream fragmentation

- **Evidence:** `src/ipc-socket.ts:280-301`; `src/transport.ts:100-125`; `tests/ipc-socket.test.ts:48-55`.
- **Cause:** Once four bytes have accumulated, the data callback immediately invokes `decodeFrame`. That decoder throws `FrameTruncated` if the complete declared payload has not arrived. The catch clears accumulated bytes and rejects instead of waiting for the rest. Successful decoding also discards bytes after the first frame by ignoring `consumed`.
- **Impact:** A valid response delivered in multiple stream callbacks can fail solely because of read segmentation. Coalesced extra frames are lost. The existing loopback test writes one complete frame and does not establish incremental-stream correctness.
- **Fix:** Use a bounded incremental parser that distinguishes incomplete input from invalid framing, retains the unconsumed suffix, and has an explicit policy for additional frames.
- **Safe test idea:** Drive a fake reader with benign framed responses split at every byte boundary, then with two coalesced frames. Assert segmentation-independent results and retained ordering; no socket or hostile traffic is necessary.

### TERM-DEV-004 — P2: Synchronous socket-write failure leaves its promise pending

- **Evidence:** `src/ipc-socket.ts:357-404`, especially `src/ipc-socket.ts:379-385` and `src/ipc-socket.ts:390-396`.
- **Cause:** The catch captures `responseFailed`, clears the timeout, and sets `settled = true` before calling the captured callback. That callback immediately returns when settled, so neither resolve nor reject occurs and no timer remains.
- **Impact:** A local write or flush exception can leave an awaiting caller pending indefinitely, contrary to the per-response timeout contract.
- **Fix:** Let one settlement routine own the flag and timer, or reject directly from the catch before marking completion. Apply the same single-owner discipline to close/error paths.
- **Safe test idea:** A fake socket whose write or flush reports an ordinary injected I/O failure should cause prompt rejection and leave no pending timer or callback. No actual broken socket was created here.

### TERM-DEV-005 — P1: New Rust redaction scanning can panic on valid Unicode

- **Evidence:** `crates/devtools-client/src/redaction.rs:62-69`, `crates/devtools-client/src/redaction.rs:187-199`, `crates/devtools-client/src/redaction.rs:224-239`.
- **Cause:** The scanners find an ASCII prefix start, then slice Rust strings using fixed byte offsets without first establishing that the end offsets are UTF-8 character boundaries. A sufficient remaining byte length does not establish a valid slicing boundary.
- **Impact:** Valid Unicode diagnostic text can panic in the redaction path rather than produce a bounded preview. This is a static process-reliability finding; no panic input or reproduction was executed or included.
- **Fix:** Match ASCII signatures on byte slices and convert only ranges proven to be valid UTF-8, or use checked string access and boundary-aware traversal. Review all sibling scanners together, not only the first occurrence.
- **Safe test idea:** In the maintainer test suite, property-check that the redactor returns normally for arbitrary valid Unicode strings, including mixed scripts and ordinary punctuation. Preserve redaction decisions through shared benign golden cases.

### TERM-DEV-006 — P2: Child-token errors include credential material

- **Evidence:** `src/auth.ts:340-351`; `crates/devtools-client/src/auth.rs:340-348`.
- **Cause:** Unknown-token and expired-token errors interpolate the supplied token into diagnostic text. Constant-time comparison changes did not remove these messages.
- **Impact:** Callers that record or display the error can retain credential material in a second channel. This review established the exported helper behavior, not a live server call chain or observed external disclosure.
- **Fix:** Use fixed error text with a non-secret request correlation identifier. Do not include token text or a credential-derived preview.
- **Safe test idea:** With a synthetic non-secret marker, assert that every rejection category omits the supplied marker from formatted diagnostics.
- **Prior review:** September 15 D01 remains, with impact narrowed to the verified client/helper boundary.

### TERM-DEV-007 — P2: Trace admission, accounting, and chunk budgets use different quantities

- **Evidence:** `src/tracing.ts:425-454`, `src/tracing.ts:472-521`; `crates/devtools-client/src/tracing.rs:426-476`.
- **Cause:** TypeScript record admission and accounting use string code units, while per-record validation and the existing chunk prefix use UTF-8 bytes. Structured admission checks only payload length, then bills the serialized event including metadata. Rust structured admission also checks payload length but bills a different serialized representation. Configured retention byte ceilings are reported at `src/tracing.ts:526-537` but are not used by these append paths.
- **Impact:** An accepted append can exceed a configured trace or chunk byte ceiling, and a tighter retention byte setting does not constrain accumulation. A subsequent fetch can reject a chunk that append accepted. This concerns the local trace model; filesystem storage is not established.
- **Fix:** Serialize/redact once, determine the actual retained UTF-8 delta, and check that delta against every effective budget before mutation. Keep event count, metadata, coalescing, and retention accounting consistent.
- **Safe test idea:** Model a small byte budget with ordinary multilingual records and metadata-bearing events; assert every accepted transition preserves both trace and chunk limits and that a rejected transition leaves state unchanged.
- **Prior review:** Refines D16; the old ASCII-only byte-count test does not cover this mechanism.

### TERM-DEV-008 — P2: Trace paging assumes fixed-size chunks that append does not create

- **Evidence:** `src/tracing.ts:398-421`, `src/tracing.ts:442-454`, `src/tracing.ts:509-519`.
- **Cause:** Appending starts a new chunk before a whole record would exceed the ceiling, so stored chunks are variable-sized. Fetch chooses `floor(offset / CHUNK_BYTES)`, never subtracts preceding actual chunk lengths, and returns the whole selected chunk without applying an intra-chunk offset. Continuation uses code-unit length against the trace counter.
- **Impact:** Byte-offset pagination can repeat a chunk, return bytes before the requested offset, or miss a later chunk. ASCII data is sufficient for the offset defect; it is independent of multilingual accounting.
- **Fix:** Store chunk start-byte positions, or traverse accumulated byte lengths. Return the requested byte range with boundary semantics defined by the protocol and a trustworthy next offset.
- **Safe test idea:** Use ordinary ASCII records that leave partially filled chunks; compare all pages at several valid offsets against one reference concatenated byte stream.

### TERM-DEV-009 — P2: Campaign preflight is observational and occurs after mutations

- **Evidence:** `src/campaign.ts:606-615`, `src/campaign.ts:999-1071`, `src/campaign.ts:1228-1232`, `src/campaign.ts:1533-1575`, `src/campaign.ts:1592-1616`.
- **Cause:** The live config does not require an explicit scratch-instance assertion. `runCampaign` runs the verb matrix, workspace creation/focus, and elevated spawn probe before evaluating optional socket preflight. A failed preflight is merely another result, not an admission gate. Elevated workspace close iterates listed workspaces, not just resources created by this run.
- **Impact:** A misconfigured live review can modify an existing instance before reporting its failed precondition. Repeat runs alter their own baseline; the runner does not restore initial focus or remove only run-owned resources.
- **Fix:** Require explicit target identity and disposable-instance consent for mutating campaigns. Validate all preconditions before dispatch, distinguish read-only probes from mutation probes, and track ownership for cleanup. Never close pre-existing workspaces as routine cleanup.
- **Safe test idea:** A recording dispatcher must receive zero mutations when target/preflight admission fails; cleanup must refer only to identifiers created by the run.
- **Prior review:** D13 is only partially addressed. The default keystroke row is now absent and explicitly gated; do not repeat the stale claim that default campaigns still send terminal text. The remaining workspace/spawn behavior is visible in current code.

### TERM-DEV-010 — P2: Workflow-import failure skips promised configuration restoration

- **Evidence:** `scripts/workflow-import.sh:106-110`, `scripts/workflow-import.sh:183-215`.
- **Cause:** The script backs up `.carryctx/config.toml`, invokes initialization/import that can rewrite it, and restores it only on the success path. Both command-failure branches exit through `fail`; the EXIT trap deletes the temporary backup without restoring configuration. Restoration also requires the destination still to exist.
- **Impact:** A failed restore operation can leave configuration changed or absent despite the script's byte-identical preservation promise, and deletes the recovery copy. This is a script control-flow conclusion; no CarryCtx operation was run.
- **Fix:** Make preservation an unconditional cleanup responsibility established immediately after backup. Restore even if the destination is absent; report restoration failure distinctly and retain recoverable evidence when necessary.
- **Safe test idea:** Use fake command and filesystem adapters to model initialization/import failure and destination disappearance; assert original configuration survives every exit. The actual workflow script must not run against an owned database for this test.

### TERM-DEV-011 — P2: Text chunking can corrupt or silently omit Unicode

- **Evidence:** `src/protocol.ts:156-179`; `crates/devtools-client/src/protocol.rs:78-100`; `tests/protocol.test.ts:102-109`.
- **Cause:** TypeScript decodes potentially incomplete UTF-8 with replacement and infers validity from re-encoded byte length. Replacement output can have the same byte length as an incomplete prefix, so the length comparison is not a validity check. Both languages break and return partial output if the permitted chunk size cannot fit the next scalar.
- **Impact:** Concatenating successful chunks need not reproduce the original text. Existing protocol chunk tests exercise ASCII only. This is separate from the recently fixed `truncateToBytes` implementation.
- **Fix:** Determine valid UTF-8 boundaries before decoding. Reject an insufficient chunk size explicitly instead of returning successful partial data, or define a minimum chunk size capable of representing every scalar.
- **Safe test idea:** Property-check concatenation identity and each chunk's byte ceiling over ordinary multilingual strings and supported chunk sizes; assert explicit errors for sizes that cannot progress.

### TERM-DEV-012 — P2: Reconnection admission counts historical requests as active connections

- **Evidence:** `src/transport.ts:532`, `src/transport.ts:539-542`, `src/transport.ts:584-594`; `crates/devtools-client/src/transport.rs:602-610`, `crates/devtools-client/src/transport.rs:644-659`.
- **Cause:** Connect passes the cumulative request counter to the concurrent-connection limiter. Disconnect clears queued frames but not the request counter.
- **Impact:** A reused headless transport eventually refuses reconnect despite no active connection. Conversely, the per-object request count cannot enforce an endpoint-wide concurrent connection ceiling. The separate live socket dial does not use this history counter for its own connect admission.
- **Fix:** Keep request telemetry separate from active connection ownership. Place the actual connection ceiling at an owner with endpoint-wide state and define idempotent connect/disconnect behavior.
- **Safe test idea:** With a drained in-memory transport, exercise successful requests and reconnect independently of request history; verify actual connection reservations are released exactly once.
- **Prior review:** D07 remains; priority narrowed to P2 for the demonstrated stub/reuse surface.

### TERM-DEV-013 — P2: Rate limiter implements the burst ceiling but not sustained refill

- **Evidence:** `src/transport.ts:203-225`; `crates/devtools-client/src/transport.rs:225-250`; `tests/transport.test.ts:49-65`; `bitty-terminal-docs/specifications/devtools-rfc.md:509-518`.
- **Cause:** Admission compares only retained timestamps against `burst`. The sustained `limitPerSec` parameter does not affect the algorithm; Rust explicitly discards it. Eviction after one second permits repeated full burst windows rather than the documented sustained rate.
- **Impact:** Client-side pacing differs from its advertised sustained/burst contract. This does not establish that the server lacks its own enforcement.
- **Fix:** Implement a documented sustained-refill token bucket with bounded burst capacity and an injected monotonic clock, or rename the helper and remove the stronger claim if it intentionally offers only a sliding-window ceiling.
- **Safe test idea:** A virtual-clock model should distinguish one allowed burst from the permitted long-run refill. Existing tests only establish burst rejection and timestamp eviction.
- **Prior review:** The rate-limit portion of D17 remains. Clearing an invalid framing buffer is not, by itself, a confirmed new data-loss vulnerability.

### TERM-DEV-014 — P2: Redaction fix leaves typed-field and case-normalization gaps

- **Evidence:** `src/redaction.ts:12-23`, `src/redaction.ts:108-113`; `crates/devtools-client/src/redaction.rs:6-29`, `crates/devtools-client/src/redaction.rs:56-74`; `tests/redaction.test.ts:10-41`.
- **Cause:** The sensitive-field sets still omit several credential-bearing field categories identified in D02. Value heuristics supplement, but cannot replace, typed classification. Rust additionally lacks the separator-free API-key field form recognized by TypeScript. Its assignment scanner locates only the lowercase first character before performing a case-insensitive comparison, contradicting the claimed case-insensitive search.
- **Impact:** The two redactors disagree on typed classifications and some ordinary case variants. A value not recognized heuristically can remain visible under a field that should be classified as sensitive. No credential payload, leak reproduction, or transmission test was performed.
- **Fix:** Use one reviewed, normalized sensitive-field policy and shared cross-language synthetic goldens. Make assignment matching consistently case-insensitive with the UTF-8-safe design from TERM-DEV-005. Keep heuristic coverage explicitly non-exhaustive.
- **Safe test idea:** Assert classification of sensitive field categories and normalization variants independently of value shape, using non-secret sentinel values; compare TS/Rust decisions on a shared corpus.
- **Prior review:** D02 is partially fixed, not unchanged: value-side patterns and an entropy heuristic are now active in both languages.

### TERM-DEV-015 — P3: Pixel-payload rejection test exercises a different rejection

- **Evidence:** `tests/automation.test.ts:478-493`; `src/automation.ts:413-427`; `src/automation.ts:389-390`.
- **Cause:** The test named for rejecting a pixel payload supplies a frame-hash result with large geometry, not an otherwise valid masked pixel-frame result containing a pixel field. Parsing rejects the snapshot discriminator before reaching the pixel-field guard.
- **Impact:** The named security regression could stay green if the actual guard were removed. This is a confirmed test-coverage defect, not a claim that the current guard is absent.
- **Fix:** Construct the correct masked-frame envelope and vary only the forbidden field; retain an adjacent valid control. Keep discriminator/geometry checks in separately named tests.
- **Safe test idea:** Use a tiny non-image sentinel in a fake transport response and verify the specific rejection category/location. No actual image collection or injection is required.

## September 15 report reconciliation

Rechecked every D01–D18 heading in `research/review/2026-09-15/08-bitty-devtools.md`. The earlier report's severities, assumed caller authority, and blanket negative claims are not inherited automatically.

| Earlier claim                                    | Current disposition                                                                                                                                                                                                                                                                                                                                                                          |
| ------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| D01 token text in errors                         | Remains at the helper boundary; TERM-DEV-006. No live logging chain established.                                                                                                                                                                                                                                                                                                             |
| D02 dead heuristic and narrow fields             | Partially fixed: live value/entropy checks exist. Residual typed and normalization defects are TERM-DEV-014; new Unicode reliability issue is TERM-DEV-005.                                                                                                                                                                                                                                  |
| D03 marker disagreement                          | Still observable at `crates/devtools-client/src/redaction.rs:400-402` versus `src/redaction.ts:128`. Marker equality cannot establish whether a redaction operation occurred; the TS changed-text comparison also misses an idempotent redaction. Treat as metadata-policy cleanup, not a separate privacy incident.                                                                         |
| D04 same-token overwrite extends lifetime        | Replacement remains at `src/auth.ts:310-318` and `crates/devtools-client/src/auth.rs:306-314`. No untrusted caller path to insert was established. Owning the store is not equivalent to merely possessing a token; the earlier TTL-bypass claim overstates established authority.                                                                                                           |
| D05 automatic cleanup absent                     | Source search finds no TS production store caller at all, only the class and its cleanup definition. Retention requires explicit owner cleanup, but a production denial-of-service chain or inevitable live-session failure is not established. Do not inherit P1.                                                                                                                           |
| D06 minimum bearer entropy                       | Shape-only validation remains at `src/automation.ts:489-497`. This consumer does not mint bearers (`src/client.ts:648-652`); security strength belongs to issuance and server verification. No client-side entropy defect established solely by this check.                                                                                                                                  |
| D07 connection cap uses request count            | Confirmed with current lines and narrower surface; TERM-DEV-012.                                                                                                                                                                                                                                                                                                                             |
| D08 no response reassembly                       | Remains in the stub at `src/transport.ts:606-626`. The new live seam explicitly handles one physical frame, while `src/client.ts:247-251` treats encoded fragments as separate request/response exchanges. Do not describe this as a working logical continuation protocol. TERM-DEV-003 addresses the additional physical-stream defect; wire continuation design remains a cross-repo gap. |
| D09 no retry/idempotency and unbounded IDs       | No automatic retry found in the TS source search; request IDs still increment (`src/inspection.ts:655-659`, `src/automation.ts:800-804`). Lack of automatic retry is not inherently wrong for mutating operations, and safe-integer exhaustion is not a demonstrated practical P1. Document caller recovery semantics instead.                                                               |
| D10 scope matrix TS/Rust drift                   | Fixed for the cited methods. Current sets agree at `src/protocol.ts:183-222` and `crates/devtools-client/src/protocol.rs:104-145`; Rust now has CTX-0159 and automation tests.                                                                                                                                                                                                               |
| D11 bounds gaps and incompatible matrix JSON     | Matrix shape divergence addressed: Rust generator at `crates/devtools-client/src/compat.rs:156-227` now mirrors TS and has a shared-golden assertion at `:273-282`; the selected TS golden test passed. Rust still implements a smaller API/bounds subset. Missing TS-only constants alone do not prove incorrect enforcement in a Rust feature that does not exist.                         |
| D12 byte/character frame validation disagreement | Rejected as stated. TS applies the UTF-8 byte bound first (`src/protocol.ts:78-85`); the later code-unit bound is redundant, not a demonstrated divergent acceptance rule. Actual Unicode chunking defect is TERM-DEV-011.                                                                                                                                                                   |
| D13 mutating campaigns lack scratch gate         | Partially fixed: default keystroke injection removed in HEAD. Remaining preflight ordering, workspace/spawn mutation, and run ownership defects are TERM-DEV-009. Process-dispatch calls already carry per-command timeouts (`src/campaign.ts:461-464`); the older statement that every step lacks a timeout is too broad.                                                                   |
| D14 conformance does not check code              | Class and expected exit remain the main assertions (`src/campaign.ts:866-900`); `expectedExitForError` is not applied to observed envelopes here. The old example treating an authentication code in the permission family as inherently inconsistent is not sufficient. Require an explicit allowed class/code table and command matching before assigning stronger findings.               |
| D15 runtime UID fallback                         | Hard-coded fallback remains at `src/client.ts:128`. CLI entry derives its UID from the process; its actual major failure is TERM-DEV-001. Explicit socket selection is not inherently unreachable merely because a default UID exists. Windows live dialing remains unavailable rather than silently working.                                                                                |
| D16 tracing units and spool paths                | Byte and paging defects confirmed/refined as TERM-DEV-007/008. Start helpers still return different hard-coded paths (`src/tracing.ts:229`, `:271`), but inspected methods perform no spool write. This is misleading metadata, not evidence of actual files with incorrect permissions.                                                                                                     |
| D17 rate limiter/framer overflow                 | Sustained-rate defect remains, TERM-DEV-013. Resetting an invalid stream buffer may be a legitimate fail-closed policy; preserving arbitrary prefixes is not automatically safer.                                                                                                                                                                                                            |
| D18 live/shadow and semantic snapshot gap        | Fallback remains explicitly injected and disconnected at `src/inspection.ts:706-718`; runtime-stats discrimination remains fail-closed at `:925-935`. Server implementation was not reread here, so “live getSnapshot always unavailable” is not renewed as a current server fact. Actual live-client routing gap is TERM-DEV-002.                                                           |

Additional recent fixes were inspected: `truncateToBytes` is now single-encode with bounded boundary backoff (`src/bounds.ts:120-140`), FIFO operations no longer use repeated front removal (`src/queue.ts:73-81`), and compatibility metadata tracks hash version 5 while explicitly describing pseudo-hashes (`src/compat-matrix.ts:135-143`). These are not repeated as stale defects.

## Optimizations, hardening, and uncertainty — not confirmed findings

- **Export versus preview:** The former literal self-comparison was replaced by `assertPreviewMatchesExport` (`src/tracing.ts:590-598`). Both call sites still derive a preview and its checked source from the same selected prefix; re-redaction is many-to-one. This is not an independent proof that all actual export bytes equal a reviewed artifact. `fetchTraceChunk` returns the raw stored chunk while showing a redacted prefix. No disk export/transmission path was audited or executed; do not claim a demonstrated external leak or a complete export safety fix. Prefer an immutable sanitized export artifact and preview its actual bytes.
- **Trace semantics:** Local append helpers do not visibly implement input-class admission, and structured JSON chunks are concatenated without JSONL separators. Drop policy and coalescing claims need a dedicated model review. Keep local synthetic event generation and conceptual spool modes separate from real collection/storage guarantees.
- **Socket lifecycle:** The TS seam has one pair of pending response callbacks, no explicit concurrent-call admission, and an empty close callback. Partial-write handling and late-open cleanup also deserve review. The Rust adapter sets read/write timeouts after a blocking connect; those are not an absolute dial/whole-operation deadline. No platform/runtime-specific timing claims were tested.
- **Peer provenance:** Caller-supplied credentials in the stub and filesystem endpoint attestation are not observations of the connected peer. This pass did not prove live OS peer-credential verification or Windows named-pipe support; no authorization-bypass claim is made.
- **Script admission:** `probe_state` in `scripts/workflow-import.sh:121-134` turns SQLite errors/timeouts into empty or zero observations. Its local non-empty protection is therefore not fail-closed. Downstream importer refusal behavior was not reviewed; no actual database replacement is asserted. Treat probe failure as a blocking unknown, not empty state.
- **CLI validation duplication:** `dispatchLive` uses casts and much weaker result checks than typed inspection (`src/cli.ts:457-539`). Consolidate schema parsing and terminal-safe presentation across live and headless paths; a TypeScript cast is not validation. Full unknown-field policy and terminal rendering safety remain unverified.
- **Automation session lifecycle:** `automationClient` snapshots scopes into a new Set (`src/client.ts:658-660`), while its adapter references mutable current transport state. Review revocation and reconnect semantics with fake session identities; server enforcement cannot be inferred from client convenience checks.
- **Compatibility evidence:** Generated `stateHash` values are FNV hashes of repeated surface names, not terminal replay hashes. Artifact `self: PASS` must not be treated as terminal compatibility evidence. Add an explicit machine-readable synthetic provenance marker. Hashing is O(n) time and O(1) auxiliary state after input construction; actual replay would have a different workload.
- **Performance:** Repeated chunk-prefix encoding/cloning in trace append copies up to a chunk per record. Cache byte lengths and mutate the last chunk without cloning where practical. With n appended records and maximum chunk size C, current repeated-prefix work can approach O(nC); counters make admission O(1) plus O(record bytes), with O(retained bytes) space. No benchmark was run.
- **Documentation/CI drift:** AGENTS and SECURITY still say pre-implementation; README claims broader live support than wiring establishes. CI quality setup has no explicit frozen dependency-install step before `just check`, whose recipes use dynamic runners. CodeQL's configuration file is not explicitly selected by its workflow. No remote workflow run, clean-clone build, or dependency-resolution experiment was performed, so these remain engineering follow-ups rather than proven CI failures.

## Coverage ledger

The tracked inventory was obtained with `git ls-files`, not inferred from a sampled directory listing. It contains **17 source TypeScript files, one TypeScript executable, 11 Rust source files, 16 TypeScript test files, one JSON fixture, two shell scripts**, and the tracked configuration/governance files classified below. Ignored worktrees, dependencies, build output, Git objects, and CarryCtx databases are outside body coverage.

**Full** means all file lines were read. **Sampled** means only the listed inclusive ranges were read; unlisted ranges remain unread. **Unread body** includes outline-only and search-only access. Running a test does not upgrade body-reading coverage.

### Full source and script reads

| File                                       | Lines |
| ------------------------------------------ | ----- |
| `bin/bitty-devtools.ts`                    | 1-30  |
| `src/bounds.ts`                            | 1-154 |
| `src/index.ts`                             | 1-27  |
| `src/ipc-socket.ts`                        | 1-420 |
| `src/protocol.ts`                          | 1-222 |
| `src/queue.ts`                             | 1-124 |
| `src/redaction.ts`                         | 1-160 |
| `crates/devtools-client/src/bounds.rs`     | 1-120 |
| `crates/devtools-client/src/inspection.rs` | 1-115 |
| `crates/devtools-client/src/lib.rs`        | 1-81  |
| `crates/devtools-client/src/protocol.rs`   | 1-233 |
| `scripts/workflow-import.sh`               | 1-220 |
| `scripts/workflow-publish.sh`              | 1-218 |

### Sampled source reads

| File                                       | Read ranges                                     |
| ------------------------------------------ | ----------------------------------------------- |
| `src/auth.ts`                              | 225-376                                         |
| `src/automation.ts`                        | 307-630, 700-920                                |
| `src/campaign.ts`                          | 61-105, 387-630, 715-1170, 1180-1270, 1360-1616 |
| `src/cli.ts`                               | 93-123, 142-242, 273-715                        |
| `src/client.ts`                            | 68-96, 120-282, 477-568, 570-685                |
| `src/compat-matrix.ts`                     | 133-269                                         |
| `src/control.ts`                           | 67-281                                          |
| `src/inspection.ts`                        | 620-744, 916-988                                |
| `src/panel-runtime.ts`                     | 21-136, 192-246                                 |
| `src/tracing.ts`                           | 118-318, 398-598                                |
| `src/transport.ts`                         | 100-191, 197-247, 400-661                       |
| `crates/devtools-client/src/auth.rs`       | 255-410                                         |
| `crates/devtools-client/src/compat.rs`     | 120-235, 244-284                                |
| `crates/devtools-client/src/control.rs`    | 130-280                                         |
| `crates/devtools-client/src/ipc_socket.rs` | 1-280                                           |
| `crates/devtools-client/src/redaction.rs`  | 1-262, 391-426                                  |
| `crates/devtools-client/src/tracing.rs`    | 340-550                                         |
| `crates/devtools-client/src/transport.rs`  | 218-263, 550-635, 640-704                       |

Every source file received at least an outline and a body slice. This does not imply full module coverage: especially large inspection parsers, transport queue mechanics, redaction helper internals, tracing stream generation, and Rust inline tests remain substantially unread.

### Tests and fixture

Full reads:

- `tests/bounds.test.ts:1-73`
- `tests/compat-matrix.test.ts:1-101`
- `tests/ipc-socket.test.ts:1-130`
- `tests/protocol.test.ts:1-110`
- `tests/redaction.test.ts:1-76`
- `tests/tracing.test.ts:1-138`
- `tests/transport.test.ts:1-249`

Sampled reads:

- `tests/auth.test.ts:95-175`
- `tests/automation.test.ts:437-543`
- `tests/campaign.test.ts:540-795`

Unread test bodies:

- `tests/cli.test.ts`, `tests/client.test.ts`: displayed folded outlines only.
- `tests/control.test.ts`, `tests/inspection.test.ts`, `tests/panel-runtime.test.ts`, `tests/queue.test.ts`: inventoried, bodies unread.
- `tests/fixtures/compat-matrix-golden.json`: not manually read; consumed by the passing existing TS golden test.

Rust inline test coverage is exactly the source ranges above. Socket loopback test bodies at `crates/devtools-client/src/ipc_socket.rs:301-342` were outlined, not read or executed. No Rust tests were run.

### Configuration, governance, and documents

Full reads within devtools:

- `AGENTS.md`, `package.json`, `tsconfig.json`, `Cargo.toml`, `crates/devtools-client/Cargo.toml`, `justfile`, `SECURITY.md`, `lefthook.yml`.
- `.github/workflows/ci.yml`, `.github/workflows/codeql.yml`, `.github/workflows/snapshot-source.yml`, `.github/codeql/codeql-config.yml`, `.github/dependabot.yml`.

Sampled: `README.md:1-220`; the remainder was not read.

Unread tracked bodies:

- `.carryctx/README.md`, `.carryctx/config.toml`, all six `.carryctx/personas/*.md`, all three `.carryctx/rules/*.md`, `.carryctx/workflows/issue-to-merge.md`. No CarryCtx state or workflow invocation occurred.
- `.gitattributes`, `.gitignore`, `.markdownlint-cli2.jsonc` in devtools.
- `.github/ISSUE_TEMPLATE/bug_report.md`, `.github/ISSUE_TEMPLATE/feature_request.md`, `.github/PULL_REQUEST_TEMPLATE.md`.
- `CHANGELOG.md`, `CONTRIBUTING.md`, `LICENSE`, `repo.toml`, `rust-toolchain.toml`, `commitlint.config.ts`, `Cargo.lock`, `bun.lock`.

External context consulted:

- `bitty-terminal-docs/specifications/devtools-rfc.md:1-240,350-579`: accepted versus implemented-only status, privacy, scope, framing, automation, and rate contracts. Other lines and linked normative security documents were not read. Its scope prose/table and version descriptions contain ambiguities; this report does not resolve them by assumption.
- `research/summary/007.md`: full, 24 lines; historical code/docs status divergence, not current implementation evidence.
- `research/review/2026-09-15/08-bitty-devtools.md`: full, 224 lines; all defect headings dispositioned above.
- September 17 terminal sibling reports: `01-terminal-engine.md` full, `03-app-ui-core-config.md` full, `02-runtime-orchestration.md:1-230` sampled. They provide coordination context only; their findings were not independently adopted as devtools evidence.
- Umbrella AGENTS full, 276 lines; research AGENTS full, 47 lines; terminal-docs AGENTS full, 145 lines; research `.markdownlint-cli2.jsonc` full, 11 lines.

The rest of the research archive and documentation corpus, all dependency source, ignored `.worktrees/`, `node_modules/`, `target/`, and server implementations are unread/outside this pass. No dependency vulnerability audit or license audit was performed.

## Checks and limits

Executed from `bitty-devtools/`:

| Check                                                                                                       | Result                                               | What it establishes                                                             |
| ----------------------------------------------------------------------------------------------------------- | ---------------------------------------------------- | ------------------------------------------------------------------------------- |
| `bun test tests/protocol.test.ts tests/transport.test.ts tests/tracing.test.ts tests/compat-matrix.test.ts` | Exit 0; 38 pass, 0 fail, 144 expectations; Bun 1.4.2 | Existing selected headless assertions only                                      |
| `bun run check:types`                                                                                       | Exit 0; `tsc -p tsconfig.json --noEmit`              | Current `src/**/*` type checking; tests and executable are excluded by tsconfig |
| `bash -n scripts/workflow-import.sh scripts/workflow-publish.sh`                                            | Exit 0                                               | Shell syntax only; neither workflow was executed                                |

No standalone `tsc` was found on PATH; the existing project script resolved its installed compiler successfully. No installation was attempted. `just check` was deliberately not run because its dynamic runners and Rust build/test recipe exceed this read-only/no-fetch verification plan.

Report-only validation: `markdownlint-cli2 --no-globs "review/2026-09-17/bitty-terminal/06-devtools.md"`, using installed version 0.23.1, exited 0 and reported zero issues for the one selected file. `--no-globs` prevents the research configuration from expanding linting to unrelated reports. Product formatting/lint, full tests, Rust compile/clippy/tests, real CLI/socket interoperability, Windows/macOS behavior, GUI or campaign behavior, performance, fault injection, and downstream importer behavior were not verified. Safe test ideas above are proposals, not executed evidence.

No custom test was written and no finding is labeled runtime-reproduced. The selected green checks must not be represented as evidence that these findings are fixed.
