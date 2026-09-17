# Bitty Whole-Workspace Architecture, Code, and Documentation Review (2026-09-16)

## 1. Executive Summary

On 2026-09-16, a comprehensive, multi-agent architectural review was conducted across the entire Bitty workspace:

- **Canonical Governance & Aggregator**: `bitty-docs` and its three Git submodules (`bitty-terminal`, `bitty-ai`, `bitty-plugins`).
- **Code Repositories**: `bitty` (19 crates), `bitty-ai` (2 crates), `bitty-plugins` (registry), `bitty-plugin-sdk`, `bitty-plugin-template`, and independent plugins (`activity`, `palette`, `statusline`, `file-manager`, `git-panel`).
- **Split Documentation Repositories**: `bitty-terminal-docs`, `bitty-ai-docs`, `bitty-plugins-docs`.
- **Research Archive**: All 43 discussion records and summaries in `research/origin/` and `research/summary/` (001 through 043).

This review moves beyond mechanical checklist audits to provide an in-depth, subjective engineering assessment of the drift between documentation specifications, research consensus, and actual production code.

---

## 2. File Index

| Report File | Focus Area | Key Findings & Evidence |
| :--- | :--- | :--- |
| **`01-docs-hierarchy-and-submodule-sync.md`** | Documentation Hierarchy & Submodule Pointers | Three-way submodule drift: `bitty-docs/bitty-ai` is **47 commits behind**; `bitty-docs/bitty-plugins` is **29 commits behind**; `bitty/docs` is **32 commits behind**. Governance stale: ADR-0003 lists 16 crates (actual: 19 crates); `p0-review-checklist.md` frozen at Phase A (2026-08-29, 32 OQs); `research-notes-coverage.md` stopped at record 016 (missing 017–043). |
| **`02-code-vs-docs-progress-audit.md`** | Code Implementation vs. Documentation Progress | **Docs-Lag**: 40+ production commits (CTX-0458~0489) missing from `bitty/CHANGELOG.md`; `file-manager` and `git-panel` removed from bundled catalog (`bitty` #725) while docs say "Split later". **Phantom Features**: `PanelRuntime` does not exist as a type; Floating/Scratchpad window modes are hardcoded to reject transitions; Sixel/iTerm2 graphics are zero-code; Command Composer is an `eprintln` stub; `RichBlock` scene graph is detached from `bitty-render` GPU pipeline. |
| **`03-research-records-traceability-audit.md`** | Research Records (001–043) Traceability Matrix | Complete ledger of all 43 research records. Only 39.5% are fully captured. **Premature capture**: Records 042 and 043 marked `.completed` in research, but PR `bitty-docs#315` is unmerged and files do not exist on `main`. **Omissions**: Record 032 (Model Provider plugins) and 038 (Pane History plugins) are completely absent from `bitty-plugins-docs`. **Dilutions**: Record 039 `ActivityStack` missing from `panel-runtime-rfc.md`; Record 008 Composer distorted into a VDOM diff engine. |
| **`04-architectural-drift-and-subjective-analysis.md`** | Architectural Drift & Subjective Engineering Analysis | Deep critical evaluation: (1) The "Documentation-First" paradox vs. 19-crate reality; (2) Git submodule operational failure; (3) Microkernel erosion via `ai_panel.rs` (1,187 lines) and `mail_panel.rs` (1,523 lines) embedded in `bitty-runtime`; (4) Security theater: `bitty-package` signature verification is a forgeable SHA-256 string stub; (5) `bitty-ai-runtime` Sans-I/O disconnect from real interactive terminal AI. |

---

## 3. Top Cross-Cutting Architectural Risks

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                       CROSS-CUTTING RISK MATRIX                             │
├──────────┬─────────────────────────────────────────────────┬────────────────┤
│ Severity │ Finding                                         │ Primary Owner  │
├──────────┼─────────────────────────────────────────────────┼────────────────┤
│ CRITICAL │ V-C Package Signature is a Forgeable SHA-256    │ bitty-package  │
│ (P0/SEC) │ String Stub (crates/bitty-package/trust.rs)     │                │
├──────────┼─────────────────────────────────────────────────┼────────────────┤
│ CRITICAL │ Submodule Pointers Severely Desynchronized      │ bitty-docs     │
│ (P0/DOC) │ (47 commits lag in AI, 32 commits in bitty)     │ bitty          │
├──────────┼─────────────────────────────────────────────────┼────────────────┤
│ HIGH     │ Core Microkernel Erosion: bitty-runtime Embeds  │ bitty-runtime  │
│ (P1/ARC) │ ai_panel.rs (1187L) and mail_panel.rs (1523L)   │                │
├──────────┼─────────────────────────────────────────────────┼────────────────┤
│ HIGH     │ Phantom Specifications vs. Gated Code Stubs     │ bitty-terminal │
│ (P1/ENG) │ (Floating modes, Composer, RichSurface, Sixel)  │ -docs          │
├──────────┼─────────────────────────────────────────────────┼────────────────┤
│ HIGH     │ Bundled Split Desync: file-manager & git-panel  │ bitty-plugins  │
│ (P1/DOC) │ Removed from Catalog, Docs Say "Split Later"    │ -docs          │
├──────────┼─────────────────────────────────────────────────┼────────────────┤
│ HIGH     │ Research 042/043 Premature Completion on        │ research       │
│ (P1/TRC) │ Unmerged PR bitty-docs#315                      │ bitty-docs     │
└──────────┴─────────────────────────────────────────────────┴────────────────┘
```

### Risk 1: The Cryptographic Trust Vacuum (P0 / SEC)

The Bitty package manager (`bitty-package`) advertises a verified package integrity chain. In reality, signature checking is a deterministic hash of the form `SHA256(key_id || manifest || artifact)`. Any actor who knows the public key ID string can generate valid signatures offline. Coupled with the lack of path traversal checks in `bitty-plugin-host`, this exposes the terminal to arbitrary untrusted code execution under the guise of verified plugins.

### Risk 2: Severe Submodule Pointer Drift (P0 / DOC)

The aggregator `bitty-docs` and code repository `bitty` point to Git submodule commits that are 47, 29, and 32 commits behind their respective documentation HEADs. Developers and automated agent harnesses reading through submodules consume obsolete, contradictory specifications.

### Risk 3: Microkernel Erosion in `bitty-runtime` (P1 / ARC)

Despite unanimous consensus across research records 001, 009, 013, and 014 that application experiences belong in external plugins, `bitty-runtime` directly compiles in monolithic email and AI chat panels (`mail_panel.rs`, `ai_panel.rs`). This grants first-party features private access to internal runtime state, violating the principle of equal capability boundaries for all plugins.

### Risk 4: Phantom Specifications (P1 / ENG)

Downstream teams (such as plugin and AI developers) are building designs on the assumption that `PanelRuntime`, floating window compositing, Sixel decoding, and Command Composer are operational. In code, non-tiled window modes are hardcoded to fail-closed, Sixel is an empty enum, `bitty-render` does not link against `bitty-rich`, and Command Composer is an `eprintln` logging statement.

---

## 4. Priority Action Roadmap for Project Recalibration

### Phase 1: Immediate Fact Reconciliation (1–2 Days, S-Effort)

1. **Sync Submodules**: Execute `git submodule update --remote` in `bitty-docs`, `bitty`, and `bitty-ai`, aligning all pointers with standalone `main` branches.
2. **Reconcile Plugin Catalog**: Update `bitty-plugins-docs/product/bundled-plugin-split-decision.md` to formally mark `file-manager` and `git-panel` as split and removed from the catalog.
3. **Update Governance Records**:
   - Amend ADR-0003 and `release-mechanics.md` to document all 19 workspace crates.
   - Backfill 40+ merged commits (CTX-0458~0489) into `bitty/CHANGELOG.md`.
   - Update `p0-review-checklist.md` to reflect 19 crates and 40 accepted OQs.
4. **Fix Research Traceability**: Either merge PR `bitty-docs#315` or revert `origin/042.md` and `043.md` completion suffixes until merged.

### Phase 2: Security & Boundary Hardening (1 Week, M-Effort)

1. **Cryptographic Signatures**: Replace the mock hash in `bitty-package/src/trust.rs` with real Ed25519 signature verification (`ed25519-dalek`), or officially downgrade documentation claims to "mock signing".
2. **Kernel Path Containment**: Port `bitty-plugin-sdk/src/path-pattern.ts` traversal and sensitive directory checks into `bitty-plugin-host/src/manifest.rs`.
3. **Fail-Closed Event Interception**: Fix `bitty-plugin-host/src/event.rs` so that timed-out interceptors fail closed (`return false`).
4. **Fix Prefix-Cache Collision**: Update `bitty-ai-runtime/src/cache_key.rs` to take structured prefix offsets instead of scanning for the raw string `b"[layer:runtime-turn len="`.

### Phase 3: Structural Realignment & Extraction (2–3 Weeks, L-Effort)

1. **Extract In-Tree Panels**: Create formal CarryCtx tasks to extract `ai_panel.rs` and `mail_panel.rs` from `bitty-runtime` into external plugins.
2. **Honesty Gating for Phantom Features**: Add explicit status disclaimers in `workspace-compositor.md` (Floating is Gated), `semantic-terminal-rfc.md` (Composer is Headless Stub), and `rich-presentation-rfc.md` (Sixel is Unimplemented).
3. **Absorb Research 039 into Terminal RFC**: Revise `panel-runtime-rfc.md` to incorporate `ActivityStack` and decouple activity session lifetimes from panel visual containers.
