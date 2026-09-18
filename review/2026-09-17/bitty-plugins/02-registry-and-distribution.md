# Registry and distribution review — 2026-09-17

## Scope, baseline, and conclusion

Read-only review of the `bitty-plugins` registry, generated catalog, storefront,
metadata download scripts, validation and integration gates, plus the local
install/update/store and package-resolution implementation in `bitty` needed to
trace distribution beyond the storefront. Standalone plugin implementations,
SDK, template implementation, general host capability enforcement, and panel
implementations are assigned elsewhere and are not audited here.

All source references below are **workspace-relative** and refer to the observed
working tree, not to remote main. No fetch, installation, source edit, commit,
CarryCtx command, nested agent, exploit, or vulnerability reproduction was
performed. The only authored file is this report. Existing bounded unit tests
were permitted; no new test or fixture was written.

| Repository           | HEAD                                       | Initial dirty baseline                                            | Recheck                                            |
| -------------------- | ------------------------------------------ | ----------------------------------------------------------------- | -------------------------------------------------- |
| `bitty-plugins`      | `95bb0bf610885e7d4464cc3056d9318c82c3e4d7` | Clean, including staged diff                                      | Same HEAD; clean; `git diff --check` passed        |
| `bitty-plugins-docs` | `9fdbcd02af835d23868a7c15c5ce12f1efcd093a` | Clean                                                             | Same HEAD; clean                                   |
| `bitty`              | `06bc1f45995fd297a3f7324688bc81b0598ecf67` | Untracked `.targets/`; no tracked modifications reported          | Same HEAD and same reported dirty path             |
| `research`           | `d70152a079c75204ec37e99a7bf3f54b8190e423` | Untracked `review/2026-09-17/`, including another author's report | Existing reports preserved; this report added only |

The umbrella directory is not a Git repository. An initial Git command there
failed without changing state; subsequent Git commands used the owning repo.

**First pass: 14 findings (3 P1, 11 P2).** The independent second pass verifies
PLUG-REG-007, PLUG-REG-010, and PLUG-REG-011 as P1 transaction defects by source
trace, with conditional failure/concurrency impacts rather than executed fault
or race evidence. Selected P2 verification and remaining unverified IDs are
listed below. Advisory integrity and deterministic trust stubs are not counted
as a demonstrated production registry authentication bypass. First-pass checks
remain historical results, not second-pass execution evidence.

Priority convention: P1 = installed-state loss/corruption or a broken transaction;
P2 = incorrect validation, resolution, metadata, or lifecycle behavior; P3 = minor
non-blocking defect. This report does not inherit the September 15 report's P0
labels for absent draft registry services.

## Independent second-pass verification

Only the three reports and their sibling README index are edited. No source,
package store, index, submodule, dependency, or workflow state was changed.
No tests, network, installs, CarryCtx, agents, fault injection, concurrency
reproduction, or signature experiments ran in this pass.

- **Stable revisions:** `bitty-plugins`, `bitty`, `bitty-plugins-docs`, and
  `research` remain at the HEADs above, with the same dirty baselines.
  `git submodule status` and `git ls-tree HEAD` confirm the nine pins and eight
  uninitialized mounts below. Local docs `rev-list` confirms the 34-commit gap;
  an initial comparison in the wrong repository failed, then succeeded in the
  docs repository. No remote freshness or pinned-content validation is claimed.
- **Verified P1, PLUG-REG-007:** `package.rs:274-280,305-306,378-435` and
  `resolution.rs:230-259,469-478` confirm same-version collision checks only
  module-tree content. With a `lua/` subtree, a changed manifest semantic hash
  can be indexed while the old body is retained. Resolution then fails closed.
  Cosmetic edits preserving the semantic hash are **not** the claimed trigger;
  root-only layouts can include the manifest in their tree digest. No install
  was executed, and no arbitrary-execution conclusion follows.
- **Verified P1, PLUG-REG-010:** `resolution.rs:127-145` unconditionally
  attempts deletion on rename error. Loss requires deletion to succeed and the
  replacement to fail or be interrupted; **not every rename failure erases the
  index**. `load_index:99-103` treats the resulting absence as empty. This is
  a source-proven failure-atomicity defect, not a platform fault-test result.
- **Verified P1, PLUG-REG-011:** traced install plus enable/uninstall
  read-modify-write bodies (`package.rs:293-435,457-501`), shared temporary
  pointer (`resolution.rs:132`), and direct CLI call (`plugin.rs:1804-1838`).
  No lock/revision check spans them. Lost updates are conditional on overlapping
  processes; no race was executed and no occurrence frequency is claimed.
- **Verified P2:** PLUG-REG-001 raw optional-type normalization loss and raw-key
  checker omissions (`registry-lib.ts:294-338,531-620` versus `schema.json`);
  PLUG-REG-002 browser optional-field guard/consumer mismatch
  (`app/src/registry.ts:80-150`, `app/src/main.ts:73-93`);
  PLUG-REG-003 ID-only metadata retention (`registry-lib.ts:1397-1410`,
  `sync-metadata.ts:281-334`); PLUG-REG-005 checkout SHA substitution
  (`validate-registry.ts:402-416`); PLUG-REG-006 raw-byte versus canonical H-B
  contract contradiction (`README.md:148-156`, `schema.json:28-32`, lifecycle
  RFC `:143-178`); PLUG-REG-008 retention loss on unchanged reinstall
  (`package.rs:429-435,737-759`); PLUG-REG-009 update enablement overwrite
  (`plugin.rs:1822`, `package.rs:418`). No behavioral reproduction was run.
- **Not independently revalidated in full:** PLUG-REG-004 (raw URL mapping
  sampled, official lookup/equality not retraced), PLUG-REG-012 through
  PLUG-REG-014, all schema numeric/length bounds within PLUG-REG-001, historical
  review dispositions, performance analysis, and the remaining assurance gaps.
  Original priorities there are first-pass claims, not newly verified findings.

### Trust and documentation qualifications

`registry-lib.ts:1362-1364` generates only unsigned/unverified status. The
conditional positive browser badge (`plugin-detail.ts:110-155`) is a future or
externally supplied index concern, not evidence that the current generator
asserts verification. Manifest metadata comes from floating HEAD
(`sync-metadata.ts:52-67`), independently of the reviewed gitlink; cached display
metadata is not authenticated release provenance.

The in-memory crate's signature mechanism is explicitly a deterministic stub
(`bitty/crates/bitty-package/src/trust.rs:275-284`). Keep it out of real trust
decisions; its mere existence does not prove an exploitable remote install chain.
The actual CLI rejects remote sources (`plugin.rs:1815-1820`), and local install
records `SourceClass::LocalPath` (`package.rs:411-419`). Registry download,
publisher authentication, deployed assets, and actual hash-to-pin equality were
not verified. The raw/H-B contradiction is a verified P2 documentation/data
contract defect, not a promoted P1/P0 authentication finding.

