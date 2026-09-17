# Code Repository Implementation vs. Documentation Progress Audit (2026-09-16)

## 1. Executive Summary

This report performs a granular, code-level and specification-level cross-examination between the active software implementations in the Bitty workspace and their corresponding documentation repositories:

- `bitty` (Rust core workspace, 19 member crates) vs. `bitty-terminal-docs`
- `bitty-ai` (Rust AI runtime, 2 crates) vs. `bitty-ai-docs`
- `bitty-plugins` (registry), `bitty-plugin-sdk`, `bitty-plugin-template`, and independent plugins (`activity`, `palette`, `statusline`, `file-manager`, `git-panel`) vs. `bitty-plugins-docs`

The analysis classifies discrepancies into two distinct failure modes:

1. **Docs-Lag (Code Leading Docs)**: Production features, refactorings, security fixes, and architectural mutations that have landed in code but remain undocumented, missing from changelogs, or contradictory to published specifications.
2. **Phantom Features (Docs Leading Code)**: Architecture contracts, RFCs, and user guides that describe subsystems as "Accepted", "Shipped", or "Integrated", but which in actual code are merely headless data models, gated stubs, or logging-only placeholders.

---

## 2. Terminal Core (`bitty` vs. `bitty-terminal-docs`)

### 2.1 Docs-Lag: Code Leading Docs

#### 1. Workspace Growth Unrecorded in Maintenance and Release Docs

- **Code Reality**: `bitty/Cargo.toml` contains 19 member crates. `bitty-compat-lab` (terminal replay harness, CTX-0074), `bitty-perf` (PB-1~PB-7 performance budget probes, CTX-0100), and `bitty-test-support` (PTY testing matrix, CTX-0267) are fully integrated workspace members.
- **Documentation**:
  - `bitty-terminal-docs/development/release-mechanics.md` (Lines 113–135) lists only 16 crates under `Crate inventory (as of CTX-0043)`. Its Group 1–4 release ladder completely omits the three verification crates.
  - `development/maintainability.md` (Line 29) continues to assert that "the workspace has grown to 16 crates".

#### 2. `bitty/CHANGELOG.md` Frozen at CTX-0445

- **Code Reality**: Between 2026-09-15 and 2026-09-16, more than 40 remediation tasks (CTX-0458 through CTX-0489) landed on `main`. These include critical P0/P1 fixes:
  - `CTX-0462`: `fix-trust-stub-forgeable` (package trust hardening)
  - `CTX-0463`: `fix-devtools-transport-auth` (DevTools UDS peer-credential verification)
  - `CTX-0464`: `fix-lua-sandbox-caps` (Lua capability boundary verification)
  - `CTX-0468`: `fix-damage-overflow-fallback` (repairing rendering damage under-paint)
  - `CTX-0479`: `fix-config-validation-trust-xdg` (preventing un-normalized XDG trust path bypass)
- **Documentation**: `bitty/CHANGELOG.md` halts at `Host bitty.process.spawn surface (CTX-0445)`. The subsequent 40+ engineering commits are completely absent from the changelog.

#### 3. `bitty-ipc` Capabilities Far Exceed "Draft Stub" Description

- **Documentation**: Described across architecture pages as a "bounded IPC/MCP framing and stdio transport stub (draft, OQ-018)".
- **Code Reality**: `crates/bitty-ipc` has evolved into an advanced, production-grade protocol crate containing:
  - Full DevTools JSON-RPC protocol implementation (`src/devtools/automation_ops.rs`)
  - Profiling, frame-capture, and attestation modules (`src/profiling.rs`, `src/auth.rs`)
  - Snapshot streaming (`src/snapshot.rs`) and peer credential enforcement over Unix Domain Sockets.

---

### 2.2 Phantom Features: Docs Leading Code

#### 1. `PanelRuntime` Does Not Exist as a Named Type

- **Documentation**: `specifications/panel-runtime-rfc.md` (accepted 2026-09-14, CTX-0181) defines `PanelRuntime` as the core abstraction managing panel lifecycles, overlays, and presentation state.
- **Code Reality**: There is **no struct or trait named `PanelRuntime`** in the `bitty` codebase. In `bitty-ui::panel`, only raw data enums exist (`PanelId`, `PanelState`, `PanelType`). The lifecycle orchestration is handled ad-hoc by `PanelRegistry` in `crates/bitty-runtime/src/registry/panel.rs`.

#### 2. Window Compositor: Only Tiled is Live; Floating and Scratchpad are Gated

