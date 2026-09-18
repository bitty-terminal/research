# Plugin documentation alignment — 2026-09-17

## Scope and baseline

Sampled read-only review of ecosystem docs, registry public contracts, research
capture, and selected terminal host seams. Only the three requested reports are
written. No source changes, CarryCtx, agents, network, installs, commits, or
behavioral reproductions. Evidence paths are workspace-relative; ranges are
inclusive. P2 denotes bounded documentation/data-contract correction, not a
proven live security incident.

| Repository            | Local HEAD                                 | Entry dirty baseline                                           |
| --------------------- | ------------------------------------------ | -------------------------------------------------------------- |
| `bitty-plugins`       | `95bb0bf610885e7d4464cc3056d9318c82c3e4d7` | Clean                                                          |
| `bitty-plugins-docs`  | `9fdbcd02af835d23868a7c15c5ce12f1efcd093a` | Clean                                                          |
| `bitty-terminal-docs` | `7947fb39acf77d308c2eb8bc57b24c481a841a17` | Clean                                                          |
| `bitty`               | `06bc1f45995fd297a3f7324688bc81b0598ecf67` | Existing untracked `.targets/`; no tracked changes             |
| `research`            | `d70152a079c75204ec37e99a7bf3f54b8190e423` | Modified `README.md`; existing untracked September 17 campaign |

The registry's recorded docs gitlink is
`ee19a0dbc10b6f1caf68d90627958a8b4cc4780c`. Source/doc mounts were not
initialized or refreshed; standalone docs are the cited contract snapshot.
SDK, template, and independent plugin HEADs were observed, but their source
bodies were not rereviewed for this documentation report. The September 17
plugin second-pass index supplies qualified contextual findings, not blanket
endorsement of its first-pass assertions or test receipts.

Cross-project routing, pin semantics, and the shared research ledger's stale
032/038 omission claims are owned only by
[the terminal report](../bitty-terminal/08-documentation-alignment.md)
(TERM-DOC-003/005), not duplicated below.

## Findings

### PLUG-DOC-001 — P2 defect: one manifest-hash field has incompatible raw and canonical meanings

- **Public schema:** `bitty-plugins/registry/schema.json:28-32` defines
  `manifest_hash` as canonical-form manifest digest H-B.
- **Procedure and actual record:** `bitty-plugins/README.md:150-156` instructs
  hashing the manifest bytes at the plugin pin. The checked official record
  says the same at `bitty-plugins/registry/official/activity.toml:7-10` and
  stores the value at `:21`.
- **Canonical contract:**
  `bitty-plugins-docs/specifications/package-lifecycle-rfc.md:143-145,158-179`
  distinguishes approved semantic content from transport bytes, with versioned
  canonical serialization for H-B. Cosmetic TOML changes must not be silently
  reinterpreted as semantic changes under this name.
- **Impact:** schema consumers and registry maintainers disagree on what the
  field binds. Advisory status limits assurance; it does not remove a present
  data-contract contradiction. This independently corroborates the
  documentation portion of PLUG-REG-006, not a second runtime defect
  (`research/review/2026-09-17/bitty-plugins/02-registry-and-distribution.md:260-281`).
- **Remedy:** name/version raw-file digests separately or populate the agreed
  canonical encoding; synchronize schema, README, generation, and records under
  the chosen contract. Keep publisher authentication separate. No digest was
  recomputed for uninitialized plugin pins and no signature chain was tested.

### PLUG-DOC-002 — P2 drift: extraction gates cite a superseded Panel Runtime pre-study

- **Docs:**
  `bitty-plugins-docs/product/bundled-plugin-split-decision.md:90,154-160`
  says Panel Runtime's pre-study is still draft and acceptance is the remaining
  gate. This is a current blocking rationale, even though other table entries
  have been updated for the completed catalog split.
- **Current canonical status:**
  `bitty-terminal-docs/specifications/panel-runtime-rfc.md:14-31` explicitly
  accepts the generic Panel Runtime contract and supersedes the pre-study,
  while preserving unresolved provider/interface questions and denying any
  automatic implemented/Verified claim.
- **Implementation evidence:**
  `bitty/crates/bitty-runtime/src/registry/panel.rs:1032-1074` supplies the
  registry mount mechanism. The current RFC distinguishes that from an exposed
  plugin-facing API at
  `bitty-terminal-docs/specifications/panel-runtime-rfc.md:693-707`.
- **Impact:** maintainers may repeat an already completed generic RFC promotion
  instead of identifying the still-missing public Lua/provider surface.
- **Remedy:** acknowledge generic RFC acceptance, then name the precise
  unresolved plugin panel registration/mount contract, owner, and delivery
  evidence needed. Do not infer that generic acceptance enables independent
  panel presentation or that a missing public surface is already delivered.

### PLUG-DOC-003 — P2 drift: accepted Git-tool documentation still points to removed bundled owners

- **Docs:**
  `bitty-plugins-docs/specifications/plugin-reuse-and-providers.md:226-245,250-253`
  uses `git_panel_manifest` in bundled source and
  `GitIntegration::is_process_spawn_git_allowed`/`runtime/git_panel.rs` as
  current evidence. Its overview still says the Git panel splits later
  (`:210-212`).
- **Contradictory maintained page:**
  `bitty-plugins-docs/product/bundled-plugin-split-decision.md:109-110,162-173`
  records removal of the bundled Git panel and the independent package.