## Architecture and inventory

The implemented paths are distinct:

1. `registry/official/*.toml` and `registry/community/*.toml` enter
   `loadEntries`, raw-key checks, entry checks, and cross-entry checks.
2. `generate-index.ts` canonicalizes identity fields and preserves previously
   synchronized metadata. It does not download packages or verify signatures.
3. `sync-metadata.ts` downloads bounded manifest text, checks the plugin ID, and
   refreshes display metadata. It is not an installer or a release resolver.
4. Vite copies `generated/registry.json` into the static site. The browser loads
   that asset, searches it, and displays a **proposed** copyable install command.
5. Actual local installation is in
   `bitty/crates/bitty-runtime/src/plugin_runtime/package.rs`, called by
   `bitty/crates/bitty-app/src/plugin.rs`. It stages a local directory, obtains
   capability consent, commits a version tree, and rewrites `current.json`.
6. `bitty-package` supplies separate in-memory source, resolver, integrity,
   lifecycle, lock, and activation models. Its README explicitly disclaims file
   I/O, networking, process spawning, VM contact, and production registry wiring.
   Those models must not be mistaken for the local installer's filesystem
   transaction or for a working download service.

The tracked `bitty-plugins` tree was inventoried with `git ls-files`: 84 entries,
including nine gitlinks. It has five TypeScript registry scripts, two workflow
shell scripts, one registry test file, six official records, an empty community
area, one JSON schema, one generated index, five workflows, and the static app.
Ignored local `node_modules` and `app/dist` were observed but not used as source
of truth or rebuilt. No hidden source tree under `store/` exists in this roster;
`app/` is the storefront.

### Submodule baseline

| Gitlink                              | Recorded revision                          | Local state                             |
| ------------------------------------ | ------------------------------------------ | --------------------------------------- |
| `bitty-plugins/docs`                 | `ee19a0dbc10b6f1caf68d90627958a8b4cc4780c` | Uninitialized                           |
| `bitty-plugins/sdk`                  | `c9c962fec9ca0595339b95d3f8089293e48ecebc` | Uninitialized                           |
| `bitty-plugins/template`             | `58b2209e38990e3a689f372ee0952484fa684b03` | Uninitialized                           |
| `bitty-plugins/plugins/activity`     | `8030805d112a0b2424e0edc0b017be44b5c24295` | Uninitialized                           |
| `bitty-plugins/plugins/file-manager` | `ada74ad978243e5808c479355a3d616e9fcb3e82` | Uninitialized                           |
| `bitty-plugins/plugins/git-panel`    | `b6b41361d0433b2be2681f0022f43d6c4b225e66` | Uninitialized                           |
| `bitty-plugins/plugins/palette`      | `3497c70ac5b22e52826304b801af302da454d262` | Uninitialized                           |
| `bitty-plugins/plugins/statusline`   | `3eab0f44b9bf76fc8c01a029176a9bd885f91d07` | Uninitialized                           |
| `bitty-plugins/plugins/wheel`        | `524ce0766c969ee84add1ce0a485bfa370a36cf9` | Initialized, matching recorded revision |

No submodule was initialized or refreshed. The sibling `bitty-plugins-docs`
checkout supplied current contracts; the mounted docs pin is 34 commits behind
that checkout by the local `rev-list` comparison. This is a measured code-mount
gap, not a claim about any remote or the separate `bitty-docs` aggregator mount.

## Confirmed defects

### PLUG-REG-001 — P2: normalization discards schema-invalid values before validation

- **Evidence:** `bitty-plugins/scripts/registry-lib.ts:294-338` defaults a
  non-string `kind` and drops incorrectly typed optional fields;
  `bitty-plugins/scripts/registry-lib.ts:531-620` checks ordinary unknown keys
  but only performs explicit type rejection for integrity fields.
  `bitty-plugins/scripts/registry-lib.ts:370-405` and
  `bitty-plugins/scripts/registry-lib.ts:497-528` omit several schema bounds.
- **Cause:** handwritten validation runs against a lossy normalized object;
  `SCHEMA_FILE` is declared but the schema is not applied to the raw entry.
  The schema requires repository length at most 200, author length at most 64,
  at most ten tags/categories, and license length at most 100
  (`bitty-plugins/registry/schema.json:22-26`, `:66-103`). Compatibility limits
  also differ between the schema's 64 characters and the parser's 128 bytes.
- **Impact:** invalid catalog edits can pass generation with a changed default
  or missing compatibility/tags, while schema-aware consumers reject the same
  source. Declared schema bounds do not bound the generated catalog.
- **Recommended fix:** validate raw types and all agreed bounds before
  normalization; keep one executable contract or a schema-parity matrix.
- **Safe regression idea:** table-driven, in-memory raw-object cases for each
  optional field's wrong type, exact accepted limit, and first rejected limit;
  assert invalid input never becomes a defaulted valid entry. Current tests
  emphasize typed `RegistryEntry` values and integrity-field type checks.

### PLUG-REG-002 — P2: browser type guard accepts optional fields that renderers cannot consume

- **Evidence:** `bitty-plugins/app/src/registry.ts:80-115` validates required
  identity fields and signature status only. Consumers call array and string
  methods at `bitty-plugins/app/src/registry.ts:146-150`,
  `bitty-plugins/app/src/components/plugin-card.ts:78-82`,
  `bitty-plugins/app/src/components/plugin-detail.ts:92-106`, and
  `bitty-plugins/app/src/search.ts:43-52`.
- **Cause:** `isRegistry` asserts a complete TypeScript type without checking
  `tags`, `categories`, `author`, `description`, `compatibility`, or `metadata`.
  It also accepts any numeric schema version. The load catch ends before page
  construction (`bitty-plugins/app/src/main.ts:73-93`).
- **Impact:** malformed optional display data can pass the loading guard and
  then abort initial rendering or search, rather than show the registry error
  page. This is a defensive input-validation defect, not a reproduced exploit.
- **Recommended fix:** validate all consumed optional shapes, supported schema
  version, and bounded lengths/counts at the browser boundary. Handle rendering
  failures explicitly rather than relying solely on the fetch catch.
- **Safe regression idea:** inert wrong-type optional fields should fail the
  guard; empty valid catalogs should render; a rejected catalog should show the
  normal error state. Existing runtime tests cover IDs, URL format, signature
  status, and copy gating, not optional-field rendering.

### PLUG-REG-003 — P2: cached metadata survives a repository change under the same ID

- **Evidence:** `bitty-plugins/scripts/registry-lib.ts:1402-1409` preserves
  metadata by ID alone. `bitty-plugins/scripts/sync-metadata.ts:281-283` starts
  from that preserved metadata; failures and missing manifests leave it in
  place (`:294-324`, `:332-335`).
