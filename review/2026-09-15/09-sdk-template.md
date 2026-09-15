# Read-only review report: plugin SDK and template

## Scope

- `bitty-plugin-sdk/src/*.ts` (12 files total: `index.ts`, `manifest.ts`, `manifest-model.ts`, `schema.ts`, `capabilities.ts`, `mock-host.ts`, `conformance.ts`, `host-surface.ts`, `host-diagnostics.ts`, `json-schema.ts`, `diagnostics.ts`, `cli.ts`)
- `bitty-plugin-sdk/lua/bitty.d.lua`, `bitty-plugin-sdk/surface/bitty-plugin-api-v1.json`, `bitty-plugin-sdk/conformance/` (`cases/` 12 items, `manifests/` 4 items, `README.md`), `bitty-plugin-sdk/tests/`, `bitty-plugin-sdk/docs/examples/`, `bitty-plugin-sdk/lua/examples/`, `bitty-plugin-sdk/scripts/generate-lua-defs.ts`
- Full `bitty-plugin-template/template/` tree (`bitty-plugin.toml`, `justfile`, `lua/@@PLUGIN_MODULE@@/init.lua`, `scripts/validate-manifest.mjs`, `README.md`, `.github/workflows/ci.yml`, `.gitignore`), plus the generator `bitty-plugin-template/scripts/generate-plugin.mjs`

## Method

- Read through `package.json`, `src/index.ts`, `manifest.ts`, `schema.ts`, and `capabilities.ts` to confirm the field whitelist, length and count limits, closed capability heads, parameter rules, and the fail-closed semantics of high-risk warnings.
- Compared `manifest-model.ts`, `mock-host.ts`, `host-surface.ts`, `json-schema.ts`, `lua/bitty.d.lua`, and `surface/bitty-plugin-api-v1.json` to check consistency between mock-host and the real host contract (capability gates, registration window, generation handles, closed event set, schema subset, quotas).
- Read through `conformance.ts` with all case names plus sampled case steps to assess coverage and runner boundaries.
- Read through the template generation chain (`generate-plugin.mjs` validation and placeholder substitution, `validate-manifest.mjs` transitional validation, `justfile`, `ci.yml`, `init.lua`, `README.md`) to check out-of-box usability, broken references, and version drift.
- Focus areas: manifest validation strictness (field whitelist, version semantics, path traversal), whether the capability model is least-privilege, mock-host vs real-host behavioral consistency, conformance coverage, and template out-of-box usability. Security items are tagged `SEC`.
- Read-only review: no source modified, no generation or test commands executed; every "may fail" below is a static deduction from the code, not verified on a live host.

## Defect list

### P0-1 Template entry point completely mismatched with the API registration signature; generated output cannot run

- Location: `bitty-plugin-template/template/lua/@@PLUGIN_MODULE@@/init.lua`, symbol: `bitty.commands.register` call site
- Symptom: the template uses the `name = "@@PLUGIN_ID@@:hello"` field with a `description` field; the contract requires `id` (short segment within the plugin) plus `title` (non-empty), plus a `run` function. Under mock semantics this fails first with an illegal command identifier or illegal definition, and never reaches the notification demo.
- Trigger: generate a repository with the generator per `README.md`, then run the entry point as-is (both mock and real host are affected; for the mock side see `bitty-plugin-sdk/src/mock-host.ts` symbol `MockHost.registerCommand`: `def.id` must match the short-segment syntax, `def.title` must be non-empty, and the assembled `pluginId:id` must already be reserved in `[lazy].commands`).
- Evidence:
  - Template entry lines 21-30 pass a qualified name via `name` and display text via `description`, with no `id` and no `title`.
  - The authoritative shapes in this repo are `BittyCommandDef` in `bitty-plugin-sdk/surface/bitty-plugin-api-v1.json` (`id`, `title`, `description?`, `args_schema?`, `result_schema?`, `run`) and the same-named class comment in `bitty-plugin-sdk/lua/bitty.d.lua`; both use `id` plus `title`.
  - The SDK's own runnable example `bitty-plugin-sdk/lua/examples/minimal-init.lua` likewise uses `id` plus `title`; the template is the only exception.
  - The template `bitty-plugin.toml` reserves `commands = ["@@PLUGIN_ID@@:hello"]` as a qualified name, which matches the mock model of "registration passes the short segment, the host adds the prefix" — making the entry point passing a qualified name all the more clearly wrong.

