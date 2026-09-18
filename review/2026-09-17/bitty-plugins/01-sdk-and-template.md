# SDK and template read-only review — 2026-09-17

## Summary and scope

Reviewed `bitty-plugin-sdk` and `bitty-plugin-template` for public-contract
consistency, manifest validation, mock-host lifecycle/events/services,
scaffolding, and test fidelity. Production code was not changed. No CarryCtx
commands, subagents, installs, network fetches, commits, exploit runs, or
vulnerability reproduction programs were used. Only this report was written.
Reading committed `.carryctx` guidance did not access or mutate workflow state.

The first pass recorded 12 IDs. The independent second pass **withdraws the
P1 interpretation of PLUG-SDK-001** and reclassifies it as P2 layout/contract
drift; PLUG-SDK-006 is an acknowledged conformance gap, not a newly discovered
implementation defect. No SDK/template P1 remains. Other original priorities
are retained, not all independently reconfirmed; see the second-pass ledger.
All findings are static deductions, not demonstrated production vulnerabilities.
First-pass test results below are historical, not second-pass execution evidence.

P1 means a fundamental author workflow is broken; P2 means a meaningful
correctness, fidelity, or validation defect; P3 means lower-impact tooling
feedback inconsistency. Acknowledged SDK limitations are explicitly identified
rather than presented as newly discovered host defects.

## Independent second-pass verification

This section supersedes first-pass severity/coverage wording where inconsistent.
Only the three review reports and their sibling README index are edited.
No product tests, generators, installs, network, CarryCtx, agents, or reproduction
programs ran in this pass. The later read/test inventory describes the first
pass only; it must not be attributed to the independent verifier.

- **Revision drift:** SDK HEAD is now
  `84d41ae96d6b733fcc57be6274935cef12c6590b`, one commit after the recorded
  `3e9ebb5`. Local `git log`/`git diff --stat` show only `package.json` and
  `bun.lock` changed (TypeScript 5.7.2 to 7.0.2). Finding source bodies are
  unchanged, but earlier typecheck/test results do not validate this dependency
  revision. Template/docs HEADs remain those below; source trees are clean.
- **PLUG-SDK-001:** P1 withdrawn; verified P2 source-layout/contract drift,
  not a current-host entry-selection failure. See corrected body.
- **PLUG-SDK-002:** verified P2 package-metadata/documented-import defect:
  `package.json:1-22`, `docs/mock-host.md:42-45`, and root-index discovery.
  No consumer install/import experiment was performed.
- **PLUG-SDK-003, PLUG-SDK-004:** verified P2 mock/RFC fidelity mismatches:
  `mock-host.ts:793-868,933-1036,1390-1437,1605-1616` versus Lua activation
  and runtime suspension contracts. These are not demonstrated host bypasses.
- **PLUG-SDK-005:** verified P2 mock cancellation defect from the due-snapshot
  loop (`:982-999`) and cancellation mutation (`:1711-1722`); not executed.
- **PLUG-SDK-006:** reclassified as acknowledged P2 conformance coverage gap;
  declaration/dispatch bodies (`:1169-1196,1513-1618`) agree with the explicit
  limitation in `docs/mock-host.md:398-401`. Do not call this a new host defect
  or mistake a green fixture for enforcement of unimplemented static schemas.
- **PLUG-SDK-008:** verified P2 mock value-isolation defect; settings return the
  stored reference (`:1262-1269`), command/service calls pass values directly
  (`:943-957,1605-1616`), contrary to runtime RFC `:193-196`.
- **Not independently revalidated:** PLUG-SDK-007, PLUG-SDK-009 through
  PLUG-SDK-012, historical September 15 dispositions, schema recursion limits,
  generator CLI edge cases, and performance observations retain first-pass
  status only. No blanket endorsement of all twelve IDs is intended.

## Baselines and authority

- `bitty-plugin-sdk`: HEAD `3e9ebb53784ed6ff089dedcc0b62938d768c92b8`.
  `git status --porcelain` was empty initially and at the pre-report recheck.
- `bitty-plugin-template`: HEAD `9fe7998bccdb751684f7586878a20543c6b9162d`.
  `git status --porcelain` was empty initially and at the pre-report recheck.