- **Cause:** metadata ownership is not keyed to the entry's source identity.
- **Impact:** a legitimate repository migration can display the former
  repository's version, license, description, and source indefinitely after
  offline/failed synchronization. The unchanged-ID generation test does not
  distinguish source changes from cosmetic entry changes.
- **Recommended fix:** invalidate cached metadata when the normalized source
  changes, or explicitly retain it as stale with old provenance separated from
  current-source metadata. Do not imply successful refresh of the new source.
- **Safe regression idea:** build an index twice with one ID and different
  repository identities; assert old metadata is removed or marked stale, while
  an equivalent URL spelling retains it. Extend
  `bitty-plugins/tests/registry.test.ts:1006-1030`.

### PLUG-REG-004 — P2: accepted repository URLs are mapped inconsistently

- **Evidence:** `bitty-plugins/scripts/sync-metadata.ts:52-67` takes only the
  first two path segments and preserves a `.git` suffix in raw URLs.
  `bitty-plugins/scripts/registry-lib.ts:822-826` also preserves `.git` in the
  basename used for official checkout lookup, while
  `bitty-plugins/scripts/registry-lib.ts:1056-1060` strips it for URL equality.
- **Cause:** separate URL helpers encode different repository identity rules.
  The schema accepts more than two path segments, including GitLab subgroups.
- **Impact:** nested GitLab repositories resolve to the wrong manifest path;
  accepted `.git`-suffixed registry URLs can miss metadata and incorrectly
  demand a different submodule directory. Existing `.git` coverage only varies
  the submodule URL, not the registry-side basename (`tests/registry.test.ts:863`).
- **Recommended fix:** centralize normalized repository identity; preserve
  GitLab namespace segments and strip only a terminal repository suffix.
  Reject unsupported host/path shapes rather than silently selecting a parent.
- **Safe regression idea:** pure URL-to-identity/manifest-path assertions for
  ordinary repositories, suffix variants, and nested namespaces; no HTTP calls.

### PLUG-REG-005 — P2: pin reachability can validate the checkout instead of the committed gitlink

- **Evidence:** `bitty-plugins/scripts/validate-registry.ts:402-416` obtains
  pins from `git submodule status` and accepts its differing-checkout marker
  without distinction. Those SHAs feed reachability at `:502-512`.
- **Cause:** working-tree checkout status is treated as the recorded parent
  revision. The documented reviewed pin is the gitlink, not whichever commit
  is currently checked out inside an initialized submodule.
- **Impact:** local validation can report the mainline reachability of a
  different commit from the one the parent repository will distribute. CI's
  normal clean checkout reduces exposure but does not correct local semantics.
- **Recommended fix:** choose and document the parent tree/index being reviewed;
  read its gitlink object ID directly and separately reject/report checkout
  drift. Do not discard status markers that affect the meaning of the SHA.
- **Safe regression idea:** unit-test pin extraction from inert recorded-tree
  and checkout-status fixtures, asserting that checkout divergence cannot
  substitute a different revision for the reviewed gitlink.

### PLUG-REG-006 — P2: official manifest hashes have two incompatible meanings

- **Evidence:** `bitty-plugins/registry/schema.json:28-32` describes
  `manifest_hash` as canonical-form H-B. The new procedure at
  `bitty-plugins/README.md:148-156` and comments at
  `bitty-plugins/registry/official/activity.toml:7-10` instead define a digest
  of raw TOML bytes at the pin. The accepted distinction is explicit in
  `bitty-plugins-docs/specifications/package-lifecycle-rfc.md:143-178`.
- **Cause:** Phase 1 raw-file pins reuse the canonical semantic-hash field
  without an encoding discriminator or contract migration.
- **Impact:** these advisory values cannot safely be consumed as H-B by a
  future installer; cosmetic TOML edits change the raw digest, while canonical
  hashing deliberately does not. This is a present schema/data contract
  contradiction, not evidence of an active signature bypass.
- **Recommended fix:** distinguish raw-manifest transport digests from the
  versioned canonical semantic digest, or populate the agreed canonical digest.
  Make the index and schema state exactly which bytes are bound.
- **Safe regression idea:** two semantically identical manifests with different
  formatting should have equal H-B and distinct raw-file digests; semantic
  changes must alter H-B. Separately compare official pins to their declared
  digest under the selected algorithm. Raw hash values were not independently
  recomputed for uninitialized plugin submodules in this review.

### PLUG-REG-007 — P1: same-version manifest-only update can succeed but become unloadable

- **Evidence:** `bitty/crates/bitty-runtime/src/plugin_runtime/package.rs:378-398`
  reuses an existing version directory after comparing only the module-tree
  digest and deletes the newly staged tree. It writes the new manifest hash
  into the index at `:411-421`. Runtime checks that hash against the retained
  manifest body at
  `bitty/crates/bitty-runtime/src/plugin_runtime/resolution.rs:230-246`.
- **Cause:** version-directory identity does not include the manifest. When Lua
  is under `lua/`, the compared content digest does not include the root
  manifest (`package.rs:634-645`).
- **Impact:** changing only manifest semantics without bumping the version can
  return a successful install/update while leaving the old manifest on disk and
  the new hash in `current.json`; subsequent resolution fails closed. This is
  an availability/transaction defect, not a claim that unapproved code runs.
- **Recommended fix:** include manifest identity in the immutable-version
  collision check. Reject any different same-version package before changing
  the index, or use a fully transactional content-addressed layout.
- **Safe regression idea:** a benign manifest-only change with unchanged Lua
  should either be rejected without state change or resolve successfully after
  commit. The existing same-version test changes a Lua file instead
  (`bitty/crates/bitty-runtime/src/plugin_runtime/package.rs:1139-1153`).

### PLUG-REG-008 — P2: idempotent reinstall deletes the retained previous version

- **Evidence:** `bitty/crates/bitty-runtime/src/plugin_runtime/package.rs:429-435`
  takes `previous_version` from the currently indexed record on every install.
  `:737-759` retains only `current` and that one previous value.
- **Cause:** an idempotent reinstall of the current version makes both retained
  names equal, forgetting the genuinely previous version.
- **Impact:** after a successful update retained the former version, reinstalling
  the unchanged current package prunes that rollback material. The test at
  `package.rs:904-924` covers reinstall before updating, not after updating.
- **Recommended fix:** preserve retention metadata independently of the current
  record; an unchanged install should not advance history or prune its predecessor.
- **Safe regression idea:** existing local install tests should assert that an
  unchanged reinstall after an update leaves both current and previous trees
  intact, without executing plugin code.

### PLUG-REG-009 — P2: updating a disabled package implicitly enables it

- **Evidence:** `bitty/crates/bitty-app/src/plugin.rs:1822` always passes
  `enable: true`. `bitty/crates/bitty-runtime/src/plugin_runtime/package.rs:418`
  writes it over the existing record regardless of prior desired state.
