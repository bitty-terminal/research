# bitty-ai review index — 2026-09-17

## Reports and disposition

| Report                                                                    | Focus                                                                                                    | Independently retained defects                                                         | Reclassified or evidence-only items                                                                          |
| ------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| [01 — Runtime and sessions](01-runtime-and-sessions.md)                   | Turn admission, tool attribution, cancellation, reconciliation, adoption, diagnostics                    | P1 AI-RUN-001; P2 AI-RUN-003/004/005/007/008                                           | P2 AI-RUN-002 integration gap, AI-RUN-006 contract/test gap; P3 AI-RUN-009 evidence overclaim                |
| [02 — Context, slices, and providers](02-context-slices-and-providers.md) | Retention, generation propagation, projection, local provider, IPC cleanup, fallback, journal, fragments | P1 AI-CTX-001/002; P2 AI-CTX-004/005/006/007/008/009                                   | P2 AI-CTX-003 quota-efficiency improvement, AI-CTX-010 defensive integration gap                             |
| [03 — Coverage and contracts](03-coverage-and-contracts.md)               | Production coverage completion across passes; prompt/cache/fingerprint/selection and host contracts      | No new correctness/contract defects survived verification                              | P3 AI-GAP-001/002 test-coverage observations; candidate directive-masking and skill-overwrite claims refuted |
| [04 — Documentation alignment](04-documentation-alignment.md)             | Sampled entry-point status, reconciliation public prose, draft-capture ledger                            | Report-level static documentation checks, not a renewed second pass of runtime defects | P2 AI-DOC-001/002/003 drift/contract corrections; AI-DOC-002 overlaps AI-RUN-006                             |

Reports 01–02 second-pass total (unchanged, not a total for all four reports): **3 P1 and 11 P2 correctness/resource defects**, **4 P2 contract/integration/improvement items**, and **1 P3 evidence-description issue**. Priority is review triage within experimental APIs, not proof of shipped exposure. IDs are preserved; withdrawals and reclassifications are explicit in each report, never renumbered.

The principal retained problems are loss of pending reconciliation lookup, retention renewal during compression, stale-generation contributions hidden by summary metadata, inconsistent cross-round tool caps, effect-status misattribution, final-callback cancellation mismatch, diagnostic/allocation bounds, ignored sampling declarations, incomplete request deadlines/parser checks, pending IPC cleanup, and journal schema admission. Recommended security work is defensive only.

## Independent verification

A separate second pass compared every P1/P2 finding with current source bodies and relevant contracts. It did not accept the original reports or test names as proof. It checked callers, returned states, ordering, negative branches, documented intentional behavior, and exact source references. Existing test assertions were read, not executed.

Important corrections:

- AI-RUN-001 loses the agent's reconciliation lookup, not necessarily every host copy. Reconcile-before-retry is per effect; it does not require blocking every unrelated turn.
- AI-RUN-002 lacks a first-class executor identity parameter, but host-side correlation is possible; unavoidable misattribution is unproven.
- AI-RUN-006 intentionally evaluates a frozen-clock schedule per invocation and preserves terminal session state. Execution-wide/no-further-retry wording is inconsistent; automatic reactivation is not a required fix.
- AI-CTX-003 documents externalization before selection. The bitmap is computed before commit, which ignores it; only output filtering follows commit. Quota waste is real, but selected-only storage is not an established contract.
- AI-CTX-007 is proven for transport-level peer errors, not correlated application refusals. Pinned upstream pending-state semantics were inspected; response-queue-full reachability was not established.
- AI-CTX-008 retains the supported fallback scratch-allocation concern. Public mutable fields are an API concern; bounded decoding and existing over-cap tests were not missing.
- AI-CTX-010 lacks aggregate admission for hand-built parts, but validated splitter output already obeys the source ceiling. No live ingress path was established.

## Coverage and documentation follow-ups

Report 03 records production-code read coverage complete across passes 01–03, including the previously open prompt, cache-key, fingerprint, selection, host, and fragment bodies. Its checked contract axes conform statically; this neither closes retained RUN/CTX defects nor establishes executed conformance. Candidate directive-masking and repeated-skill overwrite findings were refuted. AI-GAP-001/002 remain P3 test gaps for a full five-layer directive ordering test and cross-entry skill tool narrowing, not correctness defects.

Report 04 retains these qualified documentation priorities:

| ID         | Disposition                                                                                                                                                                                                                                                                                  |
| ---------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| AI-DOC-001 | P2 entry-point drift: docs and agent guidance deny an existing experimental implementation/mount. Existing crates are not a production-readiness claim.                                                                                                                                      |
| AI-DOC-002 | P2 public API-contract inconsistency: execution-wide/no-further-retry and Active-session prose conflicts with per-invocation inspection and preservation of terminal session state. Corresponds to AI-RUN-006, not an additional runtime scheduling defect or demonstrated duplicate effect. |
| AI-DOC-003 | P2 research-ledger drift: locally committed draft capture is still described as uncommitted. Local capture does not establish acceptance, current remote publication, or live task/PR state.                                                                                                 |

Selected September 16 corrections reject the naive-substring cache claim and absence-as-defect readings of explicitly deferred providers, prompt loading, storage, and multi-agent work. Gateway-file stability alone does not prove no host work. These are sampled corrections, not blanket historical closure or new performance/security evidence. Shared pin/capture governance remains in [Terminal 08](../bitty-terminal/08-documentation-alignment.md). Reports 03–04 use source `97d3125` and standalone docs `fb4cda1`; they do not replace the historical baselines below or refresh the older source-docs pin.

