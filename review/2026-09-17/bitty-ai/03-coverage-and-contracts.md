# bitty-ai review pass 03 — coverage completion and contract conformance

- Date: 2026-09-17
- Repositories:
  - `bitty-ai` HEAD `97d312585c20a5b17063084cf66ba1bb7438667c`, working tree
    clean, `git diff --check` clean.
  - `bitty-ai-docs` HEAD `fb4cda1c5e9792c848d036903c7db322ee6d1743`, working
    tree clean.
  - `research` HEAD `d70152a079c75204ec37e99a7bf3f54b8190e423`, dirty
    (`M README.md` (modified) pre-existing, `?? review/2026-09-17/` this report).
- Method: static read-only source review. No source edits, no commits, no
  network, no test execution, no installed-tool invocation beyond `rg` and
  file reads. Security observations are risk statements only; no exploit or
  reproduction material is included.
- Finding ids: this pass mints `AI-GAP-###` (prior passes used
  `AI-RUN-001..009` and `AI-CTX-001..010`; none are repeated here).

## 1. Coverage inventory

Passes 01/02 covered most of the `bitty-ai-runtime` and `bitty-ai-slice`
production surface. Pass 03 closed the ledger of unread ranges recorded in
the campaign README. Final state, by file (paths relative to `bitty-ai`):

### Fully read across passes (production code)

- `crates/bitty-ai-runtime/src/prompt.rs` (3906 lines) — 01 read the
  assemble/validate middle; 03 read the module doc and bounds (1-2180),
  declarative loaders (1780-2010, 2612-2890 region), and all in-file test
  modules. Remaining slivers 2210-2610 are the canonical-render helpers whose
  byte shape is pinned line-for-line by the tests at 2891-2960.
- `crates/bitty-ai-runtime/src/cache_key.rs` (268), `fingerprint.rs` (177),
  `lib.rs` (150), `extension.rs` (366) — complete.
- `crates/bitty-ai-runtime/src/selection.rs` (1562) — complete including all
  tests.
- `crates/bitty-ai-runtime/src/context.rs` (1464), `compression.rs` (1507) —
  production code read in 01/02; 03 read the remaining test modules.
- `crates/bitty-ai-runtime/src/fallback.rs` (364), `tool.rs` (1107),
  `provider.rs` (1051), `agent.rs` (1061), `bridge.rs` (1337) — production
  code complete across passes; residual in-file test bodies in
  `tool.rs`/`bridge.rs`/`agent.rs` were sampled, not exhaustively read, and
  are listed as coverage limits in section 5.
- `crates/bitty-ai-slice/src/local_provider.rs` (2297) — 02 read the
  production surface and key tests; 03 re-sampled fixtures. The full in-file
  test bodies remain a coverage limit.
- `crates/bitty-ai-slice/src/live_host.rs` (887), `fake_host.rs` (852),
  `fragment_transport.rs` (639), `harness.rs` (435), `bridge.rs` (308),
  `error.rs` (100), `lib.rs` (65) — complete including tests.
- `crates/bitty-ai-slice/tests/host_conformance.rs` (1393+) — 03 read the
  execution/consent/trait-swap sections (1251-1393); 02 covered the earlier
  conformance sections.

### Contract documents read in full or targeted

`prompt-layering-design.md`, `prefix-cache-context-design.md`,
`provider-plugin-boundary.md`, `context-management.md`, `ai-architecture.md`,
`multimodal-inference-boundary.md`, `code-intelligence-sharing-r4.md`,
`prototype-promotion-checklist.md`, `host-boundary-trait-design.md`,
`ipc-agent-rfc.md` (targeted), plus `research/summary/{025,027,032,033}.md`.

## 2. Findings

Two candidate findings carried in from the coverage phase were **refuted by
the code and its tests**; they are recorded here so later passes do not
re-derive them:

- Candidate "upper-layer winner masks conflicting lower-layer directives":
  refuted. Cross-layer directive conflicts resolve by precedence with a
  per-overridden-layer audit record (`merge_overrides`, sorted and
  deduplicated), and the canonical byte form renders each override
  (`override:tone winner:core-contract wlen=6:formal
overridden:project olen=6:casual`). See
  `crates/bitty-ai-runtime/src/prompt.rs:1111-1168` (merge loop),
  `:1137-1146` (winner/overridden record), and tests
  `higher_precedence_directive_overrides_lower`, `lower_layer_cannot_override_upper`,
  `override_audit_record_exact_contents`, `override_record_and_bytes_are_input_order_invariant`.