- `bitty-plugins-docs`: HEAD `9fdbcd02af835d23868a7c15c5ce12f1efcd093a`;
  clean when consulted.
- `research`: HEAD `26a46ed8c15aa4e714ca9add4ff4c07bea02ff22`;
  existing untracked baseline `review/2026-09-16/` was preserved. Concurrent
  agents may add other reports after this observation.
- The existing SDK `.worktrees/ctx-0040-feat-tools-git-conformance/` was noticed
  during inventory discovery but excluded from code analysis. Findings refer
  to the primary checkout, not unfinished work in another agent's worktree.

Read workspace and repository `AGENTS.md` for both targets, the docs repository,
and research; loaded the `ctxctl-core` skill; read target security rules and
compatibility/developer-experience reviewer guidance. The user's read-only,
no-CarryCtx, no-delegation scope overrides the default workflow instructions.
The CtxCtl MCP outline calls rejected the workspace path; local `ctxctl outline`
worked from the SDK repository. Subsequent file slices used the read tool.

Contract evidence consulted:

- `bitty-plugins-docs/specifications/plugin-api-v1-lua-surface-rfc.md:126-548`:
  read-only module, entry point, registrations, commands, storage, services,
  events, snapshot, and compatibility. Read lines 1–600; final reference and
  resolution-table tail not read.
- `bitty-plugins-docs/specifications/plugin-host-runtime-rfc.md:184-345`:
  immutable marshalling, atomic activation, suspended registrations, source
  layout, and local-path loading. Read lines 1–345, not the remainder.
- `research/summary/013.md` and `research/summary/040.md`: fully read. These
  support isolated VMs, value-based services, explicit capabilities, and the
  distinction between accepted dependencies and proposed extension manifests.
  Proposed contribution syntax was not treated as a missing SDK feature.
- `research/review/2026-09-15/09-sdk-template.md`: fully re-read and reconciled
  below. Historical severities and assumptions were not inherited automatically.

Paths in findings are repository-relative; the repository is named separately.

## Findings and contract gaps with corrected dispositions

### PLUG-SDK-001 — Reclassified P2 — Layout contract drift; loader-blocking P1 withdrawn

- **Verified template:** `scripts/generate-plugin.mjs:637-672` generates
  `lua/<id suffix>/init.lua`. It does not generate package-root `init.lua`.
- **Normative discrepancy:** plugins-docs
  `specifications/plugin-api-v1-lua-surface-rfc.md:150-163` and
  `specifications/plugin-host-runtime-rfc.md:203-218` still specify the package
  root. The template explicitly labels its layout candidate at
  `template/README.md:10-17`.
- **Actual host counter-evidence:** `bitty` at `06bc1f4`,
  `crates/bitty-runtime/src/plugin_runtime/resolution.rs:469-478`, chooses the
  package's `lua/` directory as module root when present. `mod.rs:1091-1102`
  tries both module-root `init.lua` and `<id suffix>/init.lua`. The installer
  uses those helpers at `package.rs:274-280`, and activation uses the resolver
  at `mod.rs:559-573`. The generated nested layout is therefore recognized;
  a missing package-root forwarder does **not** establish loading failure.
- **Disposition:** withdraw the fundamental-author-workflow P1. Retain P2 for
  the explicit source-layout/RFC mismatch and missing generated-host integration
  evidence, not for proven inability to activate. Entry discovery alone does
  not establish manifest compatibility, successful activation, or publication.
- **Remediation:** reconcile package root versus module root in the owning
  contract and scaffold, then add a normal generated-package integration gate.
  Do not add a forwarder blindly: `lua/` currently takes precedence, so a
  package-root file alone need not become the executed entry.

### PLUG-SDK-002 — P2 — SDK package has no resolvable public module entry

- **Repository / locations:** `bitty-plugin-sdk`, `package.json:1-22`,
  `src/index.ts:12-96`, `docs/mock-host.md:44-46`,
  `tests/packaging.test.ts:20-36`.
- **Cause:** package metadata defines the CLI bin but no `exports`, `main`,
  `module`, or `types` entry pointing to `src/index.ts`; there is no root index.
  Packaging tests cover only the executable.
- **Impact:** the documented bare import of `MockHost` from `bitty-plugin-sdk`
  has no package entry to resolve. The CLI packaging repair did not make the
  library API consumable through its documented import.
