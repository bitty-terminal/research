# Architectural Drift and Subjective Engineering Analysis (2026-09-16)

## 1. Executive Perspective

The Bitty terminal project represents an ambitious architectural experiment: building a GPU-accelerated terminal emulator with Hyprland-style compositing panels, Piccolo-based Lua plugin sandboxing, and a native AI runtime, all governed under a strict "documentation-first" paradigm.

However, an unvarnished technical inspection reveals deep architectural tensions between declared design dogma and engineering reality. This report provides a subjective, critical engineering evaluation of why the system has drifted, where the fundamental bottlenecks lie, and how the project owner should recalibrate development.

---

## 2. The "Documentation-First" Dogma vs. "19-Crate Implementation" Paradox

### 2.1 The Tension

The workspace governance rules (`AGENTS.md`) state emphatically:
> *"Documentation-first and pre-implementation: no product code until a scoped task authorizes it and design/security gates pass... Never describe a chat suggestion, empty repository, configuration file, or planned interface as implemented."*

Yet, in empirical reality:

- The `bitty` repository contains **19 member crates** and thousands of lines of sophisticated Rust code.
- It has tagged release `v0.0.20`, implements real wgpu GPU pipelines, Kitty graphics chunked decoders, Piccolo Lua VMs, ConPTY integration, and hundreds of unit and integration tests.
- Over 40 production fixes landed in code between 2026-09-15 and 2026-09-16 alone.

### 2.2 The Resulting Pathology: "Phantom Features" and "Unacknowledged Code"

Because governance dogma insists that the project is "pre-implementation" or in "early hardening", an institutional cognitive dissonance has emerged:

1. **Phantom Specifications**: Architectural documentation writes specifications for idealized, elaborate abstractions—such as `PanelRuntime`, a retained `RichBlock` scene graph compositing directly to GPU quads, multi-agent hierarchical organization trees (Mission/Org/Team/Agent), and Sixel image decoders. The documentation treats these as "Accepted" or "Specified", giving downstream developers the illusion that these foundations exist. In reality, `bitty-render` does not even depend on `bitty-rich`, `PanelRuntime` does not exist as a type, and non-tiled window modes are hardcoded to fail-closed.
2. **Unacknowledged Code Drift**: Conversely, when engineers urgently need features to make the terminal usable, they bypass the documentation process. `bitty-ipc` grew into a complex JSON-RPC automation server, `file-manager` and `git-panel` were stripped from the bundled catalog in commit `65aac5c` without updating `bundled-plugin-split-decision.md`, and 40+ bug fixes landed without being recorded in `CHANGELOG.md`.

**Subjective Verdict**: The "documentation-first" rule has inverted into its opposite. Instead of documentation guiding code, documentation has become an isolated philosophical exercise that lags weeks behind actual commit reality while describing phantom features that code has bypassed.

---

## 3. The Git Submodule Synchronization Trap

### 3.1 The Architecture

Documentation is fractured into four repositories (`bitty-docs`, `bitty-terminal-docs`, `bitty-ai-docs`, `bitty-plugins-docs`), with `bitty-docs` mounting the other three as Git submodules, and code repositories (`bitty`, `bitty-ai`) mounting their corresponding documentation repositories at `<repo>/docs`.

### 3.2 Why It Failed Operationally

- **Pointer Paralysis**: Submodules in Git do not automatically track branch heads. In an active, multi-agent, parallel development workflow, developers commit to `bitty-ai-docs` or `bitty-terminal-docs` and rarely remember to push companion commits bumping the parent submodules.
- **The Empirical Gap**:
  - `bitty-docs/bitty-ai` is **47 commits behind** `bitty-ai-docs`.
  - `bitty-docs/bitty-plugins` is **29 commits behind** `bitty-plugins-docs`.
  - `bitty/docs` is **32 commits behind** `bitty-terminal-docs`.
- **Impact on AI Pair Programming and Engineering**: An agent or developer opening `bitty/docs` reads a specification that was superseded 32 commits ago. An agent opening `bitty-docs` reads AI architecture drafts that were discarded two weeks ago.

