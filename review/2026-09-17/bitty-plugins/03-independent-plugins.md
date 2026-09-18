# Independent plugins review — 2026-09-17

## Scope and verdict

Read-only review of the independent `activity`, `palette`, `statusline`,
`file-manager`, `git-panel`, and `wheel` repositories. Product source was not
changed. No CarryCtx operations, agents, installs, fetches, commits, live plugin
activation, real process-spawn tests, or exploit/reproduction code were used.
The sole authored artifact is this report.

Ten original IDs are preserved. The independent second pass withdraws both
application P1 interpretations: PLUG-APP-001 is a P2 contract/integration mismatch;
PLUG-APP-002 is a conditional P2 recovery risk, not proven ordinary host read-I/O
data loss. PLUG-APP-007 is withdrawn as a current-host defect (mock-contract gap).
Other priorities retain first-pass status unless verified below. The **806 Lua
assertions** are first-pass results, not tests rerun by the second reviewer;
mock success does not establish real-host compatibility.

SDK internals, package generation, registry, distribution, and host enforcement
are owned by the neighboring reports. In particular, the root-entry-point
packaging mismatch is already `PLUG-SDK-001` in
[the SDK/template report](01-sdk-and-template.md); it is not counted again here.

## Independent second-pass verification

Second-pass scope is targeted source/contract review, not a repeat of the full
Lua inventory or tests below. All six plugin HEADs and the docs HEAD remain at
the recorded revisions; source trees are clean. `bitty` host evidence is at
`06bc1f45995fd297a3f7324688bc81b0598ecf67` (existing untracked `.targets/`).

- **Corrected:** PLUG-APP-001 and PLUG-APP-002 are reclassified from P1 as
  explained in their bodies; neither establishes current-host data loss.
  PLUG-APP-007's host-defect interpretation is withdrawn, not renumbered.
- **Verified P2:** PLUG-APP-003's clear command resets aggregate state but not
  session pairing; the later close reuses the pre-clear open timestamp
  (`activity/lua/activity/init.lua:229-243,287-297`), contrary to
  `activity/README.md:59-60`'s in-memory reset promise.
- **Verified P2:** PLUG-APP-004's stale cache/result-root behavior in Git Panel
  and File Manager (`init.lua:81-90` and `:85-95,126-151`, respectively).
  PLUG-APP-005's pane-cwd/result-root conflation is also source-confirmed
  (`git-panel/lua/git-panel/init.lua:249-278`). Neither establishes the actual
  subprocess working directory: the Lua bridge ignores the second argument and
  `bitty/.../plugin_runtime/spawn.rs:1032-1035` builds an argv-only request.
  Align that host/plugin seam before claiming pane-local Git execution.
- **Verified P2:** PLUG-APP-008's mounted row bypasses the 128-character total
  limit (`statusline/lua/statusline/init.lua:106-123`, `scene.lua:14-24`,
  `format.lua:177-237`, `README.md:39-48`). Host rendering/rejection was not run.
- **Verified P3 source behavior:** PLUG-APP-009 captures only `pcall` success,
  not the UI result. Its false-return scenario remains unexecuted.
- **Not independently revalidated:** PLUG-APP-006, PLUG-APP-010, the historical
  September 15 matrix, full parsing/allowlist policy, and optimization claims.
  These remain first-pass findings, not independently verified priorities.
- **Docs evidence is limited and dated:**
  `bitty-plugins-docs/docs/plugins/activity/evidence.md:23-34` expressly claims
  no independent verification or shipped compatibility. Git Panel's evidence
  page (`:32-55`) cites older `528a62b`/`e84da34` snapshots and pending host
  enforcement. Current `spawn.rs:992-1015` wires `HostToolsAuthorizer` and
  `:1040-1046` supplies failure handling. The older “bridge absent” prose cannot
  override that implementation; no full authorization audit is implied.

No plugin tests, LuaLS, parsers, live host, GUI, timer scheduler, store failure,
or real subprocess checks ran in this second pass. First-pass test counts and
blocked gates below remain attributed to that pass, not upgraded to integration
coverage. Only review Markdown and its index are changed.

## Repository baselines and inventory

Paths below are **repository-relative within the named repository**. Cross-repo
references name the owning repository separately. HEADs were read locally;
no remote freshness claim is made. All six repositories were clean in the initial
and subsequent status reads (no staged, unstaged, or non-ignored untracked files).
Ignored dependencies were present but incomplete in four repositories.

