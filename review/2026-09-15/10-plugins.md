# Plugin ecosystem review: official registry and 5 standalone plugins

Scope: `bitty-plugins/` (`registry/`, `plugins/activity|palette|statusline` placeholders, empty `sdk/` directory, empty `template/` directory, `store/app` i.e. `app/`, `scripts/`, `tests/`, `generated/`) and standalone repositories `activity/`, `palette/`, `statusline/`, `file-manager/`, `git-panel/`. Also covers `bitty-plugin-sdk/` (authoritative validation reference) and `bitty-plugin-template/` (generator reference).

Method: read-only review. In each repository read `bitty-plugin.toml`, the `lua/` entry and pure-function modules, `tests/spec/*.lua`, `tests/support/mock_host.lua`, `tests/lua-defs/negative-fixture.lua`, and `scripts/validate-manifest.mjs`; for `bitty-plugins` focus on `scripts/registry-lib.ts`, `scripts/semver.ts`, `scripts/validate-registry.ts`, `scripts/sync-metadata.ts`, `scripts/generate-index.ts`, `tests/registry.test.ts`, and `app/src/registry.ts|search.ts|router.ts|components/*.ts`; for `file-manager` and `git-panel` additionally read `validator-negative/*.toml` and empirically tested each one with the local validator to record intercept results; for `git-panel` focus on `lua/git-panel/allowlist.lua` and `lua/git-panel/listing.lua`; for `file-manager` focus on `lua/file-manager/scope.lua|listing.lua`; performance angle covers truncation and traversal upper bounds for large directories, large repositories, and oversized snapshot inputs; empty/edge states cover `scene.empty()`, `format.render({})`, `filter.filter(nil)`, missing snapshots, and missing-grant paths. No source modified; only read-only validation commands were run to obtain exit codes and diagnostic text as evidence.

## Defect list

### R1 (P0, SEC) Registry has no signatures, no hashes, no origin binding

- Paths and symbols: `bitty-plugins/registry/schema.json` whole table; `bitty-plugins/scripts/registry-lib.ts:toIndexPlugin/buildIndex`; `bitty-plugins/scripts/sync-metadata.ts:manifestMetadata/fetchText`; `bitty-plugins/generated/registry.json` consumer chain `bitty-plugins/app/src/registry.ts:loadRegistry/isRegistry`
- Symptom: entries carry only `id/name/repository/kind/author/...` with no `signature/checksum/manifest_hash` fields; `additionalProperties:false` would reject future signature fields; index generation only sorts and dedupes tags without recording any integrity value; the store frontend trusts `fetch("/registry.json")` content directly.
- Trigger: when any mirror, CDN, or commit tampers with `generated/registry.json`, or `raw.githubusercontent` returns a substituted `bitty-plugin.toml`, the client has no way to notice.
- Evidence: `schema.json` `required` is only `["id","name","repository"]` with no signature-related keys; `sync-metadata.ts` fetches the remote manifest with credential-less `fetch` and writes `metadata.version/description/license` straight through without verifying sender identity; `registry.test.ts` has no signature cases.

### R2 (P0, SEC) `sync-metadata` never verifies the fetched manifest matches the entry identity

- Paths and symbols: `bitty-plugins/scripts/sync-metadata.ts:manifestMetadata`, `rawManifestUrl`
- Symptom: `manifestMetadata(text, source, previous, fetchedAt)` only extracts `plugin.version/description/license` and never compares the fetched `plugin.id` against the current `entry.id`; a mismatch is still recorded under that `entry.id`.
- Trigger: when a repository is taken over, a branch replaced, or a lookalike repository returns another plugin's manifest, the index hangs A's version number under B's name.
- Evidence: a full read of `manifestMetadata` shows no `id` parameter and no `id`-comparison branch; `main()` does `metadataById.set(entry.id, metadata)` keyed directly by the loop's `entry.id`.

### R3 (P0, SEC) `sync-metadata` fetches with no size limit

- Paths and symbols: `bitty-plugins/scripts/sync-metadata.ts:fetchText`
- Symptom: `await response.text()` has no byte limit and no `Content-Length` pre-check, inconsistent with the manifest validator's `MANIFEST_MAX_BYTES = 256 KiB`; `FETCH_TIMEOUT_MS` limits time only, not size.
- Trigger: a malicious or corrupted remote returning tens of MB of text inflates sync-machine memory and drags index generation.
- Evidence: the `fetchText` body has only an `AbortController` timeout plus a `response.ok` check, with no length branch; by contrast `file-manager/scripts/validate-manifest.mjs` and `bitty-plugin-sdk/src/schema.ts:MANIFEST_MAX_BYTES` both enforce the 256 KiB limit.

### R4 (P0) Authoritative SDK rejects the accepted `[tools.git]`, forking from the transitional validator