- **Documentation**: `architecture/overview.md` and `specifications/workspace-compositor.md` describe a versatile compositor supporting Tiled, Floating, Overlay, Fullscreen, and Scratchpad modes.
- **Code Reality**: In `crates/bitty-ui/src/presentation.rs` (Lines 25–32):

  ```rust
  // Only PresentationMode::Tiled is live: every View starts Tiled and the layout solver ignores the field...
  // Floating, Fullscreen, and Scratchpad parse but their transitions are gated:
  // PresentationMode::can_transition rejects every cross-mode transition for now.
  ```

  Non-tiled modes cannot be activated at runtime; transitions are hardcoded to fail-closed.

#### 3. Image Protocols: Sixel and iTerm2 are Completely Unimplemented

- **Documentation**: `architecture/core-boundaries.md` and `specifications/rich-presentation-rfc.md` claim support for Kitty graphics with Sixel fallback.
- **Code Reality**:
  - `bitty-vt` contains an APC pre-scanner for Kitty Graphics, but zero parser logic for Sixel escape sequences (`\eP...q`) or iTerm2 OSC 1337 sequences.
  - `crates/bitty-rich/src/image.rs` declares `ImageSource::Sixel` as an empty enum variant with no decoding, rasterization, or placement backend.

#### 4. Rich Presentation Scene Graph Detached from GPU Renderer

- **Documentation**: `interfaces/rich-content.md` details a `RichSurface` composed of `RichBlock` and `SceneNode` elements composited into the terminal grid.
- **Code Reality**:
  - `crates/bitty-render` depends on `bitty-term-state`, `bitty-platform`, and `bitty-config`. It **does not depend on `bitty-rich`**.
  - The GPU rendering pipeline (`wgpu`) only consumes cell snapshots (`bitty-term-state::Snapshot`) and Kitty texture IDs.
  - `RichBlock`, `SceneNode`, and projection structures in `bitty-rich` exist solely as isolated, headless in-memory models.

#### 5. Command Composer Keymap is an `eprintln` Stub

- **Documentation**: `specifications/semantic-terminal-rfc.md` states that the Command Composer (multi-line input, history, external editor dispatch via `Alt+E`) is wired into keymap dispatch.
- **Code Reality**: `crates/bitty-app/src/chrome_keys.rs` (Lines 549–560):

  ```rust
  A::OpenComposer => {
      eprintln!("bitty: keymap open_composer -> composer open requested...");
  }
  ```

  Pressing `Alt+E` outputs a log line to `stderr`; no modal UI opens, and input routing remains entirely unchanged.

---

### 2.3 Architectural Boundary Erosion in `bitty-runtime`

Research 001, 002, 009, 013, and 014 establish that the terminal core must remain a minimal microkernel, with specialized UI panels (such as AI chat, mail, file browsing) implemented strictly as external plugins consuming public host APIs.

In direct contradiction to this principle, `bitty-runtime` directly embeds:

- `crates/bitty-runtime/src/ai_panel.rs` (1,187 lines of dedicated AI panel logic)
- `crates/bitty-runtime/src/mail_panel.rs` (1,523 lines of email client state)
- `crates/bitty-runtime/src/browser_panel.rs`, `palette.rs`, `statusline.rs`
- In `crates/bitty-plugin-host/src/capability.rs`, `CapabilityFamily` explicitly defines hardcoded variants: `Agent`, `Mcp`, `Ai`.

This represents a severe architectural regression: the microkernel core is bloated with application-specific panel implementations and privilege-granting enumerations.

---

## 3. AI Subsystem (`bitty-ai` vs. `bitty-ai-docs`)

### 3.1 Actual Runtime State: Sans-I/O Deterministic Skeleton

`bitty-ai` is currently not executing real LLM inference in production code. Its workspace consists of two crates:

1. **`crates/bitty-ai-runtime`**:
   - **Nature**: A 100% in-memory, single-threaded, synchronous deterministic state machine (`AI-0009`).
   - **Zero I/O Invariant**: Contains no asynchronous runtime (`tokio`), no network libraries (`reqwest`), no filesystem operations, and no wall-clock access (all timestamps `now_ms` are passed as function arguments).
   - **Model Provider**: The only production provider is `FakeProvider` (`src/provider.rs:360`), which replays hardcoded, scripted `ProviderTurn` vectors.
   - **Stubs**: Network calls, secret storage, MCP transport, multi-agent teams, and LSP integration are explicitly stubbed.

2. **`crates/bitty-ai-slice`**:
   - Serves as an experimental integration test harness (`BA-6`, `CTX-0407`).
   - Contains `local_provider.rs` (`AI-0042`), which uses `std::net::TcpStream` to issue plaintext HTTP requests strictly to loopback addresses (`127.0.0.1`, `localhost`), validating that the `ModelProvider` trait can drive an HTTP lifecycle. However, this is confined to the test slice and excluded from `bitty-ai-runtime`.
   - `LiveBittyHost` (`src/live_host.rs`) executes in-process conversions of `bitty-ipc` structures via function pointers; no actual IPC socket is connected.