| Repository   | HEAD                                       | Dirty state | Actual executable role                                                           |
| ------------ | ------------------------------------------ | ----------- | -------------------------------------------------------------------------------- |
| activity     | `104d7ef90704078c94b7809d4f61b44a7c4125f5` | Clean       | Event aggregation, local store, summary/clear commands                           |
| palette      | `47a5981a9436a1f29338b4ce3d1041997051d001` | Clean       | Settings-backed filtered text list, presentation-only overlay, toggle            |
| statusline   | `cab67f6d212a023f781eecf1e6ea2dbadc4fc7c6` | Clean       | Snapshot cwd/title/exit fragments in a statusline row                            |
| file-manager | `cd3da87015a8d7d4adc9196c513bc629024bb5d5` | Clean       | Candidate-path validation and command result shaping; no filesystem I/O          |
| git-panel    | `02cd95efe4f83531e9b0ddec1b79920d57f8497f` | Clean       | Fixed Git command adapters and result parsing against a prospective spawn bridge |
| wheel        | `524ce0766c969ee84add1ce0a485bfa370a36cf9` | Clean       | Scaffold with a notification-only `hello` command                                |

### Wheel discovery

Wheel is a real independent Git repository and plugin package, not a missing
repository or an alternative name for the SDK. Its identity is
`bitty-terminal.wheel`; `repo.toml:3-8` identifies it as the official Agent Harness
plugin, and `AGENTS.md:20-36` explicitly defers harness implementation until Bitty
AI is stable. `lua/wheel/init.lua:24-34` only registers a greeting backed by
`platform.notify`. No harness, provider/model manager, or agent dashboard exists
in this source tree; those are planned/separate responsibilities, not defects.
There is no `tests/` directory. Its manifest and Lua parser gates passed.

The workspace map has no Wheel entry although `wheel/repo.toml` declares one;
this is an umbrella inventory synchronization follow-up, not evidence that the
repository or harness implementation is absent. No umbrella file was edited.

### Read coverage

“Full” means all executable statements were read, not just a filename or test
result. “Sampled” means selected bodies/ranges or assertions were inspected.
Executing a suite does not upgrade its unread files to full coverage.

| Repository   | Production Lua fully read                                                          | Production TS |
| ------------ | ---------------------------------------------------------------------------------- | ------------- |
| activity     | `lua/activity/init.lua`, `aggregate.lua`, `redact.lua`                             | None          |
| palette      | `lua/palette/init.lua`, `filter.lua`, `scene.lua`                                  | None          |
| statusline   | `lua/statusline/init.lua`, `format.lua`, `scene.lua`                               | None          |
| file-manager | `lua/file-manager/init.lua`, `listing.lua`, `scope.lua`, `scene.lua`               | None          |
| git-panel    | `lua/git-panel/init.lua`, `allowlist.lua`, `listing.lua`, `scope.lua`, `scene.lua` | None          |
| wheel        | `lua/wheel/init.lua`                                                               | None          |

All 19 production Lua files (3,689 lines) were read. Tracked TS discovery found
only `commitlint.config.ts` in each repository, not application TS; all six
nine-line configurations were read. All six plugin manifests, package manifests,
and repository AGENTS were read. `wheel/repo.toml`, the complete workspace map,
activity README, and Git Panel MIGRATION were also read.

| Repository   | Tests fully read                                                                                | Sampled supporting material                                                                                  |
| ------------ | ----------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| activity     | `tests/spec/init_spec.lua`, `aggregate_spec.lua`, `tests/support/mock_host.lua`                 | Redaction assertions; `tests/check-lua-luals.mjs:1-150`; justfile gates                                      |
| palette      | All three `tests/spec/*_spec.lua`; `tests/support/mock_host.lua`; `tests/run.lua`               | justfile and known-gap source comments                                                                       |
| statusline   | All three `tests/spec/*_spec.lua`                                                               | Mock UI/snapshot/event bodies at `tests/support/mock_host.lua:170-230`; README behavior/gaps; justfile gates |
| file-manager | `tests/spec/init_spec.lua`, `listing_spec.lua`, `scene_spec.lua`; `tests/support/mock_host.lua` | `tests/check-manifest-negative.mjs:1-160`; justfile gates                                                    |
| git-panel    | `tests/spec/init_spec.lua`, `listing_spec.lua`, `scene_spec.lua`; `tests/support/mock_host.lua` | `tests/check-negative-fixtures.mjs:1-150`; justfile gates                                                    |
| wheel        | No behavior suite present                                                                       | README status/layout; justfile manifest/parser/control recipes                                               |