- Paths and symbols: `bitty-plugin-sdk/src/schema.ts:ALLOWED_ROOT_KEYS`; `bitty-plugin-sdk/src/manifest.ts:checkUnknownKeys`; `git-panel/scripts/validate-manifest.mjs:validateTools`; `git-panel/bitty-plugin.toml:[tools.git]`
- Symptom: the SDK `ALLOWED_ROOT_KEYS` is the six tables `plugin/compat/dependencies/services/capabilities/lazy` with no `tools`, so it reports `git-panel` as invalid with `manifest.unknown-key (tools)`; the transitional validator explicitly accepts exactly `[tools.git]`. Docs call the SDK authoritative (R-SDK-2), yet the authority would kill the only accepted Layer-2 slice.
- Trigger: anywhere `BITTY_PLUGIN_LINT` is wired to the SDK CLI and used as an authoritative gate over `git-panel`.
- Evidence: a read-only SDK CLI run over `git-panel/bitty-plugin.toml` outputs `error: manifest.unknown-key (tools): unknown key 'tools' is not part of the accepted schema`, `invalid (1 error, 0 warnings)`; the same file through the transitional validator outputs `manifest: OK ... (bitty-terminal.git-panel 0.1.0)`; `file-manager` and `activity` are both `valid` under the SDK, confirming the fork is only about the `tools` table.

### R5 (P0, SEC) `git-panel` allows many file-writing flags, blocking only three high-risk flags

- Paths and symbols: `git-panel/lua/git-panel/allowlist.lua:is_allowed_args/is_risky_flag/RISKY_FLAGS`
- Symptom: `RISKY_FLAGS` covers only `--upload-pack/--receive-pack/--exec` and their `=` forms; every other `-`-leading flag is allowed through. Flags that can redirect writes such as `--output`, `--index-file`, `--work-tree`, `--git-dir`, `-o` are not on the denylist, and `has_denied_byte` does not block letters or `-/.=/`, so `{"diff","--output=..."}`, `{"show","--index-file=..."}` and friends pass `is_allowed_args`.
- Trigger: when the caller concatenates user-controlled or remote-controlled strings into `spawn_git` arguments with the first argument being one of the seven verbs, a file write happens under a nominally read-only verb.
- Evidence: a full read of `is_allowed_args` shows five gates — length, count, shell metacharacters, first-argument verb, `is_risky_flag` — with no write-out flag table; `allowlist_spec.lua` only asserts `upload-pack/receive-pack/exec` are rejected, with no `output/index-file/work-tree/git-dir` cases. Escalated SEC as read-to-write.

### R6 (P0) `git-panel` status pipeline filters on host-absolute scope, incompatible with real `status --porcelain` relative paths

- Paths and symbols: `git-panel/lua/git-panel/init.lua:parse_porcelain/status_entries`; `git-panel/lua/git-panel/listing.lua:list_status_entries`; `git-panel/lua/git-panel/scope.lua:is_within_repo`
- Symptom: after `parse_porcelain` takes `XY SP path`, `list_status_entries` requires `scope.is_within_repo(path)`, i.e. a `~/projects/` prefix. Real `git status --porcelain` output is repository-relative paths, so normal output is discarded wholesale; tests feed `~/projects/foo.txt`-shaped input, and the artificial pass hides the mismatch.
- Trigger: running the `status` command in a real repository when `spawn` returns relative paths.
- Evidence: `init_spec.lua`'s `spawn_outputs["status --porcelain"]` three lines are all host-absolute-scope `~/projects/...` forms; `scope_spec.lua` asserts `is_within_repo` rejects everything without the `~/projects/` prefix; `listing_spec.lua` status cases likewise use absolute-scope vectors, with no relative-path case.

### R7 (P1, SEC) Store runtime validation weaker than generation-time validation; `href` and install commands exploitable via poisoned index

- Paths and symbols: `bitty-plugins/app/src/registry.ts:isPlugin/isRegistry`; `bitty-plugins/app/src/components/plugin-detail.ts:PluginDetail.render`; `bitty-plugins/app/src/components/dom.ts:externalLink/el`; `bitty-plugins/app/src/registry.ts:installCommand`
- Symptom: at generation time `REPOSITORY_PATTERN` enforces `https://` with no query credentials, but at runtime `isPlugin` only checks that `id/name/kind/repository/official` are strings and booleans without re-checking formats; `externalLink(plugin.repository, ...)` writes the untrusted URL straight through `setAttribute("href", url)`; `installCommand(id)` is a `bitty plugin add <id>` string concatenation with `id` never re-checked against `ID_PATTERN`, so a poisoned index can inject shell metacharacters to induce copy-paste execution.
- Trigger: `generated/registry.json` tampered with or replaced after build (see R1, no signatures) when the page renders a poisoned entry.
- Evidence: the `dom.ts` header comment says everything through this module avoids `innerHTML`, but `href` has no allowed-scheme whitelist; `registry.ts` has no `ID_PATTERN/REPOSITORY_PATTERN` re-check; `install-command.ts` copies via the copy button straight to `navigator.clipboard`, prompting manual selection only on failure, with no second confirmation showing the parsed result.

### R8 (P1, SEC) Versions syntax-checked but never parsed; `compat` dual-track naming can drift

