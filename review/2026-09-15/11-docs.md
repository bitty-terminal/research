# Bitty documentation repository group quality review

## Scope

- Covered repositories (4 total; `bitty-website` explicitly out of scope, untouched in this round):
  - `bitty-docs/` (canonical main docs: `docs/`, `TODO.md`, `README.md`)
  - `bitty-terminal-docs/` (`docs/`, `specifications/`, `TODO.md`, and the root topic tree)
  - `bitty-ai-docs/` (`docs/`, `specifications/`, `TODO.md`)
  - `bitty-plugins-docs/` (`docs/`, `specifications/`, `TODO.md`, plus `extensibility/`, `product/`)
- No product-code implementation reviewed — documentation quality only: map/index navigation, decision register, open-question register, security corpus, split-docs vs canonical consistency, relative-link spot checks, TODO-vs-actual-state agreement, critical documentation gaps.

## Method

- Read-only spot checks; none of the reviewed files modified:
  - Read through `bitty-docs/docs/README.md` (documentation map), `bitty-docs/README.md`, `bitty-docs/TODO.md`, `bitty-docs/docs/decisions/index.md`, `bitty-docs/docs/decisions/open-questions.md`, `bitty-docs/docs/project/repository-map.md`, `bitty-docs/docs/project/project-state.json`, the full `bitty-docs/docs/security/` set (overview, threat-model, risk-register, evidence-matrix, p0-acceptance-criteria sampled), `bitty-docs/docs/roadmap/now-next-later.md`, `bitty-docs/docs/reviews/p0-review-checklist.md`.
  - Read through the three split repositories' `README.md`, `docs/README.md`, `TODO.md`, `specifications/README.md` (terminal) and `specifications/` directory listings, `extensibility/`, `product/`, `docs/plugins/` samples, `architecture/overview.md`, `interfaces/cli.md`, `interfaces/rich-content.md`, `reference/compatibility-matrix.md` samples.
  - Relative-link validity spot checks: script-parsed `../` links in the four `docs/README.md` files and verified the targets exist; cross-repository `https://github.com/bitty-terminal/.../blob/main/...md` links pattern-checked only, with no per-link liveness verification over the network.
  - Consistency spot checks: Draft counts, OQ Accepted counts, crate count/baseline revision, submodule pointers, ownership statements, status labels (Draft/Accepted/Implemented/Verified).
  - Checked `TODO.md` check states against actual tree state (empty directories, draft states, accepted RFCs).

---

## Defect list

Severity definitions: P0 = misleads acceptance decisions or the security baseline; P1 = contradiction/staleness/broken-link risk affecting navigation and sync; P2 = experience/hygiene gap.

### P0

- **D-P0-1 Aggregator submodules unmaterialized; canonical locally incomplete**
  - Paths: `bitty-docs/bitty-terminal/`, `bitty-docs/bitty-ai/`, `bitty-docs/bitty-plugins/` (all empty directories)
  - Symptom: `bitty-docs` calls itself the aggregator and mounts project docs through three root submodules, but all three mount points are empty directories in the review workspace, with `git submodule status` showing a `-` prefix on all three lines (uninitialized); cross-repository consistency and full navigation cannot be verified locally.
  - Evidence: `bitty-docs/.gitmodules` defines the three submodules; `bitty-docs/docs/README.md` says "Land project-document changes in the owning project docs repository, then bump the corresponding submodule pointer"; workspace `ls bitty-docs/bitty-terminal/` and equivalents are all empty. Per `bitty-docs/AGENTS.md` they need `git submodule update --init` to materialize, but under the current snapshot neither reviewers nor local CI gates see the full corpus.

- **D-P0-2 Remaining-Draft counts contradict themselves**
  - Paths: `bitty-docs/README.md`; `bitty-docs/docs/README.md`; `bitty-terminal-docs/specifications/README.md`
  - Symptom: three different figures in the same snapshot. The root README says "5 `Draft` specs remain (Status/Input/Text vs AI Arch plus Plugin Reuse)"; the documentation map says "7 `Draft` specs remain (see `bitty-terminal/specifications/README.md` prioritization)"; the terminal spec index body lists 11 Drafts outside the Accepted table (Status, Input, Text, Plugin Reuse, Browser/Agent Pre-Study, AI Architecture, Semantic, UI Extensibility, UI/Compositor Gap, Feature Gap, Workspace Panel Invariants), plus an un-updated 2026-09-01 historical "7 Drafts" paragraph. Readers cannot tell the current Draft base number.
  - Evidence: the three files' wording differs verbatim; the terminal index Draft table actually has 11 rows and the Accepted table 20+ rows, matching neither "5" nor "7".