---

### 3.2 Recent Commits Audit

1. **Sampling Contract & ID Collision Fix (`harness.rs`, AI-0084)**:
   - Fixed a P2 defect where re-sampling within the same generation produced duplicate `ContextRecord` IDs (`format!("terminal-{generation}")`).
   - Updated format to `format!("terminal-{owner}-{generation}")` with fail-closed uniqueness validation.
2. **Cost Router Bounded Accounting (`accounting_bounds.rs`, AI-0083)**:
   - Resolved conflicting zero-weight semantics (router treating weight 0 as 0 vs. agent treating it as 1). Confirmed saturating arithmetic across `ADVERSARIAL_TOKENS` (65,535) and `u32::MAX`.
3. **Prefix-Cache Provider-Scoped Key (`cache_key.rs`, AI-0082)**:
   - Implemented `CacheKey` binding `(provider_id, model_id, scope, stable_prefix_hash, prefix_len)` using FNV-1a-64 hashing on prompt prefixes.
   - **Vulnerability Identified**: `cache_key.rs` (Lines 175–182) scans for the raw byte slice `b"[layer:runtime-turn len="`. If user input or project source code contains this exact string, prefix length computation is prematurely truncated, corrupting cache key generation.
4. **Extension-Point Seam Traits (`extension.rs`, AI-0079)**:
   - Codified 8 discrete extension points (`ModelPoint`, `ToolPoint`, `ContextPoint`, `MemoryPoint`, `CompactorPoint`, `AgentPoint`, `CommandPoint`, `UiPoint`) with `compile_fail` tests enforcing capability isolation.

---

### 3.3 Gaps Between `bitty-ai-docs` Specifications and Code

| Specification Domain | Specification Contract | Actual Code State |
| :--- | :--- | :--- |
| **Agent Coordination** (`agent-coordination.md`) | 4-tier tree (Mission / Org / Team / Agent); shared language server leases; OS group isolation. | Only single-agent synchronous loop in `agent.rs`. Multi-agent coordination is 100% stubbed. |
| **Context Management** (`context-management.md`) | 5-stage pipeline (L0–L4); session fact projection; semantic retrieval and global compaction. | Only L0 (structured output) and L1 (lossless pruning) implemented. L2 is an unintegrated prototype (`compression.rs`). Zero retrieval logic. |
| **Prompt Layering** (`prompt-layering-design.md`) | 5 layers assembled from `.bitty/` filesystem manifests and dynamic skill loading. | In-memory assembly only. No filesystem manifest reader exists in the core runtime. |
| **Prefix Cache** (`prefix-cache-context-design.md`) | Token-boundary alignment; vendor-specific KV-cache optimization headers. | In-memory byte hashing only (`CacheKey`). Emits no vendor-specific caching headers. |
| **Storage & Persistence** (`storage-memory-export-design.md`) | Session-sharded SQLite (`catalog.db` + per-session databases); 4-tier memory hierarchy. | `bitty-ai-runtime` has zero persistence. Prototype exists only in `bitty-ai-slice/src/journal_prototype.rs`. |

---

### 3.4 BII Delivery Window Addendums Audit (`def63dc`, `7340781`)

`bitty-ai-docs/specifications/bitty-side-delivery-verification.md` tracks host capability delivery across two commit windows in `bitty`:

- **Window 1 (`2cbb1fb..db283e6`)**: Identified deprecation of `agent.context.terminal` in favor of `terminal.snapshot` (`ai_panel.rs`), wire version negotiation, and `ContentTrust` tagging.
- **Window 2 (`db283e6..cfeffa2`)**: Audited PTY drop timeouts and performance baseline adjustments.
- **Critical Finding**: Inspection of `crates/bitty-ipc` proved that IPC wire structs remained **100% byte-identical** across these windows. The terminal host still lacks actual PTY process wiring to the agent, live publishing endpoints, and rich transport methods.

---

## 4. Plugin Ecosystem & Tooling

### 4.1 Bundled Plugin Split Desync (`bitty` #725 vs. `bundled-plugin-split-decision.md`)

- **Code Reality**: In `bitty` commit `65aac5c` (`[CTX-0399] feat(plugin): remove the bundled file-manager from the catalog (#725)`) and CTX-0400, both `file-manager` and `git-panel` were **completely removed from the bundled catalog**.
  - `bitty/CHANGELOG.md` notes that the bundled catalog is reduced to 6 plugins (`ai-panel`, `browser-panel`, `mail-panel`, `project`, `shell-integration`, `workspace`).
  - `crates/bitty-plugin-host/src/bundled.rs` enforces this 6-plugin catalog.
  - Official registry entries `file-manager.toml` and `git-panel.toml` are published in `bitty-plugins/registry/official/`.
