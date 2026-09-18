# Coverage — `bitty-docs`

Records whose governance- and umbrella-relevant conclusions informed canonical
documents in the `bitty-docs` corpus. Source: the recovered `bitty-docs`
`research-notes-coverage` ledger (records 001–055) translated to the corpus's
current topical document names. The retired `research-notes-coverage.md` page
is not cited.

The recovered ledger kept 001–016 deposit evidence and applied its labels to
017–055. Status words below follow that ledger: `Recorded`, `Owner-confirmed`,
`Needs owner confirmation`, `Partial`, `Open`.

## Mappings

| Record | Record topic (condensed)                                                              | Informing document(s) in `bitty-docs`                                                                                                          | State                                                                  |
| ------ | ------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| 003    | Mainstream terminal lineages; minimal core plus infinite composition                  | `docs/project/technology-strategy.md` — Technology and Dependency Strategy; `docs/project/reference-projects.md` — Reference Project Register  | Recorded                                                               |
| 007    | Engineering status review; code outran docs; alignment needed                         | `docs/project/project-state.json` — machine-readable project state; `docs/development/documentation-workflow.md` — Documentation workflow      | Recorded                                                               |
| 010    | Bitty-AI differentiation; spatial multi-agent; output compression                     | OQ-061..OQ-066 in `docs/decisions/open-questions.md` — Open-question register                                                                  | Recorded as candidate/direction                                        |
| 011    | CarryCtx/ctxctl as native subsystems; roles and capabilities; Lua sandbox             | `docs/decisions/index.md` — Decision register (OQ-067..OQ-070 in the open-question register)                                                   | Recorded as candidate/direction                                        |
| 012    | Batched file I/O; token governance; atomic multi-file patch                           | OQ-071 in `docs/decisions/open-questions.md`                                                                                                   | Recorded as candidate/direction                                        |
| 013    | Architecture panorama; streaming components; embodied workspace                       | OQ-080..OQ-083 in `docs/decisions/open-questions.md`                                                                                           | Recorded as candidate/direction                                        |
| 014    | Panel/Execution separation; trust levels; core ontology                               | OQ-084/OQ-085 in `docs/decisions/open-questions.md`                                                                                            | Recorded as refinements                                                |
| 015    | Secure interactive input; fail-closed agent interlock; command-risk audit             | `docs/security/threat-model.md` — Threat Model; OQ-086/OQ-087 in the open-question register                                                    | Recorded as candidate security direction                               |
| 016    | Mod/Leader key selection; Bitty Beacon spatial action engine                          | OQ-088/OQ-089 in `docs/decisions/open-questions.md`                                                                                            | Recorded as candidate/direction                                        |
| 017    | Phase assessment: terminal core hardening, validation convergence, tighter invariants | `docs/development/documentation-workflow.md`; `docs/project/project-state.json`                                                                | Recorded                                                               |
| 019    | Flat multi-repository composition with submodules and per-repository ownership        | `docs/project/repository-map.md` — Repository and Local Workspace Map; `docs/development/documentation-workflow.md`                            | Recorded                                                               |
| 024    | Stop design inflation; reconcile first, then extract and harden                       | `docs/decisions/index.md` (DIR-015); `docs/development/documentation-workflow.md` (user-doc maturity tiers); `docs/project/project-state.json` | Recorded                                                               |
| 029    | Core stays offline; extensions cross controlled boundaries                            | `docs/decisions/index.md` (DIR-016/DIR-017)                                                                                                    | Recorded for the governance corpus                                     |
| 034    | Fix the packaging/install/run chain before new distribution formats                   | DIR-020 in `docs/decisions/index.md`; `bitty-terminal-docs` `product/release-distribution.md`                                                  | Recorded                                                               |
| 035    | Image protocols are producers; appearance/compositing are independent layers          | DIR-021 in `docs/decisions/index.md`; `bitty-terminal-docs` `architecture/graphics-appearance.md`                                              | Recorded                                                               |
| 036    | Inherit pane environments via prompt-time sync, masked for agents                     | DIR-022 in `docs/decisions/index.md`; `bitty-terminal-docs` `specifications/panel-environment-state-candidate.md`                              | Recorded for the terminal corpus                                       |
| 038    | Core emits events; durable history is a plugin reusing external history               | DIR-023 in `docs/decisions/index.md`; `bitty-terminal-docs` `specifications/panel-history-candidate.md`                                        | Recorded for the terminal corpus                                       |
| 042    | Separate dependency kinds; firewall Core; own release compatibility                   | `docs/development/platform-compatibility.md` — Platform Compatibility and Dependency Governance (draft); DIR-024                               | Recorded: capture merged via PR bitty-docs#315                         |
| 043    | Native, VM, and physical test tiers with reproducible VM base images                  | `docs/development/testing-infrastructure.md` — Testing Infrastructure (draft); DIR-025                                                         | Recorded: capture merged via PR bitty-docs#315                         |
| 044    | Execution supervisor: Job/Execution first-class; mechanisms vs semantics              | `docs/development/execution-host-boundary.md` — Execution Host and Supervisor Boundary (draft); DIR-026                                        | Recorded for the bitty-owned mechanisms; AI semantics owner-pending    |
| 045    | Agent authority: Lua decides how, never whether; hard-safety boundary                 | `docs/development/agent-authority-boundary.md` — Agent Authority and Hard-Safety Boundary (draft); DIR-027                                     | Recorded for the bitty-upstream mechanisms; AI semantics owner-pending |

## Notes

- Records 048–056 were split-owner in the recovered ledger. Their
  `bitty-ai`-owned halves were distilled in `bitty-ai-docs`; the governance,
  Wheel/harness, context, tool/ACI, skill, verification, and plugin-ecosystem
  halves stayed owner-pending. They are listed here for completeness with
  `Partial`/`Open` state and no invented `bitty-docs` document.
- Records 042/043 are captured in `bitty-docs` through PR bitty-docs#315
  (merged); 044/045 through PRs bitty-docs#321/#323.
- `docs/decisions/rfcs/` and `docs/decisions/adrs/` hold the accepted
  decisions; the register and open-question register index them.
- The corpus's self-containment pass removed the `research-notes-coverage`
  ledger and now points readers here; `docs/sources/README.md` states that
  record-to-document mappings live in the `research` repository.

## Related

- [`README.md`](README.md) — coverage index.
- [`bitty-ai-docs.md`](bitty-ai-docs.md), [`bitty-terminal-docs.md`](bitty-terminal-docs.md),
  [`bitty-plugins-docs.md`](bitty-plugins-docs.md) — sibling corpora.