- Paths and symbols: `bitty-plugins/scripts/semver.ts:isValidVersionRange`; `bitty-plugins/scripts/registry-lib.ts:validateEntry`; `file-manager/scripts/validate-manifest.mjs:validateVersionRequirement`; `bitty-plugin-sdk/src/manifest.ts:VERSION_REQ`
- Symptom: the registry `isValidVersionRange` only tokenizes with a regex and its comment states it does no parsing; the transitional validator and SDK `validateVersionRequirement/VERSION_REQ` are charset-only checks, so even `>>>`- or `|||`-shaped character combinations pass; the registry `compatibility` uses `bitty/sdk` while manifests' `compat` uses `bitty/plugin-api`, and `sdk ^0.1` vs `plugin-api ^1.0` have no cross-check.
- Trigger: a manifest writing `compat.bitty = ">>>"` passes local `just manifest` and only surfaces at registry or host resolution; when `sdk` and `plugin-api` each upgrade one side while the other lags, nobody warns.
- Evidence: `registry.test.ts` asserts `isValidVersionRange("1.2.3 - 2.0.0") == false`, but that is at the registry layer; the transitional validator `ALLOWED_VERSION_REQ_CHARS` is a single charset with no structural branches; the three official entries' `sdk = "^0.1"` coexists with five manifests' `plugin-api = "^1.0"` with no mapping test.

### R9 (P1, SEC) Registry has no dependency model; manifest `[dependencies]` is an island

- Paths and symbols: `bitty-plugins/registry/schema.json:properties`; `bitty-plugins/scripts/registry-lib.ts:ENTRY_KEYS/COMPATIBILITY_KEYS`; `file-manager/scripts/validate-manifest.mjs:validateDependencies`
- Symptom: the manifest layer has `[dependencies]` (limit 8, self-dependency rejection, version-charset check), while the registry layer `ENTRY_KEYS` has no `dependencies`, so `validateRawKeys` judges an entry attempting to declare dependencies as `unknown key`; cross-plugin dependency conflicts, cycles, and version intersections are computed by nobody.
- Trigger: once any plugin declares a version dependency on another plugin and publishes, the index cannot express it and the installer can only install blind.
- Evidence: `schema.json` has no `dependencies` key; the `ENTRY_KEYS` set lacks it; neither `sync-metadata.ts` nor `generate-index.ts` has dependency-graph logic.

### R10 (P1) Official-manifest consistency checks and the SDK gate currently always spin idle

- Paths and symbols: `bitty-plugins/scripts/validate-registry.ts:checkLocalManifests/discoverSdkCli/ensureSdkDependencies`
- Symptom: `checkLocalManifests` looks for `plugins/<repo-basename>/bitty-plugin.toml`, but `bitty-plugins/plugins/activity|palette|statusline` are empty directories (submodules not expanded), so with no files it just `continue`s; `discoverSdkCli` looks for `sdk/package.json`, but `bitty-plugins/sdk/` is an empty directory while the real SDK lives in the sibling repository `bitty-plugin-sdk/`, so it always takes the `SDK manifest tooling is unavailable ... skipping SDK lint` branch.
- Trigger: every run of `validate-registry.ts` in the current checkout state.
- Evidence: directory listings show the three placeholders under `bitty-plugins/plugins/` all empty and `bitty-plugins/sdk/` empty; both `existsSync` failures in the code silently skip with only a `notice` print, leaving the exit code unaffected.

### R11 (P1) Repository-existence check silently skips on offline; `--skip-network` normalization hides 404s

- Paths and symbols: `bitty-plugins/scripts/validate-registry.ts:checkRepositoriesOnline/fetchWithTimeout`
- Symptom: the first network exception sets `offline = true; break` and remaining entries are never checked; `HEAD` has a 5 s timeout, only `404/410` error out, other `>=400` only warn; in offline CI or proxy jitter the whole existence gate is void.
- Trigger: offline CI, proxy timeouts, first-repository handshake failure.
- Evidence: the function body `catch { offline = true; console.log("notice: network unavailable ... skipping remaining") }`; `USAGE` and `REGISTRY_SKIP_NETWORK=1` provide a legitimate skip path with no failed-count distinction.

### R12 (P1) `statusline` event handlers lack `pcall`; a single snapshot rejection can break dispatch

- Paths and symbols: `statusline/lua/statusline/init.lua:refresh/snapshot`; contrast `file-manager/lua/file-manager/init.lua:refresh_cache` with three `pcall`s, `git-panel/lua/git-panel/init.lua` with three `pcall`s on the event side
- Symptom: `snapshot()` intentionally lets rejections propagate, `refresh()` calls `bitty.terminal.snapshot` directly, and the `terminal.cwd-changed/title-changed` subscription bodies are bare `refresh()` calls. On rejection the error is thrown at the dispatcher; the other two plugins in the ecosystem both `pcall(refresh_cache)` on the event side to keep the last-known value alive.
- Trigger: when authorization is revoked or the snapshot surface is temporarily unavailable while an observed event arrives.
- Evidence: `init.lua` lines 75-86 comment that it must fail closed, lines 96-105 `refresh` unprotected, lines 110-116 subscriptions without `pcall`; `init_spec.lua` asserts via `pcall(host.publish, ...)` that the error propagates, directly contradicting `file-manager`'s keep-alive semantics.

### R13 (P1) `statusline` settings unclamped; negative and huge numbers can blank or bloat output

