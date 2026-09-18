# Context, slices, and provider review — 2026-09-17

## Verdict and scope

**Second-pass disposition: 2 P1 and 6 P2 correctness/resource defects; 2 P2 improvement/contract gaps.** AI-CTX-003 and AI-CTX-010 are reclassified below, not counted as proven correctness failures. All IDs are preserved. No reproduction, attack payload, provider call, Rust test, build, installation, network fetch, source modification, CarryCtx operation, nested agent, or commit was performed.

The original coverage ledger describes the first pass, not the independent verifier's coverage. The added verification section records the second pass and its current baselines; edits are confined to these two reports and their campaign product index, `README.md`.

The requested scope was the entire `bitty-ai-slice` crate and runtime context, compression, prompt assembly, cache, fingerprinting, fallback, selection, token budgets, and adapters. The actual body-read coverage is substantially narrower, especially integration tests. This is **not exhaustive coverage or approval**. The exact ledger below distinguishes full, sampled, outline-only, and unread files. Session/tool orchestration findings already recorded in `01-runtime-and-sessions.md` are not repeated.

Paths beginning `crates/` are relative to `bitty-ai`; other paths explicitly name the owning repository. Prior reports are historical claims, not validation evidence. Priorities reflect correctness/resource/privacy consequences within these experimental APIs, not a claim of deployed exploitation.

## Baseline

| Repository      | Initial and pre-report HEAD                | Dirty baseline and recheck                                                               |
| --------------- | ------------------------------------------ | ---------------------------------------------------------------------------------------- |
| `bitty-ai`      | `97d312585c20a5b17063084cf66ba1bb7438667c` | Clean both times; `git diff --check` passed                                              |
| `bitty-ai-docs` | `bb66faf88d0a9cff4ea753671e0dbefb5fadea80` | Clean both times                                                                         |
| `research`      | `d70152a079c75204ec37e99a7bf3f54b8190e423` | Untracked `review/2026-09-17/` already present, containing the earlier report; preserved |

The umbrella workspace is not a Git repository. Initial Git commands against it failed; subsequent Git inspection used each owning repository. No baseline mutation resulted.

The implementation's checked-out docs submodule is pinned at `07169bd9108875179899ac77c3af3c0073a8506a`, different from the standalone docs checkout used for this review. No refresh/fetch was attempted and no lag count is asserted. The slice IPC dependency is pinned at `cfeffa2d8e1387029850940af2b64877dbfbe25f` (`crates/bitty-ai-slice/Cargo.toml:18`). Its upstream source was not independently reviewed here.

### New runtime files since the preceding reviewer began

A local diff from that reviewer's initial `e7cbe69f0fac8a785657b5ff0bab850b66e9f690` to current HEAD shows only:

- `src/fallback.rs`: 364 added lines.
- `src/lib.rs`: five added lines.
- `tests/fallback_envelope.rs`: 385 added lines.

This review inspected fallback production code at lines 121-353, its inline test at 355-364, and integration-test lines 1-210. AI-CTX-008 addresses this new module. The remainder of its integration suite and its module-header declarations were not body-read; the export change was diff-stat verified, not independently body-reviewed. HEAD remained unchanged through the pre-report recheck.

## Contracts and stable-prefix assessment

Workspace, source-repository, documentation-repository, and research `AGENTS.md` guidance was read. No nested guide was found under the crates or research. The ctxctl skill was loaded; installed `ctxctl outline` provided module/symbol discovery before source slices. Folded outlines are not full coverage. No CarryCtx workflow was used, as requested.

The canonical documents are proposals with important qualifications:

- `bitty-ai-docs/specifications/prefix-cache-context-design.md:60-87` places stable content before dynamic content, but explicitly calls the exact seven-layer split a proposal. Lines 94-107 subordinate append-only history to deletion, expiry, revocation, and redaction.
- Its lines 118-131 require deterministic serialization for future cache claims; lines 269-307 distinguish local prefix estimates from provider telemetry and require provider/model/configuration-aware cache scope. Lines 333-346 defer epochs, planner types, and wider routing work.
- `bitty-ai-docs/specifications/prompt-layering-design.md:85-100`, `:167-180`, and `:332-350` describe stable Core/User/Project/Skills preceding trailing runtime/current-turn material. Lines 182-211 explicitly state that prompts never grant capabilities.
- `bitty-ai-docs/specifications/context-management.md:80-82` requires deletion/expiry propagation to derived summaries, artifacts, indexes, and caches; append-only history is subordinate to those obligations. These passages were obtained through targeted search, not a full document read.
- Search excerpts from `research/summary/025.md:7-20`, `026.md:14`, `027.md:16`, `023.md:13-16`, and `028.md:8-22` support stable-before-dynamic ordering, small immutable Core text, and slice-as-harness positioning. Summaries were searched, not read in full; original research records were not consulted.

### What current code actually proves

`prompt.rs:1029-1045` rank-orders five text layers, and `:1241-1258` uses length-prefixed deterministic framing. The effective policy block follows Runtime/Turn (`:1258-1336`), keeping stable text unchanged when trailing policy changes. `cache_key.rs:202-230` walks declared section lengths instead of searching for an embedded marker. These are real implementation advances beyond the older docs snapshot at `prefix-cache-context-design.md:348-369`.

