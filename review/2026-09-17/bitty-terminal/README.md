# bitty-terminal product review — 2026-09-17 (second-pass index)

- Product tree reviewed: the umbrella directory is routing only, not a Git repository; each child (core `bitty`, docs corpora, plugins, `bitty-devtools`) is an independent repository. Source paths in this index are relative to the workspace umbrella.
- This index covers eight campaign reports. It is a verification index, not a complete product audit. Reports 01–06 retain their independent second-pass dispositions; report 07 adds compat-lab and selected runtime/configuration coverage, and report 08 adds sampled terminal/shared documentation alignment. Other product source reviews remain in their own indexes. This indexing follow-up edits only the campaign and three product READMEs, not reports or source.
- Second-pass method: ctxctl outlines followed by targeted source slices on unchanged checkouts; six sibling reports were read first, then strong claims were checked against current source. This is sampled verification, not a fresh exhaustive audit, and no product build/test/payload/reproduction was run.
- Baselines rechecked: `bitty` HEAD `06bc1f45995fd297a3f7324688bc81b0598ecf67` (only pre-existing untracked `.targets/`), `research` HEAD `d70152a079c75204ec37e99a7bf3f54b8190e423` (untracked `review/2026-09-17/`), `bitty-devtools` HEAD `77ca09c951fb3ac390f561a6e9a6afbc7a2ac64e` (pre-existing unstaged `AGENTS.md`). No source edits; no network, installs, commits, or CarryCtx; no payloads or exploit material.
- Review guidance respected: AGENTS instructions (umbrella, `research`, `bitty`, `bitty-devtools`), ctxctl-core skill, Markdown-only tooling, defensive remediation only. The workspace "pre-implementation" guidance and product docs can both drift; source files, not task prose, are the evidence standard.

## Reports

1. [01-terminal-engine.md](01-terminal-engine.md) — `bitty-vt`, `bitty-term-state`, `bitty-render`, `bitty-pty`, `bitty-platform`.
2. [02-runtime-orchestration.md](02-runtime-orchestration.md) — `bitty-runtime` lifecycle, routing, persistence, panels.
3. [03-app-ui-core-config.md](03-app-ui-core-config.md) — `bitty-app`, `bitty-ui`, `bitty-core`, `bitty-config`.
4. [04-ipc-agent-perf-testsupport.md](04-ipc-agent-perf-testsupport.md) — `bitty-ipc`, `bitty-agent`, `bitty-perf`, `bitty-test-support`.
5. [05-pluginhost-lua-package-rich.md](05-pluginhost-lua-package-rich.md) — `bitty-plugin-host`, `bitty-lua`, `bitty-package`, `bitty-rich`.
6. [06-devtools.md](06-devtools.md) — `bitty-devtools` consumer client (TypeScript plus Rust adapter).
7. [07-coverage-followup.md](07-coverage-followup.md) — compat-lab production bodies/imported harness, selected runtime production gaps, configuration merge/types/validation.
8. [08-documentation-alignment.md](08-documentation-alignment.md) — sampled terminal/shared documentation, status/routing/provenance, and selected September 16 corrections.

## Highest priorities (qualified)

These are the strongest second-pass-verified findings with their exact qualifications. Severities are static-remediation priorities, not demonstrated incidents.

- TERM-RUN-001 (P1): synchronous spawn runner joins pipe readers beyond the deadline and returns empty output on timeout — `bitty/crates/bitty-runtime/src/plugin_runtime/spawn.rs:672-831`.
- TERM-RUN-002 (P1): post-spawn errors can abandon a running child — `bitty/crates/bitty-runtime/src/plugin_runtime/spawn.rs:672-831`.
- TERM-RUN-003 (P1): plugin-store mutations publish before persistence, and rename fallback can delete the last committed file — `bitty/crates/bitty-runtime/src/plugin_runtime/store.rs:50-123,143-189`.
- TERM-IPC-001 (P1, control-assurance): live serving marks a filesystem-attested marker without live peer verification — `bitty/crates/bitty-ipc/src/auth.rs:125-168`, `bitty/crates/bitty-ipc/src/devtools/serve.rs:803-826`, `bitty/crates/bitty-app/src/ipc_serve.rs:295-354`.
- TERM-IPC-004 (P1): timed-out control work remains queued and can execute after the client times out — `bitty/crates/bitty-ipc/src/ctl.rs:1033-1054,1070-1087,1175-1217`, `bitty/crates/bitty-app/src/ctl/apply.rs:27-41`.
- TERM-DEV-001/002/005 (P1): executable wired to the stub runner, live client routing typed operations to headless models, and UTF-8-unsafe byte slicing in the Rust redactor — `bitty-devtools/bin/bitty-devtools.ts:14`, `bitty-devtools/src/client.ts:85-114,199-263`, `bitty-devtools/crates/devtools-client/src/redaction.rs:56-82,185-201,224-242`.
- TERM-ENG-001 (P1 API contract): documented `damage_since` full-redraw fallback is absent beyond the retained history window — `bitty/crates/bitty-term-state/src/state.rs:705-720,915-935`, `bitty/crates/bitty-term-state/src/damage.rs:24`.
- TERM-HOST-002 (P1, unmeasured exhaustion): Lua registration capture has no host-side admission quota — `bitty/crates/bitty-lua/src/host.rs:397-425,610-706,914-965`.

"Verified" means the cited mechanism was read in current source, not that a runtime incident, frequency, or exploit was demonstrated. Every P1 above needs remediation design plus tests; none is a confirmed production outage.

## Second-pass status by report