### P1-1 SEC manifest filesystem patterns lack traversal and sensitive-location checks

- Location: `bitty-plugin-sdk/src/manifest.ts`, symbols: `pathPatternProblem`, `validateFilesystem`
- Symptom: path patterns are only checked for non-emptiness, byte-length limit, and absence of control/whitespace characters. Patterns containing parent-directory references, absolute-form locations, and wildcards over sensitive subdirectories under the home directory all pass lint with `valid` still true.
- Trigger: `[[capabilities.filesystem]]` with `access = "read"` and `paths` taking parent-directory-reference or sensitive-directory-wildcard forms.
- Evidence: `pathPatternProblem` in full is only three checks; `manifest.test.ts` covers the filesystem only for empty arrays, non-strings, unknown keys, out-of-scope `access`, the per-`access` limit of 32 entries, cross-entry aggregation, and the 8 KiB total-text limit, with no parent-directory-reference or sensitive-location cases. The template-side `bitty-plugin-template/template/scripts/validate-manifest.mjs` symbol `validateFilesystem` adds only one extra colon rejection and likewise has no traversal check; both sides are missing it consistently.

### P1-2 Version requirements are charset-only checks; lint passes what the runtime rejects

- Location: `bitty-plugin-sdk/src/manifest.ts`, symbol: `versionReqProblem`
- Symptom: values for `compat.bitty`, `compat.plugin-api`, and `dependencies` pass as long as their characters fall in a permissive set. Pure whitespace, pure operators, and meaningless shorthands are all judged valid; but `mock-host.ts` symbol `versionSatisfies` only understands comma-separated conjunctions with full three-segment versions plus a limited operator set, so the above inputs fail at service resolution as undecidable/invalid versions. Install-time and run-time verdicts diverge.
- Trigger: `compat.bitty` taking a pure wildcard or shorthand, `dependencies` taking a natural-language version, `services.get` with `opts.version` taking a disjunction, etc.
- Evidence: the `versionReqProblem` regex is only a character whitelist; `mock-host.test.ts` covers `versionSatisfies` only for ordinary greater-than-or-equal and optional defaults, not for shapes that pass lint but are undecidable at runtime; `manifest.test.ts` asserts only ordinary ranges pass on this branch.

### P1-3 `lazy.events` is not checked against the closed event set; typos surface only at runtime

- Location: `bitty-plugin-sdk/src/manifest.ts`, symbol: `validateLazy`
- Symptom: event subscriptions are only checked for length and whitespace/control characters, not against the 17-member closed set in `host-surface.ts` symbol `EVENT_KINDS`. Any dotted string (up to 256 entries) passes lint, then fails later in `mock-host.ts` symbols `MockHost.subscribe` and `MockHost.publish` with unknown-event or undeclared-event errors.
- Trigger: misspelling an existing closed event name, or declaring a new event name out of thin air.
- Evidence: the capability branch goes through `validateCapabilityId` and rejects unknown heads with `capabilities.unknown`, while the event branch has no equivalent check; `conformance.test.ts` requires cases to publish all closed-set events, but has no reverse assertion that "lint must reject non-closed-set events".

### P1-4 SEC mock-host `settings` and deep-copy paths lack cycle protection and can stack-overflow

- Location: `bitty-plugin-sdk/src/mock-host.ts`, symbols: `settingsSet`, `deepCopy`, `containerDepth`, `countNodes`, `componentProblem`, `storeValueProblem`
- Symptom: `settings.set` deep-copies directly with no value-bound validation; several recursive helpers have no visited set, so cyclic-reference inputs cause recursive overflow rather than typed rejection. `store` does have depth and node limits, but the limit checks themselves are cycle-unaware recursion, so they overflow first just the same.
- Trigger: passing a self-referential table to `store.set`, `settings.set`, `ui.mount`, `ui.update`, or `setTerminalSnapshot`.
- Evidence: `storeValueProblem` calls `containerDepth` and `countNodes` before the serialization-length check, and all three assume tree-shaped input; `componentProblem` recurses over `children` with no cycle detection; `mock-host.test.ts` covers only chained extra-deep objects, not cyclic references.

