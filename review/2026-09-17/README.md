# Code review campaign — 2026-09-17

## Scope and method

This campaign contains **16 reports separated into three product directories**:
eight terminal reports, four AI reports, and four plugin-ecosystem reports.
Multiple scoped first-pass subagents performed static review, followed by
independent second-pass source/contract verification. The product indexes and
report-level second-pass dispositions govern this summary; first-pass prose is
not blanket endorsed. This index consolidates those records, not a new source
review or a fix-verification pass.

**Not all code was read, and no full test suite was run.** The terminal `bitty`
repository has 19 crates: 18 are assigned across reports 01–05; report 07 adds
`bitty-compat-lab` production coverage and selected runtime/configuration bodies.
Report 06 separately covers `bitty-devtools`. Assigned scope is not complete file
or line coverage. AI report 03 records production-code coverage complete across
passes 01–03, not exhaustive test-body coverage or executed conformance. Terminal
08, AI 04, and Plugins 04 add sampled documentation/source alignment checks.
These follow-ups do not blanket renew earlier findings or second-pass coverage.

Links below are relative to this campaign directory. Source path references in
product reports follow their stated repository/workspace-relative conventions.

## Product and report index

### Terminal — eight reports

[Terminal second-pass index](bitty-terminal/README.md) records qualified
priorities, independently checked IDs, and remaining first-pass leads.

| Report                                                                                           | Scoped area                                                                                   |
| ------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------- |
| [01 — Terminal engine](bitty-terminal/01-terminal-engine.md)                                     | `bitty-vt`, `bitty-term-state`, `bitty-render`, `bitty-pty`, `bitty-platform`                 |
| [02 — Runtime orchestration](bitty-terminal/02-runtime-orchestration.md)                         | `bitty-runtime`: lifecycle, process execution, routing, persistence, panels                   |
| [03 — App, UI, core, config](bitty-terminal/03-app-ui-core-config.md)                            | `bitty-app`, `bitty-ui`, `bitty-core`, `bitty-config`; selected runtime/render seams          |
| [04 — IPC, agent, performance, test support](bitty-terminal/04-ipc-agent-perf-testsupport.md)    | `bitty-ipc`, `bitty-agent`, `bitty-perf`, `bitty-test-support`                                |
| [05 — Plugin host, Lua, package, rich content](bitty-terminal/05-pluginhost-lua-package-rich.md) | `bitty-plugin-host`, `bitty-lua`, `bitty-package`, `bitty-rich`                               |
| [06 — DevTools](bitty-terminal/06-devtools.md)                                                   | `bitty-devtools` TypeScript client/executable and Rust adapter                                |
| [07 — Coverage follow-up](bitty-terminal/07-coverage-followup.md)                                | Compatibility lab, selected runtime production gaps, configuration merge/types/validation     |
| [08 — Documentation alignment](bitty-terminal/08-documentation-alignment.md)                     | Sampled terminal/shared docs, status and routing, research provenance, historical corrections |

### AI — four reports

[AI second-pass index](bitty-ai/README.md) distinguishes retained defects from
contract, integration, improvement, and evidence-description items.

| Report                                                                             | Scoped area                                                                                                       |
| ---------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| [01 — Runtime and sessions](bitty-ai/01-runtime-and-sessions.md)                   | `bitty-ai-runtime`: turn admission, tools, cancellation, reconciliation, recovery                                 |
| [02 — Context, slices, and providers](bitty-ai/02-context-slices-and-providers.md) | Runtime context/compression and `bitty-ai-slice`: projection, adapters, fallback, journal, fragments              |
| [03 — Coverage and contracts](bitty-ai/03-coverage-and-contracts.md)               | Production coverage completion across passes, prompt/cache/fingerprint/selection and host contracts; P3 test gaps |
| [04 — Documentation alignment](bitty-ai/04-documentation-alignment.md)             | Sampled entry-point status, reconciliation public contract, research ledger and historical corrections            |

### Plugins — four reports

[Plugin second-pass index](bitty-plugins/README.md) records corrected severities,
local package-store priorities, and incomplete host-integration evidence.