However, this is a **text-prefix primitive, not an integrated inference-cache implementation**:

- `context.rs:843-865` preserves caller order rather than assigning stability classes. Host ordering remains necessary.
- `agent.rs:488-502` places seed context messages before the supplied user prompt. It does not call the five-layer prompt assembler. If a host puts the Core contract inside that prompt string, dynamic seed context precedes it. There is no typed system-message role in the role mapping inspected in `local_provider.rs:576-581`.
- `cache_key.rs:56-63` provides scope _categories_, not session/turn/round identities. The key has provider/model names but no model revision, tokenizer/configuration version, or consent principal (`:129-140`). An enclosing host namespace must supply these missing boundaries before reuse across requests is safe to claim.
- The hash ends before Runtime/Turn and therefore excludes the entire effective policy block. A stable-layer directive change can keep this text-prefix key unchanged. That is not inherently a prefix-cache bug: unchanged prefix bytes are reusable. It is unsuitable as a whole-prompt, authorization, or result-equivalence key.
- `InputFingerprint` is an explicitly host-assembled digest seam, not discovery of complete environmental inputs. Its FNV-1a-64 plus length comparison is not mathematical proof of exact byte equivalence; the universal wording at `fingerprint.rs:62-66` is too strong. No collision construction was attempted.

No provider cache hits, token-prefix equality, latency improvement, or complete wire-request ordering was measured.

## Findings and second-pass dispositions

### AI-CTX-001 — P1: Compression restarts source retention

**Evidence:** `crates/bitty-ai-runtime/src/compression.rs:468-472`, `:570-600`, `:765-789`, `:1191-1257`; `bitty-ai-docs/specifications/context-management.md:82`.

**Cause:** Compression inherits a retention class but records `created_at_ms = now_ms`. Expiry compares that new timestamp with the class TTL, while passthrough records use their original collection timestamp. The source timestamps/deadlines are absent from `CompressedSpan`.

**Impact:** Compression extends the lifetime of derived material beyond its source's original expiry, despite the documented “compression never extends retention” rule. The existing test at `:1243-1245` explicitly expects survival based on compression time rather than source age; green execution of that test would not establish the intended privacy invariant.

**Remediation:** Carry the earliest applicable source expiry into each derived span, separately from its creation timestamp. Evaluate source eligibility before summarization and ensure later retention-policy changes cannot extend an inherited deadline accidentally.

**Safe test idea:** Use ordinary timestamp/class tables, including old sources compressed near expiry and sources with different deadlines. Assert that compression never delays the first applicable expiry. No sleeps or provider calls are needed.

### AI-CTX-002 — P1: Compression hides stale generations from assembly

**Evidence:** `crates/bitty-ai-runtime/src/compression.rs:721-724`, `:771-795`; `crates/bitty-ai-runtime/src/context.rs:488-509`, `:662-672`.

**Cause:** Compression validates only each record's provider/body bounds, then assigns the synthetic record the maximum source generation. It neither requires a common generation nor accepts an authoritative current generation. Assembly later sees only the synthetic generation, not its sources.

**Impact:** A span mixing old and current records can become a current-generation summary; assembly's stale-generation rejection can no longer reject the old contribution. This is a compositional invalidation defect in the documented compression-to-assembly pipeline, not a claim about a live caller currently using it.

**Remediation:** Validate every source against an explicit current-generation requirement before summarization; reject mixed generations or preserve sufficient provenance for downstream validation. Selecting the maximum is not an eligibility check.

**Safe test idea:** Table-driven generation combinations with a recording fake summarizer; rejected sets should invoke no summarizer and produce no derived record.

### AI-CTX-003 — P2 improvement: Externalization precedes projection selection

**Reclassified by independent verification:** The allocation and capacity behavior is confirmed, but `context.rs:605-617` explicitly defines externalize-then-select, and `AssembledContext::externalized` at `:592-594` counts all bodies stored during the call, not selected bodies. Omission from a projection is not deletion of underlying session data (`context-management.md:76-84`). The original classification as a proven correctness failure is withdrawn pending an explicit selected-only retention contract; keep P2 as a quota-efficiency/API improvement.

**Evidence:** `crates/bitty-ai-runtime/src/context.rs:722-774`, `:792-805`, `:823-848`.

**Cause, corrected ordering:** Every surviving large body is staged and checked against store capacity before budget selection. The `included` bitmap is computed at `context.rs:792-805`, before phase-two commit at `:823-841`. Commit ignores that bitmap; only the final output filter at `:843-848` uses it. The original assertion that selection itself occurs after commit was inaccurate.

**Impact:** A successful bounded projection retains large bodies for records omitted from that projection, consuming finite artifact slots/bytes without returning their references. Unselected bodies can also make assembly fail on artifact capacity before a smaller valid subset is considered. The September 15 atomic-error issue is fixed, but successful assembly can still waste quota.