### P1-5 SEC mock-host `settings` keys lack plugin-namespace isolation; more permissive than the contract

- Location: `bitty-plugin-sdk/src/mock-host.ts`, symbols: `settingsKeyProblem`, `settingsGet`, `settingsSet`
- Symptom: the Lua type comments say keys may only fall on relative paths inside the plugin's own namespace, but the mock only checks dotted shape (non-empty, no leading/trailing dots, no double dots, no control whitespace, byte limit), so cross-plugin dotted keys read and write successfully. If the real host isolates by namespace, the mock will miss over-privileged plugins.
- Trigger: calling `settings.set` to write a dotted key owned by another owner or another plugin name, then reading it back.
- Evidence: `lua/bitty.d.lua` symbol `BittySettingsNamespace` is commented as relative to the plugin's own namespace; `mock-host.test.ts` symbol `settings stay inside the plugin namespace` only asserts double-dot escapes are rejected, not cross-namespace writes.

### P1-6 Template transitional validator has drifted from the SDK in several places; the same manifest gets opposite verdicts

- Location: `bitty-plugin-template/template/scripts/validate-manifest.mjs`, compared against `bitty-plugin-sdk/src/manifest.ts`, `bitty-plugin-sdk/src/capabilities.ts`, `bitty-plugin-sdk/src/schema.ts`
- Symptom: the transitional script calls itself a pre-SDK-release subset, but shows two-way drift. Known divergences include at least: different capability-identifier length limits, flattened parameterized capabilities accepted on one side and rejected on the other, different SemVer strictness, empty filesystem arrays rejected on one side and allowed on the other, path colons rejected on one side and allowed on the other, and different TOML parsers (the template uses the runtime-built-in TOML, the SDK uses `smol-toml`, with possibly different date, duplicate-key, and inline-table behavior).
- Trigger: running template `just manifest` and SDK `bitty-plugin-lint` in sequence on the same `bitty-plugin.toml`.
- Evidence: the template `MAX_CAPABILITY_LEN` and SDK `MAX_CAPABILITY_LEN` constants differ; the template `validateCapabilityHead` fails heads needing parameters outright, while the SDK specifically accepts the flattened `fs.read:<params>` form with unit tests locking it in; the template SemVer lumps hyphen and plus suffixes into one character class, while the SDK uses a full SemVer2 regex with leading-zero and empty-suffix cases; the template requires a non-empty filesystem array, while the SDK emits no diagnostic for an empty array.

### P1-7 The SDK's bundled minimal Lua example cannot run directly against mock-host

- Location: `bitty-plugin-sdk/lua/examples/minimal-init.lua`
- Symptom: the example object pattern lacks explicit `additionalProperties`, a component lacks `kind`, a service provision lacks a manifest declaration, and the ownership of a keybound command is unexplained. Copying the registration logic into `MockHost` triggers pattern-illegal, component-illegal, service-undeclared, and similar failures respectively, discounting the example's teaching value.
- Trigger: wiring the example registration logic to `MockHost` (object schemas go through `json-schema.ts` symbol `schemaProblem` with its explicit additional-properties requirement; components go through `mock-host.ts` symbol `componentProblem` with its `kind` requirement; services go through the declared-provision check in `servicesProvide`).
- Evidence: `json-schema.ts` returns a problem directly for object schemas without `additionalProperties`; `lua-defs.test.ts` only does LuaLS parsing and exclusion-surface checks, with no mock-host executability assertion on this example.

### P2-1 High-risk capability set under-covers; least-privilege warnings are weak

