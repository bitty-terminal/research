# AI documentation alignment — 2026-09-17

## Scope and baseline

Sampled read-only review of standalone AI docs, implementation-facing public
contracts, and research summaries against current local source. Only the three
requested reports are authored; no implementation changes, CarryCtx, agents,
network, installs, commits, or behavioral reproductions. Evidence paths are
workspace-relative, with inclusive line ranges. Priority indicates remediation
importance within experimental APIs, not deployed exposure.

| Repository      | Local HEAD                                 | Entry dirty baseline                                           |
| --------------- | ------------------------------------------ | -------------------------------------------------------------- |
| `bitty-ai`      | `97d312585c20a5b17063084cf66ba1bb7438667c` | Clean                                                          |
| `bitty-ai-docs` | `fb4cda1c5e9792c848d036903c7db322ee6d1743` | Clean                                                          |
| `bitty-docs`    | `fc7ed9750491875bea9986be35a4df5074ad84ba` | Clean                                                          |
| `research`      | `d70152a079c75204ec37e99a7bf3f54b8190e423` | Modified `README.md`; existing untracked September 17 campaign |

The source docs gitlink is `07169bd9108875179899ac77c3af3c0073a8506a`,
not standalone docs HEAD. It was read using local `git ls-tree`, not refreshed.
The September 17 second-pass source baseline matches this review
(`research/review/2026-09-17/bitty-ai/README.md:28-41,63`). Its corrected
classification is retained; earlier test results are not rerun evidence.
Cross-product governance, research capture status, and pin semantics appear only
in [the terminal report](../bitty-terminal/08-documentation-alignment.md).

## Findings

### AI-DOC-001 — P2 drift: entry points still disagree on whether the implementation exists

- **Docs:** `bitty-ai-docs/README.md:44-50` says mounting will happen once the
  implementation repository exists. `bitty-ai/AGENTS.md:17-29` calls it
  pre-implementation and leaves workspace/MSRV undecided.
- **Current evidence:** `bitty-ai/Cargo.toml:1-21` contains the two experimental
  workspace members and MSRV; `bitty-ai/.gitmodules:1-4` defines the existing
  docs mount. `bitty-ai/README.md:7-15,34-43` accurately labels the deterministic
  runtime and experimental slice. The standalone docs map also acknowledges
  the pinned mount at `bitty-ai-docs/docs/README.md:36-39`.
- **Source:** `bitty-ai/crates/bitty-ai-runtime/src/agent.rs:900-969` contains
  actual reconciliation logic, not just proposed types.
- **Impact:** the public entry points and agent guidance can incorrectly send
  maintainers back to initialization or cause them to discount existing code.
- **Remedy:** use one dated experimental status and actual repository/mount
  relationship. Keep production-readiness disclaimers; existence of two crates
  does not establish complete AI, remote providers, or persistence.

### AI-DOC-002 — P2 defect: reconciliation public prose promises an execution-wide cap that the API does not enforce

- **Public declarations:**
  `bitty-ai/crates/bitty-ai-runtime/src/reconcile.rs:21-24,36-48,59-65`
  describes no further retry after escalation and maximum queries per execution.
  `bitty-ai/crates/bitty-ai-runtime/src/agent.rs:862-869` says resolution keeps
  the session Active and reiterates no further retry.
- **Current behavior/explicit competing contract:**
  `bitty-ai/crates/bitty-ai-runtime/src/agent.rs:873-898` defines frozen-clock,
  per-invocation evaluation and permits re-invocation after escalation.
  The body at `:913-932,940-947,959-968` resets its attempt counter for each
  invocation, updates the effect result on resolution without reactivating a
  session, and fails the session on exhaustion.
- **Canonical constraint:**
  `bitty-ai-docs/specifications/persistence-profile-r6.md:219-235` requires
  reconciliation before effect retry, not an automatic return to Active or a
  mandatory execution-wide inspection budget.
- **Impact:** adapter authors cannot infer total inspection work or session
  lifecycle from these public declarations. Status inspection is not effect
  re-execution; no duplicate live effect is demonstrated.
- **Classification relationship:** documentation/API-contract defect,
  corresponding to the narrowed AI-RUN-006 contract gap in
  `research/review/2026-09-17/bitty-ai/01-runtime-and-sessions.md:106-116`.
  It is not an additional independently counted runtime scheduling defect.
- **Remedy:** declare per-invocation query limits, reported-not-awaited delays,
  post-escalation inspection, and preservation of terminal session state. If an
  execution-wide cap is intended instead, decide that explicitly before adding
  per-effect accounting. Do not silently reactivate failed sessions.

### AI-DOC-003 — P2 drift: the maintained coverage ledger still calls tracked draft content uncommitted

- **Docs:**
  `bitty-ai-docs/specifications/research/research-coverage-ledger.md:53-61`
  says the drafts remain uncommitted/unpublished and the task awaits discussion.
  Its earlier section correctly distinguishes draft coverage from accepted
  authority (`:44-50`).
- **Local repository evidence:** both the ledger and
  `specifications/research/research-distillation-013-017-018.md` are returned by
  `git ls-files` at the recorded docs HEAD. Local history records the latter at
  `fa18390586babc2c63555c72e691ffb8463a7d76`
  (`docs(specs): reconcile reviewed AI research drafts`). The current map routes
  readers to the ledger and distillation at `bitty-ai-docs/docs/README.md:45-46`.
