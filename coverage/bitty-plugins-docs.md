# Coverage — `bitty-plugins-docs`

Records whose plugin-ecosystem conclusions informed canonical documents in
the `bitty-plugins-docs` corpus. Compiled from the recovered `bitty-docs`
`research-notes-coverage` ledger, the recovered `bitty-ai-docs` coverage ledger
(plugin boundary context), and the corpus's distillation PRs. Only current
topical document names are cited; the retired `research-*-conclusions` and
`research-distillation-*` page names are not.

The recovered `bitty-docs` ledger labelled most plugin destinations
`Needs owner confirmation`; that state is preserved here rather than upgraded.

## Mappings

| Record | Record topic (condensed)                                                                  | Informing document(s) in `bitty-plugins-docs`                                                                                   | State                                                                       |
| ------ | ----------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| 001    | Core owns mechanisms; plugins own policy; five-layer model                                | `extensibility/plugin-system.md` — Plugin system (core/plugin API boundary)                                                     | Needs owner confirmation                                                    |
| 002    | Panels as first-class app containers; IPC as unified extension protocol                   | `architecture/plugin-ipc-boundary.md` — Plugin IPC Boundary; `architecture/plugin-ecosystem-model.md` — Plugin Ecosystem Model  | Needs owner confirmation                                                    |
| 006    | Plugin and protocol risks; package management; first-party plugin discipline              | `extensibility/plugin-system.md`; `extensibility/package-management.md` — Plugin package management                             | Needs owner confirmation                                                    |
| 008    | Semantic terminal Hints as a generic plugin API; first-party capabilities                 | `extensibility/plugin-system.md` (Hints/plugin API direction)                                                                   | Needs owner confirmation                                                    |
| 011    | CarryCtx/ctxctl as native subsystems; Lua sandbox boundary                                | `extensibility/plugin-system.md`                                                                                                | Recorded as candidate/direction                                             |
| 013    | Plugin suitability; plugin reuse; plugin roadmap                                          | `packaging/plugin-reuse-and-providers.md` — Plugin Reuse and Provider Ecology RFC; `product/plugin-roadmap.md` — Plugin Roadmap | Recorded as candidate/direction                                             |
| 014    | Review refinements to 013: Panel/Execution separation, trust levels, RichSurface          | `packaging/plugin-reuse-and-providers.md`; `product/plugin-roadmap.md`                                                          | Recorded as refinements                                                     |
| 020    | Pixel mascot; per-panel scaling; plugin-side visual identity                              | `product/plugin-roadmap.md` (plugin-side visual identity)                                                                       | Needs owner confirmation                                                    |
| 029    | Core stays offline; extensions cross controlled boundaries                                | `architecture/plugin-ipc-boundary.md`; `architecture/plugin-ecosystem-model.md`                                                 | Needs owner confirmation                                                    |
| 032    | Model abstraction in core; vendor/subscription integrations in plugins                    | `packaging/plugin-reuse-and-providers.md` (Model-provider direction — candidate); AI side owner-pending                         | Recorded as captured-candidate (non-normative)                              |
| 038    | Core emits events; durable history is a plugin reusing external history                   | `packaging/plugin-reuse-and-providers.md` (History-provider direction — candidate)                                              | Recorded as captured-candidate (non-normative)                              |
| 040    | Plugins form dependency/service/extension graphs; applications expose extension platforms | `architecture/plugin-ecosystem-model.md`                                                                                        | Needs owner confirmation (AI side distilled in `bitty-ai-docs`)             |
| 041    | IPC is a second plugin boundary sharing one capability model                              | `architecture/plugin-ipc-boundary.md`                                                                                           | Needs owner confirmation (AI side distilled in `bitty-ai-docs`)             |
| 053    | Public plugin contracts; layered Lua frameworks; versioned services; dependency kinds     | `specifications/plugin-contract-direction.md` — Plugin contract direction (candidate)                                           | Plugin slice distilled (draft); Partial. SDK/packaging owners owner-pending |
| 054    | Bitty owns plugin lifecycle; Lux serves only the Lua dependency layer                     | `specifications/plugin-contract-direction.md`                                                                                   | Plugin slice distilled (draft); Partial. Manager/Lux split owner-pending    |

## Notes

- Records 053/054 are split-owner: the plugin-side capture is the draft
  `plugin-contract-direction.md`, distilled from the retired
  `research-053-054-plugin-conclusions.md`; host/SDK boundaries, Lua subset,
  packaging flow, and registry role remain owner-pending.
- The corpus's self-containment pass renamed the 053/054 conclusions page to
  `specifications/plugin-contract-direction.md` and moved the plugin model/IPC
  pages from `specifications/` to `architecture/`.
- Records 001/002/006/008/020/029 are listed because their summaries propose a
  plugin-corpus destination; the recovered ledger did not verify a plugin page
  for them, so the state stays owner-pending.

## Related

- [`README.md`](README.md) — coverage index.
- [`bitty-ai-docs.md`](bitty-ai-docs.md), [`bitty-docs.md`](bitty-docs.md),
  [`bitty-terminal-docs.md`](bitty-terminal-docs.md) — sibling corpora.
