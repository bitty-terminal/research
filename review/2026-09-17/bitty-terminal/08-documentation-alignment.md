# Terminal documentation alignment — 2026-09-17

## Scope, authority, and baseline

Read-only, sampled documentation/source review. Only this report and the two
requested sibling product reports are authored. No source fixes, other document
edits, CarryCtx, agents, network, installs, commits, or behavioral reproductions.
Paths in evidence are relative to the workspace root; ranges are inclusive.
P1 means high-priority contract/assurance correction; P2 means bounded
maintenance or contract clarification, not demonstrated production exposure.

| Local repository      | HEAD                                       | Entry dirty baseline                                                 |
| --------------------- | ------------------------------------------ | -------------------------------------------------------------------- |
| `bitty`               | `06bc1f45995fd297a3f7324688bc81b0598ecf67` | Untracked `.targets/`; no tracked changes                            |
| `bitty-docs`          | `fc7ed9750491875bea9986be35a4df5074ad84ba` | Clean                                                                |
| `bitty-terminal-docs` | `7947fb39acf77d308c2eb8bc57b24c481a841a17` | Clean                                                                |
| `bitty-ai-docs`       | `fb4cda1c5e9792c848d036903c7db322ee6d1743` | Clean                                                                |
| `bitty-plugins-docs`  | `9fdbcd02af835d23868a7c15c5ce12f1efcd093a` | Clean                                                                |
| `research`            | `d70152a079c75204ec37e99a7bf3f54b8190e423` | Modified `README.md`; untracked `review/2026-09-17/` already present |

These are local checkouts, not assertions about remote main or live PR state.
The September 17 terminal second-pass index uses the same terminal HEAD
(`research/review/2026-09-17/bitty-terminal/README.md:5-7`). Its qualified
findings, not uncorrected first-pass prose, supplied review context.

### Documentation pins are not interchangeable

Local `git ls-tree HEAD` recorded these immutable gitlinks:

| Mount                       | Pin                                        |
| --------------------------- | ------------------------------------------ |
| `bitty/docs`                | `56f70bc9b98bf7585d97ccafde33c81c5d69ac9d` |
| `bitty-docs/bitty-terminal` | `0b2fcfb5be1f65d457b0ffde0358e2ebac0a70c9` |
| `bitty-ai/docs`             | `07169bd9108875179899ac77c3af3c0073a8506a` |
| `bitty-docs/bitty-ai`       | `39b4c7568a8807e0940bd298660c972db0cfa92a` |
| `bitty-plugins/docs`        | `ee19a0dbc10b6f1caf68d90627958a8b4cc4780c` |
| `bitty-docs/bitty-plugins`  | `ee19a0dbc10b6f1caf68d90627958a8b4cc4780c` |

No mount was refreshed or initialized. Findings below cite standalone docs,
not an assumed copy at these pins. The authoritative distinction is explicit
in `bitty-docs/docs/project/repository-map.md:141-168`: code-matching docs,
governance-reviewed snapshot, and latest owning docs may intentionally differ.
Lag alone is not a defect and does not justify automatic lockstep updates.

## Findings

### TERM-DOC-001 — P1 drift: security and contributor entry points deny existing implementation

- **Docs:** `bitty-docs/docs/security/overview.md:14-19,102-105` says the
  controls are all currently unimplemented. `bitty/AGENTS.md:18-25,130-132`
  calls the repository unborn, workspace/MSRV undecided, and P0 controls
  unimplemented. These are present-tense status claims, not merely future
  acceptance requirements.
- **Counterevidence:** `bitty/Cargo.toml:1-29` defines nineteen members,
  edition, resolver, and MSRV. `bitty/crates/bitty-vt/src/bounded.rs:21-40,79-89`
  implements bounded action payloads. The canonical security matrix itself
  records implementation and separate mitigation states at
  `bitty-docs/docs/security/evidence-matrix.md:14-28`; the risk register does
  likewise at `bitty-docs/docs/security/risk-register.md:14-27`.
- **Impact:** readers cannot tell whether a control is absent or merely lacks
  complete acceptance evidence. This undermines defensive review prioritization.
  The sampled bounded wrappers do not prove complete parser/resource safety.
- **Remedy:** replace universal absence claims with the dated experimental
  implementation baseline and link each control to its evidence row. Preserve
  every normative requirement and the independent `Verified` gate; do not
  upgrade risk states on source presence alone.

### TERM-DOC-002 — P2 defect: machine project state misidentifies the implemented panel API

- **Docs:** `bitty-docs/docs/project/project-state.json:48-51` claims
  `PanelRuntime::mount` in `bitty-ui panel.rs` as implementation evidence and
  still calls Panel Runtime a draft pre-study. The file explicitly dates its
  snapshot to September 14 (`:3-12`), so being older than HEAD is not itself
  the finding.
- **Source:** the implemented operation is
  `bitty/crates/bitty-runtime/src/registry/panel.rs:1032-1074`
  (`PanelRegistry::mount_panel`), with handle checks and mapping mutation.