- **Cause:** fresh-install defaults and update policy share one unconditional
  boolean rather than preserving the user's existing enabled/disabled choice.
- **Impact:** a package deliberately disabled by the user is enabled after a
  local update or reinstall, including updates with no added capabilities and
  therefore no new consent prompt. The public package contract distinguishes
  install/update from enable/disable.
- **Recommended fix:** preserve existing enablement on updates unless explicitly
  overridden; settle fresh-install defaults separately against the onboarding
  staged-and-disabled requirement
  (`bitty-plugins-docs/product/official-plugin-onboarding.md:83-84`).
- **Safe regression idea:** a disabled record remains disabled after unchanged
  reinstall, newer-version update, and capability narrowing. No VM need run.

### PLUG-REG-010 — P1: rename-error fallback can destroy the old pointer

- **Evidence:** `bitty/crates/bitty-runtime/src/plugin_runtime/resolution.rs:127-143`
  writes a temporary pointer, attempts rename, and on **any** error removes the
  existing target before retrying. `:99-103` interprets an absent pointer as an
  empty store.
- **Cause:** a portability fallback removes the last good pointer instead of
  using a platform-correct atomic replacement or preserving it on failure.
  There is no platform restriction or error-kind restriction on this fallback.
- **Impact:** a failed replacement or interruption in the fallback can erase
  the active index; installed plugins then appear absent. This contradicts the
  atomic-switch claim and can affect all records, not just one update.
- **Recommended fix:** use a replacement primitive with an explicit
  all-or-nothing contract per platform; on failure preserve the old pointer.
  Specify durability separately and flush file/directory state where required.
- **Safe regression idea:** mocked filesystem failure points should prove the
  previous pointer remains readable after each failed commit step. Add native
  Windows replacement coverage; no filesystem fault campaign was run here.

### PLUG-REG-011 — P1: store mutations are not serialized across read-modify-write transactions

- **Evidence:** install reads the index at
  `bitty/crates/bitty-runtime/src/plugin_runtime/package.rs:293` and later
  replaces it at `:409-421`; toggles and uninstall do the same at `:462-475`
  and `:491-501`. All writers share the fixed temporary filename at
  `bitty/crates/bitty-runtime/src/plugin_runtime/resolution.rs:132-135`.
- **Cause:** no store lock or revision check spans those operations. Atomic
  replacement alone would not prevent writing a stale snapshot. The CLI
  directly calls the package operation (`bitty/crates/bitty-app/src/plugin.rs:1838`).
- **Impact:** overlapping ordinary CLI operations can lose unrelated updates;
  sharing a temporary pointer also lets writers interfere with each other's
  commit attempts. Independent in-memory resolver concurrency tests do not
  cover this filesystem transaction.
- **Recommended fix:** serialize store mutations with a cross-process lock,
  reload state under that lock, and use exclusive per-transaction temporary
  files. For long consent interactions, revalidate state on reacquisition.
- **Safe regression idea:** a deterministic mocked two-writer transaction test
  should preserve both unrelated record changes or reject one with a retryable
  conflict. Do not use the user's real plugin store.

### PLUG-REG-012 — P2: caret/tilde upper-bound arithmetic is unchecked

- **Evidence:** `bitty/crates/bitty-package/src/requirement.rs:279`, `:287`, and
  `:307` increment accepted `u32` components directly. Version parsing permits
  the full `u32` domain at
  `bitty/crates/bitty-package/src/version.rs:120-126`.
- **Cause:** overflow happens before the formatted upper bound is parsed, so
  the later `map_err` cannot implement the advertised overflow handling.
- **Impact:** boundary requirements do not reliably produce a normal validation
  error: arithmetic can panic with overflow checks or wrap without them,
  yielding incorrect resolution. No boundary reproduction was executed.
- **Recommended fix:** use checked increments and return a typed range error,
  or represent an unbounded upper edge if the accepted grammar requires it.
  Keep registry alignment diagnostics consistent with this semantic rejection.
- **Safe regression idea:** parser boundary tests should assert a typed result
  and no panic for maximum supported components in each shorthand position,
  in both normal and optimized test profiles.

### PLUG-REG-013 — P2: large numeric prerelease identifiers compare as equal

- **Evidence:** `bitty/crates/bitty-package/src/version.rs:234-242` checks
  numeric leading zeros but not a `u64` ceiling; `:256-260` compares numeric
  identifiers by parsing with `unwrap_or(u64::MAX)`.
- **Cause:** valid bounded version strings may contain numeric prerelease
  identifiers larger than `u64`, and distinct such identifiers collapse to the
  same fallback value.
- **Impact:** version precedence and comparator matching become incorrect;
  candidate selection can fall back to lexical ordering rather than numeric
  precedence (`bitty/crates/bitty-package/src/resolver.rs:115-122`).
- **Recommended fix:** compare validated digit-only identifiers by length then
  lexical digits, avoiding integer conversion. Separate precedence from total
  object identity: the current derived equality and precedence-only `Ord` also
  deserve an explicit consistency decision for build metadata.
- **Safe regression idea:** in-memory precedence tests should cover long
  numeric identifiers of equal and unequal lengths, plus comparator symmetry
  and transitivity. Existing prerelease ordering tests only cover small values.

### PLUG-REG-014 — P2: SPDX validation discards parentheses rather than parsing them

- **Evidence:** `bitty-plugins/scripts/registry-lib.ts:686` replaces all
  parentheses with spaces before its alternating identifier/operator scan.
- **Cause:** grouping balance and grouped-expression structure are erased;
  allowlisted identifiers are mistaken for a valid complete SPDX expression.
- **Impact:** invalid license expressions can pass the claimed fail-closed
  publishing gate, even though individual unknown licenses now fail correctly.
  This is catalog correctness, not a judgment that the conservative allowlist
  itself is defective.
- **Recommended fix:** parse grouping and operator precedence with a bounded
  grammar, or restrict the accepted grammar explicitly without silently
  removing syntax. Keep valid SPDX and project allowlist decisions separate.
- **Safe regression idea:** pure expression tests for balanced groups,
  unmatched/empty groups, and exception placement; retain the existing known
  identifier and exception tests at `bitty-plugins/tests/registry.test.ts:249-283`.

## Optimization opportunities, not confirmed defects

- **Catalog search/render:** `bitty-plugins/app/src/search.ts:57-79` repeatedly
  lowercases fields for each query token and sorts matches; `pages.ts:64-71`
  rebuilds the entire results DOM per input event. For N entries, Q query tokens,
  and F characters/array content per entry, search is approximately
  O(N × Q × F + M log M) time and O(M + Q) auxiliary space, plus transient
  lowercase strings; M is the match count. Precompute search text, filter
  before scoring, and consider pagination only when measured catalog size
  warrants it. Six entries do not establish a performance defect.