- **Proposed fix:** declare a supported Bun/TypeScript package entry and its
  types, with explicit subpath policy where needed.
- **Safe regression tests:** in a normal consumer fixture, import the package
  by name and construct the mock; check both runtime resolution and TypeScript
  resolution independently of CLI-bin tests. No install was attempted here.

### PLUG-SDK-003 — P2 — UI mounting escapes the activation-only registration rule

- **Repository / locations:** `bitty-plugin-sdk`, `src/mock-host.ts:1390-1437`,
  `conformance/cases/05-lifecycle-registration.json:38-65`.
- **Cause:** `uiMount` calls `assertAlive`, not `assertRegistrationOpen`.
  `assertAlive` accepts both active and suspended states. The lifecycle fixture
  actively expects a successful mount after `end-activation`.
- **Contract:** Lua surface RFC `:150-159` and runtime RFC `:203-218` include
  `ui.mount` among activation-only registrations. Updating an existing block
  is distinct from registering a new one.
- **Impact:** plugins that register new UI from callbacks can receive a green
  SDK conformance result but violate the accepted activation contract.
- **Proposed fix:** apply the registration-window guard to mount, leaving update
  subject to its separately defined lifetime and capability rules.
- **Safe regression tests:** mount during activation succeeds; mounting after
  activation or suspension fails with the registration diagnostic; updating a
  live block remains independently tested. Correct the positive late-mount case.

### PLUG-SDK-004 — P2 — Suspension does not detach executable registrations

- **Repository / locations:** `bitty-plugin-sdk`, `src/mock-host.ts:820-830`,
  `src/mock-host.ts:861-868`, `src/mock-host.ts:933-999`,
  `src/mock-host.ts:1028-1036`, `src/mock-host.ts:1605-1616`.
- **Cause:** suspension only changes the state and delivers a lifecycle event.
  Command dispatch, event publication, task draining, and timer advancement
  accept suspended state. Service closures check generation/alive but not
  suspension, and suspension does not invalidate their records.
- **Contract:** runtime RFC `:237-247` says suspended registrations are detached;
  Lua surface RFC `:437-439` explicitly makes suspended providers unavailable
  with `E_SERVICE_GONE`.
- **Impact:** tests can continue running callbacks and invoking suspended
  services when the accepted host would stop delivery. This weakens the test
  model of containment and cleanup, not evidence of a live host bypass.
- **Proposed fix:** distinguish active execution from lifecycle cleanup and
  detach ordinary dispatch while suspended; preserve grants/store as required.
  Check provider availability in previously resolved service closures.
- **Safe regression tests:** normal commands, observation handlers, task/timer
  callbacks, and resolved services must not execute after suspension; the
  suspended/disposed lifecycle events and retained store data remain testable.

### PLUG-SDK-005 — P2 — Timer cancellation during delivery is ignored

- **Repository / location:** `bitty-plugin-sdk`, `src/mock-host.ts:982-999`.
- **Cause:** `advanceTimers` snapshots eligible timers once. Its delivery loop
  does not recheck cancellation, record membership, or generation before firing
  each queued record.
- **Impact:** an earlier callback can successfully cancel a later due timer,
  yet that later callback still fires. A batch can also retain records after
  lifecycle changes initiated by host-side test callbacks. The cancellation
  return value and observed delivery disagree.
- **Proposed fix:** revalidate each queued record immediately before execution,
  including cancellation, generation, and current eligibility.
- **Safe regression tests:** use two ordinary timers due in the same advance;
  the first cancels the second, and only the first increments a counter.
  Separately verify callback-triggered disposal ends the remaining delivery.

### PLUG-SDK-006 — Reclassified P2 conformance gap — Static schemas are collected but unused

- **Repository / locations:** `bitty-plugin-sdk`,
  `src/manifest-model.ts:131-200`, `src/mock-host.ts:1169-1196`,
  `src/mock-host.ts:1513-1618`, `docs/mock-host.md:398-401`.
- **Cause:** the model stores `commandSchemas` and `providedServiceSchemas`,
  but registration and service invocation never consult them. Service methods
  are invoked directly; command registration validates only dynamic schemas.