- **Documentation**:
  - `bitty-plugins-docs/product/bundled-plugin-split-decision.md` (Lines 109–110) still records both plugins as:
    `Verdict: Split later | Gate: Panel provider contract`.
  - The implementation status section only reports on `palette` and `statusline`.
  - `bitty-plugins-docs/docs/plugins/` has no documentation directories for `file-manager` or `git-panel`.
- **Architectural Impact**: The code repository bypassed its own architectural gate (splitting them as observation-only Lua packages before the panel provider contract was finalized), but the governance documentation was never updated.

---

### 4.2 Status of 2026-09-15 Code Review Findings

| Component | Defect from 2026-09-15 Review | Current Status | Technical Evaluation |
| :--- | :--- | :--- | :--- |
| **`bitty-plugin-template`** | `init.lua` command registration used `name` instead of `id`/`title`, generating unrunnable code (P0). | **Fixed** | `init.lua` correctly declares `id = "hello"` and `title = "@@PLUGIN_NAME@@: hello"`. |
| **`bitty-plugin-sdk`** | Allowed root keys lacked `tools`, causing authoritative SDK to reject `git-panel` (R4). | **Fixed** | Added `"tools"` to `ALLOWED_ROOT_KEYS`; added `validateTools` for `[tools.git]`. |
| **`bitty-plugin-sdk`** | Manifest filesystem patterns lacked path traversal and sensitive directory checks (P1). | **Fixed** | `src/path-pattern.ts` enforces blocking of `..`, absolute paths, `.ssh`, `.gnupg`, `.aws`. |
| **`git-panel`** | Denylist omitted `--output`, `--index-file`, `-o` write flags (R5). | **Fixed** | Rewritten with strict flag allowlist (`ALLOWED_FLAGS`) and unknown flag rejection. |
| **`bitty-plugins`** | Registry metadata sync lacked entry ID verification and body size limits (R2/R3). | **Fixed** | Added `MANIFEST_MAX_BYTES = 256KiB` and `expectedId` matching in `sync-metadata.ts`. |
| **`bitty-package`** | V-C signature verification is a deterministic SHA-256 string-hash stub (P0 / SEC). | **UNFIXED** | `crates/bitty-package/src/trust.rs` (Lines 280–348) computes `expected_sig = SHA256(key_id \|\| manifest \|\| artifact)`. Anyone with the public `key_id` can forge valid signatures offline. |
| **`bitty-package`** | `check_compatibility` only checks for `.` without SemVer evaluation (P1). | **UNFIXED** | `crates/bitty-package/src/integrity.rs` (Lines 348–363) only checks `!v.contains('.')`. Version ranges in `compat.bitty` are ignored. |
| **`bitty-plugin-host`** | `FilesystemRequest::validate` only checks length and whitespace (P1). | **UNFIXED** | Unlike the TypeScript SDK, the Rust core does not block `..`, `/etc/`, or `~/.ssh/`. |
| **`bitty-plugin-host`** | `tools.rs` has argument bypasses for `-cfoo=bar` and `-C` (P1). | **UNFIXED** | Argument checks in `tools.rs` only match exact `"-c"`; `-C` is blocked only on `branch`, not on `diff` or `log`. |
| **`bitty-lua`** | `drive_chunk` has no byte length limit; compilation excluded from gas fuel (P1). | **UNFIXED** | Huge Lua scripts can stall host threads during parsing before gas accounting begins. |
| **`bitty-rich`** | Background image loading has TOCTOU file reopening (P1). | **UNFIXED** | File metadata check and `File::open` are separate operations; cache keys can be desynchronized. |

---

## 5. Remediation Priority Matrix

1. **P0 / Security**:
   - Replace `bitty-package` SHA-256 signature stub with real asymmetric cryptography (Ed25519/Minisign), or downgrade specification claims from "verified" to "mock-only".
   - Port SDK path traversal and sensitive directory protections into `bitty-plugin-host/src/manifest.rs`.
2. **P1 / Documentation Integrity**:
   - Update `bitty-plugins-docs/product/bundled-plugin-split-decision.md` to record the merged split of `file-manager` and `git-panel`.
   - Update `bitty/CHANGELOG.md` with all merged commits from CTX-0458 through CTX-0489.
   - Annotate `PanelRuntime`, `PresentationMode`, and `CommandComposer` in `bitty-terminal-docs` to explicitly state their current gated/stubbed status.
3. **P1 / Architecture Decoupling**:
   - Schedule extraction of `ai_panel.rs` and `mail_panel.rs` out of `bitty-runtime` into independent plugins.