- **Metadata sync:** `sync-metadata.ts:281-283` performs a linear lookup per
  entry, making that portion O(N²) time. An ID map makes it O(N) time and O(N)
  space; bounded networking currently dominates real cost.
- **Resolver:** `bitty-package/src/resolver.rs:499-500` clones accumulated
  constraints during backtracking. Worst-case search is combinatorial, limited
  by configured candidates/packages/solver steps; retained constraint clones
  grow with search depth and graph size. Measure before replacing copies with a
  reversible constraint trail. Do not trade deterministic diagnostics or bounds
  for speed. No benchmark or performance improvement is claimed.
- **Large library module:** `scripts/registry-lib.ts` contains 1,423 lines and
  mixes SPDX, URL, schema, network ports, gitlinks, and index logic. Splitting
  pure contracts from I/O would make shared schema testing easier, but is not
  required merely to change layout.

## Uncertain risks and assurance gaps

These are not included in the 14-defect count.

- **Advisory integrity is not authentication.** All six current records carry
  raw manifest hashes and `unsigned` status. No registry signature verifier or
  authenticated key directory exists here. That is openly staged, but any
  future installer must not promote these advisory values to proof of origin.
  The public `bitty-package` signature routine remains a deterministic stub
  (`bitty/crates/bitty-package/src/trust.rs:275-350`), not cryptographic
  authentication. Keep it isolated from production trust decisions. This review
  does not provide signing formulas, payloads, or reproduction instructions.
- **Displayed trust status:** the storefront correctly says “Index claims
  verified,” but that status still selects a positive badge and hides the
  verification disclaimer in `app/src/components/plugin-detail.ts:110-155`.
  Prefer an unconditional “not locally verified” explanation until independent
  verification exists. No browser or deployed asset assessment was performed.
- **Pin/hash freshness:** validators shape-check `manifest_hash` but do not
  compare it to pinned bytes; metadata still comes from a floating HEAD rather
  than the reviewed submodule revision. These are expressly advisory workflows,
  so no forged trust or incorrect current digest is asserted. An offline
  pin-consistency gate would make stale pins visible without adding signatures.
- **Sibling fallback is coverage, not pin proof.** Local manifest/SDK fallback
  can inspect a sibling HEAD differing from the gitlink. The registry-only CI
  checkout does not initialize submodules, whereas plugin-integration does.
  Preserve this distinction in pass/fail summaries rather than saying every
  green gate validated the pinned manifest with the pinned SDK.
- **Network limits:** metadata body streaming is bounded, but repository
  existence has no total phase budget and GitHub comparison JSON is consumed
  wholesale (`validate-registry.ts:462`). Timeouts are not byte ceilings.
  These were not exercised against any endpoint. Add bounded response parsing,
  total budgets, and explicit partial-coverage outcomes defensively.
- **Offline validation can install dependencies:**
  `validate-registry.ts:294-306` calls `bun install` when SDK dependencies are
  missing, independently of `--skip-network`. Consequently that command was
  not used for this read-only review. Separate bootstrap from validation and
  make offline/no-install behavior explicit.
- **Installer limits and cleanup:** the initial local manifest is fully read
  before its size check (`package.rs:246-250`); failures from copying or
  hardening can leave staging material (`:325-326`), while pruning deliberately
  ignores staging directories (`:749`). Use bounded reads and a transaction
  cleanup guard. No resource-exhaustion workload was run.
- **Post-commit errors:** hardening or pruning can return an error after the new
  pointer has been committed (`package.rs:427-435`). Define whether such a
  result is committed-with-warning or rollback-worthy; callers currently get a
  generic failure. No injected I/O-failure testing was performed.
- **CI drift:** workflows pin Bun 1.4.0 while `package.json:11` names 1.4.2.
  Deploy builds directly and is not locally proven to depend on successful
  registry/integration checks. Remote branch protections, secrets provisioning,
  Pages state, and actual CI results were not queried. Existing token-variable
  fallback is documented in the workflow; no credential value was inspected.

## Deliberately not yet built or not owned here

Do not reopen these as equivalent to defects in a shipped registry installer:

- Store ID-based `bitty plugin add` is labeled a design proposal at
  `bitty-plugins/app/src/components/install-command.ts:88-92`. Actual CLI
  source installation rejects remote locators at
  `bitty/crates/bitty-app/src/plugin.rs:1815-1820`.
- Git/registry/archive download adapters, registry-cache refresh, authenticated
  metadata snapshots, release selection from the static directory, publisher
  key distribution, and a production remote install chain remain absent.
  `PackageSource` is a data enum, not a fetch implementation.
- Registry dependency fields are intentionally rejected with a specific
  explanation (`bitty-plugins/scripts/registry-lib.ts:177-193`). Manifest
  dependency declarations and the Rust in-memory dependency resolver exist;
  this does not mean the web directory can resolve them or that local install
  invokes that resolver.
- `bitty-package` activation simulates phases rather than quiescing/waking VMs
  (`bitty/crates/bitty-package/src/activation.rs:340-495`). Its per-plugin
  rollback explicitly switches the whole modeled generation pending a real
  merge (`:533-562`). Production live reload/rollback commands are not proven
  by those tests.
- Research 029's Git-first registry transport is a candidate, not permission to
  discard the accepted snapshot contract. Current docs explicitly record the
  reconciliation requirement at
  `bitty-plugins-docs/extensibility/package-management.md:172-209`.
- Research 039–041's activity, extension-platform, and IPC proposals do not
  authorize new catalog fields or imply implemented lifecycle adapters. Wheel
  is explicitly described as a pre-implementation scaffold in its registry
  entry; standalone Wheel implementation is outside this review.

## Recheck of September 15 and September 16 claims

The September 15 source is
`research/review/2026-09-15/10-plugins.md`. The following is a current-source
reassessment, not an endorsement of its examples or priority labels.