- **D-P0-3 P0 review checklist baseline stale, still frozen at Phase A**
  - Paths: `bitty-docs/docs/reviews/p0-review-checklist.md`
  - Symptom: frontmatter and body frozen in several places at "16 crates `be3bdb4`, 32 OQs Accepted, soak ~808 tests", while the current canonical snapshot is 19 crates `bea338d`, 40 OQs Accepted, release `v0.0.20`. The file calls itself the "CTX-0048 coordination single source", so the stale baseline is easily mistaken for the current P0 bar.
  - Evidence: `grep` of the file hits "16 crates / 32 OQ / `be3bdb4` / 808" about 7 times; `bitty-docs/docs/project/project-state.json` (`snapshot_date 2026-09-14`) shows 19 crates / `oqs_accepted 40` / `bea338d`.

### P1

- **D-P1-1 Cross-repository absolute links all point at the `main` branch, conflicting with the pinned-revision contract**
  - Paths: `bitty-terminal-docs/specifications/README.md` and all repositories (all 56 sampled cross-repository links are `.../blob/main/...md`); `bitty-docs/docs/project/website-content-contract.md`; `bitty-docs/docs/development/documentation-workflow.md`
  - Symptom: split-to-split and split-to-canonical links uniformly use the floating `blob/main` branch, while the website content contract requires "consume only eligible documents from a pinned revision" and "must also resolve the aggregator's recorded submodule pointers". Local `just check` only validates in-repository relative links, so cross-repository breakage/drift is invisible to the local gate.
  - Evidence: of the terminal spec index's 56 links, 0 are relative and the majority are cross-repository absolute links; `bitty-docs/docs/README.md` states "No website content consumer exists yet… must also resolve submodule pointers", but the loader update is "not yet implemented".

- **D-P1-2 Ownership undecided: `status-system.md` and `default-distribution-rfc.md` claimed differently on two sides**
  - Paths: `bitty-plugins-docs/TODO.md`; `bitty-terminal-docs/specifications/`; `bitty-plugins-docs/docs/README.md`
  - Symptom: the plugins TODO "Blocked / open" says "`status-system.md` stays terminal-owned, and `default-distribution-rfc.md` remains with terminal docs pending an ownership decision" — i.e. ownership pending; but the terminal side already maintains both as its own Accepted/Draft assets, and the plugins map does not list them. The classification follow-up has no owner, no OQ, no timeline.
  - Evidence: the plugins `TODO.md` is 25 lines total with only this item in the Blocked section; the terminal spec index lists both files as its own rows with no pending marker.

- **D-P1-3 Post-OQ-053 split, `default-distribution-rfc.md` main table unsynced; amendment contradicts body**
  - Paths: `bitty-terminal-docs/specifications/default-distribution-rfc.md`
  - Symptom: the top amendment (2026-09-14, CTX-0424) already declares "`palette` and `statusline` left the bundled catalog… it is nine today", but a later table in the body still lists both plugins as "bundled, disabled" (around lines 324–325) and keeps the old catalog count. Readers trusting the table get the wrong bundled set.
  - Evidence: `grep palette/statusline/split` in the file hits both the amendment (nine) and the stale table rows (bundled, disabled); `bitty-plugins-docs/product/bundled-plugin-split-decision.md` confirms the palette removal is merged and the statusline removal is pending merge, while the terminal main table tracks neither latter state.

- **D-P1-4 `extensibility/` two drafts duplicate accepted RFCs; divergence risk**
  - Paths: `bitty-plugins-docs/extensibility/plugin-system.md`; `bitty-plugins-docs/extensibility/package-management.md`
  - Symptom: both file headers still read "pre-implementation architecture… Candidate contract", while Platform / Lua Runtime / Isolation / Package Lifecycle / Package Follow-up / Host Runtime / API v1 Surface in the same repository's `specifications/` are all Accepted. Manifest schema, capabilities, and update policy are each stated twice (e.g. plugin-system says "No manifest schema has been finalized"), with no statement that the RFC wins and no plan for when extensibility is archived or slimmed to pointer pages.
  - Evidence: both files' frontmatter `status: draft`; the plugins map lists extensibility as "Pre-implementation … contracts" side by side with the specifications Accepted table, with no dedup note.