- Paths and symbols: `statusline/lua/statusline/init.lua:options`; `statusline/lua/statusline/format.lua:components/option`
- Symptom: `max_components/component_max_chars` only check `type == "number"` and pass through; `format.components` takes `option(opts,"max_components",8)` then does `if #comps > max_components` and `for index=1,max_components` directly, so negatives yield an empty table and huge numbers make the limit nominal; negative `component_max_chars` empties every component via `char_slice(max<=0 -> "")`, with `NaN/inf` unhandled.
- Trigger: a user or synced settings writing an out-of-range number.
- Evidence: `options()` lines 64-71 have no range branch; `format.lua` lines 126-157 have no clamping; `init_spec.lua` covers only normal booleans and separators, with no numeric-boundary cases.

### R14 (P1) `statusline/format` full-scans `zones`; O(n) per refresh on large snapshots

- Paths and symbols: `statusline/lua/statusline/format.lua:latest_cwd/latest_exit/components`
- Symptom: both `latest_cwd/latest_exit` linearly scan from `#zones` backwards to the first hit, with no length limit and no cache; `components` recomputes on every event; comments say the mirror implements the bundled 8/64/128 bounds, but the bounds apply only on the output side, not the input-scan side.
- Trigger: a semantic-zone-heavy terminal snapshot (long session, many splits) with high-frequency `cwd-changed/title-changed` events.
- Evidence: lines 81-111 two loops with no `MAX_ZONES` constant; `init.lua:refresh` re-calls `snapshot()` then `format.components` every time; tests use at most single-digit `zones`, with no thousand-scale performance case.

### R15 (P1) `file-manager/scope` and `git-panel/scope` judge traversal by substring `..`, misfiring on legitimate filenames

- Paths and symbols: `file-manager/lua/file-manager/scope.lua:is_within_read_scope/raw_name`; `git-panel/lua/git-panel/scope.lua:is_within_repo`
- Symptom: `string.find(path, "..", 1, true) ~= nil` rejects immediately, so legitimate names containing consecutive dots but no traversal — `foo..bar.txt`, `my..project` — are judged out-of-scope; `raw_name` likewise substring-rejects filename segments. The correct approach is per-`/`-segment comparison with `== ".."`.
- Trigger: listing, previewing, or status-filtering a directory containing a filename with a `..` substring.
- Evidence: `scope.lua` lines 49-51 and 99-101; `git-panel/scope.lua` lines 41-43; both repositories' `scope_spec.lua` only assert `~/projects/../etc/passwd` and `foo/../bar` are rejected, with no `a..b` legitimate-name case.

### R16 (P1) `file-manager` payload bounds declared but never enforced

- Paths and symbols: `file-manager/lua/file-manager/listing.lua:MAX_ENTRIES/MAX_NAME_CHARS/MAX_PATH_BYTES/MAX_SELECTION/PAYLOAD_MAX_BYTES`; `file-manager/lua/file-manager/scene.lua:file_rows/directory/preview`
- Symptom: `PAYLOAD_MAX_BYTES = 8192` is defined and then referenced zero times in the whole library; `list_entries` limits to 128 entries but not total bytes, so 128 entries × 4096-byte paths can theoretically reach ~512 KiB; `scene.preview` limits 32 lines × 128 chars per line without converting to bytes; `file_rows` returns row tables carrying full `path`s with no payload accounting.
- Trigger: a large-directory candidate list passed via `open` or via settings `entries`.
- Evidence: in-library search for `PAYLOAD_MAX_BYTES` hits only the definition line; `listing_spec.lua` asserts only the count bound, with no byte-bound case.

### R17 (P1) `git-panel` output unbounded, fully parsed first; memory and time uncontrolled on large repositories

- Paths and symbols: `git-panel/lua/git-panel/init.lua:status_entries/branch_list/commit_list/diff_lines/spawn_git`; `git-panel/lua/git-panel/listing.lua:list_status_entries/list_commits/list_branches`
- Symptom: `spawn_git` checks `result.output` only for `type == "string"` with no length limit; `parse_porcelain` first cuts all lines with `gmatch("[^\n]+")`, `commit_list` first cuts all lines then `list_commits` truncates to 64, `diff_lines` first collects everything then truncates to 128; `status_entries` calls the 128-truncation once per status group then truncates globally, with intermediate expansion up to ~8x.
- Trigger: ten-thousand-file changelists, extra-long commit histories, huge `diff --stat` output.
- Evidence: `init.lua` lines 115-120 with no length branch; lines 149-206 four functions all looping fully first; `allowlist.MAX_TOTAL_BYTES` constrains only inputs, not outputs.

### R18 (P1) `git-panel` porcelain parsing misses renames, quotes, and empty-path variants

- Paths and symbols: `git-panel/lua/git-panel/init.lua:parse_porcelain/PORCELAIN_STATUS`
- Symptom: `string.match(line, "^..%s+(.-)%s*$")` takes the whole `R  old -> new` arrow-containing text as the path, so scope filtering later most likely drops or mislabels it; quote-wrapped paths (spaces, special characters) keep their quotes; the two-column `XY` semantics map only one column arbitrarily, with no handling of `U` unmerged details or rename-score suffixes.
- Trigger: viewing `status` in a repository with renames, space-containing filenames, or unmerged paths.
- Evidence: lines 133-147 regex and status table; `init_spec.lua` uses only a three-line `M/A/??` simple vector, with no rename or quote cases.