**Subjective Verdict**: Git submodules for documentation across an org without an automated, lockstep CI bump bot create catastrophic friction. If the polyrepo structure is retained, a GitHub Actions workflow must automatically bump aggregator submodules on every merged PR. Otherwise, documentation should be unified or fetched via sparse checkouts.

---

## 4. Microkernel Erosion: The Bloat of `bitty-runtime`

### 4.1 The Violation

Research 001, 009, 013, and 014 establish a fundamental architectural principle:
> *"Mechanisms in Core, policy and UX in Plugins. The terminal core is a microkernel owning PTY, grid, rendering, and input; all application experiences live in plugins."*

In reality, `crates/bitty-runtime` contains:

- `src/ai_panel.rs` (1,187 lines)
- `src/mail_panel.rs` (1,523 lines)
- `src/browser_panel.rs` (500+ lines)
- `src/palette.rs`, `src/statusline.rs`
- In `crates/bitty-plugin-host/src/capability.rs`, `CapabilityFamily` directly defines `Agent`, `Mcp`, `Ai`.

### 4.2 Why Did This Happen?

Developing an in-tree Rust struct that directly mutates `TerminalRegistry` and accesses internal event queues is orders of magnitude easier than designing a clean, asynchronous IPC or Lua plugin capability protocol. When faced with the friction of building a true general-purpose plugin host, development took the shortcut of hardcoding first-party application panels straight into the runtime.

### 4.3 The Architectural Hazard

By embedding `ai_panel.rs` and `mail_panel.rs` in `bitty-runtime`:

1. Bitty ceases to be a lightweight terminal emulator; it becomes an ad-hoc, monolithic IDE.
2. First-party panels enjoy private internal privileges that third-party plugins cannot access, destroying the foundational promise that "official and community plugins consume the exact same API".
3. Memory and crash footprints of experimental features (like email parsing or AI token buffering) threaten the stability of the terminal core.

**Subjective Verdict**: Extracting `ai_panel.rs` and `mail_panel.rs` out of `bitty-runtime` must be prioritized. If the plugin host is not yet expressive enough to support them externally, that is proof that the plugin host is deficient—not justification for polluting the core.

---

## 5. Security Theater: The Plugin Trust & Supply Chain Illusion

### 5.1 The Claims

The Bitty documentation extensively advertises a military-grade security architecture: capability manifests, restricted Lua stdlib via Piccolo, fuel-based CPU limits, memory quotas, and cryptographic package signing (V-C signature verification).

### 5.2 The Underlying Reality

1. **Forgeable Signatures (P0 / SEC)**:
   In `crates/bitty-package/src/trust.rs` (Lines 280–348), signature verification is implemented as:

   ```rust
   let expected_sig = compute_mock_hash(&key_id, &manifest_bytes, &artifact_bytes);
   ```

   It is a deterministic SHA-256 string concatenation. There is zero asymmetric public-key cryptography (no Ed25519, no RSA, no Minisign). Anyone who knows the `key_id` string can forge a valid signature offline.
2. **Registry Without Integrity**:
   `bitty-plugins` registry schema includes `signature` and `manifest_hash`, but they are marked advisory. The client library hardcodes `signature_status: "unsigned" | "unverified"`.
3. **Core Lacks SDK Protections**:
   While the TypeScript SDK was patched to reject path traversal (`..`) and sensitive directories (`~/.ssh`, `~/.aws`), `bitty-plugin-host/src/manifest.rs` in Rust only checks string length and whitespace. A plugin bypassing the SDK can declare `/etc/shadow` access without core rejection.
4. **Fail-Open Event Interception**:
   In `bitty-plugin-host/src/event.rs`, if an event hook times out, `should_proceed` returns `true`. A malicious plugin can bypass interception gates simply by hanging the execution thread.

**Subjective Verdict**: Declaring a sandbox secure when signature verification is a mock hash and path checking is missing from the Rust kernel is dangerous security theater. The project must either implement real Ed25519 verification immediately or demote documentation claims to explicitly state: *"Package signing is currently an unverified mock protocol."*

---