**Remediation:** Stage/select with consistent provisional footprints, then capacity-check and commit only selected externalizations; make predicted IDs and actual references agree after filtering. If retention of omitted content is intentional, expose it explicitly and account for it separately from projection assembly.

**Safe test idea, conditional:** If selected-only retention is adopted, use small ordinary records under a one-record projection budget and assert that retained count/bytes and `externalized` match selection. Otherwise pin the existing externalize-all semantics explicitly, including near-capacity refusal.

### AI-CTX-004 — P2: Local provider validates sampling but silently drops it

**Evidence:** `crates/bitty-ai-slice/src/local_provider.rs:507-536`, `:589-607`; `crates/bitty-ai-runtime/src/provider.rs:360-402`, `:427-429`.

**Cause:** `complete` validates `request.sampling`, but the request builder receives only model/messages and emits only model, messages, and `stream:false`.

**Impact, scoped:** Explicit sampling values have no effect on the outgoing request when a caller uses `ModelProvider::complete` with declarations. Validation support does not prove the backend supports every field; unsupported declarations should be refused, not silently ignored. `Agent::run_turn` currently supplies `sampling: None` (`agent.rs:529-532`), so no lost setting through that driver was established. The adapter intentionally emits a minimal chat body (`local_provider.rs:39-43`), but neither that declaration nor a response-byte cap implements a requested completion-token budget.

**Remediation:** Serialize supported fields with deliberate backend-specific mapping, and reject unsupported declared options before I/O. Preserve the distinction between undeclared fields and explicit values.

**Safe test idea:** Pure request-serialization assertions for supported fields and typed refusal for unsupported declarations. No live inference is needed.

### AI-CTX-005 — P2: Socket timeouts do not implement the request deadline

**Evidence:** `crates/bitty-ai-slice/src/local_provider.rs:618-650`, `:665-672`, `:745-794`; `crates/bitty-ai-runtime/src/provider.rs:423-426`.

**Cause:** Connect, writes, and each read receive separate timeout allowances. The read loop does not subtract elapsed time from a whole-request deadline; write-timeout setup failure is ignored. Success reports zero latency (`local_provider.rs:541`).

**Impact:** Completion can exceed the caller's requested duration while individual I/O operations remain within their own timeout. A read timeout cannot bound an earlier blocked write. The module's “binding deadline” comment is stronger than the implementation.

**Remediation:** Use one monotonic deadline inside this I/O-bearing adapter, derive remaining time for each operation, and treat failure to arm mandatory timeouts as failure. Keep caller-supplied logical time distinct from elapsed transport time.

**Safe test idea:** An injected clock/I/O abstraction can verify remaining-time calculations and timeout-arm refusal without slow sockets, live servers, or resource-stress tests.

### AI-CTX-006 — P2: Local response parsing is not fully bounded or panic-free

**Evidence:** `crates/bitty-ai-slice/src/local_provider.rs:801-839`, `:855-897`, `:1124-1132`, `:1160-1209`, `:1293-1301`.

**Cause:** The Unicode-escape branch slices a UTF-8 `str` using byte offsets before confirming character boundaries. Object/array recursion has no explicit nesting bound. A completed header block is never compared with `MAX_LOCAL_HEADERS_BYTES`; that bound is consulted only when the header terminator is missing. The aggregate receive cap remains present.

**Impact, bounded claim:** Malformed response text is not guaranteed to become a typed parser error; Unicode boundary handling can panic before digit validation. The HTTP response reaches whole-object validation at `local_provider.rs:976-983`, which calls the recursive parser, so this is not merely an unused helper. Nesting has no separately enforced depth budget, but aggregate response bytes still impose a finite input bound; stack exhaustion on a particular platform was not established. Complete headers can exceed the stated independent header ceiling. No input payload, executable reproduction, or deployment claim is provided.

**Remediation:** Inspect escape digits as bytes with checked access, enforce an explicit nesting/depth budget or use an iterative bounded parser, and check completed header length before header parsing. Consider a reviewed parser dependency at the experiment boundary rather than expanding handwritten parsing indefinitely.

**Safe test idea:** Small conformance tables for invalid Unicode escapes and explicit depth/header policy boundaries, asserting typed refusal. No stress input or attack payload is included in this report.

### AI-CTX-007 — P2: Bridge peer errors leave pending requests behind

**Evidence:** `crates/bitty-ai-slice/src/bridge.rs:269-288`, `:295-300`.

**Cause, narrowed:** After enqueue/dequeue, `peer.serve(&request)?` can return without `endpoint.complete(id)`. Cleanup exists for mismatched response IDs, but not for this transport-level peer-error path. Application-level error responses instead reach completion at `bridge.rs:295` before being returned as errors at `:301-304`. The original response-enqueue-failure example is withdrawn as an independently established path: this bridge drains every successful response synchronously and exposes no queue mutation handle.

**Impact:** Failed synchronous calls retain pending correlation entries. The bridge exposes no pending-expiry/drain operation to recover them, so repeated ordinary peer failures can consume finite pending capacity. This is distinct from the already-corrected mismatched-ID path.

**Remediation:** Centralize pending-request cleanup across all terminal exits after request admission. Preserve the original typed error and separately classify uncertain effect outcomes where applicable.