| Report                                                                          | Scoped area                                                                                           |
| ------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| [01 — SDK and template](bitty-plugins/01-sdk-and-template.md)                   | `bitty-plugin-sdk`, `bitty-plugin-template`: package entry, mocks, conformance, generation            |
| [02 — Registry and distribution](bitty-plugins/02-registry-and-distribution.md) | `bitty-plugins` registry/storefront and selected terminal local installer/store paths                 |
| [03 — Independent plugins](bitty-plugins/03-independent-plugins.md)             | Activity, Palette, Statusline, File Manager, Git Panel, Wheel; selected host seams                    |
| [04 — Documentation alignment](bitty-plugins/04-documentation-alignment.md)     | Sampled manifest-hash contract, panel extraction gates, Git-tool ownership and historical corrections |

## Evidence classes and qualified priorities

- **Confirmed/static-verified defects:** independent reading supports the cited
  mechanism under the stated conditions. This does not establish a reproduced
  incident, deployed exposure, frequency, or a passing regression test.
- **First-pass unverified leads:** claims not independently renewed remain leads,
  even where their original severity or test receipts are retained.
- **Withdrawn or reclassified claims:** second-pass corrections supersede original
  labels. Contract gaps, intentional scaffolding, and test/evidence deficiencies
  are not automatically functional defects.
- **Optimizations:** efficiency and maintenance suggestions remain distinct from
  correctness defects; no measured performance improvement is implied.

There is **no aggregate issue count**: IDs, categories, and cross-product paths
can overlap, notably terminal persistence and plugin distribution. Report counts
above count documents only. P1/P2 labels are qualified remediation priorities,
not a uniform campaign-wide exploitability or release-readiness rating.

### Strongest retained priorities

- **Terminal:** TERM-RUN-001/002/003 prioritize deadline/child ownership and
  committed-store integrity. TERM-IPC-004 covers control work executing after
  client timeout; TERM-IPC-001 is control-assurance work on a filesystem-attested
  marker, not proof of missing access control or cross-user compromise.
  TERM-DEV-001/002 concern stub/live wiring; TERM-DEV-005 is UTF-8-unsafe slicing
  in a Rust library path, not a demonstrated crash of the installed executable.
  TERM-ENG-001 is the documented damage-history fallback API contract.
  TERM-HOST-002 is a host registration-admission quota gap with practical
  exhaustion unmeasured. These are the terminal index's qualified P1 items.
- **AI:** retained P1 AI-RUN-001 loses the agent's pending reconciliation lookup,
  not necessarily every host copy, and does not imply blocking unrelated turns.
  AI-CTX-001/002 concern retention renewal during compression and stale-generation
  contributions hidden by summary metadata. Retained P2 work includes tool caps,
  effect attribution, cancellation, resource bounds, provider deadlines/parsing,
  IPC cleanup, and journal admission within experimental APIs.
- **Plugins:** retained P1 PLUG-REG-007/010/011 concern local package-store
  manifest/index mismatch, conditional old-index loss on failed replacement, and
  overlapping transaction lost updates. These are availability/integrity findings;
  no race or platform fault was executed. Stronger verified P2 work covers SDK
  package/mock fidelity, registry validation/provenance, package retention, and
  selected Activity/Git Panel/Statusline behavior. Git result-path findings do not
  prove actual subprocess cwd because the current bridge ignores Lua's second
  spawn argument.

### Coverage follow-ups and documentation alignment

- **Terminal 07:** TERM-GAP-001/002 are static-verified P2 evidence-accounting
  defects: unattempted reference comparisons can count as passes while self-failure
  remains visible; dump collection can count writes when destinations were not
  created. Neither establishes a successful compatibility run. The additional
  resolution-store rename path extends TERM-RUN-003, not a new finding.
- **Terminal 08:** TERM-DOC-001 is P1 contract/assurance drift: security and
  contributor entry points deny existing implementation, without proving controls
  meet acceptance gates. TERM-DOC-002/003/004/005 are P2 corrections for panel API
  attribution, plugin-doc routing, research record identities, and capture status.
  Local capture is not accepted design, shipped functionality, or live PR proof.
- **AI 03:** no new correctness/contract finding survived verification; candidate
  directive-masking and repeated-skill overwrite claims were refuted. AI-GAP-001/002
  are P3 test-coverage observations for full five-layer ordering and cross-entry
  skill narrowing, not defects or executed test results.