## 6. The AI Subsystem: Theoretical Elegance vs. Operational Paralysis

### 6.1 The Beauty of the Sans-I/O Architecture

`bitty-ai-runtime` was designed as a pure, synchronous, deterministic state machine (`AI-0009`). It performs no I/O, spawns no threads, uses no async runtimes, and takes `now_ms` as an input parameter. Every state transition can be serialized, replayed, and verified deterministically. From an academic software engineering standpoint, it is a textbook design.

### 6.2 The Operational Dead End

In practice, a terminal AI assistant is inherently an I/O-bound, asynchronous, real-time system:

- It must stream tokens over HTTP/SSE from remote LLM APIs.
- It must observe terminal output asynchronously.
- It must asynchronously execute bash commands and LSP queries.

By banning all I/O from `bitty-ai-runtime`, all complexity was shoved into an unwritten "bridge layer". The only working provider in the repository is `FakeProvider`, replaying hardcoded vectors. Even `bitty-ai-slice`'s `LiveBittyHost` is just an in-memory function-pointer mock.

Furthermore, Research 023 made a sound tactical decision to freeze multi-agent orchestration and focus on a single-agent loop. But the implementation went too far: it stopped at an in-memory toy that cannot even execute an Ollama call from the terminal.

Additionally, the newly merged `CacheKey` (AI-0082) contains a critical structural bug: it searches for the raw ASCII token `b"[layer:runtime-turn len="` to find the prefix boundary. Any user prompt containing this token will corrupt cache key computation.

**Subjective Verdict**: The Sans-I/O boundary is valuable for the state machine core, but `bitty-ai` needs an official, asynchronous driver layer (`bitty-ai-driver` or `bitty-ai-host`). Continuing to refine in-memory accounting and extension traits while the system cannot talk to a real LLM endpoint is diminishing-returns engineering.

---

## 7. Concrete Action Plan for Project Recalibration

To restore alignment between documentation and code, the project owner should execute the following five-step corrective sequence:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                       RECALIBRATION ACTION PLAN                             │
├─────────────────────────────────────────────────────────────────────────────┤
│ 1. REALIGN SUBMODULES & FACT MATRICES (Immediate, S-Effort)                 │
│    - Bump aggregator and code mount submodules to current HEADs.            │
│    - Amend ADR-0003 and release-mechanics to formally document 19 crates.  │
│    - Backfill 40+ merged commits into bitty/CHANGELOG.md.                   │
├─────────────────────────────────────────────────────────────────────────────┤
│ 2. RECONCILE BUNDLED PLUGIN SPLIT (Immediate, S-Effort)                     │
│    - Update bundled-plugin-split-decision.md to reflect file-manager        │
│      and git-panel removal (catalog reduced to 6).                          │
│    - Create documentation stubs under bitty-plugins-docs/docs/plugins/.     │
├─────────────────────────────────────────────────────────────────────────────┤
│ 3. ELIMINATE SECURITY THEATER (High Priority, M-Effort)                     │
│    - Replace bitty-package SHA-256 stub with ring/ed25519-dalek.            │
│    - Port SDK path traversal & sensitive directory checks to Rust kernel.   │
│    - Invert event timeout failure semantics from fail-open to fail-closed.  │
├─────────────────────────────────────────────────────────────────────────────┤
│ 4. DEMOTE PHANTOM FEATURES IN DOCUMENTATION (Medium Priority, S-Effort)     │
│    - Label Floating/Scratchpad as "Gated Prototype" in compositor specs.    │
│    - Label Command Composer as "Headless Prototype" in semantic specs.      │
│    - Annotate Sixel as "Unimplemented / Deferred" in appearance specs.      │
├─────────────────────────────────────────────────────────────────────────────┤
│ 5. EXTRACT EMBEDDED PANELS FROM BITTY-RUNTIME (Strategic, L-Effort)         │
│    - Create dedicated tasks to extract ai_panel.rs and mail_panel.rs.       │
│    - Force them to communicate strictly via public plugin IPC / Lua APIs.   │
└─────────────────────────────────────────────────────────────────────────────┘
```