- 01: TERM-ENG-001/002 mechanisms verified; TERM-ENG-003 withdrawn as a defect (unverified compatibility policy); TERM-ENG-004 withdrawn as a present defect; TERM-ENG-005 reclassified P3 test follow-up.
- 02: TERM-RUN-001/002/003 verified; TERM-RUN-004 through TERM-RUN-011 not independently reverified.
- 03: TERM-APP-001/002 verified (P2 each); TERM-APP-003/004/005 reclassified as maintenance/test/scaffold, not functional defects.
- 04: TERM-IPC-001/004 verified; TERM-IPC-008/010/011 mechanisms verified; the remaining P2/P3 not independently reverified.
- 05: TERM-HOST-001 reclassified P2 intentional-draft boundary; TERM-HOST-002/004 verified; the remaining P2 items not independently reverified.
- 06: TERM-DEV-001/002/003/004/005 verified; TERM-DEV-006 through TERM-DEV-015 not independently reverified.

## Follow-up coverage and documentation dispositions

Reports 07–08 record their own static checks, not a new independent second pass of all earlier claims. Their source baseline remains `06bc1f4`; standalone terminal docs are at `7947fb3`, shared docs at `fc7ed97`. Report 08 distinguishes standalone docs from immutable mounts; lag alone is not a defect and no mount was refreshed.

| Report / IDs      | Qualified disposition                                                                                                                                                                                  |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 07 — TERM-GAP-001 | P2 evidence-accounting defect: references blocked by self-failure can count as passed despite no comparison. Self-failure remains visible; this does not mean the entire run succeeds.                 |
| 07 — TERM-GAP-002 | P2 evidence-accounting defect: dump collection can count snapshots as written when output directories were not created. Existing-file write failures instead panic; no publication fault was executed. |
| 07 — TERM-RUN-003 | Additional affected resolution-store rename-fallback path, not a duplicate new finding or demonstrated filesystem loss.                                                                                |
| 08 — TERM-DOC-001 | P1 contract/assurance drift: security and contributor entry points deny existing implementation. Source presence is not proof of acceptance, complete parser safety, or a production incident.         |
| 08 — TERM-DOC-002 | P2 incorrect panel implementation attribution: the machine state names `PanelRuntime::mount` rather than the owning `PanelRegistry::mount_panel`. Historical age alone is not the defect.              |
| 08 — TERM-DOC-003 | P2 cross-project plugin-doc routing drift; registry and independent plugin ownership must remain distinct.                                                                                             |
| 08 — TERM-DOC-004 | P2 research ledger topic mismatch for records 007/008; other numbered rows were not certified.                                                                                                         |
| 08 — TERM-DOC-005 | P2 capture-status drift for 032/038 and 042/043. Locally tracked destinations do not establish accepted features or current remote PR state.                                                           |

Report 08 supersedes selected September 16 assertions only: intentional pin roles, corrected topology, accepted panel contract versus implementation, inert Composer, candidate rich-content APIs, captured proposals, and draft trust boundaries retain their qualifications. Its Composer editor-allowlist revision caution is not fix closure in this checkout; historical counts and missing-changelog claims were not renewed.

## Verification method and evidence limits

- CtxCtl outlines preceded targeted source slices; the slices listed in each report's "Independent second-pass verification" section are the bodies actually reread. All other original coverage claims were not re-derived.
- Claim classes: verified (source mechanism reread at unchanged HEAD), unverified (first-pass claim not renewed — treat as leads), reclassified/withdrawn (disposition changed in the report), and confirmed-mechanism-not-incident (no dynamic evidence). Severity is not exploitability.
- Historical reports 01–06 second-pass Markdown lint: `markdownlint-cli2 --no-globs "review/2026-09-17/bitty-terminal/*.md"` from `research` (installed v0.23.1, markdownlint v0.41.1, research config): final pass 7 files (six reports plus this index), 0 issues. The campaign's product tests/typechecks were not rerun and are not per-finding evidence.

Reports 07–08 ran no product builds/tests or behavioral reproductions. Report 07's full, substantive-body, production-complete/sample, outline-only, and unread classes remain distinct; named tests and declared compatibility rows are not execution receipts. This four-index follow-up uses installed `markdownlint-cli2 --no-globs` on the campaign and three product READMEs, with final read-back; historical lint receipts above are not new product evidence.

## Explicit gaps

- Compat-lab was excluded from reports 01–06 but is now reviewed in report 07: production bodies and the imported harness were read. Most test bodies, corpus bytes, captured reference dumps, and live compatibility remain unverified; prior claims are not blanket validated.
- Report 07 extends runtime/configuration coverage without upgrading previously sampled siblings. Runtime `config.rs` and `runtime/present.rs` remain body-unread there; remaining config/package/resolution ranges and inline/integration tests retain their ledger limits. Config validation was fully read; merge/types coverage remains scoped.
- Plugin-family directories (`bitty-plugins`, `bitty-plugin-sdk`, `bitty-plugin-template`, independent plugins) and `bitty-ai` source reviews remain separately indexed. Report 08 samples terminal/shared docs and cross-owner contracts, not entire docs corpora, website, or packaging.
- Not exhaustively re-audited: Windows/macOS paths, real PTY/GPU/compositor behavior, `bitty-ui` presentation, remaining config bodies, dependency supply chain, performance benches, fuzz suites, security acceptance tests, and end-to-end integration.
- These reports are static point-in-time analysis of the pinned commits. They are not an exhaustive audit, interoperability certification, release-readiness statement, or guarantee that unlisted defects do not exist. Any exploitation, reproduction, or payload work is out of scope; findings require scoped defensive fixes, tests, and documentation sync before any implementation claim.