- **AI 04:** AI-DOC-001/002/003 are P2 entry-point drift, reconciliation public
  contract inconsistency, and stale draft-capture ledger status. AI-DOC-002
  corresponds to AI-RUN-006, not another runtime scheduling defect; per-invocation
  inspection does not imply session reactivation or duplicate live effects.
- **Plugins 04:** PLUG-DOC-001/002/003 are P2 bounded documentation/data-contract
  corrections: raw versus canonical manifest hashes, superseded panel extraction
  gates, and retired Git-tool owners. PLUG-DOC-001 corroborates PLUG-REG-006,
  not a second runtime defect. Generic panel RFC acceptance does not deliver the
  public plugin surface; sampled host authorization is not full install-contract
  proof. Shared routing/capture findings remain TERM-DOC-003/005.

These are report-level static checks, not a new independent verification by this
index. The documentation reports recheck selected September 16 assertions only:
pin lag is not inherently defective, captured proposals are not delivered features,
and draft trust scaffolding does not prove a live signing bypass. Their sampled
corrections do not renew historical corpus-wide counts or closure claims.

### Corrections that must travel with the findings

- TERM-ENG-003/004 are withdrawn as present defects; TERM-ENG-005 is a P3 test
  follow-up. TERM-APP-003/004/005 are maintenance/test/scaffold items.
  TERM-HOST-001 is a P2 intentional draft/mock trust boundary, not a proven live
  signing bypass or a retained P1 production-boundary defect.
- PLUG-SDK-001's P1 entry-discovery claim is withdrawn: loader selection accepts
  the nested entry. PLUG-APP-001/002's P1 interpretations are withdrawn in favor
  of a P2 timer-contract mismatch and conditional P2 recovery risk, respectively.
  PLUG-APP-007 is a mock-contract/test gap, not a current-host defect.
- AI-RUN-002/006 are integration/contract gaps, not proven unavoidable attribution
  failure or a requirement to reactivate terminal sessions. AI-CTX-003 is quota
  efficiency, not an established selected-only-storage contract; AI-CTX-010 is a
  defensive integration gap without an established live ingress path.

## Revision drift and evidence boundaries

The campaign used a **changing shared workspace, not an isolated snapshot**.
Recorded HEAD/status observations cannot exclude concurrent or transient user
changes. This index preserves historical revision evidence; it does not assert
one stable baseline or independently recheck current source HEADs.

- Terminal reports record `bitty` at `06bc1f4` and `bitty-devtools` at `77ca09c`;
  their observations include existing host `.targets/` and a DevTools `AGENTS.md`
  modification. These are report-time observations, not campaign-wide guarantees.
- AI runtime report 01 began at `e7cbe69` and advanced to `97d3125`; report 02 and
  second-pass source verification use `97d3125`. Standalone AI docs moved from
  `bb66faf` through `57c2d7d` to `fb4cda1`, while the source docs pin remained
  `07169bd`. The product index describes the intervening changes and unchanged
  cited contracts; sibling docs and mounted docs are not interchangeable.
- SDK moved from `3e9ebb5` to `84d41ae`, a TypeScript dependency-only change from
  5.7.2 to 7.0.2. Earlier test/typecheck success does not validate the new pin.
  Registry evidence is at `95bb0bf`; eight gitlinks were uninitialized and only
  Wheel initialized. Its mounted docs pin was 34 commits behind consulted sibling
  docs; sibling contents do not establish gitlink contents.
- Follow-ups retain terminal `06bc1f4`, AI `97d3125`, registry `95bb0bf`,
  standalone AI docs `fb4cda1`, and plugin docs `9fdbcd0`; terminal docs are
  `7947fb3` and shared docs `fc7ed97`. Documentation reports distinguish local
  tracked capture from remote state and standalone docs from immutable mounts.
  No mount was refreshed; pin lag alone is not a defect.
- Research observations span `26a46ed` to `d70152a`; earlier reports record
  existing September 16 work and later observations record the untracked
  September 17 campaign. Historical September 15/16 closure claims are not
  generally revalidated or rewritten by this index.

