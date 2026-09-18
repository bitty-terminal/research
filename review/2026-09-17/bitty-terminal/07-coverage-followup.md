# Terminal coverage follow-up — 2026-09-17

## Scope and baseline

Read-only follow-up to reports 01–06, focused on `bitty-compat-lab`, selected configuration merge/types bodies, and runtime production files marked unread in report 02. Only this report is authorized for writing. Paths are relative to the umbrella workspace unless a table specifies a crate prefix. Broad review does not mean every file was read.

| Repository            | HEAD                                       | Initial dirty state                                  |
| --------------------- | ------------------------------------------ | ---------------------------------------------------- |
| `bitty`               | `06bc1f45995fd297a3f7324688bc81b0598ecf67` | Untracked `.targets/`; no tracked changes observed   |
| `research`            | `d70152a079c75204ec37e99a7bf3f54b8190e423` | Modified `README.md`; untracked `review/2026-09-17/` |
| `bitty-terminal-docs` | `7947fb39acf77d308c2eb8bc57b24c481a841a17` | Clean                                                |

The umbrella is not a Git repository; the initial umbrella Git query failed. Guidance read: umbrella, `bitty`, `research`, and `bitty-terminal-docs` AGENTS; ctxctl-core skill. No narrower AGENTS were found under `bitty/crates` or `research/review`. CtxCtl MCP rejected the checkout path; installed CLI outlines and source slices worked. No CarryCtx, nested agents, network, installations, commits, source edits, live probes, or exploit/reproduction code.

Reports 02, 03, and the product README were read; findings/headings in 01, 04, 05, and 06 were searched for overlap. Existing findings are not reissued here. Static verification establishes control flow, not a demonstrated runtime incident or test pass.

## Verified defects

### TERM-GAP-001 — P2: References not compared are counted as passing

- **Cause:** `bitty/crates/bitty-compat-lab/src/compare.rs:611-634` compares reference text only after self-consistency succeeds. When self-consistency fails but references exist, the alternative branch nevertheless assigns `reference_compared = refs.len()` without performing comparisons or recording failures. `bitty/crates/bitty-compat-lab/src/compare.rs:676-692` totals those counts and computes `reference_passed` as compared minus failures.
- **Impact:** A self-failing baseline can contribute reference passes for checks that never ran. The self-failure remains visible; this is false reference accounting, not a claim that the entire run necessarily succeeds. The reference counter can mislead compatibility evidence consumers.
- **Remediation:** Represent passed, failed, skipped, and blocked-by-self-failure outcomes explicitly. Increment compared only at the comparison, and compute passes from actual successful comparisons.
- **Safe test idea:** A pure comparator test with an intentionally inconsistent in-memory baseline and benign reference text should assert self failure, zero attempted/passed reference comparisons, and an explicit blocked outcome. No filesystem mutation or terminal process is needed with a reference-provider seam.
- **Contract:** `bitty-terminal-docs/reference/compatibility-matrix.md:136-146` distinguishes missing/skipped reference evidence from passes. The same evidence-honesty principle applies to blocked comparisons.

### TERM-GAP-002 — P2: Dump collector can report writes when no destination was created

- **Cause:** `bitty/crates/bitty-compat-lab/src/bin/collect_dumps.rs:126-130` logs directory-creation failures and continues. At `:222-229`, nonexistent output directories are skipped, but `written` increments unconditionally once per corpus. The final minimum-count assertion and success message at `:239-254` use that counter rather than successful writes.
- **Impact:** When all selected output directories remain absent after creation errors, corpus replay can finish and report snapshots written without writing any. With one unavailable destination, the message still lists that destination as though mirrored output succeeded. Existing-file write errors panic instead; that branch does not silently succeed.
- **Remediation:** Treat the canonical output directory as required, distinguish required and optional mirrors, and count successful writes per destination. Return a non-success outcome when required publication fails, and label partial mirror failures explicitly.
- **Safe test idea:** Inject directory-creation and write outcomes into a fake output sink. Cover all-destinations-unavailable, one-mirror-unavailable, and successful publication; assert truthful counts and exit status without changing real permissions or writing snapshots.