Unread or not audited in full: file-manager and git-panel scope specs; git-panel
allowlist spec; remaining activity redaction-spec bodies; most TAP helpers and
runner copies; vendored LuaLS definition copies and negative fixtures; remaining
LuaLS wrappers and manifest-report wrappers; negative TOML fixture contents;
lockfile dependency graphs; most CI/hooks, security/contributor/changelog docs,
workflow scripts, and ignored `node_modules`. No supply-chain audit is claimed.

The September 15 `10-plugins.md` report was read in full. Contract consultation
was limited to `bitty-plugins-docs/specifications/plugin-api-v1-lua-surface-rfc.md`
activation, commands, store, UI, timers, and event sections; SDK surface metadata
was searched for corroborating signatures. SDK and plugins-docs guidance was
read. No external documentation or research snapshot code was fetched or run.

The ctxctl skill was loaded. Its MCP outline calls rejected the workspace path;
bounded Read calls were used instead. Repository-local ctxctl CLI later worked
for palette outlines and command-output compression. This tooling limitation
was not interpreted as a project defect.

## Findings with corrected second-pass dispositions

### PLUG-APP-001 — Reclassified P2 — Timer contract/integration mismatch, not proven host rejection

- **Repository / evidence:** activity, `lua/activity/init.lua:103-117`,
  `lua/activity/init.lua:138-162`, `lua/activity/init.lua:304-345`;
  local mock `tests/support/mock_host.lua:257-271` and retry assertions
  `tests/spec/init_spec.lua:222-258`.
- **Cause:** observations call `schedule_flush` after activation; retries also
  create fresh timers from callback/write paths. The accepted Lua surface in
  plugins-docs, `specifications/plugin-api-v1-lua-surface-rfc.md:152-159`, explicitly
  confines timer creation to `init.lua` execution. The activity mock enforces
  only argument types and timer capacity, not the registration window.
- **Qualified impact:** a host enforcing the RFC registration window (as the
  SDK does at `src/mock-host.ts:1667-1668`) rejects these timer creations and
  takes Activity's synchronous fallback, losing coalescing and autonomous retry.
  This is a valid contract mismatch, but not the actual Rust bridge's behavior.
- **Host counter-evidence:** `bitty/crates/bitty-lua/src/host.rs:880-927`
  accepts timer creation without an activation guard. It records the callback
  and returns a numeric handle, so this path does not force the synchronous
  fallback. Targeted runtime search found only activation capture at
  `bitty/crates/bitty-runtime/src/plugin_runtime/mod.rs:692`; timer delivery
  was not established. This may leave delayed persistence unintegrated, but
  neither passing mock retries nor missing search matches prove a working
  scheduler or an executed data-loss scenario. **Original P1 withdrawn.**
- **Fix direction:** choose a persistence policy implementable with the accepted
  v1 lifecycle; do not silently assume runtime timer creation. If runtime timers
  are required, obtain an explicit contract change before changing the plugin.
  Make the local mock close registration after activation.
- **Safe test suggestion:** use an in-memory host that enforces the documented
  activation boundary and check ordinary event persistence, write failure, and
  lifecycle flush outcomes. No real store or timer daemon is necessary.

### PLUG-APP-002 — Reclassified conditional P2 — Read-error recovery risk; host-I/O P1 withdrawn

- **Repository / evidence:** activity, `lua/activity/aggregate.lua:188-200`,
  `lua/activity/aggregate.lua:230-238`; caller
  `lua/activity/init.lua:79`, `lua/activity/init.lua:120-131`.
- **Cause:** a thrown `store.get` returns a writable `new_state`, indistinguishable
  from a confirmed absent key. A later successful `store.set` serializes that
  new state over the existing value. The newer-format protection is never
  reached when the initial read failed.
- **Conditional impact:** if `store.get` throws but a later write succeeds,
  Activity can overwrite unread prior history. The local fallback is proven by
  source; ordinary transient disk-read recovery is **not** proven reachable.
- **Host counter-evidence:** `bitty` loads persistence before executing Lua
  (`crates/bitty-runtime/src/plugin_runtime/mod.rs:608-614`); load errors abort
  activation. `store.rs:50-84` loads once and `get` clones an in-memory value;
  `services.rs:198-200` wraps that read in `Ok`, with no per-get disk I/O.
  The bridge can still return a post-call deadline/reentrancy error
  (`crates/bitty-lua/src/host.rs:448-468,735-761`). No such error followed by
  write recovery was demonstrated. Retain a defensive P2 recovery/test gap,
  not an unconditional P1 history-loss claim; do not equate a thrown mock read
  with the runtime's persistence-load error path.
- **Fix direction:** retain a distinct unavailable/uninitialized state and refuse
  replacement until a read succeeds or the user explicitly purges. Reconcile
  bounded pending observations only after loading the prior state successfully.