- **D-P1-5 `interfaces/cli.md` and `interfaces/rich-content.md` lag behind accepted RFCs**
  - Paths: `bitty-terminal-docs/interfaces/cli.md`; `bitty-terminal-docs/interfaces/rich-content.md`
  - Symptom: both file headers still read "pre-implementation… candidate contracts until separately specified", while the CLI Contract (OQ-017), Rich Presentation (OQ-008/015/016), and IPC and Agent (OQ-018) have long been Accepted. The interface pages were never upgraded to an "Accepted boundary + candidate details" structure, so new readers will underestimate contract maturity.
  - Evidence: both files' opening status blocks read so verbatim; the terminal documentation map's corresponding rows explicitly mark the Accepted RFCs with the OQs they closed.

- **D-P1-6 OQ Accepted count inconsistent between sources: 40 vs 36**
  - Paths: `bitty-docs/docs/project/project-state.json`; `bitty-docs/docs/decisions/open-questions.md`; `bitty-docs/README.md`; `bitty-docs/docs/README.md`
  - Symptom: `project-state.json` (`maturity.oqs_accepted 40`) agrees with the README/map "40 OQs Accepted", but the OQ register actually measures 100 lines (OQ-001..OQ-100) with about 36 body occurrences of "Accepted:" and about 55 of "| Open:". The gap is suspected to be whether "Accepted and shipped" (OQ-033..035 etc.) counts, or a State-column lag in the register; no document defines the counting rule.
  - Evidence: script counts as above; register rows such as OQ-033/034/035 carry State-column long sentences like "Accepted and shipped…", easily missed or double-counted by counting scripts.

- **D-P1-7 Panel Runtime status contradicts itself inside canonical**
  - Paths: `bitty-docs/docs/project/project-state.json`; `bitty-terminal-docs/specifications/README.md`; `bitty-terminal-docs/specifications/panel-runtime-pre-study.md`
  - Symptom: `project-state.json`'s `panel_runtime` still reads "Panel Runtime spec remains Draft pre-study CTX-0119", while the terminal spec index says the Panel Runtime RFC was Accepted on 2026-09-14 under CTX-0181 with the prior pre-study `archived` and superseded. Two verdicts in the same snapshot.
  - Evidence: the `project-state.json` `subsystems.panel_runtime.evidence` closing sentence verbatim; the terminal index paragraph "The Panel Runtime RFC was accepted on 2026-09-14…" plus the pre-study file frontmatter `archived`.

- **D-P1-8 `bitty-ai-docs` has only the `specifications/` single tree, mismatching the README's claimed owned scope**
  - Paths: `bitty-ai-docs/README.md`; `bitty-ai-docs/docs/README.md`; `bitty-ai-docs/TODO.md`
  - Symptom: the README claims ownership of "AI-core architecture and runtime lifecycle / Provider integration / Context management / trust boundaries / reference", but the root actually holds only `specifications/` (28 files, mostly Draft) + `docs/` process pages, with no `architecture/`, `context/`, `providers/`, or `reference/` trees. The TODO explains with "empty placeholder pages are avoided… land as reviewed content" — a valid reason — but the map gives no owning OQ or landing order per missing tree, leaving a "claimed-but-missing" navigation gap.
  - Evidence: `ls bitty-ai-docs/` root shows none of the above trees; `docs/README.md`'s AI Architecture Family diagram lists 10+ Draft subsystems with no corresponding directories.

### P2

- **D-P2-1 Relative-link spot checks pass, but coverage is in-repository only**
  - Paths: `bitty-terminal-docs/docs/README.md` (all 26 `../` links hit); `bitty-plugins-docs/docs/README.md` (all 14 hit); `bitty-ai-docs/docs/README.md` (all 32 hit, including 1 anchor); `bitty-docs/docs/README.md` (both hit)
  - Symptom: the sampled relative links have no breakage — to be credited; but since cross-repository links are all absolute URLs, neither the local gate nor this offline sampling verifies whether target revisions exist or anchors drifted. Low risk but long-lived.
  - Evidence: the sampling script output all `OK`; the terminal spec index has 0 relative links with the highest cross-repository-link share.