- **Contract:** Lua surface RFC `:213-222` requires static/dynamic equivalence;
  `:422-436` requires interface validation and limits schema-validating
  consumers to table-form providers.
- **Impact:** conflicting static/dynamic command contracts and invalid service
  arguments/results receive false-positive mock success. The existing
  table-command fixture duplicates schemas on both sides, so it cannot prove
  static enforcement. **This is an acknowledged implementation gap**, not a
  newly discovered limitation; calling it host-bridge work does not make the
  mock's permissive result contract-faithful.
- **Proposed fix:** enforce canonical equivalence and service schemas, or fail
  explicitly as unsupported instead of presenting permissive conformance.
- **Safe regression tests:** compare equivalent reordered schemas, a benign
  mismatched schema, missing dynamic metadata, table-form service argument and
  result type errors, and a string-form provider requested by a validating
  consumer. Assert no callback runs on rejected arguments.

### PLUG-SDK-007 — P2 — Stored nested values are not recursively type-validated

- **Repository / locations:** `bitty-plugin-sdk`, `src/mock-host.ts:247-291`,
  `src/mock-host.ts:330-368`, `src/mock-host.ts:1272-1283`,
  `src/mock-host.ts:1296-1335`.
- **Cause:** `storeValueProblem` checks non-JSON types and finite numbers only
  at the root. The structural walk counts descendants and detects cycles but
  does not validate their scalar types or reject nested non-plain objects.
  JSON serialization can silently transform/drop invalid descendants or throw
  outside the typed diagnostic path.
- **Impact:** mock storage/settings do not meet the recursive plain-data
  contract. Persisted and returned values can differ from JSON marshalling,
  masking author bugs. The previous cycle fix does not solve recursive type
  validation.
- **Proposed fix:** integrate recursive JSON-kind and finite-number validation
  into a single bounded walk before serialization/copying; map invalid values
  to the existing typed error without partial writes.
- **Safe regression tests:** small nested arrays/objects should be validated as
  strictly as their corresponding scalar roots; retain ordinary JSON round-trip
  and previous-value-preservation assertions. No stress or malformed-input
  reproduction was executed in this review.

### PLUG-SDK-008 — P2 — Settings and service calls retain live object references

- **Repository / locations:** `bitty-plugin-sdk`, `src/mock-host.ts:1262-1269`,
  `src/mock-host.ts:1605-1616`, `src/mock-host.ts:943-957`.
- **Cause:** settings reads return the stored object directly, unlike store
  reads. Service arguments/results and command arguments/results are passed
  through without a marshalling copy.
- **Contract:** runtime RFC `:193-196` requires immutable bounded copies;
  Lua surface RFC `:426-428` says service results are values, not cross-VM
  handles.
- **Impact:** ordinary caller mutation can alter mock settings without a write,
  and service tests can accidentally depend on reference sharing that cannot
  exist across isolated VMs. Schema validation alone does not establish value
  isolation.
- **Proposed fix:** copy and validate at both call boundaries, and return
  independent settings data; freeze where the contract requires immutability.
- **Safe regression tests:** exchange a small nested JSON object and verify
  changing the caller's copy cannot mutate stored/provider-owned state; verify
  failed validation preserves the prior value.

### PLUG-SDK-009 — P2 — Schema validation accepts malformed nested schemas

- **Repository / locations:** `bitty-plugin-sdk`, `src/json-schema.ts:132-218`,
  `src/json-schema.ts:324-351`, `src/manifest.ts:547-552`.
- **Cause:** property schemas are cast rather than checked as schema objects.
  Some scalar children are treated as empty schemas; runtime validation then
  treats them as missing properties. `items` is validated only when `type`
  explicitly includes array, although value validation applies it to arrays
  even when type is omitted. Count/length bounds need not be finite integers.
- **Impact:** the manifest linter and registration validator can approve schema
  metadata outside their advertised strict subset, leading to contradictory
  dispatch behavior or untyped validation failures. This is a schema-shape
  correctness defect independent of the prior store-cycle finding.
- **Proposed fix:** recursively validate every schema-bearing keyword regardless
  of optional type annotations; accept only explicitly supported schema node
  forms; require valid finite integer count bounds and valid numeric bounds.
- **Safe regression tests:** a small table-driven schema-shape suite should
  reject non-schema property children and invalid keyword types consistently;
  schemas using type-independent array keywords must either be fully validated
  or explicitly rejected. No reproduction program was produced.