- Candidate "repeated skill versions overwrite earlier values": refuted.
  Duplicate skill names fail closed with `PromptError::DuplicateSkill`
  (`prompt.rs:1916-1920`); intra-registry directive conflicts and project
  multi-file directive conflicts also fail closed with
  `UnresolvableConflict` (`prompt.rs:1804-1818`, `:1929-1942`).

No new `AI-GAP-###` correctness or contract findings survive verification.
The production surface read in this pass conforms to the documented
contracts on every axis checked (section 3). The two residual gaps below are
test-coverage observations, not code defects.

### AI-GAP-001 (P3, test coverage): five-layer precedence has no full cross-layer ordering test

Every adjacent pair and the Core-vs-RuntimeTurn extremes are pinned
individually, and the canonical render test asserts full header order
(`prompt.rs` tests `canonical_sections_render_in_rank_order`,
`override_record_and_bytes_are_input_order_invariant`). However, no single
test drives one directive through all five layers (Core > User > Project >
SkillsProfile > RuntimeTurn) and asserts the Core value wins with exactly
four override records. The property is implied by the rank-ascending loop at
`prompt.rs:1117-1146` (input-order invariant), but a five-way sweep would
pin the full transitive chain against future rank edits.

Suggested test: a `#[test]` in `prompt.rs` with one `("tone", vN)` directive
per layer asserting `effective_directives == tone=<core value>` and
`merge_overrides.len() == 4` with descending overridden ranks.

### AI-GAP-002 (P3, test coverage): skills registry has no cross-entry tool-narrowing test

`skills_from_str` concatenates per-entry allow/deny/scope vectors and the
shared assembly layer applies narrowing (`prompt.rs:1921-1976`), and the
assembly-side narrowing is well tested (`allowed_intersection_narrows_cannot_widen`,
`budget_min_wins_cannot_widen`, `scopes_intersection_narrows_cannot_widen`).
But no test drives two skill entries with disjoint allow-sets through
`skills_from_str` + `assemble` to confirm the disjoint-allow sentinel
failure (`EMPTY_ALLOW_SET_SENTINEL`) surfaces through the registry path, not
only the programmatic path. Same one-line suggestion shape as AI-GAP-001:
a registry fixture with two entries granting disjoint `allow_tool` sets,
expecting `PromptError::PromptNotAllowed { tool: EMPTY_ALLOW_SET_SENTINEL }`.

## 3. Contract conformance results (docs/code)

All checks against `bitty-ai-docs` at `fb4cda1`:

- **Prompt layering (prompt-layering-design.md, draft)** — conforms. The
  five-layer rank order, precedence resolution, never-merge keys
  (`NEVER_MERGE_DIRECTIVE_KEYS`, fail-closed even for identical-value
  conflicts only when values differ), deny-wins/allow-intersection/budget-min/scope-intersection
  narrowing, and the "prompts never grant capabilities" rule
  (`prompt_never_grants_when_dispatcher_denies`,
  `text_claiming_capability_never_dispatches_without_grant`,
  `loaded_text_claiming_capability_never_dispatches_without_grant`) all match
  the draft's normative sections. The canonical byte form is fully
  deterministic and input-order invariant.
- **Prefix cache (prefix-cache-context-design.md)** — conforms, with the
  standing note from pass 02 that the seven-layer split is explicitly a
  proposal (:60-87); the implemented five-layer prompt is the draft's own
  baseline. `cache_key.rs` implements the documented cache scope
  (provider/model/revision/tokenizer config, no cross-provider reuse) with
  the length-aware boundary walk (`stable_prefix_len`, cache_key.rs:202-220)
  and FNV-1a-64 (offset basis/prime match the documented constants,
  cache_key.rs:261-268). The trailing-change-prefix-preservation property is
  pinned by `trailing_change_preserves_leading_prefix`.
- **Provider plugin boundary (provider-plugin-boundary.md)** — conforms.
  Core owns the `ModelProvider` contract, registry, selection (alias chain
  exclusive and fail-closed, capability-subset matching, context-window
  floor, cost ceiling with baseline-1 uncalibrated weights), and fallback
  directive taxonomy (advance on transient, stop on caller-config/auth,
  stop-and-reconcile on unknown). `selection.rs` tests pin each gate.