### R19 (P1) `git-panel` commits re-sorted by hash lexicographic order, losing time order

- Paths and symbols: `git-panel/lua/git-panel/listing.lua:list_commits`
- Symptom: `table.sort(commits, function(a,b) return a.hash < b.hash end)` turns the reverse-chronological `git log --oneline` order into hash order; `init.lua:commit_list` hardcodes `log --oneline -n 10` while `MAX_COMMITS = 64`, so the limit is unreachable on this call path.
- Trigger: any multi-commit `log` display.
- Evidence: lines 261-263 sort branch; `listing_spec.lua` asserts `commits[1].hash == "abc1234"`, i.e. dictionary order, contradicting `log` semantics; `init_spec.lua`'s two-commit case happens to be in dictionary order, hiding the problem.

### R20 (P1) `palette` has no input size bound; oversized candidate tables can block switching

- Paths and symbols: `palette/lua/palette/init.lua:entries/refresh`; `palette/lua/palette/filter.lua:filter`
- Symptom: `entries()` returns the settings table as-is, and `filter` traverses everything with `ipairs` before truncating to 128. Output is bounded but input is not: ten-thousand-candidate tables do full `string.lower` plus substring search on every `toggle/focus.changed`.
- Trigger: `entries` written with an oversized array, then switching the panel or moving focus.
- Evidence: `filter.lua` lines 79-96 loop first and only `if #out >= MAX` to break, with no input-length pre-check upstream; `filter_spec.lua` maxes at 300 entries with no ten-thousand-scale case; `init.lua:refresh` has no `pcall` around `bitty.ui.update`, so no degradation on large tables.

### R21 (P1) `palette/init` command lacks argument and result schemas; `toggle` update unprotected

- Paths and symbols: `palette/lua/palette/init.lua:bitty.commands.register`
- Symptom: the `toggle` registration carries only `id/title/description/run` with no `args_schema/result_schema`, while `activity`'s `summary/clear` both declare closed objects and string results; `refresh()` bare-calls `bitty.ui.update`, so mid-session grant revocation throws at the command caller.
- Trigger: the host tightening schema validation, or `ui.rich` revoked then `toggle` called.
- Evidence: lines 71-83 registration table; compare `activity/lua/activity/init.lua` lines 144-193 dual-schema declarations; `init_spec.lua` never asserts schema presence.

### R22 (P1) `activity` event payloads lack type validation; malformed events can pollute aggregates

- Paths and symbols: `activity/lua/activity/init.lua:track_session_open/track_session_close` with four event subscriptions; `activity/lua/activity/aggregate.lua:on_cwd_changed/on_process_exited/on_session_duration`
- Symptom: non-numeric `terminal_id` only silently returns on the pairing side, yet the `terminals_opened/closed` counters still increment, so open/close counts and duration buckets can drift apart long-term; non-string `cwd` becomes `<unknown>` via `redact` yet still counts a `cwd_events` and builds a bucket; non-numeric `exit_code` falls to `unknown` yet still counts toward event totals.
- Trigger: the host or tests injecting missing-field or wrong-type payloads.
- Evidence: lines 116-142 type guards only protect the pairing table; lines 195-225 subscription bodies have no payload validation; `init_spec.lua` deliberately publishes an `opened` without `terminal_id` and asserts the count is 2, baking in the drift.

### R23 (P1) `activity` write-failure count only grows; `retention` semantics overridden by stored value

- Paths and symbols: `activity/lua/activity/init.lua:write_state/settings_number/store_command_args/retention_days`; `activity/lua/activity/aggregate.lua:normalize/load/prune/render`
- Symptom: `write_errors` is never zeroed on success, so `render` shows the historical failure count forever; `retention_days` is read once at startup, then `normalize` overrides it with the stored `raw.retention_days`, so M2M sync or a stale value can hijack the user's narrowing intent long-term; `summary` unconditionally sets `dirty = true` after every `prune`, forcing a write even with no changes.
- Trigger: recovering after one write failure; the user narrowing retention while the stored value is larger.
- Evidence: lines 68-78 `write_state` success branch has no `write_errors = 0`; lines 141-149 `normalize` overrides the argument with the stored value; lines 150-164 `summary` dirty path; `init_spec.lua` covers update-version rejection and prune folding, but not failure-zeroing or settings-priority.

### R24 (P1) `validator-negative` all valid but zero automation; regressions have no gate

- Paths and symbols: `file-manager/validator-negative/bad-cap.toml|bad-param.toml|bare-fs.toml|double-colon.toml|fs-bool.toml|unknown-key.toml|base.toml`; `git-panel/validator-negative/bad-param.toml|bare-spawn.toml|double-colon.toml|fs-bool.toml|tools-no-required.toml|tools-no-version.toml|tools-rg.toml|base.toml`; `file-manager/justfile:manifest`; `git-panel/justfile:manifest`
- Symptom: read-only runs show the transitional validator rejects all 6+7 negative cases with nonzero exits and parameter-shape-precise diagnostics, and both `base.toml`s pass, proving the vectors themselves are high quality; but `just manifest` only validates the root `bitty-plugin.toml`, `just test` only runs `test-lua/test-luals/test-manifest`, and no task walks that directory, so adding/removing/editing negative cases is executed by nobody.
- Trigger: later changes loosening validation, mistyping a regex, or deleting a branch.
- Evidence: read-only runs: `file-manager` six negatives report unknown capability, illegal parameter segment, bare `fs.read` missing parameters, double colon, wildcard parameter, unknown top-level key respectively, `base` reports OK; `git-panel` seven negatives report parameter shape, bare `process.spawn`, double colon, misused `fs` parameters, non-boolean `required`, non-string `version`, unknown `tools.rg` respectively, `base` reports OK; whole-repo search for `validator-negative` hits only research notes, with no just/test/CI references.