- **D-P2-2 Decision register stops at DIR-020; new OQ-084..100 have no DIR handoff note**
  - Paths: `bitty-docs/docs/decisions/index.md`; `bitty-docs/docs/decisions/open-questions.md`
  - Symptom: the DIR stops at DIR-020 (release distribution) while OQs already extend to OQ-100, of which security/architecture-critical candidates such as OQ-084 (core ontology), OQ-085 (trust levels), OQ-086 (sensitive-input), OQ-087 (command-risk) exist only as candidate paragraphs in the threat-model/IPC RFC, with no DIR statement of intent. A normal "undecided" state, but the map never says when a DIR is needed.
  - Evidence: the DIR dedupes to exactly 20 entries; the maximum OQ number is OQ-100; OQ-084..087 States are all Open candidate.

- **D-P2-3 Security corpus body interleaves candidate models; easily misread as accepted**
  - Paths: `bitty-docs/docs/security/threat-model.md`; `bitty-ai-docs/specifications/ipc-agent-rfc.md` (candidate sections); `bitty-docs/docs/security/overview.md`
  - Symptom: after the threat-model boundary diagram come the OQ-085 trust-level candidate and OQ-086 detection candidate sections; although qualified with "candidate… does not change any current boundary", sharing the page with normative invariants invites fast readers to mistake them for current controls. The overview's 10 invariants are themselves clear — this is a layout risk, not a content error.
  - Evidence: the threat-model paragraph "A candidate trust-level model…" verbatim; OQ-086/087 are likewise candidates in the IPC/AI Architecture RFCs.

- **D-P2-4 Per-plugin pages cover only three plugins; later split candidates have no placeholder note**
  - Paths: `bitty-plugins-docs/docs/plugins/` (only `activity/`, `palette/`, `statusline/` + `TEMPLATE.md`); `bitty-plugins-docs/product/plugin-roadmap.md`
  - Symptom: consistent with OQ-053 "`file-manager`, `git-panel`, `ai-panel`, `mail-panel` split later" — the missing pages are expected; but the plugin index never lists these "planned, pageless" candidates with their prerequisite contracts (panel-provider contract), so new contributors may create duplicate pages.
  - Evidence: the `docs/plugins/` directory listing; the split-later list in `bundled-plugin-split-decision.md`.

- **D-P2-5 TODOs broadly match reality, but two wordings need convergence**
  - Paths: `bitty-docs/TODO.md`; `bitty-terminal-docs/TODO.md`; `bitty-ai-docs/TODO.md`; `bitty-plugins-docs/TODO.md`
  - Symptom: broadly matching, to be credited: the main TODO's unchecked "final cross-repository documentation review after all initial repositories have their first commits" is exactly this review, and its preconditions are now met; all three split-repository TODOs honestly record migration completion and open items. But the plugins TODO ownership-pending item (see D-P1-2) has no corresponding OQ/CTX; and the ai TODO defers G-1..G-6 to external tracking without a pointer table, inconveniencing cross-repository tracking.
  - Evidence: all four TODOs are short (29/26/25/289 lines) with checks matching actual directories; the ai TODO's closing section declares pressure-test gaps externally owned verbatim.

---

## Recommendations