- **Safe test suggestion:** an in-memory store with existing aggregates and a
  temporary read failure should preserve the original value through observation
  and recovery. Include an unread newer-format value in the same policy tests.

### PLUG-APP-003 — P2 — Activity clear retains pre-purge session pairing state

- **Repository / evidence:** activity, `lua/activity/init.lua:171-172`,
  `lua/activity/init.lua:229-243`, `lua/activity/init.lua:287-297`;
  purge promise `README.md:59-60`.
- **Cause:** clear deletes the persisted key, cancels the pending flush, and
  replaces `state`, but leaves `open_sessions` and `open_session_count` intact.
  It also leaves `write_errors` and retry bookkeeping unchanged.
- **Impact:** a later close can record a duration spanning activity from before
  the explicit purge. Old open sessions continue occupying the bounded pairing
  table, and a summary can continue reporting pre-purge write failures despite
  a successful clear. The in-memory reset is incomplete.
- **Fix direction:** reset pairing and persistence bookkeeping as part of the
  successful clear transaction; preserve existing state if deletion fails.
- **Safe test suggestion:** use the existing virtual clock/mock store to verify
  that a successful purge removes all pairing history and old diagnostics, while
  a failed purge leaves the prior state intact.

### PLUG-APP-004 — P2 — Cached cwd leaks across focused-terminal changes

- **Repositories / evidence:** git-panel, `lua/git-panel/init.lua:81-90`,
  `lua/git-panel/init.lua:130-132`, `lua/git-panel/init.lua:275-278`;
  file-manager, `lua/file-manager/init.lua:85-94`,
  `lua/file-manager/init.lua:123-135`, `lua/file-manager/init.lua:145-151`.
- **Cause:** a successful snapshot without cwd does not clear `cache.cwd`; neither
  cache records terminal/runtime/generation identity. A pane with no shell cwd
  therefore inherits the last pane's cwd. Git supplies it as a second spawn
  argument and uses it for result paths; file-manager uses it as its implicit
  root when no explicit root is supplied. The current Rust spawn bridge consumes
  only argv, not that cwd argument (`bitty/crates/bitty-lua/src/host.rs:865-871`),
  so a wrong actual subprocess cwd is not proved by this cache trace.
- **Impact:** commands can report the previous pane's repository or resolve a
  candidate against the wrong directory. In file-manager this is incorrect
  result shaping, not unauthorized filesystem I/O: that plugin performs none.
  A genuinely failed read and a successful snapshot with absent cwd must not be
  conflated.
- **Fix direction:** bind cached state to snapshot identity and invalidate it on
  identity changes or successful snapshots lacking required metadata. Require
  current cwd for Git commands; retain explicit-root headless file-manager use.
- **Safe test suggestion:** mock focused-terminal changes with and without cwd,
  including an unavailable snapshot, and assert identity-correct roots or a
  clear unavailable result rather than reuse of another terminal's directory.

### PLUG-APP-005 — P2 — Git status uses pane cwd as repository root

- **Repository / evidence:** git-panel, `lua/git-panel/init.lua:64-85`,
  `lua/git-panel/init.lua:249-278`, `lua/git-panel/init.lua:328-330`,
  `lua/git-panel/listing.lua:238-244`.
- **Cause:** semantic cwd is passed both as the working directory for Git and
  as the root for joining porcelain paths. Porcelain v1 paths are repository-root
  relative, as the plugin's own comments state; a terminal cwd can instead be a
  repository subdirectory. No root-discovery step exists.
- **Impact:** normal status output viewed from a nested working directory is
  assigned incorrect paths (including duplicated subdirectory segments). The
  September 15 fix admits relative paths but has not established the right base.
- **Fix direction:** resolve the repository root through an accepted read-only
  host/tool contract and keep it separate from terminal cwd; do not infer one
  from the other. Coordinate this with the still-pending spawn bridge.
- **Safe test suggestion:** an in-memory adapter fixture should distinguish pane
  cwd, repository root, and root-relative status records, asserting correct
  results for top-level and nested working directories.

### PLUG-APP-006 — P2 — Git porcelain decoder still changes legitimate path names

- **Repository / evidence:** git-panel, `lua/git-panel/init.lua:171-203`,
  `lua/git-panel/init.lua:220-242`; existing narrow coverage
  `tests/spec/init_spec.lua:139-166`.
- **Cause:** rename parsing splits at the last textual arrow separator without
  tracking quoting, so a destination containing that text is split incorrectly.
  Quoted paths are decoded in two substitutions: octal decoding first, then
  backslash decoding over the transformed string. That can reinterpret bytes
  produced by the first pass or match within an escaped backslash sequence.
  The generic whitespace pattern also does not implement the fixed record
  delimiter as a grammar.