**Safe test idea:** A benign peer that returns a typed service error should leave `pending_count()` unchanged; a later successful call should still work. No effectful host operation is required.

### AI-CTX-008 — P2: New fallback envelope bounds are bypassable and checked after allocation

**Evidence:** `crates/bitty-ai-runtime/src/fallback.rs:121-157`, `:183-197`, `:312-346`; `crates/bitty-ai-runtime/tests/fallback_envelope.rs:36-145`.

**Cause:** The fields are public, permitting direct construction or mutation without `new`; `to_bytes` does not revalidate before allocating/encoding. Separately, `scrub_text` grows a string across the entire input, and potentially allocates a lossy UTF-8 copy, before `fallback_for` rejects an over-cap result. Initial capacity is not a hard cap.

**Impact, narrowed:** Public-field mutation followed by encoding lacks the constructor's invariant; this is an API-hardening observation, not proof that validated decoding accepts over-cap envelopes. `from_bytes` explicitly checks field bounds (`fallback.rs:211-233`). The retained P2 resource defect is in the supported `fallback_for` path: scratch memory grows with input before refusal. Tests already cover cap-adjacent refusal (`tests/fallback_envelope.rs:295-314`) and decoder rejection (`:329-385`); they do not establish a scratch-allocation ceiling. The original “constructor-only” description was too broad.

**Remediation:** Make invariant-bearing fields private or validate at every encoding boundary. Scrub incrementally into bounded storage; if exact output length is required in errors, count remaining characters without retaining them.

**Safe test idea:** API-level tests covering all supported constructors/encoders and ordinary cap-adjacent fallback inputs. Verify errors and bounded retained output; large allocations are unnecessary.

### AI-CTX-009 — P2: Journal schema admission checks column names, not required semantics

**Evidence:** `crates/bitty-ai-slice/src/journal_prototype.rs:89-116`, `:206-236`, `:389-422`.

**Cause:** Schema verification compares only column-name lists for tables that already exist. It ignores column types, primary-key/unique/not-null constraints, missing portions of an existing schema, and the stored profile value. `CREATE TABLE IF NOT EXISTS` cannot repair incompatible constraints on an existing table.

**Impact, narrowed:** A database with matching column names but different constraints can be admitted and modified, although append order and duplicate-ID protection rely on those constraints. Unrelated or partial databases can also receive missing prototype tables, but whether every such database must be refused is a separate admission-policy question, not a proven constraint failure.

**Remediation:** Identify a new empty store separately from an existing journal; verify schema/profile version and required constraints before any write. Define migrations explicitly rather than treating name matches as compatibility.

**Safe test idea:** Tiny local schema fixtures with one changed constraint or missing table, asserting typed refusal and unchanged data. The second pass read the wrong-column-name regression at `journal_prototype.rs:605-629`; it was not executed and does not cover matching names with incompatible constraints.

### AI-CTX-010 — P2 hardening gap: Reassembly has no independent aggregate ceiling

**Reclassified by independent verification:** Missing aggregate validation is confirmed, but `fragment_transport.rs:36-39` assigns the 64 KiB refusal to the two splitting entry points, not every reassembler. Its `:428-436` promises byte equality when parts came from `pre_split_fragment`; that bounded path preserves the source limit. Public hand-built collections can exceed it, but no deployed ingress using such collections was established (`:60-64` explicitly says there is no live path). Withdraw the original implication of a demonstrated splitter-to-reassembler correctness failure; retain P2 as defensive admission work before untrusted transport integration.

**Evidence:** `crates/bitty-ai-slice/src/fragment_transport.rs:364-378`, `:478-555`.

**Cause:** The split entry point checks the runtime fragment byte cap, but reassembly validates only each part's size and consistency with the supplied `part_count`. It has no aggregate reconstructed-byte bound or independently derived maximum part count, and grows the output during validation.

**Impact:** Public reassembly can accept a consistent part collection that could never have come from one permitted runtime fragment. Its output allocation is bounded by supplied part count times per-part bytes, not by `MAX_FRAGMENT_BYTES`.

**Remediation:** Validate aggregate checked length against the source-fragment limit before allocation, and bound part count according to the documented splitting contract. Keep identity/order checks intact.

**Safe test idea:** Small boundary-focused part collections, including an aggregate just beyond the source cap; assert refusal without retaining a partial reconstructed output.

## Earlier findings rechecked

### September 15

Source: `research/review/2026-09-15/07-bitty-ai.md`, read in full.