### R25 (P1) Template generates an outdated command shape; new repositories are non-compliant out of the box

- Paths and symbols: `bitty-plugin-template/template/lua/@@PLUGIN_MODULE@@/init.lua:bitty.commands.register`; `bitty-plugin-template/template/bitty-plugin.toml:[capabilities]`; contrast `activity/lua/activity/init.lua`, `palette/lua/palette/init.lua`, `file-manager/tests/support/mock_host.lua`
- Symptom: the template registers with `name = "@@PLUGIN_ID@@:hello"`, while all five current plugins and the Mock host require `id = "<resource>"` with non-empty title and `run`; the `name` key is judged definition-illegal; the template comments say the surface is a pre-SDK-settlement sketch, but the generator is neither marked outdated nor given a post-generation lint gate.
- Trigger: creating a plugin from the template then running `test-lua` or authoritative lint directly.
- Evidence: template `register({ name = ..., description = ..., run = ... })`; the `file-manager` Mock requires `def.id` to be a string and `title` non-empty, else `E_DEF_INVALID`; all of `activity/palette/statusline/file-manager/git-panel` use the `id` shape.

### R26 (P2) Registry entry-level validation loose in three places

- Paths and symbols: `bitty-plugins/scripts/registry-lib.ts:validateEntry/validateSlugArray/validateRegistry/checkLicense`
- Symptom: first, the same repository URL can be reused by different `id`s with no duplicate-repository warning, so mirrors and forks are easily confused; second, tag dedup is case-sensitive, so `["a","A"]` is rejected on pattern error rather than duplicate error with unstable diagnostics; third, unknown SPDX only warns, so `Some-Custom-License` can enter the index since warnings never block generation.
- Trigger: growing community entries and bulk imports.
- Evidence: `validateRegistry` builds only a `byId` table with no `byRepository` table; `validateSlugArray` uses a raw-value `Set`, and tests on `["a","A"]` only assert length greater than zero; `checkLicense` returns `warnings` for non-allowlisted names, and `generate-index.ts` refuses to write only when `error > 0`.

### R27 (P2) Manifest transitional validator charset-only on version requirements

- Paths and symbols: `file-manager/scripts/validate-manifest.mjs:validateVersionRequirement`; `git-panel/scripts/validate-manifest.mjs:validateVersionRequirement`; `bitty-plugin-sdk/src/manifest.ts:VERSION_REQ`
- Symptom: all three are single-line charset regexes with no comparator-structure, empty-branch, or `||`-trailing-branch checks; `">>>"`-class inputs pass the transitional gate and are only rejected at the registry layer, postponing error discovery by one stage.
- Trigger: hand-writing a `compat` range.
- Evidence: `ALLOWED_VERSION_REQ_CHARS/VERSION_REQ` single-line charsets; `semver.ts` has structural parsing but is only used on the registry side.

### R28 (P2) `statusline` reactive table inconsistent with manifest subscriptions; `has_status` and `components` disagree

- Paths and symbols: `statusline/lua/statusline/format.lua:is_reactive_event/has_status/latest_exit/components`
- Symptom: `is_reactive_event` returns true for `focus.changed/terminal.bell`, but the manifest `events` declares only `cwd-changed/title-changed`, and the function has no callers; `has_status` treats non-numeric `exit_code` as having status, while `components` only emits a segment when `type(exit) == "number"`.
- Trigger: reading the code for event extension, or a snapshot carrying a dirty `exit_code` type.
- Evidence: lines 187-192 four-true-two-false; lines 175-181 vs 144-149 type-criteria difference; `init_spec.lua` has no `has_status` dirty-type case.

### R29 (P2) `file-manager` assorted boundaries and dead code

- Paths and symbols: `file-manager/lua/file-manager/scope.lua:is_path_bounded/file_name/parent_dir/is_directory_path`; `file-manager/lua/file-manager/listing.lua:list_entries/entry_from_path`; `file-manager/lua/file-manager/init.lua:rename_pair`
- Symptom: `is_path_bounded` counts code points as `MAX_NAME_CHARS * 4` with no conversion to `MAX_PATH_BYTES = 4096` and is referenced zero times library-wide; `list_entries` iterates `ipairs(paths or {})` with no table guard, so a string input throws an argument error; `rename_pair` does not reject no-op `src == dst` renames; `is_directory_path` strips the trailing slash before judging scope for `~/projects/foo/`, yet `entry_from_path` keeps the original slash path for directories, so the same path has two coexisting normalizations.
- Trigger: abnormal settings, no-op rename confirmations, mixed directory-slash variants.
- Evidence: lines 68-85 boundary function with zero references; lines 78-81 loop; lines 144-158 `rename_pair` with no equality branch; `scope_spec.lua` has no `is_path_bounded` case, `listing_spec.lua` has no non-table-input case.