## Actual baselines and campaign movement

| Repository or dependency   | First-pass baseline                                                                                                    | Second-pass observed baseline                                                      |
| -------------------------- | ---------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| `bitty-ai`, report 01      | `e7cbe69f0fac8a785657b5ff0bab850b66e9f690`, advanced during first pass                                                 | `97d312585c20a5b17063084cf66ba1bb7438667c`                                         |
| `bitty-ai`, report 02      | `97d312585c20a5b17063084cf66ba1bb7438667c`                                                                             | Same; clean on entry                                                               |
| Standalone `bitty-ai-docs` | `bb66faf88d0a9cff4ea753671e0dbefb5fadea80`                                                                             | `57c2d7d49c83f8094c5586ddf7338f686d77209e`; clean on entry                         |
| Source docs submodule      | Report 02 recorded `07169bd9108875179899ac77c3af3c0073a8506a`                                                          | Same recorded/checked-out pin; not refreshed                                       |
| `research`                 | Report 01 began `26a46ed8c15aa4e714ca9add4ff4c07bea02ff22`; both ended/used `d70152a079c75204ec37e99a7bf3f54b8190e423` | `d70152a079c75204ec37e99a7bf3f54b8190e423`; campaign already untracked             |
| Slice IPC dependency       | `cfeffa2d8e1387029850940af2b64877dbfbe25f`                                                                             | Same immutable local Git object inspected for pending lifecycle, not terminal HEAD |

The source diff between report 01's initial HEAD and the second pass adds fallback, its integration tests, and five export lines only. RUN references did not shift. Standalone docs added host-boundary/promotion material and updated index/triage; the disputed context-management, persistence R6, and implementation-profile documents were unchanged. Standalone-docs citations must not be mistaken for the older submodule snapshot.

This campaign uses a changing shared workspace, **not an isolated snapshot**. HEAD/status observations delimit evidence but cannot exclude transient edits. Historical report baselines are preserved rather than silently replaced by a stable-snapshot claim.

## Coverage and test gaps

Reports 01–02 contain separate first-pass ledgers and targeted second-pass coverage. Those second-pass checks cover their numbered P1/P2 findings, not every line of every crate. Read scope includes orchestration, compression/assembly, adapter, parser, bridge, fallback, journal, and reassembly bodies plus selected test assertions and contracts. Report 03 closes the recorded production-read gaps across passes; its ledger does not retroactively expand the earlier second-pass sample.

Residual reading limits in report 03 are sampled inline tests in runtime `tool.rs`, `bridge.rs`, `agent.rs`, and slice `local_provider.rs`, plus integration suites outside the passes 01/02 list. Prompt/cache/fingerprint/selection production reads are no longer open gaps; FakeHost/LiveHost, fragment, and host-conformance reads add static evidence, not executed certification. Still not certified: full executed parser/adapter conformance and platform stack behavior; production adapter correlation; live network/deadline behavior; durable deletion/recovery; full upstream IPC/security corpus; cross-platform behavior; historical September 15/16 closure claims beyond report 04's selected corrections. Documentation coverage remains sampled, not corpus-wide. No tokenizer, cache-hit, latency, memory benchmark, or deployed exploit claim was measured.

No Rust builds, tests, typecheck, clippy, executable fixtures, reproduction code, payloads, network calls, installations, source edits, commits, CarryCtx operations, or nested agents were performed. The runtime remains an experimental synchronous skeleton; the slice includes a loopback provider and a journal prototype, not a complete production provider/persistence system.

## Validation and owned files

The current four-index follow-up uses installed `markdownlint-cli2 --no-globs` with the campaign and three product README paths, followed by final content read-back. It runs no product tests; formatting checks do not certify source contracts.

The original second pass owned this index and reports 01–02 under `research/review/2026-09-17/bitty-ai/`; reports 03–04 were authored separately. This indexing follow-up changes only the campaign and three product READMEs, not any report or the root research index. No implementation-repository README was created or overwritten. The following command and result are historical second-pass receipts, not checks rerun on the reports by this indexing task.

Markdownlint is run with installed CLI 0.23.1 / markdownlint 0.41.1 and `--no-globs` to prevent the repository's configured broad glob from expanding coverage. Validation command, from `research`:

```sh
markdownlint-cli2 --no-globs review/2026-09-17/bitty-ai/01-runtime-and-sessions.md review/2026-09-17/bitty-ai/02-context-slices-and-providers.md review/2026-09-17/bitty-ai/README.md
```

Result: **3 files linted, zero issues**, no fix flag. Edited report sections and this index were read back. `git diff --check` passed in source/docs/research; it does not validate untracked Markdown, which was covered by Markdownlint.

Historical second-pass closing observations: source remains `97d312585c20a5b17063084cf66ba1bb7438667c`, clean, with unchanged docs pin. Standalone docs advanced during verification from `57c2d7d49c83f8094c5586ddf7338f686d77209e` to `fb4cda1c5e9792c848d036903c7db322ee6d1743`, clean; the additional diff adds git-wrapper design and updates its index, leaving the cited contracts unchanged. Research remains `d70152a079c75204ec37e99a7bf3f54b8190e423`; the three owned files are untracked. No source change or commit was made by this verifier. Rust checks are intentionally not represented by Markdownlint success.