| Earlier item                                            | Current disposition                                                                                                                                                                                                                                 |
| ------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| P1-3 snapshot JSON interpolation                        | Fixed in inspected production code: `harness.rs:65-83`, `:111-124` escape string values. Inline regressions were outlined, not executed.                                                                                                            |
| P1-4 global content substring extraction                | Original issue fixed: path lookup at `local_provider.rs:976-1001`, `:1039-1117`. This is not parser-wide approval; AI-CTX-006 is separate.                                                                                                          |
| P1-5 sequence restart / transport mismatch              | Runtime batches were addressed in the preceding report; current cursor at `fragment_transport.rs:596-614` assigns dense transport sequence values and splits before ingestion. Whole-turn integration test was outlined, not body-read or run here. |
| P1-8 assembly mutation on error                         | Original issue fixed by staging/cap checks before commit, `context.rs:722-841`. AI-CTX-003 concerns omitted bodies retained on success, not the old partial-error claim.                                                                            |
| P2-2 unused detail/priority and stale/duplicate records | Request fields removed; reserved `DetailLevel` explicitly documented as unused at `context.rs:261-289`. Duplicate IDs and generation now reject at `:651-672`. Compression can still hide source generations: AI-CTX-002.                           |
| P2-3 prompt budget and empty allow-set                  | Fixed in inspected implementation: `prompt.rs:1075-1081`, `:1192-1198`. Empty scopes are a distinct policy shape; no unsupported claim that every empty intersection must error is made.                                                            |
| P2-4 hyphenated skill names                             | Separate skill validator is called at `prompt.rs:2050-2052`; the validator body itself was not read, so full namespace correctness is not claimed.                                                                                                  |
| P2-6 alias fallback / zero weights                      | Alias branch is exclusive at `selection.rs:575-591`; selection filtering and cost estimation use zero-as-one at `:649-652`, `:747-764`.                                                                                                             |
| P2-8 path and IPv6 formatting                           | Original defects fixed in inspected code: allowlist at `local_provider.rs:425-447`, bracketed IPv6 at `:455-463`, bracketed localhost refusal at `:288-300`.                                                                                        |
| Undefined protocol-to-client-ID rule                    | A concrete bound-principal mapping now exists, `bridge.rs:87-136`; it is not authentication evidence by itself.                                                                                                                                     |
| Pin drift tooling absent                                | README and justfile now describe a local-only drift gate; script implementation was not reviewed or executed.                                                                                                                                       |

### September 16

The September 16 report filenames were inventoried, but their source bodies were **not independently read in this review**. The September 17 predecessor's summaries at `01-runtime-and-sessions.md:168-174` were read and treated as secondary evidence only. Therefore no blanket confirmation or closure of September 16 findings is claimed.

Specific inherited themes were rechecked against current source: the old marker-scan claim is superseded by length-aware `cache_key.rs:202-230`; the slice has a loopback HTTP provider and SQLite journal prototype, so an unqualified whole-slice “no network/no persistence” description is inaccurate. Runtime remains a distinct synchronous skeleton. Any remaining September 16 claim, including cross-repository lag counts or architectural viability judgments, requires direct follow-up rather than repetition here.

## Optimizations and maintenance observations

These are separate from confirmed correctness defects:

1. Context dedupe uses a linear survivor scan and clones retained records (`context.rs:690-719`). For N records and B bounded comparison bytes, worst-case comparison work is O(N²·B), with O(N·B) retained scratch space. N is capped at 128; benchmark before replacing the simple representation.
2. JSON validation decodes strings that are subsequently decoded again during path lookup. For response bytes B and depth D, basic traversal is O(B) per pass with O(D) stack; multiple fixed-path passes add allocation churn. A bounded parser or borrowing scanner could reduce copies after correctness repairs.
3. Prefix parsing/hash construction is O(P) time for P canonical bytes and constant parser scratch space, excluding owned route strings. Fingerprinting is O(B) time and O(1) hash state. Cryptographic or equality-verifying designs should be selected according to the actual cache's trust model, not justified solely by speed.
4. The stable policy block is serialized after dynamic text, so stable directives do not contribute to the reusable leading prefix. Separate stable and dynamic policy serialization is a potential optimization, not a requirement to move changing budgets into the prefix.
5. `crates/bitty-ai-slice/README.md:33-40` lists only the journal exception to its deterministic/no-network statement, while `:50` and `:87-88` describe the networked local provider. Tighten the scope of that introductory statement; no source fix was made.

## Gaps and hypotheses, not additional confirmed findings

- Journal tombstoning explicitly leaves payload rows intact (`journal_prototype.rs:32-35`, `:276-279`), whereas canonical context guidance requires payload deletion/expiry propagation. Treat this prototype operation as logical hiding, not a user-deletion implementation. Product persistence needs reviewed consent, redaction, file permissions, bounded reads, physical deletion semantics, backups, and crash behavior. None is approved by this review.
- `Journal::read_all` is proportional to all surviving rows; per-field caps do not impose a total journal/read budget. No disk-growth or memory-stress test was run.
- Context record `id` and supersede strings are not length-validated in the inspected `ContextRecord::validate`; rendering also adds labels and IDs outside `footprint_bytes`. Artifact references are store-local and contain no store/session namespace. Hosts must not mix stores or treat references as capabilities. Broader integration consequences were not traced.
- `ContextRequest` uses bytes/4 estimates (`context.rs:295-310`), not tokenizer counts. The word “conservative” is not substantiated for arbitrary text/model families. Agent calls currently set `max_tokens: None`; no complete token admission or output-reservation pipeline was verified.
- The local provider ignores context references and tools, and models tool observations as user text. It advertises Text only, so absence of tool execution is not itself a defect. The `/api/generate` response fallback does not establish native Ollama request compatibility: the builder always emits a chat body.
- Local response semantic checks do not inspect assistant role/finish reason, and JSON content-type admission uses substring matching. These are adapter conformance gaps requiring an explicit supported-protocol contract; no deployed failure is asserted.
- Cache/fingerprint tests were mostly outline-only or unread. Model revision/tokenizer/principal scoping, complete-input enumeration, invalidation granularity, schema availability, and real provider request serialization remain integration work, not established guarantees.
- Trust labels, scoped-ID derivation, caller-supplied generations, and fake consent fixtures are not authentication/redaction proofs. The upstream IPC implementation, shared security corpus, real host binding, and provider retention policies were not independently inspected.