- Location: `bitty-plugin-sdk/src/capabilities.ts`, symbol: `HIGH_RISK_HEADS`
- Symptom: only 6 heads trigger high-risk warnings; sensitive heads such as terminal management, process spawning, network connections, file writes, clipboard reads, agent and external calls trigger no equivalent warning. Lint passes valid with no warning, so authors may underestimate the grant.
- Trigger: declaring the above sensitive capabilities without declaring an already-listed high-risk head.
- Evidence: `manifest.test.ts` only locks in the original read triggering a warning, not sensitive writes and execution-class heads.

### P2-2 Terminal snapshot without scope fails; divergence risk against default semantics

- Location: `bitty-plugin-sdk/src/mock-host.ts`, symbol: `terminalSnapshotRead`
- Symptom: omitting `scope` reports unsupported scope; if the real host defaults to a semantic snapshot, the mock is stricter than the real machine and kills legitimate plugins in mock.
- Trigger: calling `terminal.snapshot` with no argument or an empty table.
- Evidence: `host-surface.ts` symbol `SNAPSHOT_SCOPE_ONLY` is the only semantic value; conformance case `10-ui-terminal.json` covers only explicit semantic and explicit raw scopes, not the default.

### P2-3 Empty-payload events allow extra fields; more permissive than the strict contract

- Location: `bitty-plugin-sdk/src/host-surface.ts` symbol `EVENT_PAYLOAD_FIELDS`, `bitty-plugin-sdk/src/mock-host.ts` symbol `MockHost.publish`
- Symptom: event names with no required entries have no entry definitions, so publishing only type-checks known fields and does not reject extra keys. Signal- and lifecycle-class events can carry arbitrary extra fields past the size check.
- Trigger: publishing a table with extra keys to a payload-less event.
- Evidence: the payload table lacks terminal-bell, config-reload, handler-violation, and lifecycle entries; `publish` has no extra-key rejection branch for non-intercepted classes, only the intercepted class has a whitelist check.

### P2-4 Tasks and timers can only be created during the activation window; runtime semantics unclear

- Location: `bitty-plugin-sdk/src/mock-host.ts`, symbols: `tasksSpawn`, `timersCreate`
- Symptom: both assert the registration window; the docs say the host owns tasks and one-shot timers, without stating whether creation is activation-only. If the real machine allows runtime creation, the mock kills legitimate plugins; if the real machine is equally activation-only, the Lua comments should say so.
- Trigger: calling `tasks.spawn` or `timers.create` after activation has ended.
- Evidence: conformance cases `05-lifecycle-registration.json` and `11-services-tasks-timers.json` both create within the activation window, with no positive/negative cases outside it.

### P2-5 Grants linger across generations; no re-authorization needed after reload

- Location: `bitty-plugin-sdk/src/mock-host.ts`, symbols: `grant`, `revoke`, `dispose`, `beginActivation`
- Symptom: `dispose` clears subscriptions, commands, keybindings, UI blocks, tasks, timers, and services, but not the grant set, so a new generation inherits old grants. If the real machine requires fresh consent on reload, the mock overestimates permission continuity.
- Trigger: authorize, then go through suspend-dispose into a new activation window, then call a gated surface without re-authorizing.
- Evidence: the `dispose` body has no grant cleanup; `mock-host.test.ts` has a revoke-then-fail case but no cross-generation grant-residue case.

### P2-6 UI exclusivity claims and mounts have no cross-check

- Location: `bitty-plugin-sdk/src/mock-host.ts`, symbol: `uiMount`
- Symptom: mounting only checks slot names, rich text, and overlay capability, not `lazy.claims`. A full manifest declares exclusive claims, but the corresponding slot can be mounted with no declaration at all.
- Trigger: mounting a claim-gated slot without declaring the corresponding claim.
- Evidence: `conformance/manifests/full.toml` contains `claims`, while case `10-ui-terminal.json` does not cover mounting with the declaration missing.

### P2-7 Key-chord case rules may be stricter than the real configuration

- Location: `bitty-plugin-sdk/src/mock-host.ts`, symbol: `chordProblem`
- Symptom: requires all-lowercase with spaces stripped, and single characters must carry a modifier; the Lua comments say modifier case is insensitive. If the real machine accepts conventional mixed-case spellings, the mock kills them wrongly.
- Trigger: suggesting a chord spelled with a capital `Ctrl` or `Shift`.
- Evidence: conformance and unit tests use only all-lowercase chords, with no case-equivalence case.