- **Fingerprint / disable-on-unknown (fingerprint.rs, AI-0095)** —
  conforms: complete-input FNV-1a-64, typed `FingerprintError` with
  `MissingInput`/`UnknownComponent` arms, disable-on-unknown construction.
- **Injection defense (context-management contract, AIQ-11 evidence)** —
  conforms and is unusually well tested: directive-laden untrusted bodies
  and summaries are inert data (maintenance-fingerprint equivalence against
  same-shape benign controls), untrusted supersede links are ignored,
  untrusted priority escalation is clamped and provably equivalent to
  Normal, untrusted duplicates never displace trusted originals regardless
  of order or timestamp, and oversized hostile bodies cannot reallocate
  budget. Compression inherits trust via full provenance chains and never
  mints supersede links or escalates priority.
- **Artifact store (context.rs:342-440)** — conforms: bounded
  byte/count/total caps, typed absences, two-phase commit with
  `next_id()` observable so tests prove zero mutation on failure
  (`artifact_store_exhaustion_fails_closed`,
  `duplicate_record_ids_fail_closed`, `stale_and_future_generations_fail_closed`).
- **Host conformance (host_conformance.rs)** — conforms: consent ledger,
  execution truncation, target mismatch, opt-in denial
  (`EffectRequiresExplicitConsent`), and trait-swap identity are shared
  between `FakeHost` and `LiveBittyHost`; live-provider tests are fenced
  with front/back count fences against the process-global store.
- **Prototype promotion gates (prototype-promotion-checklist.md, draft)** —
  the checklist's gate anchors (AI-0084 length-aware walk, AI-0090
  `schema_digest`, AI-0095 fingerprint, AI-0097 minimal-envelope fallback)
  all correspond to the code shapes described. House style (typed errors
  with `Display` + `std::error::Error`, `#[must_use]`, `# Errors` docs,
  `#![deny(unsafe_code)]`, std-only) is consistently observed in every file
  read this pass. The checklist itself is correctly marked draft and claims
  no accepted behavior.
- **Sampling/provider validation (provider.rs:635-770)** — conforms:
  `MP-2` id shape, `MP-5` declared-only sampling validation with the
  Anthropic reasoning/completion budget ordering rule, `MP-8` timeout
  ceiling; absent fields are never defaulted.

## 4. Optimization / spec-gap observations (non-blocking)

- `DirectiveOverride` sorting uses `(key, overridden rank, overridden value,
winning rank, winning value)` (`prompt.rs:1152-1167`). The `value` fields
  participate in the sort only for determinism; they do not affect the
  semantic ordering. No change suggested — noting the shape for future
  schema reviewers.
- `select_breakpoints` is O(n) over records per call with the span cap
  walk; no hot-path concern at documented bounds (records are already
  budget-assembled), recorded as context only.
- The live-provider test fences (`live_host.rs:545-575` region) rely on a
  process-global `OnceLock<Mutex<()>>` per test module; a second test module
  touching the live store would need the same lock. Currently only one
  module exists — spec-level note for the HostBoundary draft.

## 5. Remaining coverage limits and next steps

Not exhaustively read (in-file test bodies only; production code complete):

- `tool.rs`, `bridge.rs`, `agent.rs` in-file test modules (sampled).
- `local_provider.rs` full test bodies (sampled; production surface read in
  pass 02).
- `tests/` integration files not in the pass 01/02 list (e.g.
  `accounting_bounds.rs`, `batch_evidence.rs`, `subscription_bounds.rs`,
  `mcp_fail_closed.rs`, `fallback_envelope.rs`, `turn_semantics` suites)
  were not opened; the promotion checklist pins their gate evidence at
  specific merges, and pass 03 verified only the production modules those
  suites target.

Suggested pass 04: integration-suite read-through against the
`prototype-promotion-checklist.md` gate table (each named test file vs. its
merge pin), plus the three sampled test modules.

## 6. Verification

- Reads/commands used: `rg`, `ctxctl read`, `git rev-parse`/`git status`/
  `git diff --check` (captured before this report), file reads. No command
  failed; no test or gate was executed (out of scope per task).
- After writing this file, `research` `git status` must show only the
  pre-existing `M README.md` (modified) plus `?? review/2026-09-17/`.