- **Current source owner:**
  `bitty/crates/bitty-plugin-host/src/bundled.rs:550-559` contains the remaining
  six catalog entries; Git panel is not one. Host verb/tool authorization now
  runs through `HostToolsAuthorizer` at
  `bitty/crates/bitty-runtime/src/plugin_runtime/spawn.rs:416-456`, constructed
  by the backend at `:1008-1049`.
- **Impact:** reviewers following the accepted contract's implementation links
  are sent to retired owners instead of the current enforcement seam.
- **Remedy:** link immutable historical evidence as history and identify the
  current host authorizer separately from independent plugin policy. Keep
  installation-time table validation, executable/version discovery, capability
  grant binding, and dispatch-time verb validation distinct. The broad deferred
  install-enforcement wording at
  `bitty-plugins-docs/docs/plugins/git-panel/evidence.md:49-55` cannot be
  used to claim that no host dispatch enforcement exists, but the sampled
  authorizer alone does not prove the entire `[tools.*]` install contract.

## September 16 recheck and non-findings

| Earlier assertion                                                                                                                                                           | Current disposition                                                                                                                                                                                                                                                                                                                                                           |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Split decision still says file-manager/Git panel split later and no per-plugin pages exist (`research/review/2026-09-16/02-code-vs-docs-progress-audit.md:172-183`)         | Superseded: `bitty-plugins-docs/product/bundled-plugin-split-decision.md:109-110,144-173` records the split and deferred presentation; Git Panel evidence page exists. Current bundled catalog at `bitty/crates/bitty-plugin-host/src/bundled.rs:550-559` corroborates six remaining entries. Residual gate/evidence drift is PLUG-DOC-002/003, not the old blanket omission. |
| No provider/history capture in plugin docs (`research/review/2026-09-16/03-research-records-traceability-audit.md:36-43`)                                                   | Superseded by `bitty-plugins-docs/specifications/plugin-reuse-and-providers.md:534-602`. It preserves candidate status and assigns AI/provider and plugin responsibilities; historical summaries 032/038 align in direction. This is not implemented provider/history infrastructure.                                                                                         |
| SDK/template entry absence proves generated plugin cannot load                                                                                                              | Do not carry forward: September 17 second pass withdrew the P1 loader claim and retained P2 layout drift (`research/review/2026-09-17/bitty-plugins/README.md:49-55`). No generated plugin activation was attempted in this review.                                                                                                                                           |
| Advisory registry integrity plus mock trust establishes a production signing bypass (`research/review/2026-09-16/04-architectural-drift-and-subjective-analysis.md:85-108`) | Unsupported as a live-chain conclusion. `bitty-plugins-docs/docs/README.md:110-115` explicitly disclaims verified status/tamper defense; package lifecycle keeps real signature verification draft (`specifications/package-lifecycle-rfc.md:186-189`), consistent with `bitty/crates/bitty-package/src/trust.rs:273-284`. Raw/H-B ambiguity remains PLUG-DOC-001.            |
| Broad host capability/timer/mock findings all imply broken independent plugins                                                                                              | Preserve the corrected September 17 distinctions, not stronger first-pass labels (`research/review/2026-09-17/bitty-plugins/README.md:47-58,88-109`). No independent plugin runtime, scheduler, or integration check was run here.                                                                                                                                            |

**Intentional unresolved compatibility:**
`bitty-plugins/README.md:185-232` openly documents resolver/SDK/registry
range differences and defers convergence. That is an acknowledged cross-owner
contract question, not a newly discovered undocumented implementation claim.
The five-entry historical wording was not elevated to a finding; this review
sampled only Activity's registry entry.

## Precise sampled coverage

- Read workspace, research, registry, host, and all standalone docs AGENTS and
  ctxctl-core. Source reading used repository-local CLI outlines and narrow
  slices; the MCP path guard did not admit this external workspace.
- Standalone plugin docs: split decision `:85-174`; Git Panel evidence
  `:12-60`; plugin reuse/provider contract `:206-285,530-604`;
  package lifecycle `:143-190`; docs map `:90-117`. Targeted searches located
  research 032/038 and integrity discussion; not all returned pages were read.
- Cross-owner contracts: terminal Panel Runtime `:14-39,691-738`; shared
  documentation map, decisions, project state, and security sections precisely
  listed in the terminal report. No exhaustive per-plugin page-set review.
- Registry: README `:150-269`, schema `:20-51`, Activity entry `:1-24`, and
  `.gitmodules:1-28`; local Git HEAD/status and docs-gitlink checks. No generated
  index, storefront behavior, metadata fetch, registry release chain, or
  plugin-hash-to-pin equality verification.
- Source: catalog outline plus catalog function; spawn outline and
  `:416-456,1000-1052`; trust outline and `:270-290`. These are narrow checks,
  not a renewed installer/sandbox audit. The body of the six-entry catalog is
  read back during report validation.
- Research: complete September 16 reports 01–04; September 17 plugin index;
  registry report `:90-174,260-282`; full summaries 032 and 038. Independent
  plugin report contents were not fully rereviewed; original research records
  were filename-checked only.

## Checks and boundaries

Only the requested English Markdown reports changed. Installed Markdownlint
runs from `research` with `--no-globs` and all three explicit paths, without
fixes; the final result is recorded in the task response. No Rust/TypeScript
changes, builds, typechecks, product tests, live registry/provider connections,
or reproduction material. Existing dirty paths are preserved. Report lint is
not proof of host integration, security acceptance, or remote delivery state.
