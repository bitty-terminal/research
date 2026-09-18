# Record-to-document coverage index

This directory is the archive-side traceability record from each numbered
research record to the canonical documents it informed. It replaces the
per-corpus coverage ledgers that previously lived inside the docs repositories
and were removed when those corpora became research-free (the workspace
self-containment rule). The docs corpora are now intentionally research-free:
they carry no research references, record numbers, or coverage ledgers, so the
mappings live only here.

The mappings are **provenance**, not authority. A record is historical design
input; when a record and the maintained corpus disagree, the corpus wins. A
document named here is not thereby normative, accepted, or an implementation
claim; the document's own status field decides that.

## Per-corpus files

| Corpus file                                        | Corpus                | Records covered                                                                                 |
| -------------------------------------------------- | --------------------- | ----------------------------------------------------------------------------------------------- |
| [`bitty-ai-docs.md`](bitty-ai-docs.md)             | `bitty-ai-docs`       | 004, 010–013, 015, 017, 018, 021, 025–028, 030–033, 036–038, 039–056                            |
| [`bitty-docs.md`](bitty-docs.md)                   | `bitty-docs`          | 003, 007, 010–017, 019, 024, 029, 034–036, 038, 042–045                                         |
| [`bitty-terminal-docs.md`](bitty-terminal-docs.md) | `bitty-terminal-docs` | 001, 002, 005, 006, 008, 009, 013, 014, 016, 017, 020, 029, 034–036, 038, 039, 044–047, 053–055 |
| [`bitty-plugins-docs.md`](bitty-plugins-docs.md)   | `bitty-plugins-docs`  | 001, 002, 006, 008, 011, 013, 014, 020, 029, 032, 038, 040, 041, 053, 054                       |

A record can appear in several corpora because its conclusions were split
across owners. Records 048–056 are split-owner captures: the AI-side slice is
distilled in `bitty-ai-docs`, the terminal-side slice in `bitty-terminal-docs`,
the plugin-side slice in `bitty-plugins-docs`, and the governance-side slice in
`bitty-docs`; the remaining owner halves stay owner-pending.

## Reading the mappings

- Documents are named by their **current** title and corpus-relative path. The
  docs repositories renamed the original distillation pages after the coverage
  ledgers were written; this index uses the merged topical names, not the
  retired `research-distillation-*` paths.
- `Distilled` means a draft discussion synthesis informed by the record exists
  in the corpus; it is not an accepted decision.
- `Recorded` means the named canonical document carries the direction (as
  recorded when the mapping was written).
- `Owner-confirmed` means the destination owner confirmed the capture.
- `Owner-pending` means the corpus or owner was not verified; the topic is
  routed, not captured.
- Every record's own per-record pointer remains its `summary/NNN.md`
  Destination and Open-items fields. This index aggregates those pointers and
  the recovered ledgers; it does not replace them.

## Related

- [`origin/`](../origin/) — immutable original records.
- [`summary/`](../summary/) — trimmed English mirrors.
- [`README.md`](../README.md) — archive index and status table.