### R30 (P2) `git-panel` branch names miss the `@{` forbidden shape; selection bound fights branch bound

- Paths and symbols: `git-panel/lua/git-panel/listing.lua:is_valid_branch_name/MAX_BRANCHES/MAX_SELECTION`; `git-panel/lua/git-panel/scene.lua:branch_rows/branches`
- Symptom: `is_valid_branch_name` blocks `.. ~ ^ : ? * [`, `//`, `.lock`, leading/trailing `/`, `.`, but not `@{` or lone `@`, differing from the simplified `git check-ref-format` shape; `MAX_BRANCHES = 32` while `MAX_SELECTION = 64`, with `scene.branch_rows(branches, 64)` defaulting to 64 yet `scene.branches` explicitly passing 32 — two upper bounds for the same view.
- Trigger: a malformed branch line containing `@{n}` entering the list, or a long branch table going through different scene entries.
- Evidence: lines 86-131 character loop with no `@` branch; lines 18-21 dual constants; lines 26-54 two functions passing different values; `listing_spec.lua` has no `@{` case.

### R31 (P2) `git-panel/file-manager` shared comments outdated: host-bridge missing surfaces changed, comments not synced with failure-code contracts

- Paths and symbols: `git-panel/lua/git-panel/init.lua:spawn_git`; `file-manager/lua/file-manager/init.lua:snapshot`; `git-panel/tests/support/mock_host.lua`; `file-manager/tests/support/mock_host.lua`
- Symptom: comments say the real bridge has no `bitty.process` yet the Mock already provides that surface with seven-verb gating, tests assert the `E_SPAWN_UNAVAILABLE` fallback path, and docs never say when that code retires; `file-manager` comments say observer handlers never touch the filesystem, yet `open/preview/rename` on the command side `refresh_cache` first then read settings candidates, and the event/command failure codes (`E_SNAPSHOT_UNAVAILABLE/E_FS_DENIED/E_CAPABILITY_DENIED`) are scattered across comments with no unified error-code-table test.
- Trigger: the host adding `process.spawn` or tightening snapshot grants, requiring plugin and test changes in lockstep.
- Evidence: `init.lua` lines 97-120 suppression note plus three failure codes; Mock header comments calling themselves non-host implementations; `init_spec.lua` has one case each for missing surface, missing grant, and over-scope, but no error-code-table snapshot test.

## Recommendations