## Uncertainty and optimization boundaries

- Self-consistency and deterministic replay are not independent terminal conformance. The report generator explicitly emits `test-present` for named tests (`bitty/crates/bitty-compat-lab/src/report.rs:566-576,697-705`), not their execution result. Local rows are declared coverage, not live probe receipts (`:706-713`). Do not upgrade either to current execution evidence.
- `bitty/crates/bitty-runtime/src/plugin_runtime/resolution.rs:127-145` has the same destination-deleting rename fallback mechanism already reported for the key/value store as TERM-RUN-003. This is additional affected-path coverage, not a duplicate new finding; package-index consequences need their own scoped remediation tests.
- Existing report 02 already lists global search/selection attribution and pending-paste target ownership as hypotheses. The newly read search facade still uses `self.state`; this follow-up does not duplicate that lead or claim complete live UI attribution proof.
- Existing background-resource object-binding and plugin permission findings remain owned by report 05. Re-reading callers does not establish their exploitability or renew an exhaustive security audit.

## Newly reviewed coverage ledger

Full means all file lines read. Production-complete means the entire non-test body was read but inline tests were not; it is recorded as sampled at file level. Outlines and search hits alone do not count as body coverage. Exact ranges below are inclusive.

### Compatibility lab

Prefix: `bitty/crates/bitty-compat-lab/`.

| Path                         | Coverage                  | Ranges                                          |
| ---------------------------- | ------------------------- | ----------------------------------------------- |
| `src/lib.rs`                 | Full                      | 1–91                                            |
| `src/compare.rs`             | Full substantive body     | 1–373, 375–827; omitted line 374 is a separator |
| `src/matrix.rs`              | Full                      | 1–331                                           |
| `src/report.rs`              | Full                      | 1–762                                           |
| `src/bin/collect_dumps.rs`   | Full                      | 1–256                                           |
| `src/bin/compat_report.rs`   | Full                      | 1–78                                            |
| `Cargo.toml`                 | Full                      | 1–21                                            |
| `README.md`                  | Full                      | 1–39                                            |
| `tests/compare.rs`           | Outline only; body unread | None                                            |
| `tests/compat_matrix.rs`     | Outline only; body unread | None                                            |
| `tests/dogfooding_corpus.rs` | Outline only; body unread | None                                            |
| `tests/report.rs`            | Outline only; body unread | None                                            |
| `tests/harness.rs`           | Outline only; body unread | None                                            |
| `tests/live_compat.rs`       | Unread                    | None                                            |

The production library imports code outside its `src/` tree: `bitty/tests/compat/harness.rs:1-192` was fully read, including inline tests. Thus the imported production harness is included, not overlooked because of its test-directory name. Corpus bytes and captured reference dumps were not inspected or executed.

### Runtime

Prefix: `bitty/crates/bitty-runtime/src/`.