- **Impact:** ordinary unusual filenames can be dropped or reported with the
  wrong destination/name. Existing tests cover simple quoted names and simple
  renames but not lossless decoding of the supported Git representation.
- **Fix direction:** adopt an unambiguous, documented machine-output format when
  the accepted tool policy allows it, or implement a single-pass quote-aware
  parser. Preserve path bytes separately from display transformations.
- **Safe test suggestion:** add pure parser fixtures for quoted destinations,
  embedded separator text, literal backslashes, and supported escapes; assert
  byte-preserving decode. Do not create filesystem fixtures or execute commands.

### PLUG-APP-007 — Withdrawn as host defect — Mock result/error contract mismatch

- **Repository / evidence:** git-panel, `lua/git-panel/init.lua:132-154`,
  command consumers `lua/git-panel/init.lua:275-319`;
  mock result contract `tests/support/mock_host.lua:136`.
- **Cause:** `spawn_git` checks only that the result is a table with string
  `output`; the `status` member returned by its own modeled adapter is ignored.
  An unsuccessful Git invocation with empty output therefore becomes an ordinary
  empty listing or zero-count open result.
- **Second-pass correction:** the mock's unsuccessful-result table is not the
  current production backend contract. `bitty/crates/bitty-runtime/src/
plugin_runtime/spawn.rs:998-1006,1040-1046` maps non-completed outcomes to
  bridge errors before returning a success table; nonzero exits become
  `E_SPAWN_FAILED`. `bitty/crates/bitty-lua/src/host.rs:334-339,865-874`
  describes `exit_code`, not the mock's `status`, and propagates those errors.
  `spawn_git` does not catch them. Thus the claim that current-host command
  failure becomes successful empty data is withdrawn. Retain only a mock
  alignment/test gap; no real process was run.
- **Fix direction:** settle the result contract with the bridge owner, require a
  successful outcome before parsing, and propagate a bounded diagnostic for
  failure instead of reporting successful empty data.
- **Safe test suggestion:** mock successful empty output, unsuccessful empty
  output, and unsuccessful partial output, checking that only the first is
  accepted as an empty successful result.

### PLUG-APP-008 — P2 — Statusline's actual UI path bypasses its total text limit

- **Repository / evidence:** statusline, `lua/statusline/init.lua:107-117`,
  `lua/statusline/scene.lua:14-23`, `lua/statusline/format.lua:195-212`,
  `lua/statusline/format.lua:229-236`; declared behavior `README.md:44-48`.
- **Cause:** the entry point calls `format.components` and `scene.row`, not
  `format.render`, which is where the 128-character total limit is enforced.
  The row inserts the settings-provided separator verbatim. Per-value bounds
  alone do not bound prefixes, separators, and the combined row.
- **Impact:** even two maximum-length values plus prefixes exceed the stated total
  limit. Long separators can further increase the emitted scene, producing
  oversized presentation or a host rejection with a frozen last-good row. The
  pure `format.render` tests do not validate the mounted scene's aggregate text.
- **Fix direction:** apply one cumulative code-point budget at scene construction,
  including prefixes and separators, and bound the separator before allocation.
  Keep the helper and the production row on the same policy.
- **Safe test suggestion:** inspect text across all children of the mock-mounted
  row for ordinary long cwd/title values, multibyte values, and custom separators;
  assert the documented total bound and valid character boundaries.

### PLUG-APP-009 — P3 — Statusline treats a false UI update as success

- **Repository / evidence:** statusline, `lua/statusline/init.lua:114-123`;
  contract in plugins-docs,
  `specifications/plugin-api-v1-lua-surface-rfc.md:332-342`;
  local mock `tests/support/mock_host.lua:198-204`.
- **Cause:** the single `updated` variable captures the success boolean from
  `pcall`, not `bitty.ui.update`'s return value. A non-throwing false return
  therefore replaces `last_good` even though nothing was presented.
- **Impact:** the exposed `refresh` result and subsequent fallback no longer
  represent the last successfully rendered components. Existing tests cover
  thrown capability errors, not the documented false-return path.
- **Fix direction:** inspect both protected-call success and the update result;
  preserve the prior cache when the host declines the update. Do not attempt to
  remount outside the registration window.
- **Safe test suggestion:** have the in-memory UI return false for an update and
  check that cached and returned last-good components remain unchanged.

### PLUG-APP-010 — P3 — File-manager preview returns contradictory directory fields