## Exact coverage ledger

“Full” means every file line was returned and read. “Sampled” lists exact body-read ranges; outlines, directory listings, and search matches do not upgrade coverage. Unlisted portions are unread. The entire requested scope was considered for inventory, not exhaustively reviewed.

### Slice source and metadata

All paths in this table are under `bitty-ai/crates/bitty-ai-slice/`.

| File                        | Coverage                                   | Body-read lines                                 |
| --------------------------- | ------------------------------------------ | ----------------------------------------------- |
| `src/bridge.rs`             | Full                                       | 1-308                                           |
| `src/local_provider.rs`     | Sampled; all production implementation     | 1-1396; tests 1397-2297 unread                  |
| `src/harness.rs`            | Sampled; production implementation         | 1-280; inline tests outlined only               |
| `src/fake_host.rs`          | Sampled                                    | 106-711; header/imports and inline tests unread |
| `src/live_host.rs`          | Sampled                                    | 149-463; header/imports and inline tests unread |
| `src/fragment_transport.rs` | Sampled; mapping/reassembly implementation | 335-639; types/errors outlined only             |
| `src/journal_prototype.rs`  | Sampled; all production implementation     | 1-425; inline tests outlined only               |
| `src/lib.rs`                | Unread                                     | None                                            |
| `src/error.rs`              | Unread                                     | None                                            |
| `README.md`                 | Full                                       | 1-211                                           |
| `Cargo.toml`                | Full                                       | 1-22                                            |

### Slice integration tests

All paths are under `bitty-ai/crates/bitty-ai-slice/tests/`.

| File                   | Coverage                    |
| ---------------------- | --------------------------- |
| `vertical_slice.rs`    | Outline only, no body reads |
| `client_id_binding.rs` | Unread                      |
| `fake_host.rs`         | Unread                      |
| `fragment_mapping.rs`  | Unread                      |
| `host_conformance.rs`  | Unread                      |
| `pinned_surface.rs`    | Unread                      |

### Scoped runtime source

All paths are under `bitty-ai/crates/bitty-ai-runtime/`.

| File                                                                                                      | Coverage                                | Body-read lines                |
| --------------------------------------------------------------------------------------------------------- | --------------------------------------- | ------------------------------ |
| `src/cache_key.rs`                                                                                        | Full                                    | 1-268                          |
| `src/fingerprint.rs`                                                                                      | Full                                    | 1-177                          |
| `src/context.rs`                                                                                          | Sampled                                 | 250-880                        |
| `src/compression.rs`                                                                                      | Sampled                                 | 457-820, 1190-1449             |
| `src/prompt.rs`                                                                                           | Sampled                                 | 972-1441, 1480-2094            |
| `src/selection.rs`                                                                                        | Sampled                                 | 235-818                        |
| `src/fallback.rs`                                                                                         | Sampled                                 | 121-364                        |
| `src/provider.rs`                                                                                         | Sampled                                 | 360-489                        |
| `src/agent.rs`                                                                                            | Sampled, context/provider assembly only | 458-567                        |
| `src/lib.rs`                                                                                              | Unread body                             | Diff stat and predecessor only |
| `src/bridge.rs`, `src/extension.rs`                                                                       | Unread body                             | Predecessor report only        |
| `src/session.rs`, `src/tool.rs`, `src/stream.rs`, `src/reconcile.rs`, `src/adoption.rs`, `src/fencing.rs` | Excluded orchestration review           | Predecessor report only        |
| `README.md`, `Cargo.toml`                                                                                 | Unread in this review                   | Predecessor report only        |

### Runtime integration tests

All paths are under `bitty-ai/crates/bitty-ai-runtime/tests/`.

| File                                                                                                                                                                                     | Coverage                                      |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------- |
| `fallback_envelope.rs`                                                                                                                                                                   | Sampled, 1-210; remaining tests outlined only |
| `cache_key.rs`                                                                                                                                                                           | Outline only                                  |
| `cache_invalidation.rs`                                                                                                                                                                  | Unread                                        |
| `granularity_denial.rs`                                                                                                                                                                  | Unread                                        |
| `input_fingerprint.rs`                                                                                                                                                                   | Unread                                        |
| `accounting_bounds.rs`                                                                                                                                                                   | Unread                                        |
| `subscription_bounds.rs`                                                                                                                                                                 | Unread                                        |
| `schema_invalidation.rs`                                                                                                                                                                 | Unread                                        |
| `sampling.rs`                                                                                                                                                                            | Unread                                        |
| `cost_ceiling.rs`                                                                                                                                                                        | Unread; predecessor report only               |
| `result_schema_disclosure.rs`                                                                                                                                                            | Unread                                        |
| `extension_points.rs`                                                                                                                                                                    | Unread                                        |
| `agent_turn_semantics.rs`, `turn_lifecycle.rs`, `unknown_reconcile.rs`, `runtime_fail_closed.rs`, `batch_evidence.rs`, `writer_fencing.rs`, `mcp_fail_closed.rs`, `recovery_adoption.rs` | Not re-reviewed; predecessor scope            |