- **Current contract:**
  `bitty-terminal-docs/specifications/panel-runtime-rfc.md:14-31,693-701`
  distinguishes accepted contract terminology from the actual registry type.
  Its accepted status and explicit current-type correction contradict the
  machine fact's API attribution.
- **Impact:** machine-fed summaries and onboarding direct maintainers to a
  nonexistent named implementation instead of the owning runtime module.
- **Remedy:** preserve the historical snapshot, but correct or supersede its
  erroneous evidence with exact type/module/revision attribution and separate
  contract acceptance from implementation completeness. Do not rename source
  merely to match conceptual RFC terminology.

### TERM-DOC-003 — P2 drift: cross-project routing still sends plugin docs to the wrong mount

- **Docs:** `bitty-docs/docs/projects/README.md:19-23` gives the plugin
  code-repository mount as `bitty-plugins/<plugin>/docs`.
- **Repository evidence:** `bitty-plugins/.gitmodules:10-13` defines the
  registry repository's actual `docs` mount. The flat independent-repository
  topology is already corrected in
  `bitty-docs/docs/project/repository-map.md:41-46,80-104`.
- **Impact:** the principal routing table can direct edits or reads to a
  nonexistent ownership path; registry mounts and independent plugin repositories
  are different things.
- **Remedy:** route the ecosystem corpus to `bitty-plugins/docs`, and describe
  individual plugin documentation ownership separately without assuming every
  independent plugin has a docs submodule. Keep cross-cutting governance in
  `bitty-docs`; this finding is not duplicated in the plugin report.

### TERM-DOC-004 — P2 defect: research ledger assigns the wrong topics to records 007 and 008

- **Ledger:** `bitty-docs/docs/sources/research-notes-coverage.md:65-66`
  labels 007 as Starship/statusline and 008 as Lua-engine selection.
- **Actual sampled summaries:** `research/summary/007.md:1-24` is engineering
  status reconciliation; `research/summary/008.md:1-24` is semantic folding,
  hints, and Command Composer. Its destination is terminal/plugin docs.
- **Corroborating contract/source:**
  `bitty-terminal-docs/specifications/semantic-terminal-rfc.md:168-205`
  records Composer and the inert UI path; that path is visible at
  `bitty/crates/bitty-app/src/chrome_keys.rs:549-560`.
- **Impact:** a reader following the canonical provenance table is sent to
  unrelated design decisions and may repeat the September 16 claim that the
  Composer concept was lost. A name shared with scene composition is not proof
  of conceptual replacement.
- **Remedy:** reconcile these numbered identities and destinations against the
  archive, preserving original records. Validate identity/title/destination
  joins mechanically before expanding the ledger. No assertion is made that
  all other numbered rows are accurate or inaccurate.

### TERM-DOC-005 — P2 drift: capture status contradicts locally present destinations

- **Canonical omission claims:**
  `bitty-docs/docs/sources/research-notes-coverage.md:34-40,90,96` says records
  032 and 038 are owner-confirmed omissions in plugin docs.
- **Current destinations:**
  `bitty-plugins-docs/specifications/plugin-reuse-and-providers.md:534-564,566-602`
  explicitly captures each as candidate direction, matching
  `research/summary/032.md:10-18` and `research/summary/038.md:10-20`.
  These are captured proposals, not delivered provider/history features.
- **Opposite stale status:** `research/summary/042.md:3,23-24` and
  `research/summary/043.md:3,22-23` still attach an open/unmerged capture-PR
  qualifier. Both destination files exist in local tracked canonical source:
  `bitty-docs/docs/development/platform-compatibility.md:14-23` and
  `bitty-docs/docs/development/testing-infrastructure.md:14-22`.
  Local history records their capture at
  `d7e67cb2c29d2f95dd843c18f19f103e62fd3db6`; the ledger also records capture
  at `bitty-docs/docs/sources/research-notes-coverage.md:100-101`.
- **Impact:** follow-up planning can re-open already captured material or mistake
  historical PR prose for a present remote-state check.
- **Remedy:** record destination revision and draft/accepted status per corpus;
  replace transient PR-state qualifiers with durable local capture evidence.
  Preserve dates and historical reviews, linking forward to the new disposition.
  No network check or live PR-state conclusion was made here.

## September 16 assertions rechecked

The four September 16 reports were read in full; only the following claims
received current evidence checks. No blanket renewal of their corpus-wide counts
or security conclusions is intended.