- **Repository / evidence:** file-manager, `lua/file-manager/init.lua:166-175`,
  `lua/file-manager/listing.lua:38-45`, `lua/file-manager/listing.lua:60-73`,
  `lua/file-manager/scope.lua:143-162`;
  existing assertion `tests/spec/init_spec.lua:178-179`.
- **Cause:** preview resolves a relative directory path before passing it to
  `entry_from_path`. Resolution removes its trailing separator; inference then
  sets `kind` to `file`. Preview separately derives `is_dir` from the original
  path and sets it true, but never corrects `kind`.
- **Impact:** consumers receive conflicting metadata for the same preview.
  Equivalent absolute and relative directory inputs can disagree. This is a
  command-result defect, not missing filesystem enumeration or panel rendering.
- **Fix direction:** infer kind once from the original candidate or pass an
  explicit consistent kind after resolution; normalize presentation metadata
  together.
- **Safe test suggestion:** compare the complete preview result for equivalent
  relative and absolute directory candidates, including both `kind` and `is_dir`.

## Recheck of September 15 plugin findings

The old numbering below refers to `review/2026-09-15/10-plugins.md`. Old priorities
and security labels are not inherited without reassessing current code and
reachability.

| Old finding                                | Current assessment                                                                                                                                                                                                                                                                                                                                                               |
| ------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| R4: tools.git validator fork               | Transitional validators are removed; all six packages pin SDK commit `c3fa9b0`. Git Panel's local SDK gate is unavailable, so successful current tools.git validation was not re-established here. SDK report owns schema analysis.                                                                                                                                              |
| R5: permissive Git flags                   | The cited allow-unknown-flags implementation is gone: `lua/git-panel/allowlist.lua:45-58,150-187` now uses a closed flag set and a dense sequence check. Not a blanket read-only certification: command-specific operand semantics and host mediation still need contract review before exposing caller-controlled invocations. Current command handlers construct fixed arrays. |
| R6: relative status dropped                | Simple root-relative records are now accepted and tested. Partially resolved: the cwd/root confusion remains PLUG-APP-005.                                                                                                                                                                                                                                                       |
| R12: statusline event exceptions           | Fixed for snapshot/compose/update throws: `lua/statusline/init.lua:106-138` contains them. Boolean failure handling is a separate residual, PLUG-APP-009. Activation mount denial intentionally fails closed and is tested; that is not the old dispatch defect.                                                                                                                 |
| R13: numeric settings                      | Fixed for production component count/value-length options by `format.lua:152-192`. Total row/separator budgeting remains PLUG-APP-008.                                                                                                                                                                                                                                           |
| R14: full zone scan                        | Fixed in statusline by the newest-256 window at `format.lua:89-140`. No timing benchmark was run.                                                                                                                                                                                                                                                                                |
| R15: substring traversal checks            | Production root-relative handling uses segments; legitimate consecutive dots are no longer universally rejected. Historical Git grant helper remains, but traversal false-positive claim is no longer accurate.                                                                                                                                                                  |
| R16: unused file-manager payload cap       | No longer unused: `listing.lua:86-95` accounts for path plus name bytes. This is not proof of an 8 KiB serialized command result; see uncertainties. Old scene builders were deliberately removed.                                                                                                                                                                               |
| R17: unbounded Git parsing                 | Parsing now receives at most 8 KiB and local cuts discard a partial line, `init.lua:136-153`; count limits remain. Host capture allocation and host-originated truncation signaling are unresolved, not proven fixed by local slicing.                                                                                                                                           |
| R18: porcelain gaps                        | Simple relative paths, quoted spaces, rename destinations, and unmerged status pairs are covered. Lossless path decoding remains incomplete: PLUG-APP-006.                                                                                                                                                                                                                       |
| R19: hash-sorted commits                   | Fixed: `listing.lua:284-302` preserves input order; hash sorting is a separate unused display helper. Requesting ten commits under a maximum of 64 is a choice, not a defect.                                                                                                                                                                                                    |
| R20: palette input count                   | Fixed: `filter.lua:103-118` examines at most 1,024 candidates and emits at most 128 rows; a 10,000-candidate test passes. Per-string byte work and discarded truncation stats remain optimization topics.                                                                                                                                                                        |
| R21: palette schemas/update throws         | Schemas are present and host UI calls are guarded. Missing schemas were not intrinsically invalid anyway: accepted command schemas are optional. Toggle-state behavior on a rejected close remains a disclosed limitation.                                                                                                                                                       |
| R22: activity payload types                | The cited nil/string/non-finite cases are guarded and tested. Fractional identities still pass finite-number checks, but accepted host payloads use integers; no new live-input security claim is made.                                                                                                                                                                          |
| R23: write errors/retention/no-op summary  | Explicit finite settings win over stored retention, successful writes reset errors, and no-op summaries do not dirty the state. Clear bookkeeping remains PLUG-APP-003; timer-backed recovery remains PLUG-APP-001.                                                                                                                                                              |
| R24: negative fixture automation           | Both justfiles now include required negative-fixture gates with positive controls and expected diagnostics. Gates were read but not executed because their pinned SDK dependencies are missing.                                                                                                                                                                                  |
| R25: template command shape                | Wheel uses `id`, `title`, and `run`; the obsolete `name` shape is absent. Generator/package issues belong to the SDK/template report.                                                                                                                                                                                                                                            |
| R27 / plugin part of R8: version validator | No independent transitional validator remains in these repositories; SDK report owns current version-range correctness. No duplicate application finding.                                                                                                                                                                                                                        |
| R28: statusline reactive/type drift        | Fixed: production subscriptions use `format.REACTIVE_EVENTS`, match the manifest, and `has_status` uses the same numeric exit-code rule. Tests explicitly cover these cases.                                                                                                                                                                                                     |
| R29: file-manager boundaries/dead code     | Non-table listing guard and basic equal-source/destination rejection exist; root-parameterized path limits replace the old unused predicate. Directory representation is still inconsistent in preview: PLUG-APP-010. Lexical normalization is not a realpath guarantee.                                                                                                         |
| R30: branch validation/bounds              | Cited reflog-marker/leading-dash omissions are addressed; default branch rows and ingestion share 32. This is still a simplified display validator, not full Git reference validation.                                                                                                                                                                                           |
| R31: bridge comments/error contracts       | Git Panel MIGRATION explicitly records missing spawn/tools enforcement. File-manager explicitly defers I/O/panels. Missing planned surfaces are not defects. Exact future spawn result/error semantics remain a contract gap.                                                                                                                                                    |