### PLUG-SDK-010 — P2 — Conformance timeout starts after synchronous work ends

- **Repository / locations:** `bitty-plugin-sdk`, `src/conformance.ts:359-378`,
  `src/conformance.ts:801-807`, `docs/mock-host.md:316-319`.
- **Cause:** the async IIFE runs every synchronous step before returning the
  promise passed to `withTimeout`. The timer is therefore installed after the
  step loop has already completed. Even moving timer creation earlier would
  not preempt a blocking synchronous step on the same thread.
- **Impact:** the advertised per-case wall-clock bound is not enforced. The
  file and step-count caps help but do not provide the claimed execution
  deadline. This is a defensive harness reliability concern, not a measured
  resource-exhaustion claim.
- **Proposed fix:** enforce monotonic deadline checks between bounded steps;
  if hard interruption is required, isolate execution under a host-controlled
  cancellable deadline. State clearly which phases the budget covers.
- **Safe regression tests:** inject a deterministic clock to verify expired
  budgets prevent the next harmless step; verify cleanup and a timeout result
  without long loops, large fixtures, or timing-dependent stress tests.

### PLUG-SDK-011 — P2 — Plugin API compatibility is read but never enforced

- **Repository / locations:** `bitty-plugin-sdk`,
  `src/manifest-model.ts:226-235`, `src/mock-host.ts:693-706`,
  `src/mock-host.ts:793-817`.
- **Cause:** `pluginApiRange` is retained in the manifest model, but neither
  construction nor activation checks it against the mock's `1.0.0` bridge.
- **Contract:** Lua surface RFC `:534-541` requires the manifest Plugin API
  range and runtime bridge version to agree at activation.
- **Impact:** a manifest requiring a different API major can activate in the
  mock and produce green tests using an unsupported surface.
- **Proposed fix:** evaluate Plugin API compatibility before opening activation;
  make host-application compatibility an explicit configurable harness feature
  rather than silently claiming it is checked.
- **Safe regression tests:** supported, unsupported-major, and absent-range
  cases should have documented outcomes, with no registrations committed after
  a version mismatch.

### PLUG-SDK-012 — P3 — Structured filesystem writes miss high-risk warnings

- **Repository / locations:** `bitty-plugin-sdk`, `src/manifest.ts:595-711`,
  `src/manifest.ts:719-741`, `src/capabilities.ts:100-107`,
  `src/capabilities.ts:313-321`.
- **Cause:** structured filesystem declarations bypass `validateCapabilityId`,
  the only path emitting `capabilities.high-risk`. The expanded warning set
  includes `fs.write`, but the structured write path never uses it.
- **Impact:** equivalent write authority is highlighted only in the flat form.
  This affects author/reviewer feedback; warnings do not grant authority and
  this is not a demonstrated consent bypass in the production host.
- **Proposed fix:** normalize both forms before applying risk classification,
  while preserving source-specific diagnostic paths and avoiding duplicates.
- **Safe regression tests:** ordinary equivalent flat and structured write
  declarations should receive equivalent warnings; read-only declarations
  retain their documented warning behavior.

## Uncertainties and deliberately limited behavior

These are not additional proven production defects:

- **Host implementation parity:** the first pass did not inspect Rust host
  source. The second pass traced entry selection and selected bridge paths,
  correcting PLUG-SDK-001. It did not execute the host or establish complete
  lifecycle, UI, service, or manifest parity with the RFC.
- **Version grammar:** `docs/manifest.md:428-458` now openly documents wider and
  narrower resolver grammar and the pending `DEC-0008` decision. The zero-major
  caret bug is fixed; remaining structural divergence is an acknowledged
  compatibility limitation, not the old charset-only defect. Concrete service
  versions with prerelease/build metadata still exceed the mock resolver's
  documented numeric-only subset (`src/version-range.ts:135-148`).
- **Filesystem policy:** `src/path-pattern.ts:20-29` implements a conservative
  static subset. Static segment checks are not an authorization boundary;
  production real-path resolution, approved roots, sensitive-resource policy,
  and bounded file handling remain necessary. No path bypass analysis or
  payloads are included here.