| Earlier assertion                                                                                         | Current disposition                                                                                                                                                                                                                                                                                                                                                                                                             |
| --------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Report 01: mismatched pins are inherently critical; plugin code mount is N/A (`:28-45`)                   | Withdraw blanket inference. Pin roles are intentional; the registry does have `docs`. See baseline and TERM-DOC-003.                                                                                                                                                                                                                                                                                                            |
| Report 01: ADR/crate inventory, P0 checklist, and repository map remain at old topology (`:51-75,87-110`) | Materially corrected: `bitty-docs/docs/decisions/adrs/ADR-0003-core-workspace-topology.md:32-37,59-78`; `bitty-docs/docs/reviews/p0-review-checklist.md:25-35`; `bitty-terminal-docs/development/release-mechanics.md:113-141`; `bitty-terminal-docs/development/maintainability.md:29,53-61`; repository map above. Historical sixteen-crate paragraphs are not defects merely for remaining. Draft counts were not recounted. |
| Report 02: missing named PanelRuntime, gated modes, Composer imply phantom accepted features (`:51-95`)   | Current RFC explicitly separates name, contract, and implementation. `bitty/crates/bitty-ui/src/presentation.rs:102-113` still gates cross-mode transitions. Composer remains inert; default Alt+E activation was overstated by the old audit, as corrected in `bitty-terminal-docs/specifications/semantic-terminal-rfc.md:195-205`.                                                                                           |
| Report 02: rich-content architecture claims an implemented rich GPU pipeline (`:76-82`)                   | `bitty-terminal-docs/interfaces/rich-content.md:14-16,37-47` explicitly describes candidate interfaces, not implemented APIs. No new defect inferred from that conceptual diagram.                                                                                                                                                                                                                                              |
| Report 03: ActivityStack missing, 042/043 absent, 32% traceability failure (`:49-68,117-128`)             | ActivityStack captured as non-accepted direction at `bitty-terminal-docs/specifications/panel-runtime-rfc.md:714-738`; capture absence no longer holds. Percentage not reproduced and must not be carried forward.                                                                                                                                                                                                              |
| Report 04: embedded panel file names prove private privilege and architectural violation (`:54-81`)       | Insufficient evidence. DIR-018 already scopes decoupling (`bitty-docs/docs/decisions/index.md:47`); plugin split decision distinguishes mechanisms and deferred extraction. No API-parity or authorization failure established by file presence.                                                                                                                                                                                |
| Report 04: mock signing proves a live authentication bypass (`:85-108`)                                   | Do not carry forward. Source calls it a stub (`bitty/crates/bitty-package/src/trust.rs:273-284`); accepted package docs retain real-signature deferral (`bitty-plugins-docs/specifications/package-lifecycle-rfc.md:186-189`). September 17 corrected it to a draft-boundary concern (`research/review/2026-09-17/bitty-terminal/README.md:39`).                                                                                |

**Revision-applicability caution:**
`bitty-terminal-docs/specifications/semantic-terminal-rfc.md:223-229` describes
an editor allowlist update, whereas local `bitty` still has
`resolve_editor -> Option<String>` returning trimmed input at
`bitty/crates/bitty-rich/src/composer.rs:1072-1080`. This is a documented newer
source-line claim, not proof that a fix is in this reviewed checkout or absent
from another branch. Identify the implementing revision before treating it as
closure. The September 16 claim of forty-plus missing changelog fixes was not
revalidated against other local refs and is not renewed.

## Precise sampled coverage

- Guidance: workspace, research, terminal, shared docs, and all three standalone
  docs AGENTS; ctxctl-core skill. No mounted-doc content was substituted for
  standalone docs.
- Shared corpus: documentation map `:12-160`; repository map `:12-190`;
  project-state `:1-160`; decision register introductory rules and DIR-001..022
  rows; OQ register `:12-46`; project routing `:12-80`; ADR-0003 `:14-78`;
  P0 review checklist `:12-46`; source coverage `:12-106`; platform compatibility
  `:12-46`; testing infrastructure `:12-46`.
- Security: overview `:12-111`; threat model `:12-66`; risk register `:12-49`;
  evidence matrix `:12-96`; P0 criteria `:12-49`. Remaining risk/criterion rows
  were not exhaustively read or certified.
- Standalone terminal docs: Panel Runtime `:14-39,691-738`; semantic terminal
  `:168-229`; rich-content interface `:12-66`; Lua/XDG `:12-56`; targeted
  search results for implementation notes in compositor/rich-presentation and
  inventory notes in maintainability/release-mechanics. Search hits do not
  imply whole-document reads. No broad user-guide, CLI, packaging, website,
  migration, or link-corpus audit.
- Research: complete September 16 reports 01–04; September 17 campaign index
  `:1-150` and three product second-pass indexes; full summaries 007, 008, 023,
  025, 032, 037, 038, 042, 043. Original records were filename-checked only,
  not reread or rehashed; no 43-record completeness claim.
- Source: Cargo metadata plus the exact terminal functions/ranges cited above;
  outlines of panel registry, presentation, chrome dispatch, Composer, bundled
  catalog, spawn, VT module/bounds, event, and trust. Additional plugin/AI
  source checks are confined to their reports. No full product tests or builds.

## Checks and limitations

Installed Markdownlint is the sole report quality gate; the three explicit
report paths are linted with `--no-globs` and no fix flag. No Rust/TypeScript
changes require compilation or typecheck. Lint success is not implementation,
security, integration, or compatibility evidence. Final command/result is
recorded in the task response; existing dirty files are preserved.
