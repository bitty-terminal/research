# Coverage — `bitty-ai-docs`

Records whose AI-relevant conclusions informed canonical documents in the
`bitty-ai-docs` corpus. Source: the recovered `bitty-ai-docs`
`research-coverage-ledger` and the recovered `bitty-docs`
`research-notes-coverage` ledger, translated to the corpus's **current**
topical document names. The retired `research-distillation-*` page names are
not cited.

Records 013, 017, 018, 025, and 039–056 carried detailed line-range
traceability in the recovered ledger; it is condensed here to the
record-to-document mapping. The record itself remains the evidence, not the
document.

## Mappings

| Record | Record topic (condensed)                                                              | Informing document(s) in `bitty-ai-docs`                                                                                            | State                                                |
| ------ | ------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------- |
| 004    | Hermes/Warp lessons: skills, memory, context, delegation, execution targets           | `architecture/ai-architecture.md` — AI Architecture                                                                                 | Recorded; agent direction remains candidate          |
| 010    | Spatial multi-agent, minimal-sufficient context, evidence chain                       | `architecture/ai-architecture.md`; open questions OQ-061..OQ-066 in `bitty-docs`                                                    | Recorded as candidate/direction                      |
| 011    | Native agent services, Lua policy / Rust enforcement, capability sandbox              | `architecture/ai-architecture.md`                                                                                                   | Recorded as candidate/direction                      |
| 012    | Batched tool calls, atomic multi-file patch, LSP/lint adapters                        | `architecture/ai-architecture.md`                                                                                                   | Recorded as candidate/direction                      |
| 013    | Architecture panorama; AI workspace and embodied multi-agent                          | `architecture/ai-architecture.md`; `specifications/ai-runtime-boundaries-candidate.md` — AI runtime boundaries (candidate)          | Distilled as draft synthesis (013/017/018 companion) |
| 015    | Echo-off detection, graded interaction, secret minimization (agent interlock)         | `specifications/ipc-agent-rfc.md` — IPC and Agent RFC; `architecture/ai-architecture.md`                                            | Recorded as candidate security direction             |
| 017    | M1–M8 assessment; optional AI integration; generic-primitives pressure test           | `specifications/ai-runtime-boundaries-candidate.md`; `product/ai-vertical-slice-pressure-test.md` — AI Vertical Slice Pressure Test | Distilled as draft synthesis (013/017/018 companion) |
| 018    | Standalone AI runtime owning context, permissions, events, code intelligence          | `specifications/ai-runtime-boundaries-candidate.md`; `architecture/ai-architecture.md`                                              | Distilled as draft synthesis (013/017/018 companion) |
| 021    | Shared processes and toolchains; organized collaboration; compilable context          | `specifications/shared-workspace-services-candidate.md` — Shared workspace services and agent coordination (candidate)              | Distilled as draft synthesis                         |
| 022    | Mechanisms in core, workflows in plugins; context compressed by view                  | `context/context-management.md` — Context management architecture (owner-pending)                                                   | Owner-pending                                        |
| 023    | Docs good enough; move into runtime implementation                                    | `architecture/ai-architecture.md` (owner-pending)                                                                                   | Owner-pending                                        |
| 025    | Prefix-cache-friendly context: stable content first, appends last                     | `context/prefix-cache-context-design.md` — Prefix-Cache-Friendly Context Design                                                     | Distilled as draft synthesis; record Captured        |
| 026    | Built-ins teach semantics; runtime owns capabilities                                  | `context/prompt-layering-design.md` — Prompt Layering Design                                                                        | Distilled as draft synthesis                         |
| 027    | Core prompt around a thousand words; capabilities on demand                           | `context/prompt-layering-design.md` (budget evidence)                                                                               | Distilled as draft synthesis                         |
| 028    | Remove empty shells and duplicate implementations; fix identity ownership             | `architecture/ai-architecture.md`; `product/implementation-profile-v0.1.md` (owner-pending)                                         | Owner-pending                                        |
| 030    | Keep kernel minimal; third parties only in adapters                                   | `providers/dependency-strategy.md` — Dependency Strategy                                                                            | Distilled as draft synthesis                         |
| 031    | Hard gaps blocking real AI integration after parallel development                     | `architecture/ai-architecture.md` (owner-pending)                                                                                   | Owner-pending                                        |
| 032    | Model abstraction in core; vendor/subscription integrations in plugins                | `providers/provider-plugin-boundary.md` — Provider plugin boundary                                                                  | Distilled as draft synthesis                         |
| 033    | Unified multimodal lifecycles without one request shape                               | `providers/multimodal-inference-boundary.md` — Multimodal inference boundary                                                        | Distilled as draft synthesis                         |
| 036    | Inherit pane environments via prompt-time sync, masked for agents                     | `interfaces/panel-environment-awareness.md` — Panel environment awareness                                                           | Distilled as draft synthesis                         |
| 037    | Split config/data/state/cache; shard storage per session                              | `persistence/storage-memory-export-design.md` — Storage memory and export design                                                    | Distilled as draft synthesis                         |
| 038    | Core emits events; durable history is a plugin reusing external history               | `persistence/history-consumption-boundary.md` — History consumption boundary                                                        | Distilled as draft synthesis                         |
| 039    | Panel as app container; activity keeps sessions alive                                 | `specifications/panel-workspace-candidate.md` — Panel and agent workspace boundary (candidate)                                      | AI slice distilled (draft); Partial                  |
| 040    | Plugin dependency/service/extension graphs; applications expose extension platforms   | `specifications/plugin-extension-model-candidate.md` — Plugin and extension model (candidate)                                       | AI slice distilled (draft); Partial                  |
| 041    | IPC as a second plugin boundary sharing one capability model                          | `specifications/ipc-extension-boundary-candidate.md` — IPC extension boundary (candidate)                                           | AI slice distilled (draft); Partial                  |
| 044    | Job/Execution first-class under a supervisor; mechanisms vs semantics                 | `specifications/execution-supervisor-candidate.md` — Execution supervisor (candidate)                                               | AI slice distilled (draft); Partial                  |
| 045    | Lua decides how, never whether; capability intersection, budgets, secrets             | `specifications/lua-core-safety-boundary-candidate.md` — Lua and Core safety boundary (candidate)                                   | AI slice distilled (draft); Partial                  |
| 046    | Wheel scoped to the Coding agent harness; personal/office agents are future plugins   | `specifications/wheel-scope-and-framework-candidate.md` — Wheel scope and framework illustration (candidate)                        | AI slice distilled (draft); Partial                  |
| 047    | Caller attribution for provider calls; LLM-provider layer splits out of Wheel         | `agent/caller-attribution-design.md` — Caller attribution design                                                                    | Draft candidate; owner-pending beyond draft          |
| 048    | Quality is Model × Harness × Context × Tools × Verification                           | `specifications/quality-and-context-compiler-candidate.md` — Quality formula and context compiler (candidate)                       | AI slice distilled (draft); Partial                  |
| 049    | Context Compiler over Cold/Warm/Hot layers; stability zones; cache planning           | `specifications/quality-and-context-compiler-candidate.md`                                                                          | AI slice distilled (draft); Partial                  |
| 050    | Portable capabilities in `.agents`; harness behavior in `.wheel`                      | `specifications/wheel-config-and-context-git-model-candidate.md` — Wheel configuration and context git model (candidate)            | AI slice distilled (draft); Partial                  |
| 051    | Git-inspired context DAG: checkpoints, branches, typed merges, GC                     | `specifications/wheel-config-and-context-git-model-candidate.md`                                                                    | AI slice distilled (draft); Partial                  |
| 052    | Minimal Wheel kernel with Lua-plugin composition; bitty-ai as distribution            | `specifications/wheel-core-plugin-boundary-candidate.md` — Wheel core and plugin boundary (candidate)                               | AI slice distilled (draft); Partial                  |
| 053    | Public plugin contracts; layered Lua frameworks; versioned services                   | `specifications/wheel-core-plugin-boundary-candidate.md` (AI-relevant framework slice)                                              | AI slice distilled (draft); Partial                  |
| 054    | Bitty owns plugin lifecycle; Lux serves only the Lua dependency layer                 | `specifications/wheel-core-plugin-boundary-candidate.md` (AI-relevant slice)                                                        | AI slice distilled (draft); Partial                  |
| 055    | Wheel as Event-Sourced Agent Workspace: headless agents, event log, versioned context | `specifications/event-sourced-agent-workspace-candidate.md` — Event-sourced agent workspace (candidate)                             | AI slice distilled (draft); Partial                  |
| 056    | Split stored history from compiled context; commit rationale, not chain-of-thought    | `specifications/wheel-context-storage-and-reasoning-candidate.md` — Wheel context storage and reasoning management (candidate)      | AI slice distilled (draft); Partial                  |

## Notes

- Records 048–056 are split-owner: the AI-side slice is distilled here as a
  draft discussion synthesis, not an accepted decision; the terminal, plugin,
  governance, and storage owner halves remain owner-pending.
- `specifications/ai-runtime-boundaries-candidate.md`,
  `specifications/panel-workspace-candidate.md`,
  `specifications/plugin-extension-model-candidate.md`,
  `specifications/ipc-extension-boundary-candidate.md`,
  `specifications/shared-workspace-services-candidate.md`, and the
  `research-distillation-021` material were renamed by the corpus's
  self-containment pass; the original ledger cited the pre-rename titles.
- `product/ai-unresolved-questions.md` — AI Unresolved Questions and
  `architecture/*-rN.md` (R1–R6 disposition pages) hold the open questions and
  disposition state that the record distillations reference but do not own.

## Related

- [`README.md`](README.md) — coverage index.
- [`bitty-docs.md`](bitty-docs.md), [`bitty-terminal-docs.md`](bitty-terminal-docs.md),
  [`bitty-plugins-docs.md`](bitty-plugins-docs.md) — sibling corpora.