- **Overlay updates:** the surface table/tests intentionally place conditional
  `ui.overlay` on mount only, while the RFC's UI prose can be read more broadly.
  Reconcile revocation behavior with the owning contract before treating an
  update-gate change as a defect fix.
- **Snapshot fidelity:** the harness supplies a single arbitrary snapshot;
  `terminal_id` is validated but does not select a different snapshot
  (`src/mock-host.ts:1470-1510`). Treat multi-terminal selection as unmodeled,
  not as proof of a production routing defect.
- **Conformance filesystem boundary:** the runner uses lexical containment and
  pre-read `stat`; stronger symlink/descriptor/read-budget guarantees were not
  established. Treat fixture trees as reviewed local inputs until the boundary
  is specified and hardened. No escape or race reproduction was attempted.
- **Activation failure cleanup:** there is no transaction wrapper/rollback
  harness, and `dispose` refuses activating state (`src/mock-host.ts:833-840`).
  Partial registration rollback is not established by current tests.
- **Read-only API table:** TypeScript readonly declarations do not freeze
  `host.bitty` at runtime (`src/mock-host.ts:705-758`). This remains a mock
  fidelity gap relative to the injected Lua table, not ambient host authority.

## Performance and maintainability suggestions

- `src/mock-host.ts:153-189`, `src/mock-host.ts:209-291`, and
  `src/json-schema.ts:79-116` duplicate copy/serialization/traversal logic.
  Consolidate bounded plain-data validation and copying so fixes apply to
  schemas, storage, services, events, and snapshots consistently. Recursive
  schema depth inspection and snapshot copying still lack the same explicit
  depth guard as store values; this is a defensive hardening recommendation,
  not a stress-test result.
- `src/mock-host.ts:1630-1635` and `:1687-1692` rescan retained task/timer
  records on creation. With R retained records and C successive creations,
  repeated counting costs O(CR) time; cancelled/spent records retain callbacks
  until disposal, O(R) space. Track live counts and remove spent records when
  practical. `advanceTimers` additionally sorts due records, O(R log R) time.
- The generator's substantial hand-maintained display-width/table formatter
  (`scripts/generate-plugin.mjs:178-584`) is larger than its scaffolding logic.
  Keep its documented Unicode limitations and pin-drift tests explicit rather
  than broadening this unrelated implementation during contract repairs.
- Add generator CLI tests: current 22 tests cover formatting and SDK-pin
  invariants, not invocation, refusal of existing destinations, unknown flags,
  partial-write recovery, or loadable output. `parseArgs` accepts unknown and
  repeated option names (`scripts/generate-plugin.mjs:82-103`).
- The SDK Lua example recorder is **not** a live Lua-to-mock adapter:
  `tests/example.test.ts:80-108` replaces callbacks with inert stubs;
  `tests/lua/minimal-example-recorder.lua:88-103` supplies canned return values.
  It checks activation call shapes, not command-body execution, returned-value
  branches, or service isolation. Narrow the “end to end” description in
  `docs/mock-host.md:416-425` and add callback execution evidence separately.
- Template Lua gates use `bunx` despite claiming the dependency step is the only
  network step (`template/justfile:17-43`). Check the actual local parser binary
  before invocation; do not infer offline behavior from `node_modules/` alone.
  The negative parser control accepts any nonzero failure, including a missing
  executable, rather than distinguishing an expected parse diagnostic.
- Generated CI still specifies Bun 1.4.0, whereas repository metadata specifies
  1.4.2. This drift is visible at `template/.github/workflows/ci.yml:30` and
  `package.json:8`; it was not tested as an incompatibility here.

## Recheck of the 2026-09-15 review

- **P0-1 command registration:** fixed. Template uses `id = "hello"` and
  `title`, not the old `name`/`description` substitution. Do not reopen it.
  PLUG-SDK-001 is the separate package-layout issue.
- **P1-1 filesystem traversal/sensitive segments:** the old missing checks are
  present in `src/path-pattern.ts:57-79` and reused by both forms. This is
  static remediation evidence, not proof of all runtime filesystem policy.
- **P1-2 charset-only requirements:** replaced by shared structural parsing in
  `src/version-range.ts`. Six version tests passed. Known resolver divergence
  is documented rather than silently assumed fixed.
