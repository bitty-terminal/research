# research agent guide

## Scope and authority

- This file governs only the independent `research` Git repository, mirrored
  at the org `research` repo.
- This repo is a Markdown-only archive: design discussion records and
  code-review campaigns backing the `*-docs` corpora (`bitty-docs`,
  `bitty-terminal-docs`, `bitty-ai-docs`, `bitty-plugins-docs`) that planning
  tasks build on. No product code lives here.

## Layout

- `origin/` — immutable full original records, numbered `NNN.md`. Never edit
  these files; the only allowed mutation is renaming `NNN.md` to
  `NNN.md.completed` once its conclusions land in `*-docs`.
- `summary/` — trimmed English mirrors, `NNN.md` per `origin/NNN.md`, in the
  7-field format (Status / Date / One-liner / Background / Key conclusions /
  Destination / Open items).
- `review/` — one directory per code-review campaign date, each with its own
  `README.md` describing the reports inside.
- Top-level `README.md` is the index: the status table for all records plus
  the review index.

## Status workflow

- `origin/NNN.md.completed` = conclusions recorded in `*-docs` (done); no
  suffix = open, still to be captured.
- New research: add `origin/NNN.md` (next free number) + `summary/NNN.md` +
  one `README.md` index row. On capture: rename the origin file, flip the
  summary Status to Captured, update the row.
- Keep `origin/`, `summary/NNN.md`, and the `README.md` row in sync; a rename
  without the other two updates is incomplete.

## Language and hygiene

- English-only for everything except `origin/` (originals keep their source
  language).
- No host-specific absolute paths (`/home/…`, `/mnt/…`), usernames, or
  machine layout anywhere durable.
- `origin/` is excluded from lint; everything else must pass `markdownlint-cli2`.

## Toolchain and delivery

- Markdownlint only (`.markdownlint-cli2.jsonc`), no CI/CD.
- Docs-only content pushes straight to `main`: no Issue/PR/checks cycle.
  Verify with `markdownlint-cli2 "**/*.md"` before pushing.
