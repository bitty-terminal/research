# Documentation Hierarchy and Submodule Synchronization Audit (2026-09-16)

## 1. Scope and Objective

This report audits the documentation repository group of the Bitty project:

- `bitty-docs` (the root canonical aggregator and governance authority)
- `bitty-terminal-docs` (mounted as `bitty-docs/bitty-terminal` and `bitty/docs`)
- `bitty-ai-docs` (mounted as `bitty-docs/bitty-ai` and `bitty-ai/docs`)
- `bitty-plugins-docs` (mounted as `bitty-docs/bitty-plugins`)

The objective is to verify:

1. Submodule pointer synchronization across aggregator, split repos, and code repos.
2. Internal consistency of canonical governance (ADRs, Open Questions, Decision Register, Project State).
3. Documentation metadata, draft counts, and checklist currency.

---

## 2. Three-Way Submodule Divergence Analysis

The Bitty architecture uses a split-documentation model where individual documentation repositories are maintained independently and mounted into `bitty-docs` (the aggregator) and into code repositories (`bitty/docs`, `bitty-ai/docs`).

An empirical audit of Git commit pointers reveals widespread divergence:

### 2.1 Submodule Pointer Audit Matrix

| Documentation Corpus | Standalone Repo `HEAD` | `bitty-docs` Submodule Pointer | Lag (Commits Behind) | Code Repo Mount Pointer | Code Mount Lag |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **`bitty-terminal-docs`** | `6100ec9` (2026-09-16 16:09) | `0b2fcfb` (2026-09-16 10:44) | **4 commits behind** | `bitty/docs`: `56f70bc` (2026-09-15 18:37) | **32 commits behind** |
| **`bitty-ai-docs`** | `66af2d4` (2026-09-16 16:29) | `39b4c75` (2026-09-14 20:20) | **47 commits behind** | `bitty-ai/docs`: `9610415` (2026-09-16 02:40) | **4 commits behind** |
| **`bitty-plugins-docs`** | `a751372` (2026-09-16 15:50) | `ee19a0d` (2026-09-15 00:08) | **29 commits behind** | N/A (no code mount) | N/A |

### 2.2 Critical Findings on Submodule Synchronization

1. **`bitty-docs/bitty-ai` is Severely Stale (47 Commits Behind)**:
   - The aggregator submodule in `bitty-docs` points to `39b4c75`, predating the distillation of Research 021, 025, 028, 030, 031, 032, 033, 037, 039, 040, 041, the BII delivery window addendums (`def63dc`, `7340781`), and the AIQ triage ledger (`aiq-triage-2026-09-16.md`).
   - Anyone reading `bitty-docs` locally sees an obsolete snapshot of AI architecture that still includes abandoned exploratory designs.

2. **`bitty-docs/bitty-plugins` is 29 Commits Behind**:
   - Points to `ee19a0d`, missing the entire capture of Research 039/040 (`specifications/plugin-ecosystem-model.md`), Research 041 (`specifications/plugin-ipc-boundary.md`), and the contributor docs refresh.

3. **`bitty/docs` Submodule Mount is 32 Commits Behind**:
   - The primary code repository `bitty` mounts `bitty-terminal-docs` at `docs/`, but its pointer is pinned to `56f70bc` (from 2026-09-15).
   - It lacks the Graphics and Appearance model updates (DIR-021, `82402cb`), the latest CI secret scan gates, and the architectural diagram fixes (`0b2fcfb`). Developers working inside `bitty` read outdated specifications.

---

## 3. Canonical Governance Consistency Audit

### 3.1 ADR-0003 vs. Actual Workspace Topology (16 vs. 19 Crates)

- **Path**: `bitty-docs/docs/decisions/adrs/ADR-0003-core-workspace-topology.md`
- **Discrepancy**:
  - ADR-0003 specifies a 10-crate core topology, annotated with an implementation note from 2026-08-29 (`be3bdb4`) stating the workspace resolves to 16 member crates.
  - The actual `bitty/Cargo.toml` workspace has contained **19 member crates** since release `v0.0.20` (`d9f5b49`, 2026-09-11):
    `bitty-agent`, `bitty-app`, `bitty-compat-lab`, `bitty-config`, `bitty-core`, `bitty-ipc`, `bitty-lua`, `bitty-package`, `bitty-perf`, `bitty-platform`, `bitty-plugin-host`, `bitty-pty`, `bitty-rich`, `bitty-vt`, `bitty-term-state`, `bitty-test-support`, `bitty-render`, `bitty-runtime`, `bitty-ui`.
  - The three verification and harness crates—`bitty-compat-lab`, `bitty-perf`, and `bitty-test-support`—are completely absent from ADR-0003's crate table and dependency rules.
  - ADR-0003 has never been amended to acknowledge these crates, leaving their architectural boundaries and dependency constraints formally unspecified.