| Path                              | Coverage                     | Ranges                                                    |
| --------------------------------- | ---------------------------- | --------------------------------------------------------- |
| `error.rs`                        | Full                         | 1–76                                                      |
| `inspect.rs`                      | Sampled, production-complete | 1–546                                                     |
| `palette.rs`                      | Sampled, production-complete | 1–234                                                     |
| `paste.rs`                        | Sampled, production-complete | 1–225                                                     |
| `project.rs`                      | Sampled, production-complete | 1–148                                                     |
| `project_scope.rs`                | Sampled, production-complete | 1–60                                                      |
| `queries.rs`                      | Sampled, production-complete | 1–513                                                     |
| `registry.rs`                     | Full substantive body        | 1–518                                                     |
| `shell_integration.rs`            | Sampled, production-complete | 1–123                                                     |
| `statusline.rs`                   | Sampled, production-complete | 1–160                                                     |
| `tabs.rs`                         | Full                         | 1–158                                                     |
| `workspace.rs`                    | Sampled, production-complete | 1–147                                                     |
| `plugin_runtime/manifest_toml.rs` | Sampled, production-complete | 1–503                                                     |
| `plugin_runtime/package.rs`       | Sampled, production-complete | 1–830                                                     |
| `plugin_runtime/resolution.rs`    | Sampled, production-complete | 1–718                                                     |
| `runtime/animations.rs`           | Sampled, production-complete | 1–370; initially truncated leading output reread at 1–169 |
| `runtime/background_images.rs`    | Full                         | 1–137                                                     |
| `runtime/help.rs`                 | Sampled, production-complete | 1–317                                                     |
| `runtime/kitty_images.rs`         | Full                         | 1–209                                                     |
| `runtime/mouse_chrome.rs`         | Full                         | 1–277                                                     |
| `runtime/resize.rs`               | Sampled, production-complete | 1–653                                                     |
| `runtime/scrollbar.rs`            | Full                         | 1–296                                                     |
| `runtime/search.rs`               | Full                         | 1–277                                                     |
| `runtime/close_confirm.rs`        | Sampled, production-complete | 1–273                                                     |
| `config.rs`                       | Outline only; body unread    | None                                                      |

Report 02's previously sampled files remain sampled there; they have not become fully reviewed merely because their siblings were read. In particular, `runtime/present.rs` remains body-unread in this follow-up. Runtime integration tests and the inline-test tails not listed above remain unread in this follow-up.

### Configuration

Prefix: `bitty/crates/bitty-config/src/`.

Body coverage recorded after the report text was drafted (explicitly recorded source reads):

| File                                       | Coverage class                        | Ranges read                          |
| ------------------------------------------ | ------------------------------------- | ------------------------------------ |
| `merge.rs`                                 | Sampled, production-complete segments | 1–618, 619–850, 1380–1640, 2290–2441 |
| `types.rs`                                 | Sampled segments                      | 824–924, 987–1148, 2546–2986         |
| `validation.rs`                            | Full                                  | 1–173                                |
| `bitty/crates/bitty-runtime/src/config.rs` | Outline only; body unread             | None                                 |

No new defect was identified in the read config bodies against `bitty-terminal-docs/specifications/configuration-model-rfc.md:153-209` (one merge class per field, conflict reporting, attribution survival, non-overridable policy, profile `extends` with cycle detection). The remaining unread config bodies remain residual gaps.

### Intended behavior consulted

- `bitty-terminal-docs/product/compat-lab.md:1-143`: draft scaffold, deterministic and bounded harness, reference comparison intent, larger corpora must be split. Historical roadmap claims were not treated as current implementation evidence.
- `bitty-terminal-docs/reference/compatibility-matrix.md:1-180`: maintained matrix, CI/local/partial/gap distinctions and live-probe exclusions; 181–235 unread.
- `research/summary/017.md:1-18`: full; systematic compatibility proof and workspace invariants prioritized over feature expansion.
- Other research summaries were keyword-searched only, not body-reviewed.

## Checks and residual limitations

This is a static report, not a code fix. No product tests, Rust build, clippy, rustfmt, typecheck, real PTY/GPU activity, or fault injection was run. Test ideas are proposed defensive checks, not executed evidence. The only intended validation execution is Markdown lint of this report; its result will be recorded after execution. Executed: `markdownlint-cli2 07-coverage-followup.md` from `research/` (repo config `.markdownlint-cli2.jsonc`), result: 0 issues.

Residual gaps include remaining config bodies, remaining package/resolution bodies, runtime presentation and other previously sampled modules, most tests, corpus correctness, independent terminal normalization, platform behavior, dependencies, filesystem concurrency, and live UI integration. Nothing here constitutes release certification or an exhaustive every-file audit.