| ID   | Proposed fix                                                                                                                                                                                                                                                                                                                                                                                                     | Expected benefit                                                            | Effort |
| ---- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- | ------ |
| R-1  | Materialize and verify aggregator pointers: run `git submodule update --init` once in `bitty-docs` and record the three submodules' `main` SHAs; add an actual pin table (repo, path, SHA, behind-count) to the "Submodule pin semantics" subsection in `docs/project/repository-map.md`. Land CTX-0199's proposed `just docs-status` if cheap, else start with a handwritten table.                             | Eliminates D-P0-1; makes cross-repository consistency locally reproducible. | S      |
| R-2  | Unify the Draft count: take the `bitty-terminal-docs/specifications/README.md` Draft table as the single source of truth, change `bitty-docs/README.md` and `bitty-docs/docs/README.md` to reference that table (no more per-file numbers), and delete the 2026-09-01 historical count paragraph or explicitly mark it historical.                                                                               | Eliminates D-P0-2; the map stops drifting.                                  | S      |
| R-3  | Archive or refresh the P0 review checklist: change `bitty-docs/docs/reviews/p0-review-checklist.md` frontmatter to `archived` with a front-page note "superseded by project-state + evidence-matrix", or refresh the baseline to 19 crates / 40 OQs / `bea338d`.                                                                                                                                                 | Eliminates D-P0-3; prevents the stale bar from being cited.                 | S      |
| R-4  | Converge `default-distribution-rfc.md`: fold the body's stale bundled table into a "historical (pre-2026-09-14)" collapsed section, keep only the current 9 in the live table, and annotate the `palette` removal as done and the `statusline` removal as pending merge (linking `bitty` PR #678/#680), matching the amendment.                                                                                  | Eliminates D-P1-3; the default set is unambiguous again.                    | S      |
| R-5  | Rule on ownership and open an OQ/CTX: write the `status-system.md` vs `default-distribution-rfc.md` ownership either-way decision into the decision register or a new OQ, add owner plus next-review-point to the plugins TODO Blocked item, and close the dangling state.                                                                                                                                       | Eliminates D-P1-2; cross-repository duties clear.                           | S      |
| R-6  | Slim `extensibility/` to pointer pages: add "Authoritative contracts live in specifications/…; this page is orientation only" to both files' front pages with per-section links to the corresponding Accepted RFCs; delete duplicated manifest/schema definitions, keeping links only. Upgrade `interfaces/cli.md` and `rich-content.md` likewise to an "Accepted boundary + Draft details" structure.           | Eliminates D-P1-4, D-P1-5; single source of truth.                          | M      |
| R-7  | Define the OQ counting rule and align: define at the top of documentation-workflow or the OQ register that "Accepted count = rows whose State column contains Accepted (including Accepted and shipped)", then fix the `project-state.json` generator script or the register wording so 40 aligns — or change 40 to the actual value with the gap explained.                                                     | Eliminates D-P1-6; roadmap and OQ stop disagreeing.                         | S      |
| R-8  | Sync the Panel Runtime status: flip `panel_runtime` to Accepted (CTX-0181) in the next `just state-refresh` of `project-state.json`, keeping the pre-study as historical evidence; sync the sentence citing the old state in `bitty-docs/docs/roadmap/now-next-later.md`.                                                                                                                                        | Eliminates D-P1-7.                                                          | S      |
| R-9  | Add a navigation skeleton to `bitty-ai-docs` (no empty placeholder pages): add a "Planned trees" table in `docs/README.md` listing `architecture/`, `context/`, `providers/`, `reference/` each with owning OQs (e.g. OQ-054/055/066 etc.), prerequisite RFCs, and order — stating "no page = not started".                                                                                                      | Mitigates D-P1-8; gaps visible and plannable.                               | S      |
| R-10 | Harden cross-repository links (two steps): short term, add an "absolute cross-repo link allowlist" check to each repository's `just check` (only `blob/main` allowed, filenames must be on the allowlist); long term, implement pinned-revision consumption per website-content-contract (`sync:docs --pin` + submodule SHA resolution), then switch cross-repository links to pins or relative submodule paths. | Mitigates D-P1-1, D-P2-1; breakage becomes gateable.                        | M      |
| R-11 | Land the first DIR-015 shipped-factual guides: write `installation` and `getting-started` in the terminal repository per the CTX-0199 plan (with a `v0.0.20` qualifier); keep other empty user-tree pages in their empty-state declarations.                                                                                                                                                                     | Fills the user-docs gap; honors the TODO promise.                           | M      |
| R-12 | Security-corpus layout: converge all candidate paragraphs into a closing "Candidate models (not normative)" section, keeping only normative content in the body; apply the same to threat-model, IPC/Agent RFC, and AI Architecture.                                                                                                                                                                             | Eliminates the D-P2-3 misreading risk.                                      | S      |
| R-13 | Add the "planned" list to the plugin index: list split-later items (`file-manager`, `git-panel`, `ai-panel`, `mail-panel`) with the panel-provider contract prerequisite in `docs/plugins/README.md`, marked "no pages yet".                                                                                                                                                                                     | Eliminates the D-P2-4 information gap.                                      | S      |