- **Impact:** a maintained provenance entry directs readers to an obsolete
  delivery state despite the capture being present in local committed history.
  This is not a claim about current publication, remote PRs, or task state.
- **Remedy:** replace the undated live-workflow qualifier with a historical
  snapshot and immutable capture revision. Retain the distinct statement that
  draft capture does not accept the architecture or verify the product.

## September 16 assertions and intentional future work

| Earlier assertion                                                                                                                                                                          | Current sampled disposition                                                                                                                                                                                                                                                                                                                                                                                |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Report 02 calls the runtime a deterministic sans-I/O skeleton (`research/review/2026-09-16/02-code-vs-docs-progress-audit.md:116-129`)                                                     | Broad scope remains appropriately experimental. `bitty-ai/README.md:7-30` distinguishes runtime from host seams. The slice does contain a loopback HTTP adapter; actual `TcpStream::connect_timeout` appears in `bitty-ai/crates/bitty-ai-slice/src/local_provider.rs:611-643`. No network request was made.                                                                                               |
| Report 02/04: prefix cache uses naive substring scanning (`research/review/2026-09-16/02-code-vs-docs-progress-audit.md:140-142`; `04-architectural-drift-and-subjective-analysis.md:130`) | Superseded by current source. `bitty-ai/crates/bitty-ai-runtime/src/cache_key.rs:202-255` walks ordered length-framed sections. This is a static correction of that exact claim, not a complete cache/security audit or new performance evidence.                                                                                                                                                          |
| Missing multi-agent, remote providers, layered filesystem prompt loading, and durable SQLite prove documentation failure (report 02 `:148-156`)                                            | Not established: `bitty-ai-docs/specifications/implementation-profile-v0.1.md:14-17,27-52,105-110` explicitly limits experimental scope; `persistence-profile-r6.md:237-264` defers background scheduling and durable storage. `storage-memory-export-design.md:14-41` expressly describes a proposal. Preserve these future designs instead of counting absence as a defect.                              |
| Provider-plugin ownership conflicts with a Rust local-provider experiment                                                                                                                  | No conflict established. `bitty-ai-docs/specifications/provider-plugin-boundary.md:14-38` labels handoff and ownership as draft; the experimental slice is not a shipped plugin system. `research/summary/032.md:10-18` is historical direction, not authority to change the core boundary.                                                                                                                |
| BII gateway file stability proves no real host work (report 02 `:160-166`)                                                                                                                 | Overbroad inference. `bitty-ai-docs/specifications/bitty-side-delivery-verification.md:93-120` explicitly distinguishes unchanged gateway files from added host bridge, projection, runtime binding, and spawn authorization. Current local host uses `HostToolsAuthorizer` at `bitty/crates/bitty-runtime/src/plugin_runtime/spawn.rs:1008-1049`. Complete socket/PTY/AI integration was not traced here. |
| Prefix-cache and storage research must already be production implementations (report 03 rows 025/037)                                                                                      | Withdraw that reading. `bitty-ai-docs/specifications/prefix-cache-context-design.md:14-23,37-58` disclaims shipped behavior and measurements; storage draft does likewise. `research/summary/025.md:10-23` and `research/summary/037.md:10-19` record direction, not proof of delivery.                                                                                                                    |

The September 17 AI index's qualified retention, generation, cancellation,
resource, and reconciliation findings remain separate source-review work
(`research/review/2026-09-17/bitty-ai/README.md:18-26`). This review does not
relabel those defects as documentation-only, nor independently renew every one.

## Precise sampled coverage and checks

- Guidance: workspace/research/AI/shared-docs/standalone-docs AGENTS and
  ctxctl-core. The CLI provided repository-local outlines after the MCP outline
  tool rejected the external workspace path.
- AI docs read: root README `:1-90`; map `:12-51`; implementation profile in
  full after frontmatter; delivery verification `:12-136`; coverage ledger
  `:12-96`; persistence R6 `:219-266`; prefix-cache design `:12-66`;
  provider-plugin boundary `:12-66`; storage-memory design `:12-59`.
  Other architecture/R1–R5/prompt/transport/provider documents were not reviewed
  in full. Discovery of filenames and map links is not document coverage.
- Research: full summaries 023, 025, 032, 037; shared capture samples documented
  in the terminal report; September 16 reports 01–04; September 17 AI index and
  targeted RUN-006 review passages. Original records and upstream reference
  snapshots were not reread or rehashed.
- Source: workspace metadata and docs gitlink; outlines of agent, reconcile,
  cache key, and local provider; exact source ranges cited above. The terminal
  spawn seam was checked at `06bc1f45995fd297a3f7324688bc81b0598ecf67`
  (pre-existing untracked `.targets/`, no tracked changes). No complete crate,
  provider parser, persistence, or integration audit.
- No Rust build/test/clippy/typecheck, live provider, timing, cache-hit, or
  memory measurement. Only Markdown reports changed. Installed Markdownlint
  runs on the three explicit report paths with `--no-globs`, without fixes or
  network; final result is recorded in the task response. Lint does not certify
  any source contract or historical test receipt.