### Guidance, docs, and historical evidence

| Path                                                                            | Coverage                                                                |
| ------------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| Workspace `AGENTS.md`                                                           | Full tool output                                                        |
| `bitty-ai/AGENTS.md`                                                            | Full, 1-152                                                             |
| `bitty-ai-docs/AGENTS.md`                                                       | Full, 1-147                                                             |
| `research/AGENTS.md`                                                            | Full, 1-47                                                              |
| Installed `ctxctl-core` skill                                                   | Full loaded guidance                                                    |
| `bitty-ai/Cargo.toml`                                                           | Full, 1-32                                                              |
| `bitty-ai/justfile`                                                             | Full, 1-103                                                             |
| `research/.markdownlint-cli2.jsonc`                                             | Full, 1-11                                                              |
| `bitty-ai-docs/specifications/prefix-cache-context-design.md`                   | Sampled, 1-400 of 426                                                   |
| `bitty-ai-docs/specifications/prompt-layering-design.md`                        | Sampled, 1-350 of 503                                                   |
| `bitty-ai-docs/specifications/context-management.md`                            | Targeted search matches only                                            |
| `research/summary/*.md`                                                         | Context/cache/slice-related search matches only, not full-file coverage |
| `research/review/2026-09-15/07-bitty-ai.md`                                     | Full, 1-185                                                             |
| `research/review/2026-09-17/bitty-ai/01-runtime-and-sessions.md`                | Full, 1-286                                                             |
| `research/review/2026-09-16/*.md`                                               | Filenames only; no direct body review                                   |
| Other canonical specs, security corpus, research originals, upstream IPC source | Unread                                                                  |

## Independent second-pass verification

The second pass checked every P1 and P2 finding's causal path, not just its cited expressions. Reclassifications above supersede the first pass's blanket “confirmed” label. First-pass coverage tables remain historical; no full-file coverage is inherited by the second pass.

### Baseline and movement

Source was read at `97d312585c20a5b17063084cf66ba1bb7438667c`, clean at entry: identical to this report's original source baseline, but newer than report 01's `e7cbe69f0fac8a785657b5ff0bab850b66e9f690`. The intervening local diff contains only fallback, its tests, and exports. Standalone AI docs were `57c2d7d49c83f8094c5586ddf7338f686d77209e` at second-pass entry, not the original `bb66faf88d0a9cff4ea753671e0dbefb5fadea80`; the three disputed context-management/R6/implementation-profile contracts were unchanged. Source's docs pin remains `07169bd9108875179899ac77c3af3c0073a8506a`. Research remains based on `d70152a079c75204ec37e99a7bf3f54b8190e423`, with this campaign already untracked. No snapshot isolation, fetch, reset, or commit was performed; see the [index](README.md) for closing rechecks.

### Verified dispositions and corrected references

| ID         | Independent result                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| AI-CTX-001 | Retained P1: span expiry uses compression time (`compression.rs:570-581`), passthrough uses collection time (`:590-600`); the fixture source is timestamped 100 at `:825-831`, then compressed at 1000 and retained at 1050 (`:1235-1245`). This confirms renewal, not just a missing test. P1 concerns the experimental retention primitive, not proven shipped retention.                                                                                                                             |
| AI-CTX-002 | Retained P1 compositional defect: `compress_records` passes source summaries to the summarizer (`compression.rs:746-759`), retains only maximum generation (`:771-795`), and exposes `view.records` as assembly-ready (`:475-490,683-705`). Assembly checks the synthetic generation (`context.rs:662-672`), not source membership. Mixed old/current generations are sufficient; all-stale summaries still reject against a newer generation. No current product caller or deployment was established. |
| AI-CTX-003 | Reclassified P2 quota-efficiency/API improvement: externalize-then-select is documented. Corrected the order: selection bitmap precedes commit, output filtering follows it. Omitted projection data is not necessarily expired/deleted data.                                                                                                                                                                                                                                                           |
| AI-CTX-004 | Retained P2 at the public provider seam: explicit sampling is accepted/validated but not serialized or refused. Agent currently sends None, and validation support is not backend support for every field.                                                                                                                                                                                                                                                                                              |
| AI-CTX-005 | Retained P2: caller deadline (`provider.rs:423-426`) is not a cumulative elapsed deadline; each blocking operation has a fresh allowance and write-timeout setup failure is ignored. Zero success latency is corroborating telemetry loss, not a measured delay.                                                                                                                                                                                                                                        |
| AI-CTX-006 | Retained P2 for Unicode slice safety and completed-header ceiling; parser is reached through `local_provider.rs:976-983`. Depth is bounded indirectly by aggregate bytes, not an explicit safe depth policy. Platform stack exhaustion was not demonstrated.                                                                                                                                                                                                                                            |
| AI-CTX-007 | Retained P2 for transport-level `HostPeer::serve` errors only. Correlated application error responses complete first. Response-queue-full reachability is withdrawn from the proven causal path. Pinned dependency inspection closes the pending-tracker assumption below.                                                                                                                                                                                                                              |
| AI-CTX-008 | Retained P2 for supported `fallback_for` scratch growth before refusal. Public-field encoding is a separate API-hardening concern; bounded decoder admission remains intact. Corrected the inaccurate constructor-only test characterization.                                                                                                                                                                                                                                                           |
| AI-CTX-009 | Retained P2: name-only `PRAGMA table_info` inspection (`journal_prototype.rs:404-420`) does not establish constraints used by append (`:246-274`); `open` writes the profile at `:231-236`. Existing negative test (`:605-629`) checks wrong names only. New-store creation is intentional; blanket refusal of every partial/foreign store needs explicit policy, rather than being inferred solely from missing tables.                                                                                |
| AI-CTX-010 | Reclassified P2 defensive integration gap: no aggregate check for public hand-built part sets, but the documented validated splitter path enforces the source bound. No live ingress defect established.                                                                                                                                                                                                                                                                                                |