---

## Missing documentation list

The following are all honestly-missing gaps with no empty placeholders; grouped by theme, priorities for planning reference:

### Architecture

- M-A1 Core ontology and identity model: OQ-084 (ownership, lifetime, persistence, permission of Instance/Workspace/Panel/Surface/ExecutionContext/Terminal/Session/Resource/Service/Agent, plus PanelId/WorkspaceId/ResourceId/ExecutionContextId/AgentId/GenerationId relations) has no standalone ontology document, scattered only across AI Architecture and Core boundaries. (Cf. D-P2-2)
- M-A2 AI architecture subsystem tree: `bitty-ai-docs` lacks `architecture/`, `context/`, `providers/`, `reference/`; 10+ Drafts such as Context Management, Command/Tool, Agent Coordination, Persistence exist only as single files with no layered navigation. (Cf. D-P1-8)
- M-A3 Terminal P0 spec trio not Accepted: Status System (P0 slice diagnostics), Input and Pointer Contract (P0 slice input), Text and Rendering RFC (P0 slice rendering, OQ-090..100) remain Draft, of which Text/Rendering has only `HeadlessRasterizer` non-user-ready evidence. (Terminal spec index Draft table P0 rows)
- M-A4 Workspace Panel invariants candidate: `workspace-panel-invariants.md` (OQ-058, M4 hardening) remains Draft with only `ctx-0405` PR test evidence. (Terminal spec index Draft table)

### Security

- M-S1 Trust-tier model: OQ-085 (level 0–4 × capability domains matrix) is only a candidate paragraph; threat-model + P0 acceptance revisions never happened. (Cf. D-P2-3)
- M-S2 Sensitive-input interlock: OQ-086 (termios no-echo vs heuristics, secret/destructive/safe trichotomy, agent-input interlock, snapshot-exclusion) mechanism, typed denials, and acceptance evidence undecided. (OQ register Open)
- M-S3 Command risk tiers and audit: OQ-087 (tiers, hard-deny, consent-ledger, Tool Bus composition) has no parser/tier engine/deny policy. (OQ register Open)
- M-S4 Key storage and credential references: OQ-054 (`api_key_env` vs `api_key_cmd` resolution order and project-override boundaries), OQ-055 (secret tiers, consent/audit/redaction per tier) are only AI Architecture MPC candidates, with no storage contract beyond ADR 0006. (OQ register Open)

### Interfaces

- M-I1 Release-ready CLI/output contract: `interfaces/cli.md` is a draft; the Accepted CLI Contract's action/output schema and exit codes 0–8 were never settled into a stable user/automation reference page. (Cf. D-P1-5)
- M-I2 Rich/image user contract: `interfaces/rich-content.md` is a draft; the Rich Presentation Accepted boundary was never rewritten as an author-usable presentation interface page. (Cf. D-P1-5)
- M-I3 Compatibility guarantee page: `reference/compatibility-matrix.md` is Draft evidence (`4135803` CTX-0404); `requirements/`, `reference/`, `migrations/`, `troubleshooting/`, `tutorials/`, `how-to/`, `examples/` are all explicit empty states; per the risk-evidence gate, `Compatible`/`Release-ready` claims must not promise stable behavior before the gate, so the gaps are expected but must stay on the map. (Terminal empty-tree state)
- M-I4 Plugin SDK/manifest/lifecycle formal reference: the `sdk/`, `manifests/`, `lifecycle/`, `isolation/`, `reference/` trees are not yet built (plugins map Planned structure); currently only specifications + extensibility drafts + three plugin pages. (Plugins map)
- M-I5 Installation and getting-started shipped-factual guides: CTX-0199 already opened the policy door (DIR-015 qualifier), but the `v0.0.20` AUR/Releases/`bitty init`/`bitty doctor` guides are not yet written into the owning project docs. (Main TODO Follow-ups)

---

Date: 2026-09-15

Covered repositories: `bitty-docs`, `bitty-terminal-docs`, `bitty-ai-docs`, `bitty-plugins-docs` (`bitty-website` out of scope for this round, not reviewed).