Registry-only R1–R3, R7, R9–R11, and R26 are outside this report and belong to
[registry/distribution](02-registry-and-distribution.md). Registry aspects of R8
are likewise not duplicated.

## Optimization and maintainability opportunities

These are separate from confirmed application defects and carry no PLUG-APP ID.

- **Bound work, not just output.** Palette lowercases each full candidate before
  matching and character counting scans the full matching string. With at most
  1,024 candidates, time is still O(B) in examined string bytes, and temporary
  memory can be O(L) for the longest candidate. A bounded byte scan/display
  representation would make the policy more explicit. No hostile workload or
  latency benchmark was run.
- **Snapshot scanning.** File-manager and Git Panel search all zones newest-first
  in `snapshot_cwd` (O(Z) time, O(1) auxiliary space). Statusline's bounded window
  is a useful precedent. A protected call contains errors, not execution cost.
- **File listing work.** `file-manager/listing.lua:87-104` bounds accumulated valid
  output bytes but still examines duplicate/invalid candidates until input ends.
  Time is O(B + K log K), where B is examined path bytes and K accepted entries
  before count truncation; auxiliary space is O(K) plus stored path bytes.
  Add an independent examined-candidate budget if large external candidate
  arrays are to be supported.
- **Expose truncation meaningfully.** Palette discards the filter statistics in
  `init.lua:85`; Git Panel stores a last-spawn bit separately from command
  results, and `open` overwrites branch truncation with the subsequent status
  call's flag. A bounded partial-result indicator would avoid implying complete
  data. Keep this compatible with the eventual result schemas.
- **Mocks need boundary coverage.** Local stubs do not faithfully model generation
  teardown, registration windows, immutable snapshots, all budgets, or the real
  bridge. The Git mock calls the same allowlist module as production, so it is
  not an independent authorization oracle. Keep pure tests, but add narrowly
  scoped accepted-contract conformance at the integration boundary.
- **Avoid dead presentation claims.** Git Panel's remaining branch scene is still
  not imported by `init.lua`. It is tested utility code, not a wired panel.
  Wheel needs behavior/conformance tests when functionality is authorized, not
  an invented harness now.

## Uncertainties and deliberate limitations

- **Shipping and integration:** repository-local passing tests do not prove
  installability or an available UI/process bridge. Palette command enumeration,
  keyboard picker interactions, status providers, statusline Git/task fragments,
  file-manager filesystem I/O/panel rendering, Git panel mounting, and Wheel
  harness logic are explicitly deferred. They are not counted as missing-feature
  defects. Package-root entry wiring is covered only by PLUG-SDK-001 elsewhere.