For AI-CTX-007, the locally available Git object at dependency revision `cfeffa2d8e1387029850940af2b64877dbfbe25f` was inspected, not the terminal repository's changing HEAD. `bitty/crates/bitty-ipc/src/channel.rs:511-528` inserts the pending record; `:543-545` only dequeues requests; `:567-568` removes pending state on completion. `drain_expired` exists at `:579-601` but the slice bridge neither calls nor exposes it. Thus a transport-error return at slice `bridge.rs:276` leaves pending state. The private endpoint and synchronous response dequeue also explain why a response-queue-full branch alone is insufficient evidence. No peer was executed.

### Second-pass coverage and residual gaps

CLI outlines guided targeted reads; early agent reads, supporting provider/runtime-bridge slices, and the immutable pinned dependency object did not receive an outline first. The dependency was read via local Git. Main body ranges: runtime `compression.rs:1-35,136-175,213-254,457-490,535-614,683-818,825-838,1191-1257,1400-1507`; `context.rs:434-520,580-602,604-879`; `fallback.rs:1-60,117-233,303-353`; `provider.rs:352-430`. Slice `local_provider.rs:1-80,479-544,589-683,748-841,855-898,969-1040,1120-1210,1248-1310`; `bridge.rs:143-160,225-307`; `journal_prototype.rs:1-50,89-116,206-274,389-423`; `fragment_transport.rs:1-64,356-401,428-556`. Runtime agent orchestration and shared guide/contract reads are recorded in report 01's second-pass section.

Additional test bodies: journal `:605-629` and `tests/fallback_envelope.rs:1-45,280-385`; compression fixture/tests as listed above. Other test bodies, full parser conformance, full fragment-mapping/host conformance, cache/prompt/fingerprint/selection claims, historical-review closure tables, and the full security corpus were not independently re-reviewed. The pinned IPC channel file was returned in full, but dependency verification is limited to the pending lifecycle above, not IPC-wide approval. No Rust test or reproduction was executed. The aggregate verdict remains partial static review, not product approval.

## Validation and limitations

- **Rust execution: none.** No selected tests, full suite, clippy, typecheck, benchmark, or build was run. Test references are static observations, never a claim of passing tests. Existing default-target runtime binaries were not found by the targeted lookup; no alternative cache search or build followed.
- **First-pass static baseline checks:** source/docs HEAD remained unchanged during that pass; both repositories remained clean; `git diff --check` passed in `bitty-ai`. The predecessor-to-HEAD diff identified the new fallback module and test file.
- **Method limitations:** missing test-body coverage is material. In particular this review cannot certify current invalidation-granularity tests, full FakeHost/LiveHost conformance, complete fallback tests, or historical September 16 closure.
- **Safety/scope:** no new executable fixture or reproduction was written. Recommended tests above are defensive contract checks for future authorized implementation work. No source fix was attempted.
- **First-pass report validation:** the original Markdownlint invocation expanded to 68 files via repository configuration and reported zero issues. That result is historical, not the second pass's scope.
- **Second-pass validation:** installed CLI 0.23.1 / markdownlint 0.41.1 with `--no-globs` linted exactly both reports and `README.md`: 3 files, zero issues; no fix flag. Edited sections/index were read back. `git diff --check` passed but excludes untracked files; Markdownlint covered these three untracked files.
- **Closing drift check:** source remains `97d312585c20a5b17063084cf66ba1bb7438667c`, clean. Standalone docs advanced to `fb4cda1c5e9792c848d036903c7db322ee6d1743`, clean; its diff from the second-pass entry adds git-wrapper design/index only, not the cited contracts. Research HEAD remains `d70152a079c75204ec37e99a7bf3f54b8190e423`. This is observed movement, not a stable-snapshot review.