### 3.2 Stale P0 Review Checklist Frozen at Phase A

- **Path**: `bitty-docs/docs/reviews/p0-review-checklist.md`
- **Discrepancy**:
  - Frontmatter and body remain frozen at: `"16 crates be3bdb4, 32 OQs Accepted, soak ~808 tests Implemented but not yet Verified"`.
  - The file calls itself the "CTX-0048 coordination single source".
  - As of 2026-09-16, the project is at 19 crates (`06bc1f4`), 40 OQs Accepted, 100 OQs registered, release `v0.0.20`.
  - This document misleads reviewers on the current baseline acceptance criteria.

### 3.3 Remaining-Draft Count Contradictions

- **Paths**:
  - `bitty-docs/README.md`: claims `"5 Draft specs remain"` (Status/Input/Text vs AI Arch plus Plugin Reuse).
  - `bitty-docs/docs/README.md`: claims `"7 Draft specs remain (see bitty-terminal/specifications/README.md prioritization)"`.
  - `bitty-terminal-docs/specifications/README.md`: contains an active table of 11 Draft specifications, plus a historical 2026-09-01 paragraph citing 7 drafts.
- **Root Cause**: Hardcoded numeric counts maintained independently across three files without a single automated single-source-of-truth generation script.

### 3.4 Decision Register (DIR) vs. Open Questions (OQ)

- **Path**: `bitty-docs/docs/decisions/index.md` and `open-questions.md`
- **Current State**:
  - `open-questions.md` registers 100 entries (OQ-001 through OQ-100).
  - `index.md` stops at DIR-023 on `main` (DIR-021: Graphics/Appearance from note 035, DIR-022: Panel Environment from note 036, DIR-023: Panel History from note 038).
  - Branch `ctx-0213/docs-research-042-043` introduces DIR-024 (Platform Compatibility) and DIR-025 (Testing Infrastructure), but remains unmerged.
  - OQs from OQ-039 through OQ-100 (including OQ-084 ontology, OQ-085 trust levels, OQ-086 password detection, OQ-087 command risk) have no corresponding DIR tracking entries.

### 3.5 Outdated Local Workspace Layout in Repository Map

- **Path**: `bitty-docs/docs/project/repository-map.md` (Lines 84–89)
- **Discrepancy**:

  ```text
  └── bitty-plugins/                  # local grouping only, never parent Git repo
      ├── activity/                   # independent repo: first official plugin
      ├── bitty-plugin-sdk/           # independent repo
      ├── bitty-plugin-template/      # independent repo
      └── <plugin-name>/              # one independent repo per plugin
  ```

  - `repository-map.md` depicts `bitty-plugins/` as a local grouping directory enclosing `activity/`, `bitty-plugin-sdk/`, and plugins.
  - In actual reality (per `$BITTY_WORKSPACE/AGENTS.md` and the filesystem), the workspace mirrors the GitHub organization flatly: every repository is a direct child of `$BITTY_WORKSPACE` (`activity/`, `palette/`, `statusline/`, `bitty-plugins/`, `bitty-plugin-sdk/`, etc.).
  - `bitty-plugins` is an independent Git repository (the registry and store frontend), NOT a parent directory.

### 3.6 Research Coverage Provenance Ledger Stopped at Record 016

- **Path**: `bitty-docs/docs/sources/research-notes-coverage.md`
- **Discrepancy**:
  - States: `"The local workspace keeps a numbered research-note series under recording/research/ (001 through 016)..."`.
  - The table only covers notes `001` through `016`.
  - Notes `017` through `043` are completely absent from the canonical provenance ledger in `bitty-docs`. Traceability for 27 research records is broken at the umbrella governance layer.

---

## 4. Remediation Checklist

1. **Submodule Pointer Realignment**:
   - Bump `bitty-docs` submodules:
     - `bitty-terminal` -> `6100ec9`
     - `bitty-ai` -> `66af2d4`
     - `bitty-plugins` -> `a751372`
   - Bump `bitty/docs` -> `6100ec9`.
   - Bump `bitty-ai/docs` -> `66af2d4`.
2. **ADR-0003 Amendment**:
   - Formally document `bitty-compat-lab`, `bitty-perf`, and `bitty-test-support` in the workspace member table and define their dependency boundaries.
3. **P0 Review Checklist & Draft Count Normalization**:
   - Update `p0-review-checklist.md` to reflect 19 crates and 40 accepted OQs.
   - Remove hardcoded draft counts from `README.md` files; point directly to the specifications index table.
4. **Repository Map Tree Fix**:
   - Update `repository-map.md` ASCII diagram to show the flat workspace layout matching `workspace.toml` and `AGENTS.md`.
5. **Research Notes Coverage Expansion**:
   - Extend `research-notes-coverage.md` to map records `017` through `043`.