- **Storage semantics:** activity prunes cwd labels only when summary runs;
  counters and collapsed totals are cumulative, despite summary text saying
  “window.” README documents summary-driven pruning. Clarify wall-clock label
  retention versus lifetime totals before asserting an automatic-retention
  requirement or treating every retained counter as a privacy violation.
  Successful purge is separately covered above.
- **Payload sizing:** file-manager counts raw path/name bytes, not serialized
  keys, metadata, separators, and escaping. Git output may expand when joined
  to long roots and encoded as command results. The precise command-result byte
  budget and error behavior were not traced end to end; no full serialized-bound
  compliance claim is made.
- **Spawn capture:** plugin truncation occurs after the host has returned a
  string. The local Git mock itself can cut at 8 KiB without reporting a flag;
  a real bridge must define bounded capture, partial-record handling, exit status,
  and truncation signals. Current plugin slicing alone proves none of those
  host properties. No real subprocess was launched by plugin code.
- **Least privilege:** Git Panel still declares panel and filesystem grants that
  its current Lua does not directly exercise; filesystem authority may be needed
  by future mediated Git execution. Its literal home-project scope is not an
  arbitrary-root authorization guarantee. Do not expand grants based on the
  root-parameterized presentation helper; host policy remains authoritative.
- **Unicode/path portability:** POSIX-style slash and tilde handling does not
  establish Windows drive/UNC support or realpath containment. Invalid UTF-8
  fallback behavior and all special filesystem-name cases were not exhaustively
  tested. Activity's basename reduction is not anonymization of a sensitive
  basename; current documentation expressly permits storing bounded directory
  labels, so this is not presented as a new policy violation.
- **Palette close failure:** toggled state changes before a render succeeds;
  its test intentionally preserves the old visible scene after a rejected close.
  Decide whether the command reports desired or actually presented state.
  This report does not equate a presentation-only overlay with a focus trap.
- **Wheel metadata:** manifest license is unset while package metadata says MIT;
  license is optional at this scaffold stage. Workspace-map omission is a
  bookkeeping gap. Neither is proof of a runtime defect.

## Validation actually performed

Existing recipes were read before execution. Tests were run from their owning
repository with the already installed Lua interpreter and available local tools;
no dependency installation or alternative SDK checkout was substituted.

| Repository                         | Command/check                                                               | Observed result                                                                                                 |
| ---------------------------------- | --------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| activity                           | `just test-lua manifest lua` through ctxctl                                 | Lua 169 passed, 0 failed; manifest stopped at missing `bitty-plugin-lint`; parser not reached                   |
| palette                            | `just test-lua manifest lua` through ctxctl                                 | Lua 83 passed, 0 failed; manifest valid with 0 errors/0 warnings; all three Lua parser commands passed          |
| statusline                         | `just test-lua manifest lua` through ctxctl                                 | Lua 143 passed, 0 failed; manifest stopped at missing `bitty-plugin-lint`; parser not reached                   |
| file-manager                       | `just test-lua manifest lua` through ctxctl                                 | Lua 160 passed, 0 failed; manifest stopped at missing `bitty-plugin-lint`; parser not reached                   |
| git-panel                          | `just test-lua manifest lua` through ctxctl                                 | Lua 251 passed, 0 failed; dependency recipe stopped at missing `bitty-plugin-lint`; manifest/parser not reached |
| wheel                              | `just manifest lua` through ctxctl                                          | Manifest valid with 0 errors/0 warnings; entry-point Lua parser passed                                          |
| activity, statusline, file-manager | Separate `just lua`                                                         | Each failed closed: installed `luaparse` script unavailable                                                     |
| All six                            | `git rev-parse HEAD`, `git status --porcelain=v1`, tracked Lua/TS discovery | Revisions above; source repositories clean                                                                      |

The ctxctl wrapper printed underlying recipe errors; those checks are recorded
as blocked/failed, not passed because an outer tool invocation returned output.
Missing local dependencies are environment limitations, not confirmed source
bugs. Lua behavior execution also parses required modules, but is not a substitute
for the unavailable pinned Lua 5.1 grammar gate.

Not run: full `just check`, LuaLS/typecheck (wrappers create scratch workspaces),
negative-manifest suites, Git Panel manifest JSON test, Wheel parser negative
control, supply-chain tools, CI, real host reload/suspend/dispose, GUI rendering,
real Git process adapters, packaging/install tests, cross-platform runs, or
performance measurements. No custom regression or reproduction code was written
or executed. Suggested tests above are defensive, pure/in-memory follow-ups.

Report validation: repository Markdownlint configuration was read; the scoped
report is checked with `markdownlint-cli2` only, without formatting other files.
The research repository already had an untracked September 17 campaign directory;
other reports were preserved. Final report lint and source status are recorded
in the completion response.