| Old claim                                                                   | Current disposition and evidence                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| --------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| R1: no hashes/signatures/origin binding                                     | Partly superseded. Optional integrity fields, generated status, and six official raw hashes now exist. Verification remains openly advisory. The new raw/H-B conflict is PLUG-REG-006; no production registry trust chain is claimed.                                                                                                                                                                                                                                 |
| R2: fetched ID not checked                                                  | Fixed for parsed plugin tables: `sync-metadata.ts:192-199` rejects missing/mismatched IDs; existing tests at `tests/registry.test.ts:1362-1381` pass. A wholly missing plugin table returns no metadata rather than an identity error (`sync-metadata.ts:190`), so the diagnostic is not exhaustive.                                                                                                                                                                  |
| R3: manifest body unbounded                                                 | Main remote path fixed: 256 KiB header/stream guard, timeout through body read, cancellation on excess at `sync-metadata.ts:74-155`; existing bounded tests pass. This does not prove every network response elsewhere is bounded.                                                                                                                                                                                                                                    |
| R7: browser IDs/URLs/copy unsafe                                            | Original unchecked-ID/URL claim is superseded by guards at `app/src/registry.ts:80-103`, `:168-202` and external-link defense in `components/dom.ts:52-64`. Remaining optional-field shape failure is PLUG-REG-002.                                                                                                                                                                                                                                                   |
| R8: ranges never structurally checked; mapping absent                       | Structural range checking and explicit compatibility-key mapping now exist. Closed-resolver differences are warning-only by a documented decision. The local installer normalizes partial comparators (`bitty/.../package.rs:539-608`), so those warnings do **not** prove current local installs universally fail. Registry `sdk` is documented as Plugin API range, not simply the SDK package release number. No standalone manifest parity audit is claimed here. |
| R9: no dependency model anywhere                                            | Overbroad. Registry dependencies are deliberately unsupported with specific diagnostics and tests; a bounded Rust resolver with transitive/cycle/yank/prerelease coverage exists. Production directory-to-installer dependency wiring remains absent.                                                                                                                                                                                                                 |
| R10: official checks always idle                                            | Superseded as an absolute claim: SDK override/submodule/sibling and manifest sibling fallback now exist, and unresolved entries are counted. This checkout still has eight uninitialized gitlinks; coverage and pin identity remain distinct.                                                                                                                                                                                                                         |
| R11: offline skip loses visibility                                          | Improved: checked/skipped counts and a warning survive a network exception (`registry-lib.ts:949-987`, `validate-registry.ts:96-107`). The phase still stops after an exception deliberately; offline degradation is specified, not proof of verified repository existence.                                                                                                                                                                                           |
| R26: duplicate repo, case-sensitive duplicate tags, unknown SPDX only warns | Original three points fixed: normalized duplicate-repository warning, case-insensitive slug duplicate diagnostic, unknown SPDX hard error. SPDX grouping remains wrong independently: PLUG-REG-014.                                                                                                                                                                                                                                                                   |
| R4–R6, R12–R25, R27–R31                                                     | Standalone plugins, SDK, template, or host behavior outside this report's audit boundary. Not marked fixed based on old reports; assigned reviewers must reassess them.                                                                                                                                                                                                                                                                                               |

September 16 files were searched for registry/distribution claims and relevant
sections were checked against current source:

- `02-code-vs-docs-progress-audit.md:195` says metadata identity and body bounds
  were fixed: **confirmed**, with the scope/diagnostic qualifications above.
- `04-architectural-drift-and-subjective-analysis.md:102` says integrity fields
  are advisory and generated status is unsigned/unverified: **still correct**.
  September 17 added six raw manifest hashes; it did not implement verification.
- The deterministic signature-stub claim remains true in the in-memory crate,
  but the broad end-to-end arbitrary-execution assertion in the September 16
  README is **not established by this review**. The local install path retains
  `local-path` provenance and does not use that stub to authenticate downloads.
- `02-code-vs-docs-progress-audit.md:172-182` reports stale split verdicts and
  missing file-manager/git-panel docs: **superseded** in the current sibling
  docs checkout. `product/bundled-plugin-split-decision.md:109-110` now says
  Split, and both per-plugin page sets exist. Current bundled catalog functions
  at `bitty/crates/bitty-plugin-host/src/bundled.rs:550-615` omit those two IDs.
- The old “29 commits behind” figure is historical. The code repository's
  `docs` pin is now measured as 34 commits behind the inspected sibling docs
  HEAD. No new claim is made about the aggregator's separate mount.
- September 16's missing research capture and PanelRuntime architecture claims
  are outside the distribution audit; summaries 029 and 039–041 were consulted
  for accepted-versus-candidate boundaries, not used as implementation proof.

## Explicit file coverage ledger

Depth: **R** = relevant bodies/data read and traced; **O** = symbol outline or
inventory-level review, not a full body audit; **C** = configuration/contract
reference; **X** = deliberately excluded. A grouped row names the exact files
sharing that depth; it is not a claim to have read every file beneath a directory.

### Registry and storefront repository

