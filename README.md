# Bitty Research Archive

Discussion records and code reviews behind Bitty's design. Development tasks are
planned from the `*-docs` repos (`bitty-docs`, `bitty-terminal-docs`,
`bitty-ai-docs`, `bitty-plugins-docs`); this archive keeps the raw reasoning
that led there.

## Rules

- **English only** — everything in this repo except `origin/` is written in
  English. `origin/` holds immutable original discussion records (any language);
  never edit them.
- **No absolute paths** — never commit host-specific paths (`/home/…`,
  `/mnt/…`, usernames, machine layout). Use repo-relative paths.
- **Minimal toolchain** — markdownlint only, no CI/CD (see
  `.markdownlint-cli2.jsonc`).

## Layout

| Path        | Content                                                                       |
| ----------- | ----------------------------------------------------------------------------- |
| `origin/`   | Full original records, numbered `NNN.md`. Immutable.                          |
| `summary/`  | Trimmed English summaries, `NNN.md` mirrors `origin/NNN.md`.                  |
| `review/`   | Code-review campaigns, one directory per date, each with its own `README.md`. |
| `coverage/` | Record-to-document coverage index, one file per docs corpus.                  |

## Status workflow

`origin/NNN.md` + `.completed` suffix = conclusions have been recorded in the
`*-docs` repos (done). No suffix = open, still to be captured. A record whose
capture is split across owners is marked `Partial` in the table and its origin
file stays unrenamed until every owned part is captured.

To add new research:

1. Drop the full record at `origin/NNN.md` (next free number).
2. Add a trimmed English summary at `summary/NNN.md` (same 7-field format:
   Status / Date / One-liner / Background / Key conclusions / Destination /
   Open items).
3. Add one row to the index table below.
4. When its conclusions land in `*-docs`: rename to `origin/NNN.md.completed`,
   flip the summary's Status to Captured, and update the row below.

## Index — discussion records