1. Registry integrity phase 1: add optional `signature { algorithm, value, signer }` and `manifest_hash` to `schema.json`, add signature verification to `registry-lib.ts` (warn when offline without keys, mandatory verification with keys), have `generate-index.ts` write signature status into the index, and have the store grey out verification-failed entries and disable one-click copy of install commands. Expected benefit: cuts the index-poisoning-to-copy-paste-execution chain. Effort M. Covers R1, R7.
2. Bind fetched identity plus size to sync: `sync-metadata.ts` must compare `plugin.id` after parsing, recording an error on mismatch and refusing to write that entry's `metadata`; `fetchText` adds a `Content-Length` pre-check with 256 KiB truncation, recording over-limit as a warning and keeping old `metadata`. Expected benefit: prevents version mis-hanging and large-body sync DoS. Effort S. Covers R2, R3.
3. Merge versions and dependencies to a single source: reuse the `semver.ts` structural parser to replace the three charset checks, sharing one function between `validate-registry` and the transitional validator; add optional registry `dependencies { id: range }` with version-intersection and cycle detection, or explicitly declare dependencies unsupported and have the manifest validator reject the table. Expected benefit: errors move earlier; the installer has evidence to work from. Effort M. Covers R8, R9, R27.
4. Fix the hollow official gate: materialize `plugins/*` and `sdk` inside `bitty-plugins` via `git submodule` or workspace-relative references, or change `validate-registry.ts` local-manifest comparison to take standalone-repository paths explicitly; support a `BITTY_PLUGIN_SDK_DIR` override in `discoverSdkCli`. Expected benefit: restores id/name/license consistency checks plus authoritative lint. Effort S. Covers R10.
5. Tiered network checks: on offline skip only unchecked entries with a warning count and exit zero, keep `404/410` as errors, have `--skip-network` report the skipped count; add a duplicate-repository warning. Expected benefit: readable CI signal; no more whole-skip on first timeout. Effort S. Covers R11, R26.
6. Store runtime hardening: have `isRegistry` reuse `ID_PATTERN/REPOSITORY_PATTERN`, add an `https:` scheme whitelist to `externalLink`, and have `installCommand` return empty and hide the copy button for illegal `id`s. Expected benefit: a poisoned index degrades to unclickable, uncopyable. Effort S. Covers R7.
7. `git-panel` allowlist to default-deny flags: allow only a known read-only flag table (e.g. `--porcelain/--stat/--oneline/-n/-a/--abbrev-ref/--others`, etc.), rejecting all unknown `-`-leading flags; add an 8 KiB truncation with a truncation bit to `spawn_git` output; merge `open`'s snapshot read plus two derived calls into one snapshot plus batched derivation. Expected benefit: closes read-to-write; bounds large-repository memory. Effort M. Covers R5, R17.
8. Fix `status --porcelain` path semantics: define relative paths as relative to the repository root, change `list_status_entries` to relative-path validity plus host-side absolutization, or join with the cached `cwd` as base in the `init` layer before scope checks; add rename-arrow, quote, and C-style-escape branches to `parse_porcelain`; keep input order in `list_commits` with a separate hash-sorted display-only function. Expected benefit: real-repository status becomes usable; renamed files no longer vanish. Effort M. Covers R6, R18, R19.
9. Unify event-failure semantics: `statusline` observer handlers likewise `pcall(refresh)` for keep-alive, or document that this plugin deliberately breaks dispatch with host-side isolation requirements; clamp `options` `max_components` to 1-8 and `component_max_chars` to 8-64, rejecting `NaN/inf`; cap `zones` scanning (e.g. first 256 zones) with truncation recorded. Expected benefit: a single rejection no longer spreads; abnormal settings no longer blank output. Effort S. Covers R12, R13, R14.
10. Fix scope substring false positives: change both `scope`s to per-`/`-segment `== ".."` checks, rejecting empty segments and trailing-dot variants; `file-manager` implements `PAYLOAD_MAX_BYTES` accounting or deletes the constant, `list_entries` adds a table guard, `rename_pair` rejects `src == dst`. Expected benefit: legitimate filenames work; payload bounds mean what they say. Effort S. Covers R15, R16, R29.
11. `palette/activity` input and error convergence: `palette` length-pre-checks `entries` growth (e.g. truncate past 1024 with a bit recorded), `pcall`-downgrades `ui.update` inside `refresh`; `palette toggle` gains closed schemas; `activity` zeroes on successful writes, prefers settings over stored retention, dirties only on change, and adds payload type guards to events. Expected benefit: large inputs never block; failure counts trustworthy. Effort S. Covers R20, R21, R22, R23.
12. Gate the negative cases and template: add `test-negative` to both repositories' justfiles walking `validator-negative/*.toml` and asserting only `base.toml` passes; change template `init.lua` to the `id` shape with schemas, auto-running transitional validation plus `luaparse` after generation. Expected benefit: high-quality negative cases no longer sleep; the template works out of the box. Effort S. Covers R24, R25.
13. Add branch and status boundary cases: branches gain `@{`, lone `@`, leading-`-` cases; status gains relative-path, rename, quote cases; commits gain shuffled-input order-preservation cases. Expected benefit: locks this round's findings into regressions. Effort S. Covers R18, R19, R30.

## Test gaps

- Registry: no signature verification, no mismatched-manifest, no oversized-remote, no duplicate-repository, no `sdk` vs `plugin-api` drift, no offline-tiering assertions; `loadEntries` corrupt-TOML and missing-file coverage only light.
- Versions and dependencies: no real version-intersection resolution, no cyclic dependencies, no `>>>`-class malformed ranges as transitional-validator negatives; `semver` snapshots syntax only.
- Store frontend: no poisoned-index runtime cases (`javascript:` repository, illegal-`id` install command, extra-long descriptions/tags), no illegal-route segments, no empty index, no page-level degradation cases for malformed `registry.json`.
- `activity`: no malformed-event-payload matrix, no `write_errors` zeroing, no settings-vs-stored retention conflict, no timer-budget-exhausted sync writes, no 64+-session or 32+-entry long-soak cases.
- `palette`: no ten-thousand-candidate performance case, no non-table `entries`, no oversized-query with mixed multibyte boundaries, no `ui.update` throw degradation, no host-tightening case for missing schemas.
- `statusline`: no oversized-`zones` (e.g. thousand-scale) timing assertion, no negative and `NaN/inf` for `max_components/component_max_chars`, no dirty `exit_code` types, no contract test for `focus/bell` judged reactive yet unsubscribed.
- `file-manager`: no large-directory (e.g. ten-thousand-candidate) timing or payload-byte assertions, no `PAYLOAD_MAX_BYTES`, no `a..b` legitimate names, no non-table inputs, no `src == dst`, no slash-variant normalization, no settings-candidate vs explicit-parameter priority conflict.
- `git-panel`: no `--output/--index-file/--work-tree/--git-dir` negatives, no oversized `status/log/diff` truncation, no real relative-path vectors, no rename or quote paths, no commit order preservation, no `@{` branches, no `open` double-derive counts or timing, no missing-surface plus missing-grant combination matrix.
- Validators: `validator-negative` zero automation; no SDK-vs-transitional-validator consistency matrix (especially `[tools.git]`); no end-to-end lint of template-generated output.
- LuaLS negatives: `negative-fixture.lua` covers only alias namespaces, old event names, bare snapshots, and the singular task surface — not `os.execute/io.popen/dofile/loadstring`, native module loading, `bitty.process` over-privileged direct calls, or `bitty.api` variant spellings.

Date: 2026-09-15

Covered repositories: `bitty-plugins/`, `activity/`, `palette/`, `statusline/`, `file-manager/`, `git-panel/`, with `bitty-plugin-sdk/` and `bitty-plugin-template/` as references.