| Exact relative files                                                                                                                                                                                                                                                                                                                            | Depth       | Coverage                                                                                                                                                                                               |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `bitty-plugins/scripts/registry-lib.ts`                                                                                                                                                                                                                                                                                                         | R           | Entry discovery/normalization, keys/bounds, integrity shapes, SPDX, compatibility mapping, duplicate rules, SDK/manifest lookup, URL/submodule identity, network ports, pins, parse/build/render index |
| `bitty-plugins/scripts/semver.ts`                                                                                                                                                                                                                                                                                                               | R           | Structural and resolver-alignment grammars, version/identifier bounds and normalized shorthand                                                                                                         |
| `bitty-plugins/scripts/validate-registry.ts`                                                                                                                                                                                                                                                                                                    | R           | CLI orchestration, local manifest/SDK checks, dependency bootstrap, network existence, pin extraction/API checks                                                                                       |
| `bitty-plugins/scripts/generate-index.ts`                                                                                                                                                                                                                                                                                                       | R           | Validation, idempotence, check-only path, write boundary                                                                                                                                               |
| `bitty-plugins/scripts/sync-metadata.ts`                                                                                                                                                                                                                                                                                                        | R           | Source URL mapping, bounded body reading, identity, cache handling, partial failures and writes                                                                                                        |
| `bitty-plugins/tests/registry.test.ts`                                                                                                                                                                                                                                                                                                          | R           | All test groups inventoried; targeted bodies for schema, versions, source lookup, pins, index, integrity, metadata and browser guards; all 82 existing tests executed                                  |
| `bitty-plugins/registry/schema.json`                                                                                                                                                                                                                                                                                                            | R           | Complete field/type/length/cardinality contract                                                                                                                                                        |
| `bitty-plugins/registry/official/activity.toml`, `bitty-plugins/registry/official/file-manager.toml`, `bitty-plugins/registry/official/git-panel.toml`, `bitty-plugins/registry/official/palette.toml`, `bitty-plugins/registry/official/statusline.toml`, `bitty-plugins/registry/official/wheel.toml`                                         | R           | All six complete entries; IDs, source URLs, descriptions, compatibility and advisory digest fields                                                                                                     |
| `bitty-plugins/registry/community/.gitkeep`                                                                                                                                                                                                                                                                                                     | O           | Empty community area; no community record to audit                                                                                                                                                     |
| `bitty-plugins/generated/registry.json`                                                                                                                                                                                                                                                                                                         | R           | All six records and metadata; check-only freshness passed                                                                                                                                              |
| `bitty-plugins/app/src/registry.ts`                                                                                                                                                                                                                                                                                                             | R           | Browser types/validation, loading, lookup, facets, copy/link guards                                                                                                                                    |
| `bitty-plugins/app/src/search.ts`, `bitty-plugins/app/src/router.ts`                                                                                                                                                                                                                                                                            | R           | Scoring/filtering, route parsing, History API/link interception                                                                                                                                        |
| `bitty-plugins/app/src/main.ts`, `bitty-plugins/app/src/pages.ts`                                                                                                                                                                                                                                                                               | R           | Boot/error/render path, page construction, empty results, informational pages                                                                                                                          |
| `bitty-plugins/app/src/components/dom.ts`, `bitty-plugins/app/src/components/plugin-card.ts`, `bitty-plugins/app/src/components/plugin-detail.ts`                                                                                                                                                                                               | R           | Text/link construction, optional-field consumption, advisory status display                                                                                                                            |
| `bitty-plugins/app/src/components/plugin-search.ts`, `bitty-plugins/app/src/components/plugin-filters.ts`, `bitty-plugins/app/src/components/install-command.ts`                                                                                                                                                                                | R           | Input/change events, labels, copy fallback and proposal disclosure                                                                                                                                     |
| `bitty-plugins/app/vite.config.ts`, `bitty-plugins/app/index.html`                                                                                                                                                                                                                                                                              | R           | Static index emission/dev serving, shell, navigation and noscript behavior                                                                                                                             |
| `bitty-plugins/app/src/styles/base.css`, `bitty-plugins/app/src/styles/components.css`, `bitty-plugins/app/src/styles/main.css`, `bitty-plugins/app/src/styles/tokens.css`                                                                                                                                                                      | O           | Inventoried; no CSS/layout/contrast audit or browser rendering                                                                                                                                         |
| `bitty-plugins/app/public/_redirects`, `bitty-plugins/app/public/favicon.svg`, `bitty-plugins/app/public/robots.txt`                                                                                                                                                                                                                            | O           | Inventoried; deployment behavior not tested                                                                                                                                                            |
| `bitty-plugins/.github/workflows/registry-check.yml`, `bitty-plugins/.github/workflows/plugin-integration.yml`, `bitty-plugins/.github/workflows/deploy.yml`                                                                                                                                                                                    | R           | Triggers, checkout depth, gates, Bun pin, bounded jobs, publication separation                                                                                                                         |
| `bitty-plugins/.github/workflows/codeql.yml`, `bitty-plugins/.github/workflows/snapshot-source.yml`                                                                                                                                                                                                                                             | O           | Inventoried; not executed or audited as package distribution                                                                                                                                           |
| `bitty-plugins/scripts/workflow-import.sh`, `bitty-plugins/scripts/workflow-publish.sh`                                                                                                                                                                                                                                                         | R (limited) | Read dispatch/side-effect paths through line 190 to classify as CarryCtx state distribution, not package lifecycle; final tails not audited; never invoked                                             |
| `bitty-plugins/justfile`, `bitty-plugins/package.json`, `bitty-plugins/app/package.json`, `bitty-plugins/.gitmodules`                                                                                                                                                                                                                           | C/R         | Gate side effects, installed scripts/dependencies, all gitlink URL declarations                                                                                                                        |
| `bitty-plugins/tsconfig.json`, `bitty-plugins/app/tsconfig.json`, `bitty-plugins/bunfig.toml`, `bitty-plugins/bun.lock`                                                                                                                                                                                                                         | O/check     | Inventoried; configs consumed by passing typecheck/tests; no lockfile/dependency supply-chain audit                                                                                                    |
| `bitty-plugins/AGENTS.md`, `bitty-plugins/.carryctx/rules/delivery.md`, `bitty-plugins/.carryctx/rules/documentation.md`, `bitty-plugins/.carryctx/rules/security.md`, `bitty-plugins/README.md`                                                                                                                                                | C           | Applicable guidance and scoped status/hash/range/dependency claims                                                                                                                                     |
| `bitty-plugins/.carryctx/README.md`, `bitty-plugins/.carryctx/config.toml`, `bitty-plugins/.editorconfig`, `bitty-plugins/.gitattributes`, `bitty-plugins/.gitignore`, `bitty-plugins/.markdownlint-cli2.jsonc`, `bitty-plugins/.prettierignore`, `bitty-plugins/commitlint.config.ts`, `bitty-plugins/lefthook.yml`, `bitty-plugins/repo.toml` | O           | Tracked administrative/config inventory; no CarryCtx state access                                                                                                                                      |
| `bitty-plugins/CHANGELOG.md`, `bitty-plugins/CONTRIBUTING.md`, `bitty-plugins/SECURITY.md`, `bitty-plugins/TODO.md`, `bitty-plugins/LICENSE`                                                                                                                                                                                                    | O           | Inventoried; source and canonical scoped contracts take precedence; not a prose/license audit                                                                                                          |
| `bitty-plugins/.github/ISSUE_TEMPLATE/bug_report.md`, `bitty-plugins/.github/ISSUE_TEMPLATE/feature_request.md`, `bitty-plugins/.github/ISSUE_TEMPLATE/registry_entry.md`, `bitty-plugins/.github/PULL_REQUEST_TEMPLATE.md`, `bitty-plugins/.github/dependabot.yml`                                                                             | O           | Administrative inventory only                                                                                                                                                                          |
| Nine gitlinks listed above; ignored `bitty-plugins/node_modules`, `bitty-plugins/app/dist`, Git internals and worktrees                                                                                                                                                                                                                         | X           | Pins recorded; vendored/standalone implementations and generated local build artifacts not audited                                                                                                     |

### Cross-repository distribution trace