- **P1-3 unknown lazy events:** fixed at `src/manifest.ts:895-903`.
  Corresponding tests were sampled, not executed individually.
- **P1-4 cycles/settings bounds:** cycle checks and settings bounds now exist;
  old reproduction claims are stale. PLUG-SDK-007 concerns nested type
  validation, a distinct remaining gap. Cycle/stress tests were not run.
- **P1-5 settings namespace:** the prior claim about another owner's dotted
  relative key was not established: relative keys remain within each mock
  instance's own map. A conservative leading `plugins` guard now exists.
  PLUG-SDK-008 concerns live returned references, not namespace escape.
- **P1-6 transitional validator drift:** obsolete; the validator is removed and
  generated repositories use commit-pinned SDK lint. Pin unit tests passed.
- **P1-7 minimal Lua example:** repaired object schemas, node kind, companion
  service declaration, and qualified command are present. Recorder coverage is
  limited as described above; the Lua example suite was not run.
- **P2-1 risk set:** expanded; do not repeat “only six heads.” Structured-write
  warning asymmetry remains as PLUG-SDK-012.
- **P2-2 default snapshot scope:** fixed; selected default-scope test passed.
- **P2-3 payload-less extra fields:** fixed in `src/mock-host.ts:899-909`.
- **P2-4 task/timer activation-only creation:** accepted contract requires it;
  not a defect. UI mount is the remaining inconsistency, PLUG-SDK-003.
- **P2-5 persistent grants:** old premise was wrong for the accepted persistent
  manifest-hash grant record. Mock disposal now clears grants as an explicitly
  documented stricter harness simplification; suspension correctly retains them.
- **P2-6 exclusive claims:** repaired for `tabline`; selected claim test passed.
- **P2-7 chord case:** repaired; selected case/alias test passed.
- **P2-8 zero-major caret:** repaired; the existing matrix and resolver tests
  passed. No generic SemVer assumption overrides this project's documented rule.
- **P2-9 Lua parser invocation:** the lockfile confirms a `luaparse` CLI and the
  recipe now has a negative parser control. The old speculative “library only”
  finding is withdrawn. Generated parser execution was not run here.
- **P2-10 generator version grammar:** now mirrors strict SemVer 2. No generator
  process was invoked, so this is a source-level recheck only.
- **P2-11 manifest pre-read size:** repaired at `src/conformance.ts:407-415`.
  The scratch-writing regression test was deliberately filtered out. The
  ineffective timeout recommendation remains relevant as PLUG-SDK-010.

## File inventory and actual read coverage

“Fully read” means the complete file content was inspected, not merely listed,
outlined, imported by a test, or counted by an assertion. “Sampled” means only
specified regions were inspected. Execution coverage is separate below.

### SDK — fully read

- All **14** current `src/*.ts` files: `capabilities.ts`, `cli.ts`,
  `conformance.ts`, `diagnostics.ts`, `host-diagnostics.ts`, `host-surface.ts`,
  `index.ts`, `json-schema.ts`, `manifest-model.ts`, `manifest.ts`,
  `mock-host.ts`, `path-pattern.ts`, `schema.ts`, `version-range.ts`.
- `package.json`, `tsconfig.json`, `justfile`, `AGENTS.md`.
- `docs/mock-host.md`, `lua/bitty.d.lua`, `lua/examples/minimal-init.lua`.
- `tests/version-range.test.ts`, `tests/packaging.test.ts`,
  `tests/conformance.test.ts`, `tests/lua/minimal-example-recorder.lua`.
- `conformance/cases/05-lifecycle-registration.json`,
  `conformance/cases/11-services-tasks-timers.json`,
  `conformance/cases/12-lazy-commands-table.json`,
  `conformance/manifests/lazy-table.toml`.
- `.carryctx/rules/security.md`,
  `.carryctx/personas/compatibility-reviewer.md`.

### SDK — sampled, outlined, or not read

- `tests/mock-host.test.ts`: read 1–100, 238–677, 680–1099; remaining portions
  outline-only. `tests/manifest.test.ts`: read 1–90 and 740–1039; remainder
  outline-only. No claim of full test-source review.