### P2-8 Caret version semantics wrong for zero major versions

- Location: `bitty-plugin-sdk/src/mock-host.ts`, symbol: `versionSatisfies`
- Symptom: the caret implementation means same-major plus greater-than-or-equal, but for zero majors it should tighten to same-minor; currently it judges incompatible minor bumps as satisfied.
- Trigger: provider major version zero with a minor roll-forward, consumer resolving with a caret requirement.
- Evidence: `mock-host.test.ts` service cases only use major-one ranges, with no zero-major caret case.

### P2-9 Template `just lua` recipe references a highly suspect parser invocation; the out-of-box gate may fail directly

- Location: `bitty-plugin-template/template/justfile`, symbol: `lua` recipe
- Symptom: the recipe invokes a Lua parser through a package runner with version plus quiet and file arguments; the package shape looks more like a library than a CLI, and the argument style is not evidenced anywhere inside the template. `just check` after generation may fall over at this gate.
- Trigger: running `just lua` or `just check` after generation.
- Evidence: no successful-output archive of that invocation exists in the template; the SDK-side Lua gate goes through the in-house generator plus language-service checks (`bitty-plugin-sdk/justfile` symbol `lua-defs-check`), while the template side is a separate isolated chain.

### P2-10 Generator version check weaker than the SDK; generated output may be rejected by its own lint

- Location: `bitty-plugin-template/scripts/generate-plugin.mjs`, symbol: `validateVersion`
- Symptom: the generator accepts by splitting the core three segments plus a full-string charset check, accepting empty suffixes and underscore suffixes that strict SDK SemVer rejects; generation succeeds but the artifact fails the SDK gate.
- Trigger: invoking the generator with a boundary version string, then running SDK validation on the artifact.
- Evidence: SDK `manifest.test.ts` explicitly rejects underscore-suffix and empty-suffix shapes; the generator charset includes underscores and never validates non-empty suffix segments.

### P2-11 SEC conformance has no pre-read size gate for oversized manifests; memory amplification issue

- Location: `bitty-plugin-sdk/src/conformance.ts`, symbol: `runConformanceCaseFile`
- Symptom: case files have state-size and step-count limits, but manifest files are read directly and only rejected afterwards by lint's byte limit, so oversized fixtures occupy memory first.
- Trigger: placing an oversized TOML under `conformance/manifests/` and referencing it from a case.
- Evidence: the function `statSync`-limits cases to 1 MiB, with no equivalent pre-check for `manifestFile`.

## Recommendations

1. Fix the template entry point and examples first so generated output runs by default
   - Plan: change the template entry to short-segment `id` plus `title`, remove the mistaken `name` and `description` usage, and note in comments that the host assembles the qualified name from the manifest; fix the SDK minimal Lua example's object-pattern additional properties, component `kind`, service declaration, and keybinding ownership notes; add a docs step for "walk registration plus dispatch once with mock-host after generation".
   - Expected benefit: eliminates the out-of-box P0 and unifies the three authoritative shapes (surface table, Lua definitions, template).
   - Effort: S

2. Tighten manifest semantics and share one parser with the runtime
   - Plan: upgrade version-requirement checks from charset checks to semantic parsing from the same source as `versionSatisfies` (three-segment completeness, operator whitelist, documenting comma-vs-disjunction policy as one of two options); validate `lazy.events` against the closed event set; add normalization plus traversal rejection to filesystem patterns (parent-directory references, absolute forms, tightening sensitive directories per host policy — at minimum reject traversal first); unify length measurements as bytes.
   - Expected benefit: lint agrees with install/runtime verdicts; typos and over-grants fail at the earliest gate. SEC benefit is blocking traversal-style file grants.
   - Effort: M

3. Align the template transitional validator or retire it soon
   - Plan: one of two options. Short term, add a drift-comparison table to the transitional script and align each divergence one by one (limit constants, SemVer, flattened parameters, empty arrays, colon policy, parser differences); long term, switch the template to the SDK `bitty-plugin-lint` as the template comments intend, keeping only a thin wrapper. During the transition, add a template-CI gate running both validators on the same fixtures and requiring agreement.
   - Expected benefit: eliminates the dual-lint split; users stop editing manifests back and forth.
   - Effort: M

