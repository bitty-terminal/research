# Terminal Engine Review — bitty (2026-09-17)

- Scope: `bitty-vt`, `bitty-term-state`, `bitty-render`, `bitty-pty`, `bitty-platform` in the `bitty` repository.
- Method: static code reading only. No builds, tests, or reproductions were run (out of scope for this task); every behavioral claim below is a reading conclusion, not a demonstrated runtime failure.
- Report language: English. Defect IDs: `TERM-ENG-###`, severities P1 (correctness/data-loss visible to users), P2 (incorrect edge behavior, bounded leaks, perf hazards), P3 (robustness, testing gaps).

## Revisions reviewed

| Repo                  | HEAD                                       | Dirty                                                              |
| --------------------- | ------------------------------------------ | ------------------------------------------------------------------ |
| `bitty`               | `06bc1f45995fd297a3f7324688bc81b0598ecf67` | untracked `.targets/` only                                         |
| `research`            | `26a46ed8c15aa4e714ca9add4ff4c07bea02ff22` | untracked `review/2026-09-16/` (other reviewers' files; untouched) |
| `bitty-terminal-docs` | `cb9e9e42e8807ce5eb9678c690e925465e471a67` | clean                                                              |

Specs consulted: `terminal-state-rfc.md` (accepted), `text-rendering-rfc.md` (draft/experimental — treated as non-normative), `reference/compatibility-matrix.md`, `security/audits/clipboard-2026-09.md`, and `research/summary/006/043/044/045.md` for context.

## Coverage

Read in depth (near-fully or symbol-complete):

- `bitty-term-state`: `src/state.rs` (apply/dispatch, damage batching, print, erase, resize/reflow, OSC 8, replies, resets), `src/grid.rs`, `src/damage.rs`, `src/charsets.rs`, `src/cell.rs`, `src/search.rs`, `src/state/hash.rs`, `src/state/invariants.rs`.
- `bitty-render`: `src/grid.rs` (atlas, render, place/emit glyph), `src/frame.rs` (plan_frame/coalesce), `src/cache.rs`, `src/batch.rs`, `src/pipeline.rs`, `src/gpu.rs`, `src/software.rs`, `src/fallback.rs`, `src/crossfont_backend.rs`.
- `bitty-vt`: `src/parser.rs`, `src/parser/dispatch.rs`, `src/parser/sgr.rs`, `src/kitty_apc.rs`.
- `bitty-pty`: `src/pty.rs`, `src/reader.rs`, `src/writer.rs`, `src/builder.rs`, platform outlines (`src/platform/unix.rs`, `windows.rs`).
- `bitty-platform`: `src/clipboard.rs`, `src/keyboard.rs`, `src/event.rs`, `src/app.rs`, `src/surface.rs`, `src/url.rs`, `src/dpi.rs`.
- Tests sampled: `bitty-term-state/tests/replay_determinism.rs`, `resize_scrollback.rs`, `property_invariants.rs` (grep), `bitty-vt/src/parser/tests.rs`, `bitty-pty/tests/spawn_smoke.rs` (outline + partial).

Not covered / coverage gaps: `bitty-runtime` composition and the real present path (so "runtime emits glyphs before/after eviction" claims below are hypotheses about call order, marked as such); Windows-specific PTY code paths (outline only); font shaping beyond single-scalar caching in `bitty-render`; accessibility/IME surfaces; benchmark suites; `bitty-platform` winit event-loop corner cases under real display servers (gui-tests feature excluded from CI by design).

## Findings

### New defects

#### TERM-ENG-001 (P1, confirmed) — `damage_since` drops damage when the retained window overflows

- Evidence: `bitty/crates/bitty-term-state/src/state.rs:710` (`damage_since` concatenates only batches whose `generation > generation`), `bitty/crates/bitty-term-state/src/state.rs:915` (`finalize_batch` unconditionally pushes a batch per applied action, even when the coalesced region set is empty, after popping the oldest entry at `DAMAGE_HISTORY_BATCHES` capacity; window constant in `bitty/crates/bitty-term-state/src/damage.rs:24`, value 64).
- Behavior: when a consumer misses more batches than the retained window, older damage is discarded. The retained batches are **newer**, not older, than that consumer's last-seen generation. If those retained batches contain only empty damage, the result is empty despite discarded visible changes; otherwise it can be incomplete. The documented full-grid fallback is absent. This proves an API-contract defect, not that a particular live renderer currently misses such a window; runtime consumer reachability was not verified in this second pass.
- This is the same root cause as prior-review D01 (see recheck section); it remains unfixed.
- Fix: detect an actual history gap before concatenating retained batches and return conservative full damage. With contiguous generation numbers and oldest retained generation `g`, the immediately preceding generation `g - 1` still has complete coverage; older requests do not. The original proposed `b.generation <= generation` predicate had the comparison reversed. Account for scrollback conservatively rather than inferring discarded scroll damage from retained batches alone.
- Regression test goal: separately assert conservative coverage when a consumer predates retained history. The existing `damage_since_reconstructs_the_full_stream` (`bitty/crates/bitty-term-state/tests/replay_determinism.rs:196-210`) generates only 1..40 actions and asserts the within-window bound at line 209. Preserve that concatenation property; add a distinct history-gap property instead of simply removing/inverting the bound.

#### TERM-ENG-002 (P2, hypothesis on trigger timing, mechanism confirmed) — whole-atlas `evict_all` invalidates glyph slots already emitted in the same frame

- Evidence: `bitty/crates/bitty-render/src/grid.rs:1292` (`evict_all` resets layout, clears `slots`, zeroes texels, and clears `pending` uploads), reached from `ensure` at `grid.rs:1272-1285` when allocation fails; `emit_glyph` at `grid.rs:1778` reads `self.atlas.ensure(...)` and then pushes a `GlyphInstance` holding `GlyphSource::Atlas { slot }` plus `uv: slot.uv(...)` computed from the pre-eviction slot.
- Mechanism: a later cell's `ensure()` can evict the atlas while the current draw list still retains slots emitted for earlier cells. `evict_all()` clears CPU texels, slot ownership, and pending uploads without rebuilding that list. This establishes stale slot references under mid-pass exhaustion. Not every earlier instance necessarily becomes visibly wrong: some texels could coincide or remain in the GPU texture. The actual upload/present path and possible effects on retained pixels in later partial frames were not traced; the original universal-corruption and same-frame-only claims were too strong.
- Why hypothesis at the trigger level: this needs an allocation failure at a mid-frame `ensure()` — i.e., a frame whose dirty region mixes many distinct large glyphs such that the atlas fills mid-pass. The code path itself is unambiguous; whether it is reachable with default font/size is untested here (static review, no reproduction).
- Prior-review D02 equivalence: same defect; still present.
- Fix options: (a) on eviction, mark the current `DrawList` as requiring a full re-pass (re-run `render` from a clean plan), (b) move eviction to the start of the frame before any `ensure` in the pass, or (c) keep evicted glyphs alive for the current frame by snapshotting `pending`/texels before reset and letting the in-flight instances reference the snapshot.
- Regression test idea: unit test that shrinks the atlas (test constructor), renders a frame whose first dirty cell rasterizes a large glyph and whose later cells exhaust the atlas, and asserts every emitted `GlyphSource::Atlas` instance's UV region contains that glyph's texels in the atlas texture after uploads are applied.

#### TERM-ENG-003 (reclassified, unverified compatibility policy) — unsupported underline-style fallback

- Source behavior verified: `bitty/crates/bitty-vt/src/parser/sgr.rs:99-109,134-140,158-163` maps unsupported underline styles to Single and maps SGR 21 to Double.
- **Withdrawn as a confirmed P2 defect.** The original heading and body contradicted each other about xterm. Neither an authoritative unsupported-style contract nor a comparator result was established. The claimed ECMA/ITU negotiation requirement and xterm behavior are withdrawn, not replaced with another unsupported assertion.
- Follow-up: establish the intended compatibility policy and then document/test it. Do not prescribe ignoring unsupported styles as a proven fix. No runtime compatibility check was performed.

#### TERM-ENG-004 (withdrawn as a defect; optional invariant hardening) — cursor-report arithmetic

- `bitty/crates/bitty-term-state/src/state.rs:1692-1707` uses ordinary subtraction in origin mode. The original report itself acknowledged the cursor invariant prevents underflow today; a hypothetical future relaxation is not causal evidence of a present defect.
- No reachable invariant violation was established in this pass. Keep origin/scroll-region transition tests as defense in depth, not a confirmed failure. Do not silently saturate invalid state without deciding whether invariant violations should instead be exposed.

#### TERM-ENG-005 (reclassified P3 test-coverage follow-up, not a product defect) — resize/region/alternate-screen combinations

- Corrected evidence: `bitty/crates/bitty-term-state/src/state.rs:356-363` resets the scroll region to full-screen; it does **not** reset origin mode there. Saved-cursor clamping and spacer repair are at `state.rs:403-436`.
- The sampled test `bitty/crates/bitty-term-state/tests/resize_scrollback.rs:181-205` checks height changes and bounds with default modes, despite its preservation-oriented name. The file extends through line 533; scrollback monotonicity is at lines 208-285, not only through 210.
- Add or locate coverage combining alternate screen, saved cursors, origin mode, narrowed region, and resize. Absence across the entire suite was not established; this is a sampled gap, not proof that resize is incorrect. No product code change is implied.

### Prior-review recheck (research/review/2026-09-15)

| Prior finding                                                                    | Verdict at current HEAD                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Evidence                                                                                                                       |
| -------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| 01 D01 — damage history window overflow loses full-redraw fallback               | Still valid, confirmed                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | TERM-ENG-001 above (`state.rs:710`, `state.rs:915`, `damage.rs:24`; proptest excludes overflow at `replay_determinism.rs:209`) |
| 01 D02 — mid-frame whole-atlas eviction corrupts already-emitted glyph instances | Still valid, mechanism confirmed / trigger reachability unproven                                                                                                                                                                                                                                                                                                                                                                                                                                          | TERM-ENG-002 above (`grid.rs:1292`, `grid.rs:1272-1285`, `grid.rs:1778-1830`)                                                  |
| 02 P1-1 — `Drop for Pty` blocks unboundedly, unbounded leak                      | Partially superseded: blocking drop is now a documented, deliberate contract ("Leak-free by default", `crates/bitty-pty/src/pty.rs:7-12`), and kill-then-wait on SIGKILL-equivalent termination is bounded in practice for healthy children (`pty.rs:399-408`). Residual concern remains only for uninterruptible child states (D-state on Linux); that is inherent to reaping, not a defect at current design. Recommend keeping a note in docs that `Drop` may block, which the module docs already do. | `pty.rs:1-12`, `pty.rs:399-408`                                                                                                |
| 02 PTY wait-hang (Windows ConPTY `cmd /C exit` ~23 min)                          | Addressed in code: `Pty::wait_timeout` polls `try_wait` with a 5 ms cadence and documents the exact incident (`pty.rs:212-245`, `WAIT_POLL_INTERVAL` at `pty.rs:25`)                                                                                                                                                                                                                                                                                                                                      | `pty.rs:212-245`                                                                                                               |
| 02 clipboard claims (wl-copy stdin pipe, CTX-0388, bounded size)                 | Consistent with current code: `CLIPBOARD_MAX_BYTES` = 8192 (`crates/bitty-platform/src/clipboard.rs:78`); wl-clipboard-rs data-control backend selected via the `wayland-data-control` arboard feature (`crates/bitty-platform/Cargo.toml:31-36`). No new defect introduced; the CTX-0388 audit covers this surface.                                                                                                                                                                                      | `clipboard.rs:78`, `Cargo.toml:31-36`                                                                                          |

Prior claims not re-listed above were either cosmetic, already fixed, or not reproducible against the current tree from the static read; none of the rechecked files contradicted the remaining prior findings in a way that warrants new IDs.

### Security notes (high-level, defensive only)

- Untrusted-input bounding is consistently applied in the reviewed surface: OSC 8 hyperlink table capped (`state.rs:1643-1668`, `HYPERLINK_TABLE_MAX` degrade-to-no-link), reply queue bounded with an overflow flag (`state.rs:742-754`), zone records capped, process names stripped/truncated (`pty.rs:260-278`), clipboard payload capped, PTY builder strips graphics fingerprints and caps env (`builder.rs`). No new exploit-class issue found; no exploit details included by design.
- Kitty graphics/ APC handling remains inert in state (`state.rs:901-902`) — bounded by design; parser-side `kitty_apc.rs` parsing was reviewed and stays allocation-bounded.
- Do not infer absence of all P0 controls from stale pre-implementation guidance. The second pass observed a safe-config branch at `bitty/crates/bitty-app/src/config_cli.rs:117-135` and capability checks at `bitty/crates/bitty-runtime/src/plugin_runtime/services.rs:217-242`. Neither these samples nor historical guidance establish complete security acceptance.

## Test limitations

No tests, builds, benchmarks, or linters were run against the product code as part of this review (task constraint: static reading only). All "confirmed" labels mean the defect mechanism is visible in the cited code paths and follows from the types and control flow as written; TERM-ENG-002's real-world trigger frequency is untested. The only validation performed was Markdown linting of this report.

## Summary

- Retained: TERM-ENG-001 (P1 API-contract mechanism; live consumer impact unverified) and TERM-ENG-002 (conditional P2 atlas-lifetime mechanism; default trigger and presentation unverified).
- Reclassified: TERM-ENG-003 is an unresolved compatibility policy, TERM-ENG-004 is withdrawn as a present defect, TERM-ENG-005 is a sampled P3 test follow-up only. IDs are preserved.
- Prior PTY/clipboard dispositions are first-pass conclusions, not independently renewed here. Do not infer cross-platform closure from static code presence.
- Highest-value follow-up: implement and test a conservative history-gap fallback. Keep the existing within-window concatenation property and add a separate out-of-window property; merely removing its assertion would leave the equality contract wrong for a full-redraw fallback.

## Independent second-pass verification

- Source baseline rechecked: `bitty` HEAD `06bc1f45995fd297a3f7324688bc81b0598ecf67`, only pre-existing untracked `.targets/`. No source edits or product execution.
- Source slices independently read: `state.rs:258-280,335-365,395-450,705-720,915-935,1311-1348,1474-1496,1562-1575,1680-1720`; `damage.rs:17-25`; `parser/sgr.rs:99-165`; `resize_scrollback.rs:181-205`; `replay_determinism.rs:190-212`; render `grid.rs:1242-1310,1575-1640,1778-1865`. Paths resolve within the crates named above. Outlines preceded source slices; unrelated ranges remain unread.
- TERM-ENG-001/002 mechanisms independently verified; TERM-ENG-003 fallback behavior verified but defect claim unsupported; TERM-ENG-004 invariant clamp and homing counterevidence verified; TERM-ENG-005 sampled test/code discrepancy corrected.
- This is not an exhaustive terminal or standards audit. No new tests, payloads, reproductions, benchmarks, network access, installs, commits, or CarryCtx use. Markdown-only validation is recorded in [the product index](README.md).