- `tests/example.test.ts`: outline plus 63–262.
- `tests/cli.test.ts`, `tests/lua-defs.test.ts`: outlines/search results only.
- `scripts/generate-lua-defs.ts`: outline plus 462–658; validation body was not
  fully read. `surface/bitty-plugin-api-v1.json`: read 900–1128; other type
  definitions were inspected through generated Lua declarations, not by fully
  reading their JSON source. The drift check executed successfully.
- `docs/manifest.md`: read 200–389 and 398–465; other portions not read.
- Nine other conformance cases (01–04 and 06–10), three other conformance
  manifests, five TOML test fixtures, two `docs/examples/*.toml` files, and
  `lua/examples/minimal-init.bitty-plugin.toml`: inventoried; some were read by
  existing test execution, but their full text was not manually inspected.
- `scripts/check-lua-luals.ts`, `tests/lua-defs/negative-fixture.lua`,
  `conformance/README.md`, `docs/lua-defs.md`: inventoried, not manually reviewed.
- Workflow import/publish scripts, repository CI/security automation, root
  README/contributor/release/security prose, root lockfile/dependency contents,
  other governance files, `.git`, and worktrees are outside the detailed read
  coverage. No dependency or supply-chain audit is claimed.

### Template — fully read

- All **8** files in `template/`: `.github/workflows/ci.yml`, `.gitignore`,
  `README.md`, `bitty-plugin.toml`, `bun.lock`, `justfile`,
  `lua/@@PLUGIN_MODULE@@/init.lua`, `package.json`.
- All four scaffolding/pin scripts: `scripts/generate-plugin.mjs`,
  `scripts/generate-plugin.test.mjs`, `scripts/refresh-sdk-pin.mjs`,
  `scripts/verify-sdk-pin-refresh.mjs`.
- `README.md`, `AGENTS.md`, `package.json`, `justfile`,
  `.github/workflows/ci.yml`, `.carryctx/rules/security.md`,
  `.carryctx/personas/developer-experience-reviewer.md`.
- The two workflow import/publish scripts and remaining root governance,
  automation, contributor/security/release prose, lockfile, dependency contents,
  and Git internals were inventoried or excluded, not fully reviewed.

## Validation actually performed

All commands below ran from the named repository using existing Bun 1.4.2 and
installed dependencies. No flags requested snapshots, coverage files, or writes.

1. SDK: `bun test tests/version-range.test.ts tests/packaging.test.ts`
   — **9 pass, 0 fail, 132 assertions**.
2. SDK: `./node_modules/.bin/tsc -p tsconfig.json --noEmit`
   — **exit 0**, no diagnostics. Run directly because `just type-check` depends
   on `install`, prohibited for this review.
3. Template: `bun test scripts/generate-plugin.test.mjs`
   — **22 pass, 0 fail, 29 assertions**.
4. SDK: `bun test tests/conformance.test.ts -t 'every case passes|fixtures publish|fixtures cover|accepted surface agreement'`
   — **6 pass, 1 filtered out, 0 fail, 119 assertions**. The existing 12 case
   files ran through the runner. This proves their expectations, not complete
   semantic parity. The excluded test writes an oversized temporary manifest.
5. SDK: `just lua-defs-check`
   — **exit 0**, generated definitions match the source table: **19 functions,
   17 events**. No regeneration occurred.
6. SDK: `bun test tests/mock-host.test.ts -t 'registration window and lifecycle|observation events deliver frozen|interception handlers veto|throwing handlers are recorded|services resolve declared|tasks and timers enforce|settings stay inside|terminal snapshot defaults|exclusive-claim UI|key chords are trimmed'`
   — **16 pass, 31 filtered out, 0 fail, 74 assertions**.

Total: **53 selected existing tests passed, 0 failed**. No new test or defect
reproduction code was created. Selected tests establish current behavior; none
is represented as a regression test for every finding above.

Not run: complete `bun test`, install-dependent `just check`, formatting writes,
LuaLS (which may write scratch), Lua example execution, generated-tree creation,
clean-generation/install/network pin refresh, Rust host tests, cross-platform
CI, fuzzing, resource-exhaustion probes, and security exploit tests. This is not
all-code, branch, dependency, runtime-host, or cross-platform coverage.

Report validation: installed `markdownlint-cli2` is available; the assigned file
will be checked alone after writing. Final outcome is recorded below after the
actual run. No other report is authorized for modification.