4. Converge mock-host and real-host permission and window semantics
   - Plan: decide and test, for each item, whether the mock or the real machine is stricter: `settings` namespace isolation, `terminal.snapshot` default scope, whether tasks/timers are activation-only, whether grants cross generations, whether UI exclusivity claims gate mounting, keybinding case handling, empty-payload strictness, caret zero-major handling. Write the verdicts into `docs/mock-host.md`, align behavior with the real machine, and add explicit unit tests for the differences.
   - Expected benefit: the mock is no longer biased in both directions ("sometimes stricter, sometimes looser"); plugin authors can trust it.
   - Effort: M

5. Add fail-closed structural validation and cycle protection to the data plane
   - Plan: reuse the `store` value bounds for `settings.set`; add visited sets to all recursive copies, depth/node counts, and component validation so cyclic inputs return typed failures instead of overflowing; keep existing byte gates for oversized UI and snapshots.
   - Expected benefit: eliminates plugin- or fixture-triggerable stack-overflow DoS. Direct SEC benefit.
   - Effort: S

6. Review the high-risk set and add least-privilege guidance
   - Plan: re-review `HIGH_RISK_HEADS`, bring execution, write, network, outbound, and sensitive-input-read capabilities into scope or write down why each is excluded; sync the template and docs with "no table means no permission, one capability at a time, prefer exact keys over broad prefixes" examples.
   - Expected benefit: more complete informed consent at grant time; less over-granting.
   - Effort: S

7. Fix the template quality-gate chain and CI coverage
   - Plan: verify and pin the true executable form of `just lua` (switch if the package has no binary; rewrite per the parser's real help if the flags are wrong), pin `bun` and `just` versions; add generator round-trip tests to CI (generate, leftover-placeholder check, dual lint, Lua parsing) so there is more than the single `just check` gate.
   - Expected benefit: the template goes from "looks runnable" to "actually runnable", with version drift exposed early.
   - Effort: S

8. Harden conformance runner boundaries
   - Plan: add a pre-read file-size gate for manifests equal to the case gate; document the in-root constraint and symlink policy for `manifest` references; change the synchronous long-step timeout to a step-count plus per-step budget or sharded execution to avoid synchronous long loops bypassing the wall clock.
   - Expected benefit: fixtures can no longer be memory/time amplifiers. SEC benefit is boundary completeness.
   - Effort: S

## Test gaps

- Filesystem capability end-to-end: none of the 4 existing conformance manifests declares filesystem, so the `readCapabilities` mapping of filesystem entries into parameterized-capability branches is never exercised by a case; traversal patterns, per-access limits, and total-text limits are unit-tested only, never verified through the declare-plus-grant-plus-call chain.
- Closed-event-set reverse: cases prove all 17 event kinds publishable, but no case proves lint rejects an 18th event name; the typo scenario is missing.
- Version-semantics reverse: ordinary ranges have passing cases, but the "lint-must-reject or runtime-must-reject" matrix for pure whitespace, pure operators, missing segments, and disjunctions is missing; zero-major caret is missing.
- Permission boundaries: cross-namespace settings, claim-less mounting, cross-generation grants, case-variant chords, default snapshot scope, and extra keys on empty payloads have only positive or no cases.
- Structural robustness: the mock-host typed-failure matrix for cyclic references, non-finite numbers, non-plain objects, oversized single values, oversized notification payloads, and oversized components is incomplete; `settings` value bounds are entirely missing.
- Template chain: no negative automation is visible for the generator ("illegal id, illegal version, reserved placeholder prefix, target already exists, leftover placeholders in template" — the code has branches but template CI never runs them); no dual-run consistency gate for the transitional validator vs the SDK; no evidence for the real executability of `just lua`.
- Lua definitions: generator drift checks and language-service checks are complete, but there is no mock-host assertion on example/template-entry executability consistency, leaving a gap where docs pass and runs fail.

Date: 2026-09-15