Full revisions, source slices, and contract references remain in the product
indexes and individual reports. No source fixes, commits, or CarryCtx operations
were performed for this campaign/indexing work.

## Test execution and uncovered areas

Execution results below are **historical first-pass receipts**, not tests rerun
by independent verifiers or by this indexing task. Follow-up rows record static
review limits only. No full workspace/product suite was run.

| Report(s)               | Recorded execution and limits                                                                                                                                                                    |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Terminal 01, 02, 03, 05 | Static review only; no product builds/tests                                                                                                                                                      |
| Terminal 04             | 284 selected Agent/IPC/test-support tests passed; no full workspace check, performance run, fuzzing, or cross-platform validation                                                                |
| Terminal 06             | 38 selected tests, TypeScript no-emit check, and two shell syntax checks passed; not live-client or Rust-path integration proof                                                                  |
| Terminal 07, 08         | Static coverage/documentation follow-ups only; no product builds/tests, live probes, or fault injection                                                                                          |
| AI 01, 02, 03, 04       | No Rust builds/tests/typecheck/clippy; selected test assertions and contracts read, not executed                                                                                                 |
| Plugins 01              | 53 selected SDK/template tests, SDK no-emit check, and generated-definition freshness check passed at the earlier SDK pin; complete suite/generation/host activation not run                     |
| Plugins 02              | 82 registry tests, registry freshness check, and registry/app typechecks passed; no app build, deployed storefront, installer fault/race, or end-to-end trust validation                         |
| Plugins 03              | 806 Lua assertions; Palette/Wheel manifest/parser gates passed. Other gates were blocked by missing `bitty-plugin-lint` or `luaparse`; full checks, LuaLS, and real host/Git integration not run |
| Plugins 04              | Sampled documentation/source checks only; no product tests, generated activation, or live registry/integration checks                                                                            |

Tests and assertions are different units and are not summed into coverage.
All second passes were static and ran no product tests. Markdownlint validates
report formatting only, not source correctness, test coverage, or finding fixes.

Explicit gaps across the campaign:

- `bitty-website` remains uncovered; the plugin storefront's sampled review is
  not a review of the separate website. Terminal 07 now covers compat-lab
  production bodies and the imported harness, but not corpus/reference-dump
  correctness, most test bodies, or live compatibility execution.
- Workspace-level scripts, CI/build/release/packaging systems, and dependency or
  supply-chain audits are not comprehensively reviewed or validated. Selected
  registry/DevTools scripts and configuration reads do not close these gaps.
- File-level coverage is limited to each report's ledger: full bodies, sampled
  ranges, outlines/search-only access, and unread files are different evidence.
  Second-pass targeted slices do not inherit full first-pass coverage. Unlisted
  ranges, most test bodies, and uninitialized gitlink contents remain unverified.
- Real PTY/GPU/compositor/UI behavior, Windows/macOS paths, end-to-end integration,
  live provider/network deadlines, durable recovery, scheduler/timer delivery,
  filesystem faults/races, fuzz/security acceptance suites, and performance or
  memory benchmarks are not established by these reports.
- AI 03 closes the recorded production-read gaps, including cache, prompt,
  fingerprint, selection, and host/fragment bodies. Its static conformance checks
  do not establish complete executed adapter/parser conformance; sampled inline
  tests and unopened integration suites remain explicit limits. Plugin
  Palette/Wheel second-pass behavior and remaining first-pass optimization/risk
  sections are not blanket endorsed. Documentation alignment is now sampled in
  Terminal 08, AI 04, and Plugins 04, not audited as entire corpora.

This is a report-only review archive, not release approval or assurance that
unlisted defects are absent. The earlier indexing pass covered this campaign
`README.md` and the archive's [root index](../../README.md). This follow-up changes
only this campaign README and the [terminal](bitty-terminal/README.md),
[AI](bitty-ai/README.md), and [plugin](bitty-plugins/README.md) product indexes.
The root index, all reports, September 16 work, and source files are outside this
edit scope. No network, installation, product execution, source fix, commit, or
CarryCtx operation is part of indexing. Its quality gate is installed
`markdownlint-cli2 --no-globs` on those four explicit Markdown files, followed
by final content read-back.