| NNN | Status   | One-liner                                                                                |
| --- | -------- | ---------------------------------------------------------------------------------------- |
| 001 | Captured | Mechanisms in Core, policy and UX in Plugins                                             |
| 002 | Captured | Panels as app containers, IPC as extension protocol                                      |
| 003 | Captured | Minimal core plus infinite composition for terminals                                     |
| 004 | Captured | Warp sessions plus Hermes growth runtime for agents                                      |
| 005 | Captured | Five terminal routes; Bitty aims programmable shell                                      |
| 006 | Captured | Prove terminal foundation before plugin ecosystem                                        |
| 007 | Captured | Code outran docs; global alignment needed                                                |
| 008 | Captured | Commands as objects; keyboard addresses everything                                       |
| 009 | Captured | Layered complexity defines composable workspace identity                                 |
| 010 | Captured | Spatial multi-agent with minimal-sufficient context                                      |
| 011 | Captured | Native agent services with Lua policy, Rust enforcement                                  |
| 012 | Captured | Batch tool calls, parallel rounds, reuse LSP                                             |
| 013 | Captured | Programmable terminal workspace panorama with multi-agent support                        |
| 014 | Captured | Calibrate panorama, freeze core ontology and invariants                                  |
| 015 | Captured | Echo-off password detection with graded agent safety                                     |
| 016 | Captured | Alt modifier with spatial badges for one-handed actions                                  |
| 017 | Captured | Harden terminal, validate workspace, tighten invariants                                  |
| 018 | Captured | Standalone AI runtime owning context, permissions, events                                |
| 019 | Captured | Submodule composition with isolated ownership, integrated workspace                      |
| 020 | Captured | Pixel mascot with modern graphics, per-panel scaling                                     |
| 021 | Captured | Shared toolchain, organized collaboration, compilable context                            |
| 022 | Captured | Mechanisms in core, workflows in plugins, compressed views                               |
| 023 | Captured | Docs are good enough; move into runtime implementation                                   |
| 024 | Captured | Stop design inflation; reconcile, extract, harden                                        |
| 025 | Captured | Stable content first, appends last, hit rate first                                       |
| 026 | Captured | Built-ins teach semantics, runtime owns capabilities                                     |
| 027 | Captured | Core around a thousand words, capabilities on demand                                     |
| 028 | Captured | Remove empty shells, deduplicate, fix identity ownership                                 |
| 029 | Captured | Core stays offline, extensions cross controlled boundaries                               |
| 030 | Captured | Keep kernel minimal, third parties in adapters                                           |
| 031 | Captured | Terminal snapshot and tool gateway block AI integration                                  |
| 032 | Captured | Core defines models, plugins handle vendors and subscriptions                            |
| 033 | Captured | Unify multimodal lifecycles, keep per-modality request shapes                            |
| 034 | Captured | Fix existing packaging chain before adding distribution formats                          |
| 035 | Captured | Separate image protocols from appearance and compositing system                          |
| 036 | Captured | Inherit pane environment via prompt sync, masked for agents                              |
| 037 | Captured | Split config/data/state/cache, shard storage per session                                 |
| 038 | Captured | Core emits events, history plugins reuse external shell history                          |
| 039 | Captured | Panel is a workspace container; activity keeps sessions alive                            |
| 040 | Captured | Plugins form dependency, service, and extension graphs                                   |
| 041 | Captured | IPC is a second plugin boundary sharing one capability model                             |
| 042 | Captured | Separate dependency kinds; firewall Core; own release compatibility                      |
| 043 | Captured | Native, VM, and physical test tiers with QEMU/libvirt VMs                                |
| 044 | Captured | Job/Execution first-class under a supervisor; bitty mechanisms captured, ai pending      |
| 045 | Captured | Lua composes workflows; Core enforces capability, resource, lease, and secret boundaries |
| 046 | Captured | Wheel scoped to Coding Agent Harness; Personal/Office agents are separate future plugins |
| 047 | Captured | Caller attribution for provider calls: OpenRouter Apps mechanism + TurnRequest identity  |
| 048 | Captured | Agent quality as Model x Harness x Context x Tools x Verification; bitty-ai distilled    |
| 049 | Captured | Context Compiler between Agent Runtime and provider over Cold/Warm/Hot layers            |
| 050 | Captured | Portable capabilities in `.agents`, harness behavior in `.wheel`; bitty-ai distilled     |
| 051 | Captured | Git-inspired context DAG with checkpoints, branches, merges, GC; bitty-ai distilled      |
| 052 | Captured | Minimal Wheel kernel with Lua-plugin composition; bitty-ai distilled                     |
| 053 | Captured | [Plugin contracts and Lua frameworks](summary/053.md)                                    |
| 054 | Captured | Bitty owns plugin lifecycle; Lux serves only the Lua dependency layer at build time      |
| 055 | Captured | Wheel as Event-Sourced Agent Workspace: headless agents, event log, versioned context    |
| 056 | Captured | Split stored history from compiled context; commit rationale, not chain-of-thought       |
| 057 | Captured | Workspace-native UI runtime: retained tree over wgpu, in-VM Lua UI, window chrome        |
| 058 | Captured | Beacon targeting framework: spatial and semantic addressing via Command Registry         |
| 059 | Captured | Wheel context runtime: tiered evidence, knowledge DAG, and information refinement        |
| 060 | Captured | Phodopus runtime strategy: fork Piccolo, modular stdlib, and sandbox architecture        |
| 061 | Captured | Bitty Remote: unified remote infrastructure and the remote client                         |

Records 044–061 are **Captured** (owner directive 2026-09-18: the `*-docs` corpora are the working corpus agents build on; the research archive records discussion provenance, not task state). Record-to-document coverage now lives here, in the [`coverage/`](coverage/README.md) index (one file per docs corpus), together with each record summary's Destination/Open-items fields. The `*-docs` corpora are intentionally research-free — they carry no research references, record numbers, or coverage ledgers — so the mappings are kept only in this archive. The origin files stay byte-identical; only the `.completed` suffix and this row status changed.

Records 042 and 043 are captured as draft pages in `bitty-docs`
(`development/platform-compatibility.md`, `development/testing-infrastructure.md`)
through PR bitty-docs#315 (merged).

## Index — code reviews

| Date                                      | Scope                                                                                                                                                                                                                                                                                                                         |
| ----------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `review/2026-09-15/`                      | Full-workspace review, 11 reports: `bitty` crates (6), `bitty-ai`, `bitty-devtools`, SDK + template, plugins, docs. `bitty-website` out of scope. See its `README.md`.                                                                                                                                                        |
| [2026-09-17](review/2026-09-17/README.md) | Scoped multi-subagent static review with independent verification: 11 reports in terminal (6), AI (2), and plugins (3) directories. 18 of 19 terminal crates scoped; compat-lab excluded. Coverage, corrected priorities, revision drift, and limited test receipts in campaign index; not all-code or full-suite validation. |