| Exact relative files                                                                                                                                                                                                                            | Depth        | Coverage                                                                                                                                                  |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `bitty/crates/bitty-runtime/src/plugin_runtime/package.rs`                                                                                                                                                                                      | R            | Local installer, compatibility normalization, version collision, consent snapshot, enable/remove, copy/hardening/retention; selected existing test bodies |
| `bitty/crates/bitty-runtime/src/plugin_runtime/resolution.rs`                                                                                                                                                                                   | R (targeted) | Index read/write, record resolution, provenance, digest checks, root containment (`99-321`); remaining symbols outlined                                   |
| `bitty/crates/bitty-app/src/plugin.rs`                                                                                                                                                                                                          | R (targeted) | Whole symbol inventory; source classification and install dispatch (`1784-1854`), imports and CLI/test interfaces; managed-state codec not fully audited  |
| `bitty/crates/bitty-package/src/source.rs`                                                                                                                                                                                                      | R (targeted) | Source enum and validation; digest/provenance helper inventory                                                                                            |
| `bitty/crates/bitty-package/src/resolver.rs`                                                                                                                                                                                                    | R            | Candidate construction/insertion/validation, root constraints and backtracking (`27-172`, `266-630`); tests outlined                                      |
| `bitty/crates/bitty-package/src/version.rs`, `bitty/crates/bitty-package/src/requirement.rs`                                                                                                                                                    | R (targeted) | Version parsing/precedence, shorthand expansion and comparator parsing; test inventory; not every requirement parse branch read                           |
| `bitty/crates/bitty-package/src/lockfile.rs`                                                                                                                                                                                                    | R (targeted) | Validation/insertion/digest (`119-206`); remaining types/tests outlined                                                                                   |
| `bitty/crates/bitty-package/src/lifecycle.rs`, `bitty/crates/bitty-package/src/activation.rs`                                                                                                                                                   | R (targeted) | State transitions and modeled activation/rollback; remaining generation/retention symbols outlined                                                        |
| `bitty/crates/bitty-package/src/trust.rs`                                                                                                                                                                                                       | R (targeted) | Stub verifier classification (`275-354`); trust/key/test symbols outlined; no reproduction                                                                |
| `bitty/crates/bitty-package/src/integrity.rs`, `bitty/crates/bitty-package/src/manifest.rs`                                                                                                                                                     | O            | Complete symbol outlines for model boundaries; no full crypto/canonicalization/manifest audit                                                             |
| `bitty/crates/bitty-package/src/lib.rs`, `bitty/crates/bitty-package/src/error.rs`, `bitty/crates/bitty-package/Cargo.toml`                                                                                                                     | O            | Tracked inventory; package README used for declared crate boundary                                                                                        |
| `bitty/crates/bitty-package/README.md`                                                                                                                                                                                                          | C            | Draft, network-free, no-I/O model distinction                                                                                                             |
| `bitty/crates/bitty-package/tests/hostile.rs`, `bitty/crates/bitty-package/tests/transaction.rs`, `bitty/crates/bitty-runtime/tests/plugin_package.rs`, `bitty/crates/bitty-app/tests/cli_plugin.rs`                                            | O            | Existing test functions inventoried; no Rust tests run; names are not execution evidence                                                                  |
| `bitty/crates/bitty-plugin-host/src/bundled.rs`                                                                                                                                                                                                 | O            | Catalog and test inventory to recheck removed bundled entries; individual plugin manifests not audited                                                    |
| `bitty/crates/bitty-runtime/src/plugin_runtime/store.rs`                                                                                                                                                                                        | O/X          | Outlined and identified as plugin key/value persistence, not distribution store; excluded from substantive findings                                       |
| `bitty/crates/bitty-runtime/src/plugin_runtime/mod.rs`, `bitty/crates/bitty-runtime/src/plugin_runtime/manifest_toml.rs`, `bitty/crates/bitty-runtime/src/plugin_runtime/services.rs`, `bitty/crates/bitty-runtime/src/plugin_runtime/spawn.rs` | X            | Inventoried to establish boundaries; VM lifecycle/manifest/capability implementation belongs to other review scopes                                       |
| `bitty/crates/bitty-runtime/tests/plugin_runtime.rs`, `bitty/crates/bitty-runtime/tests/plugin_store.rs`, `bitty/crates/bitty-runtime/tests/runtime_plugin.rs`, remaining runtime tests/fixtures                                                | X            | Inventoried, not executed or reviewed in depth                                                                                                            |

### Guidance, contracts, and prior evidence

Read applicable workspace `AGENTS.md`, `research/AGENTS.md`,
`bitty-plugins-docs/AGENTS.md`, `bitty/AGENTS.md`, the registry's three rules,
and `bitty/.carryctx/rules/{delivery,documentation,security,performance}.md`.
The explicit read-only/no-CarryCtx task overrides normal delivery automation.
The ctxctl skill was loaded. Its MCP adapter rejected workspace paths, so the
installed CLI was used from each repository. Shell outlines were unsupported;
those scripts were read directly and not executed.

Contracts consulted by targeted sections:

- `bitty-plugins-docs/extensibility/package-management.md:14-240`.
- `bitty-plugins-docs/specifications/package-lifecycle-rfc.md:140-249`.
- `bitty-plugins-docs/specifications/package-followup-rfc.md`: targeted search
  for source identity, canonical H-B, resolver, and registry status.
- `bitty-plugins-docs/product/official-plugin-onboarding.md:25-124`.
- `bitty-plugins-docs/product/bundled-plugin-split-decision.md:90-169`.
- Per-plugin docs filenames and all canonical plugin-doc Markdown paths were
  inventoried; individual standalone design/evidence bodies were not audited.
- `research/summary/029.md`, `039.md`, `040.md`, `041.md` were read; the summary
  corpus was searched for distribution/plugin context. Other summaries were
  discovery context only, not independently verified conclusions.
- `research/review/2026-09-15/10-plugins.md` was read in full.
- The five September 16 review Markdown files were searched for relevant
  claims; the claim recheck above explicitly scopes what was verified.
- `research/.markdownlint-cli2.jsonc` and the installed linter help were read
  to ensure lint targets only this report and performs no fixes.

## Checks and limits

| Check                                                                                                        | Result                                                                                                          |
| ------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------- |
| `bun test tests/registry.test.ts` in `bitty-plugins`                                                         | PASS: Bun 1.4.2; 82 pass, 0 fail, 332 expectations, one file                                                    |
| `just registry-check` in `bitty-plugins`                                                                     | PASS: six entries, generated index current; six unsigned warnings and six partial-comparator alignment warnings |
| `bun run type-check` in `bitty-plugins`                                                                      | PASS: installed TypeScript, no emit                                                                             |
| `bun run --cwd app type-check` in `bitty-plugins`                                                            | PASS: installed TypeScript, no emit                                                                             |
| `git diff --check` in `bitty-plugins`                                                                        | PASS; no source diff                                                                                            |
| Source HEAD/status rechecks                                                                                  | Stable HEADs and dirty baselines as listed above                                                                |
| `markdownlint-cli2 --no-globs review/2026-09-17/bitty-plugins/02-registry-and-distribution.md` in `research` | PASS: markdownlint-cli2 0.23.1; one file linted, zero issues                                                    |

Not run: registry metadata sync, ordinary generation, repository/network/pin
verification, SDK-dependent registry validation, `just check`, app build,
integration-smoke, dependency installation/audits, Cargo compilation/tests,
Rust lint/format, standalone plugin/SDK/template suites, browser/clipboard/
accessibility/Lighthouse checks, HTTP/Pages/deployment checks, fuzzing, fault
injection, stress tests, concurrency reproduction, GitHub/remote checks,
signature experiments, or full canonical-doc links/lint. Aggregate gates would
install/fetch, emit build output, execute excluded suites, or exceed the allowed
write scope. No platform-specific behavior is claimed as experimentally proved.

The coverage ledger deliberately separates read bodies from outlines and
inventory-only files. This is a thorough registry/distribution review with a
targeted cross-repository lifecycle trace, not an exhaustive audit of all Bitty
runtime source or a verification of the installed product's trust boundary.
