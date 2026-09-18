# Plugin host, Lua, package, and rich-content review

## Independent second-pass verification

- Unchanged `bitty` HEAD `06bc1f45995fd297a3f7324688bc81b0598ecf67`; only pre-existing untracked `.targets/`. First-pass coverage and prior-review tables below are historical evidence, not an exhaustive second audit.
- **Verified and reclassified:** TERM-HOST-001 is a disclosed mock/draft boundary, P2 pre-integration work, not a proven live signing bypass. Source contract sampled at `crates/bitty-package/src/trust.rs:273-300`; draft disclosure and call site at `crates/bitty-plugin-host/src/install.rs:1-38,264-307`. Cryptographic implementation and live registry deployment were not established or tested.
- **Verified source mechanisms:** TERM-HOST-002 (P1 explicit host-capture quota gap, practical exhaustion unmeasured) and TERM-HOST-004 (P2 post-effect timeout ambiguity). The former must not imply unlimited work per VM slice or feasible timer-ID exhaustion; those claims lack causal evidence. The latter proves a conditional outcome-contract defect, not that the current store usually exceeds the deadline.
- Exact samples: `crates/bitty-lua/src/host.rs:397-425,460-468,610-706,914-965`; `crates/bitty-runtime/src/plugin_runtime/mod.rs:642-714,999-1057`; `plugin_runtime/services.rs:197-252`. Registration vectors have no local admission quotas, and runtime capture validation is post-execution; the checked event validation does not reject duplicates. These observations do not certify Piccolo's heap accounting or end-to-end resource exhaustion.
- **Not independently reverified:** TERM-HOST-003/005/006/007/008/009. Inherited compatibility and rollback findings remain explicitly draft-model limitations; no live installer failure is newly established. Filesystem races, ACL effects, upstream VM behavior, and complete plugin authorization were not tested.
- No source edits, product execution, network, installs, commits, CarryCtx, payloads, or reproductions. Markdown validation and cross-report limitations are in [the product index](README.md).

## Scope and disposition

- Date: 2026-09-17. Read-only, static defensive review of `bitty-plugin-host`, `bitty-lua`, `bitty-package`, and `bitty-rich`, with narrowly sampled runtime call sites to qualify reachability.
- Sole authorized output: `research/review/2026-09-17/bitty-terminal/05-pluginhost-lua-package-rich.md`, relative to the workspace. No product source changes, commits, installations, repository/network fetches, CarryCtx operations, nested agents, exploit execution, or reproduction code/steps.
- First-pass inventory retained with second-pass qualification: nine IDs, one P1 resource-accounting gap and eight P2 items including draft integration-readiness gaps, not nine demonstrated product defects. TERM-HOST-001 is an intentional mock, not a verified live trust-boundary failure. Source confirmation and second-pass coverage are distinguished below; no P0 or production exploitability conclusion is made.
- The earlier reviews mix real defects, deliberately deferred functionality, and speculative attack paths. Their broad claims of arbitrary execution, universal fail-open URL handling, absent traversal tests, and absent rollback blast-radius tests should not be carried forward unchanged.

## Baseline and method

| Repository | HEAD at review                             | Dirty baseline                                      |
| ---------- | ------------------------------------------ | --------------------------------------------------- |
| `bitty`    | `06bc1f45995fd297a3f7324688bc81b0598ecf67` | `?? .targets/`; tracked and staged diffs empty      |
| `research` | `d70152a079c75204ec37e99a7bf3f54b8190e423` | `?? review/2026-09-17/`; existing reports preserved |

The umbrella is not a Git repository. References beginning `crates/` are relative to the `bitty` repository at the above HEAD. References beginning `review/` are relative to `research`. Documentation references explicitly identify their owning repository. Working-tree bytes, rather than historical task names or claims of merged fixes, are authoritative.

Read the workspace `AGENTS.md`, `bitty/AGENTS.md`, `research/AGENTS.md`, and the `ctxctl-core` skill. No nested `AGENTS.md` was found beneath the scoped crates or research review directory. The repository guidance contains stale pre-implementation assertions; it was not used as evidence that current Rust code is absent. CarryCtx-specific workflow was not performed, as expressly excluded by this review.

Method: directory inventory, CtxCtl outlines followed by selected source ranges, exact-line readbacks for key findings, cross-crate symbol searches, previous-review comparison, and local normative-document consultation. The CtxCtl MCP adapter rejected the checkout as outside its configured root; local `ctxctl` worked. No dependency/reference code was executed. A local search did not locate the Piccolo 0.3.3 base-library source at the checked Cargo-cache location; raw-table behavior remains unverified.

## Current architectural evidence

- The policy host and executable Lua integration are distinct layers. `PluginHost::activate` requires current-manifest declarations and matching grants for nonempty capability sets (`crates/bitty-plugin-host/src/host.rs:271`). `GrantStore::is_granted` intersects declaration, stored hash, and grant (`crates/bitty-plugin-host/src/grant.rs:147`). Capability heads delegate to the package crate's closed vocabulary (`crates/bitty-plugin-host/src/capability.rs:367`). This is meaningful enforcement, not an allow-all sandbox.
- `LuaVm` constructs `Lua::core()` and the retained library subset (`crates/bitty-lua/src/lib.rs:375`). Execution checks fuel, elapsed time, and Piccolo-accounted heap between slices (`crates/bitty-lua/src/lib.rs:657`). These are cooperative VM boundaries, not OS process isolation or a complete accounting of Rust allocations.
- Host authority is delegated to `HostServices` implementations (`crates/bitty-lua/src/host.rs:310`). Sampled runtime services explicitly gate terminal snapshots, notifications, and Git spawning (`crates/bitty-runtime/src/plugin_runtime/services.rs:217`, `:239`, `:277`). Bridge namespace immutability is not the authorization mechanism.
- The queued policy host enforces per-plugin and global admission checks (`crates/bitty-plugin-host/src/event.rs:1010`). The executable runtime also has direct synchronous Lua delivery (`crates/bitty-runtime/src/plugin_runtime/mod.rs:901`); policy-host queue tests alone do not establish end-to-end scheduling isolation for that path.
- `bitty-package` has no declared dependencies and models verification, activation, and rollback in memory. The live local-directory installer is elsewhere: `crates/bitty-runtime/src/plugin_runtime/package.rs:229`. It parses actual manifest bytes, scans a module tree, and checks real version requirements through its own helpers. The scoped seven-stage draft verifier is not demonstrated to be the production registry installer.
- Rich-content checks include finite image dimensions, bounded decoding buffers, regular-file/root policy, token-scoped clipboard reads, and validated hyperlink presentation. These controls must not be confused with an audited platform opener, filesystem-handle sandbox, or complete renderer integration.

## Confirmed findings

Second-pass status: TERM-HOST-001 is reclassified below; TERM-HOST-002/004 mechanisms were independently re-read; remaining headings keep their first-pass records.

### TERM-HOST-001 — Reclassified P2 integration-readiness gap — signed verification is an intentional draft mock

**Evidence:** `crates/bitty-package/src/trust.rs:284`, `crates/bitty-package/src/trust.rs:312`, `crates/bitty-package/src/trust.rs:334`; invoked by the `Signed` branch at `crates/bitty-plugin-host/src/install.rs:274`.

**Cause:** The verifier checks key presence and revocation but does not use `KeyRecord.public_key_hex` to authenticate the release. Its success criterion is a deterministic digest-based mock. The mock is not confined to test compilation, and the ordinary signed-install API can return success through it.

**Impact / boundary correction:** A passing mock result is not cryptographic authenticity. However, `crates/bitty-plugin-host/src/install.rs:1-14` explicitly describes a proposed draft, says nothing is shipped until acceptance/release, and `crates/bitty-package/src/trust.rs:273-284` explicitly labels the verifier a stub. Non-test visibility is not production deployment. No live registry download-to-execution caller was established, so withdraw the P1 production-boundary classification. Retain P2 integration-readiness work: prevent this model from being consumed as real authentication. This is intentional scaffolding with an unsafe future integration risk, not proof of current unauthorized installation.

**Fix:** Make signed mode return an explicit unsupported/unimplemented error outside test-only scaffolding until a reviewed asymmetric verifier and authenticated key policy exist. Separate mock types from production trust evidence. When implemented, bind algorithm/version, publisher identity, manifest, artifact, revocation, and rotation policy; never label checksum-only acceptance as authenticated signing.

**Safe test idea:** Use vetted cryptographic library known-answer fixtures when the real verifier exists; verify key identity and revocation decisions independently. In the interim, ordinary signed-install requests must receive the explicit unsupported result. Do not use mock-generated successes as cryptographic evidence.

### TERM-HOST-002 — P1 — Lua registration capture is outside explicit host-memory quotas

**Evidence:** `crates/bitty-lua/src/host.rs:397`, `crates/bitty-lua/src/host.rs:610`, `crates/bitty-lua/src/host.rs:662`, `crates/bitty-lua/src/host.rs:914`; runtime capture validation occurs after execution at `crates/bitty-runtime/src/plugin_runtime/mod.rs:692`, with validation at `:999`.

**Cause:** Commands, event subscriptions, and timers are appended to Rust-owned vectors without admission count/byte limits. Registration strings are copied without the general bridge marshalling budget. Timer handles use unchecked increment. The runtime validates declared command/event names only after initialization; repeated event handlers are collected rather than rejected by that validation, and timers have no equivalent validation there.

**Impact:** Piccolo heap accounting does not account for these Rust-owned strings/vector storage. VM execution time and instruction budgets limit individual execution work but do not provide a declared cumulative host-memory ceiling or prevent excessive capture before rejection. The risk is host allocation and callback growth, not a proven cross-plugin privilege escalation. No pressure workload was run.

**Fix:** Enforce per-generation entry and aggregate-byte budgets before copying/stashing/appending; validate field lengths and registration phase immediately; reject or intentionally deduplicate repeated event registrations; use checked timer-handle allocation. Include capture and callback ownership in disposal accounting.

**Safe test idea:** Parameterize capture limits to small values and test ordinary admission, exact-limit acceptance, next-entry rejection, duplicate policy, and generation cleanup. Assert rejection precedes allocation/registration side effects; no large stress input is required.

### TERM-HOST-003 — P2 — Compilation is not covered by the VM execution budget

**Evidence:** `crates/bitty-lua/src/lib.rs:615`, `crates/bitty-lua/src/lib.rs:685`; `crates/bitty-lua/src/config.rs:473`; live caller limit at `crates/bitty-runtime/src/plugin_runtime/mod.rs:1105`.

**Cause:** `drive_chunk` copies and compiles source before the execution timer and memory pre-check, without a source-byte ceiling at this shared boundary. Required modules also compile inside a synchronous callback (`crates/bitty-lua/src/host.rs:1019`).

**Impact:** The public VM/config APIs do not independently uphold a total load-and-run bound. The live plugin reader does have a 1 MiB metadata-based source ceiling, so “all current plugin source is unbounded” is inaccurate; the VM comment's 8 KiB plugin-event payload reference is not a compilation bound. Parse-time heap and latency overshoot were not measured.

**Fix:** Enforce explicit source limits before ownership copies and compilation, distinguish compile and execute budgets in reporting, and bound module compilation cumulatively. If a strict wall deadline is required, synchronous compilation cannot be made preemptible merely by moving `Instant::now()`; use an appropriate supervised execution boundary or a compiler with cooperative budgeting.

**Safe test idea:** With deliberately small configured source limits, use benign chunks to verify pre-compilation rejection and unchanged VM state. Instrument load and execute accounting separately rather than running a parser-exhaustion case.

### TERM-HOST-004 — P2 — Bridge timeout can report failure after a mutation succeeded

**Evidence:** `crates/bitty-lua/src/host.rs:460`; trait contract at `crates/bitty-lua/src/host.rs:310`; runtime mutation implementations at `crates/bitty-runtime/src/plugin_runtime/services.rs:202` and `:239`.

**Cause:** `bounded` invokes the service first, then converts an otherwise successful result into `E_TIMEOUT` when elapsed time exceeds the deadline. It neither cancels nor rolls back service work.

**Impact:** A write or notification can be applied while Lua receives an undifferentiated failure. Retrying can duplicate effects or leave plugin state inconsistent. This is an outcome-contract failure, not proof that capability enforcement is bypassed. The current timeout test exercises a delayed settings read, not mutation reconciliation (`crates/bitty-lua/tests/host_bridge.rs:225`).

**Fix:** Define whether deadlines govern admission or completion. Preserve completed mutation outcomes, or return an explicit indeterminate/maybe-applied result with reconciliation or idempotency support. Use cancellable transaction boundaries where strict fail-closed side-effect semantics are required.

**Safe test idea:** Use an injected clock and an in-memory service to check committed, rejected-before-commit, and indeterminate outcomes deterministically. Avoid sleeps and external effects.

### TERM-HOST-005 — P2 — Validated file paths are reopened without object binding

**Evidence:** Lua module acquisition at `crates/bitty-lua/src/host.rs:1207` and `:1243`; rich validation returns a path at `crates/bitty-rich/src/loader.rs:259`; background metadata/key/read sequence at `crates/bitty-rich/src/background.rs:936` and `:964`.

**Cause:** Root/type/metadata checks and reads operate on separate path lookups. Lua uses `read_to_string` after a metadata length check. Background loading derives its key from pre-open metadata and later opens the path again.

**Impact:** With concurrent modification of an approved tree, the opened object and bytes need not be those that were validated. Lua's returned bytes are not bounded again at read time. Background's encoded-byte limit is independently enforced by bounded reading and a final length check, so the earlier statement that its 4 MiB cap is bypassed is incorrect; the remaining issue is object/root/type assurance and cache identity. Actual unauthorized access or a race was not reproduced.

**Fix:** Acquire through a root-relative, handle-based policy, validate the opened object's type/metadata, and read with an explicit cap from that same handle. Account for intermediate path components, not merely the final symlink. Bind the cache identity to the opened object or actual content; treat mutable development roots distinctly from immutable installed content.

**Safe test idea:** Inject a file-provider abstraction returning benign objects and metadata; assert validation, bounded reading, and cache keying use the same object token. Test oversized reader rejection with a small configured limit, without racing real files or accessing unrelated paths.

### TERM-HOST-006 — P2 — Draft compatibility stage does not compare version requirements

**Evidence:** `crates/bitty-package/src/integrity.rs:330` and `:497`; activation preflight discards host version arguments at `crates/bitty-package/src/activation.rs:361`.

**Cause:** The verification stage checks presence and superficial version shape, never matches `manifest.compat` ranges. The in-memory activation preflight also does not perform compatibility evaluation.

**Impact:** This draft verification/activation API can report success for incompatible package/host combinations. It must not serve as compatibility evidence. The live local installer is a counterexample to the earlier blanket claim: `crates/bitty-runtime/src/plugin_runtime/package.rs:291` calls `check_compat`, and `:539` parses and matches requirements.

**Fix:** Share an authoritative version/range implementation and policy between local installation, draft verification, activation, and rollback. Carry validated compatibility information into generation preflight rather than accepting and ignoring host-version arguments.

**Safe test idea:** Table-driven pure-data tests for ordinary matching/nonmatching versions, malformed versions, absent required versions, prereleases, and rollback after a host-version change. No package execution is necessary.

### TERM-HOST-007 — P2 — Per-plugin rollback performs a full environment switch

**Evidence:** `crates/bitty-package/src/activation.rs:538`–`:563`; existing test at `crates/bitty-package/tests/transaction.rs:364`–`:447`.

**Cause:** `rollback_per_plugin` checks target membership, then activates the entire target generation. Its documentation admits that the actual per-plugin merge is deferred.

**Impact:** The public function's targeted-operation promise does not match its mutation scope. Unrelated packages and capability snapshots can revert with the target generation. This is a documented draft limitation but still an unsafe success contract for a caller expecting surgical rollback. The earlier claim that a blast-radius test is missing is false: the current test explicitly expects a full switch.

**Fix:** Return unsupported for per-plugin rollback until a resolver produces a fresh, dependency-consistent generation with explicit scope; alternatively expose this operation only as full-environment rollback with confirmation. Preserve compatibility and consent checks for every changed package.

**Safe test idea:** With a small in-memory two-package environment, assert a targeted request either leaves unrelated state unchanged or returns unsupported without mutation. Keep full-switch tests under the full-rollback API.

### TERM-HOST-008 — P2 — Aggregate timeout loses a completed interceptor veto

**Evidence:** `crates/bitty-plugin-host/src/event.rs:1438` and `:1452`; aggregate wrapper at `crates/bitty-runtime/src/runtime/plugin.rs:332`.

**Cause:** Decisions are reduced to veto-wins, but an aggregate `timed_out` boolean then overrides the reduced decision. Timeout identity is no longer associated with an individual handler.

**Impact:** A completed veto and another handler's timeout cannot be represented faithfully by this API: the timeout flag causes approval despite the completed veto. This conflicts with the documented multiple-handler veto-wins rule. It does **not** mean that every timeout should become a security denial. Ordinary timeout-as-abstention is the accepted availability contract, and mandatory core authorization must remain separate.

**Fix:** Represent each handler's outcome separately, map that handler's timeout/error to abstention, and aggregate the remaining completed decisions. Keep independent consent and capability gates outside plugin interception. The current URL wrapper is separately fail-closed on timeout (`crates/bitty-runtime/src/runtime/plugin.rs:365`).

**Safe test idea:** Pure enum-based aggregation tests covering completed decisions, abstentions, and per-handler unavailable outcomes; assert permutation independence and preservation of completed vetoes. No handler hangs, command launches, or URL opens are needed.

### TERM-HOST-009 — P2 — Composer owner-only permissions are not guaranteed at creation

**Evidence:** `crates/bitty-rich/src/composer.rs:1197`–`:1204`, `:1226`–`:1236`.

**Cause:** `create_new` prevents overwriting but does not specify Unix creation mode. Restriction to `0600` happens afterward through a pathname-based helper whose error is ignored. The non-Unix helper is a no-op.

**Impact:** The promise of owner-only temporary drafts depends on ambient permissions and a successful later chmod, rather than being enforced by the operation. A failed restriction is followed by writing content and can still report success. A concurrently accessible temporary directory makes the creation interval relevant. This corrects the Sept 15 assertion that Unix uses atomic `create_new + 0600`. Default Windows temp ACLs were not inspected, so universal cross-user readability is not claimed.

**Fix:** Specify restrictive permissions atomically at Unix creation, propagate permission errors, use descriptor-based operations, and require a private parent directory. Implement or explicitly gate platform-specific ACL guarantees before claiming equivalent confidentiality on Windows.

**Safe test idea:** In a private test directory, inspect the benign file's creation permissions and inject permission-setting failure to assert no content is written and no success is returned. Add platform ACL checks in native CI; do not inspect another user's files.

## Recheck of Sept 15 report

Compared against `review/2026-09-15/06-lua-pluginhost-package-rich.md`. “Unconfirmed” below is not “safe”; it means the prior security conclusion outruns the inspected evidence.

| Prior claim                                     | Current disposition and evidence                                                                                                                                                                                                                                                                            |
| ----------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Lua source compile quotas absent                | Confirmed at shared VM boundary, qualified by live 1 MiB caller limit; TERM-HOST-003.                                                                                                                                                                                                                       |
| Post-hoc host timeout                           | Confirmed outcome ambiguity, downgraded from privilege-boundary language; TERM-HOST-004.                                                                                                                                                                                                                    |
| Spawn skips bridge timeout                      | Intentional delegated timeout contract, not independently a confirmed defect. Runtime has a timeout/kill/wait loop and clears environment (`plugin_runtime/spawn.rs:672`, `:745`). Full descendant/reaper behavior remains unreviewed.                                                                      |
| Registration capture unbounded                  | Confirmed explicit host-quota gap; TERM-HOST-002.                                                                                                                                                                                                                                                           |
| Read-only proxy escape                          | Unconfirmed upstream raw-table behavior; `debug` absence is explicitly tested at `bitty-lua/src/lib.rs:1059`. A per-VM namespace mutation alone does not prove added host authority. No escape test was executed.                                                                                           |
| `reset` retains heap                            | True and explicitly documented (`bitty-lua/src/lib.rs:498`); no current generation-reuse defect established. Runtime activation creates a fresh VM (`plugin_runtime/mod.rs:642`). Treat naming/API clarity as optimization.                                                                                 |
| Lossy UTF-8 conversion                          | Confirmed conversion at `bitty-lua/src/host.rs:178`, including some direct string arguments. Auth-key escalation not traced; retain as representation-hardening work, not confirmed privilege escalation.                                                                                                   |
| Module TOCTOU                                   | Confirmed source-level object-binding gap, conditional on concurrent mutation; TERM-HOST-005.                                                                                                                                                                                                               |
| Shared `os.clock` security side channel         | Process-global baseline exists (`bitty-lua/src/stdlib.rs:40`); no cross-plugin data leak established. Per-VM origins would not remove timing observations. Time/date access is deliberately retained.                                                                                                       |
| Unlimited config `views` map                    | Explicit no-whole-table-cap contract comment at `bitty-lua/src/config.rs:680`; capture occurs after execution checks (`:506`, `:686`). Host-side extraction accounting is a gap to measure, not evidence of unlimited total memory.                                                                         |
| Filesystem patterns allow broad paths           | Validator accepts bounded opaque patterns (`bitty-plugin-host/src/manifest.rs:348`), but declaration is not authorization or filesystem access. Current bridge trait has no generic filesystem primitive (`bitty-lua/src/host.rs:320`). No core path-access bypass established.                             |
| Intercept timeout is itself a bypass            | Reject blanket conclusion: timeout-as-abstention is accepted policy. Preserve completed veto issue separately as TERM-HOST-008; URL wrapper already denies timeout.                                                                                                                                         |
| Git global-option and inherited-pager bypasses  | Not confirmed. The validator requires the subcommand first (`bitty-plugin-host/src/tools.rs:141`); global options cannot be assumed to retain global semantics afterward. Spawn uses `env_clear` and piped output. Repository config/helper behavior and per-verb grammar need a dedicated defensive audit. |
| Single-capability revoke permits reprompt loops | Removal without full denial marker remains (`bitty-plugin-host/src/grant.rs:206`), but no automatic plugin-driven prompt loop was traced. UI/consent-state uncertainty, not a confirmed exploit.                                                                                                            |
| Zero-capability activation/test helper backdoor | Empty authority is intentionally allowed (`bitty-plugin-host/src/host.rs:309`); helper is crate-private (`:366`). Hypothetical future visibility or parsing bugs are not present findings.                                                                                                                  |
| Signature stub                                  | Confirmed, with production reachability qualification and existing draft disclosures; TERM-HOST-001.                                                                                                                                                                                                        |
| Compatibility ignored                           | Confirmed in draft package API, not live local installer; TERM-HOST-006.                                                                                                                                                                                                                                    |
| Source URL validation enables arbitrary fetch   | Data validation is shallow (`bitty-package/src/source.rs:69`); no fetcher in the scoped crate. Treat transport/provenance policy as a pre-integration gap, not demonstrated fetching or command execution.                                                                                                  |
| Local digest omits filesystem metadata          | Helper accepts caller-supplied path/byte pairs (`bitty-package/src/source.rs:158`); it does not inspect filesystem objects. Do not infer symlink traversal from this pure helper. Tree-format and metadata policy remain integration gaps.                                                                  |
| Per-plugin rollback changes all packages        | Confirmed documented draft API mismatch; existing test covers full switch. TERM-HOST-007.                                                                                                                                                                                                                   |
| `raw_bytes_len` self-report                     | True (`bitty-package/src/manifest.rs:663`, `:671`), but caller is Rust/parser code, not an identified plugin-controlled constructor. Other manifest fields have bounds. Parser ownership/validated-input types are hardening work.                                                                          |
| Background read bypasses encoded cap            | Reject cap-bypass claim: bounded read and final check remain at `bitty-rich/src/background.rs:971`. Object/cache binding issue remains TERM-HOST-005.                                                                                                                                                       |
| Editor selection executes arbitrary binary      | Running the user's chosen editor without a shell is intended; no untrusted project-to-editor authority path established (`bitty-rich/src/composer.rs:1072`, `:1256`).                                                                                                                                       |
| Composer `.sh` and Windows permissions          | Suffix alone does not execute anything. Windows exposure depends on ACLs. Actual creation/permission error handling deserves TERM-HOST-009, including Unix.                                                                                                                                                 |
| Kitty 320 MB limit is a bypass                  | It is an explicit configurable aggregate ledger policy, not the single-shot placeholder limit (`bitty-rich/src/kitty.rs:65`–`:80`). Peak/capacity accounting remains optimization/verification work; no large transfer was executed.                                                                        |
| `file:` hyperlink goes directly to OS           | Presentation validates, it does not open (`bitty-rich/src/hyperlink.rs:27`). Runtime uses a separate file-approval path and gesture token (`runtime/plugin.rs:375`, `:398`). Blanket direct-open claim not sustained.                                                                                       |
| Clipboard capture is unauthorized forwarding    | Capturing bounded data is not a platform write (`bitty-rich/src/clipboard.rs:271`). Forwarding callers were not comprehensively reviewed; keep as integration gap.                                                                                                                                          |
| Image metadata admission defeats pixel limits   | `ImageStore` is a metadata model (`bitty-rich/src/image.rs:427`); concrete Kitty decoder validates dimensions and lengths before pixel allocations (`kitty_decode.rs:341`, `:395`). No actual buffer/accounting mismatch traced.                                                                            |

### Corrections to the prior test-gap list

- `crates/bitty-lua/tests/host_bridge.rs:131` and `:154` already test rooted module resolution and traversal rejection. `:81` tests ordinary namespace writes; this is not a raw-table guarantee.
- `crates/bitty-lua/tests/host_bridge.rs:225` tests typed read timeout, and `:516` tests deliberate slow-spawn delivery. Neither proves rollback/cancellation of completed mutations.
- `crates/bitty-package/tests/transaction.rs:364` explicitly tests per-plugin API full-switch behavior. Desired surgical behavior is missing, not all blast-radius coverage.
- Kitty decoder outlines include normalization, malformed-data, exact-length, dimension, and area tests. The earlier blanket absence of these classes is unsupported. Outlines are test inventory, not evidence those tests pass.
- The test named `verify_pipeline_each_stage_independently_fails` samples stages 2–4 only (`crates/bitty-package/tests/transaction.rs:577`–`:656`), not all seven stages.
- The global-budget measurement at `crates/bitty-plugin-host/tests/measurement.rs:590` assumes 1024-capacity subscriptions can collectively reach 8192 events with ten subscribers. Whether constructor clamping invalidates that assumption was not resolved in this sample. Do not cite it as freshly passing evidence.

## Recheck of Sept 16 security claims

Sources searched: `review/2026-09-16/02-code-vs-docs-progress-audit.md`, `04-architectural-drift-and-subjective-analysis.md`, `03-research-records-traceability-audit.md`, and the date's `README.md`.

| Claim                                                                           | Assessment against this HEAD                                                                                                                                                                                       |
| ------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Signature stub and superficial compatibility checks remain                      | Substantively true for the draft APIs; TERM-HOST-001/006. The absence of public-key authentication is confirmed without reproducing a signature.                                                                   |
| Mentioned fix branches/task names imply merged hardening                        | Not accepted as evidence. This checkout is at `06bc1f4`; no other branch, worktree, or remote was fetched or tested.                                                                                               |
| Weak filesystem pattern validation plus stub signing proves arbitrary execution | Not established. It combines separate data-model shortcomings without tracing a live authorized install/access path. Distinguish grant request, granted authority, filesystem operation, and executed code.        |
| Git option spellings prove repo escape                                          | Not established without respecting argv position and per-subcommand semantics. The claimed inherited environment is contradicted by the sampled spawn implementation.                                              |
| Force all timeout handling to return false                                      | Not an appropriate blanket fix: conflicts with accepted ordinary interception availability policy. Preserve core authorization and completed vetoes; distinguish the already fail-closed URL path.                 |
| Background metadata/open split                                                  | Confirmed conditionally, but encoded read cap remains effective; TERM-HOST-005.                                                                                                                                    |
| Sandboxed Piccolo means privilege escalation is blocked                         | Too broad as assurance. Restricted standard libraries and sampled service grants are meaningful; compilation, host allocations, mutable filesystem acquisition, and integration boundaries remain material limits. |
| Rich rendering and Sixel/composer implementation-status claims                  | Not fully re-audited here. The scoped metadata/headless models do not establish end-to-end GPU or UI delivery, but absence of a direct dependency alone is insufficient proof of no integration.                   |

## Documentation and research consulted

Only local documents were consulted; no web or repository fetch was performed under the no-fetch restriction.

- `bitty-docs/docs/security/overview.md:79` explicitly distinguishes VM isolation from an OS sandbox; `:99` requires equal capability treatment for official/community plugins.
- `bitty-docs/docs/security/threat-model.md`, `p0-acceptance-criteria.md`, `risk-register.md`, and `evidence-matrix.md` were searched for plugin, resource, filesystem, signature, and Lua controls. These are sampled policy/evidence references, not newly verified acceptance results. The overview still says controls are unimplemented while the risk register reports several mitigations; historical status prose cannot substitute for source/test evidence.
- `bitty-plugins-docs/specifications/plugin-platform-rfc.md:490`–`:592` defines the four interception points, bounded metadata, queue delivery, timeout-as-abstention, and multi-handler veto-wins. Its old global-queue status prose is not authoritative over current implementation.
- `bitty-plugins-docs/specifications/package-lifecycle-rfc.md:14`–`:43` explicitly leaves real signatures and key-directory contracts draft. This materially narrows Sept 16 claims that documentation universally represents real signing as implemented.
- `bitty-plugins-docs/specifications/isolation-resource-rfc.md:227` was searched for accepted budgets and historical measurements. No cited historical test result was treated as a test run at this HEAD.
- The Sept 15 report was read in full; relevant Sept 16 passages were searched and compared as historical hypotheses, not instructions or current proof.

## Coverage ledger

**Definitions:** Full means all bytes/lines of that small text file were read. Sampled means only the named ranges/symbols, outline, or search hits were inspected; everything else in that file is unread. Unread means no substantive body review. No full source-file coverage or exhaustive audit is claimed. Files are relative to `bitty` unless qualified otherwise.

### Full text coverage

- The four scoped `Cargo.toml` files.
- `crates/bitty-plugin-host/README.md`, `crates/bitty-lua/README.md`, `crates/bitty-package/README.md` through the initial crate-map reads.
- Workspace `AGENTS.md`, `bitty/AGENTS.md`, `research/AGENTS.md`, the loaded CtxCtl skill, and `research/.markdownlint-cli2.jsonc`.
- `research/review/2026-09-15/06-lua-pluginhost-package-rich.md`.
- No Rust source file was read end-to-end.

### Sampled policy host coverage

| File                                                  | Inspected coverage                                                                                |
| ----------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| `crates/bitty-plugin-host/src/capability.rs`          | Outline; 160–233, 367–407: parser and closed-set delegation                                       |
| `crates/bitty-plugin-host/src/grant.rs`               | Outline; 124–178, 193–282: insert/check/revoke/narrowing                                          |
| `crates/bitty-plugin-host/src/host.rs`                | Outline; 271–378, 429–510, 540–607: activation/reload/grants/subscriptions                        |
| `crates/bitty-plugin-host/src/event.rs`               | Folded outline; 950–1180, 1420–1468: subscription/admission/drain and interception                |
| `crates/bitty-plugin-host/src/install.rs`             | Outline; 77–125, 206–315: all install input fields and verifier body; test names/search hits only |
| `crates/bitty-plugin-host/src/manifest.rs`            | Folded outline; 348–392, 537–630: filesystem requests and capability expansion/validation         |
| `crates/bitty-plugin-host/src/tools.rs`               | Outline; 108–261: full argv/tool admission helpers; test inventory only                           |
| `crates/bitty-plugin-host/src/lib.rs`                 | Install exports/status search hits only                                                           |
| `crates/bitty-plugin-host/tests/measurement.rs`       | Outline; 590–650: selected global-bound tests                                                     |
| `crates/bitty-plugin-host/tests/dogfood_isolation.rs` | Outline only                                                                                      |
| `crates/bitty-plugin-host/tests/bundled_dogfood.rs`   | Outline only                                                                                      |

Unread bodies: `crates/bitty-plugin-host/src/bundled.rs`, `error.rs`, `registry.rs`. All unlisted ranges and inline-test bodies remain unread. Registry internals, complete safe-mode transition behavior, command ownership, and generation disposal are therefore not independently accepted by this review.

### Sampled Lua coverage

| File                                        | Inspected coverage                                                               |
| ------------------------------------------- | -------------------------------------------------------------------------------- |
| `crates/bitty-lua/src/lib.rs`               | Folded outline; 375–414, 498–508, 615–735, 760–820, 1059–1072                    |
| `crates/bitty-lua/src/host.rs`              | Folded outline; 154–207, 310–359, 397–489, 610–742, 865–974, 978–1104, 1180–1265 |
| `crates/bitty-lua/src/config.rs`            | Folded outline; 473–537, 604–713: execution/result capture and table limits      |
| `crates/bitty-lua/src/stdlib.rs`            | Outline; 26–105: retained installation and time functions                        |
| `crates/bitty-lua/tests/host_bridge.rs`     | Outline; 81–99, 131–194, 225–254, 516–560                                        |
| `crates/bitty-lua/tests/measurement_lua.rs` | Outline only                                                                     |

Unread: remaining VM final-outcome checks, most bridge callback bodies and host installation, most config schema extraction, utf8/string/table implementations and tests, upstream Piccolo internals. No claim of complete VM/fuel/GC correctness.

### Sampled package coverage

| File                                        | Inspected coverage                                                                 |
| ------------------------------------------- | ---------------------------------------------------------------------------------- |
| `crates/bitty-package/src/trust.rs`         | Outline; 154–369: signature/key records and verifier; 312–351 line-number readback |
| `crates/bitty-package/src/integrity.rs`     | Outline; 330–529: compatibility, store check, pipeline inputs and orchestration    |
| `crates/bitty-package/src/activation.rs`    | Outline; 290–302, 340–375, 440–563: staging, preflight, confirmation, rollback     |
| `crates/bitty-package/src/source.rs`        | Outline; 69–205: validation, local digest, drift and promotion helpers             |
| `crates/bitty-package/src/manifest.rs`      | Folded outline; 653–737: input fields and validation                               |
| `crates/bitty-package/src/lib.rs`           | Trust exports/status search hits only                                              |
| `crates/bitty-package/tests/hostile.rs`     | Folded outline and signature helper search hits only                               |
| `crates/bitty-package/tests/transaction.rs` | Outline; 364–447, 577–656                                                          |

Unread bodies: `error.rs`, `lifecycle.rs`, `lockfile.rs`, `requirement.rs`, `resolver.rs`, `version.rs`; also checksum implementation and manifest canonicalization. No package builder/archive extractor exists in the inventoried scoped crate files; external SDK/build/registry producers were not inspected. Reproducible packaging, archive confinement, transport framing, lock provenance, and resolver correctness remain open review scope.

### Sampled rich-content coverage

| File                                    | Inspected coverage                                                   |
| --------------------------------------- | -------------------------------------------------------------------- |
| `crates/bitty-rich/src/background.rs`   | Folded outline; 821–837, 930–1010: cache key, acquisition, admission |
| `crates/bitty-rich/src/loader.rs`       | Outline; 259–378: complete path validator body                       |
| `crates/bitty-rich/src/kitty.rs`        | Folded outline; 65–80, 289–435: intake/chunks/eviction               |
| `crates/bitty-rich/src/kitty_decode.rs` | Folded outline; 273–293, 341–432: bounds/raw/PNG decoding            |
| `crates/bitty-rich/src/image.rs`        | Folded outline; 427–504: metadata admission                          |
| `crates/bitty-rich/src/composer.rs`     | Folded outline; 1072–1085, 1149–1306; 1185–1232 exact-line readback  |
| `crates/bitty-rich/src/clipboard.rs`    | Outline; 233–328: grants, capture, redemption                        |
| `crates/bitty-rich/src/hyperlink.rs`    | Outline; 1–82: URI validation and lookup                             |

Unread bodies: `blocks.rs`, `geometry.rs`, `hints.rs`, `kitty_place.rs`, `lib.rs`, `presentation.rs`, `projection.rs`, `scene.rs`, `shell.rs`; `README.md`; `tests/background_peak_memory.rs`; all five `tests/fixtures/bg3/` image files. Background sniff/decode implementation, full peak-memory accounting, composition/placement, and GPU integration were not reviewed. Existing fixture names are inventory, not decoded/validated evidence.

### Boundary and documentation samples

- `crates/bitty-runtime/src/plugin_runtime/mod.rs`: outline; 526–714, 755–769, 901–942, 999–1057, 1105–1128.
- `crates/bitty-runtime/src/plugin_runtime/services.rs`: outline; 197–289.
- `crates/bitty-runtime/src/plugin_runtime/spawn.rs`: outline; 672–705, 720–780; selected environment/authorization search hits. Not a complete process-runner audit.
- `crates/bitty-runtime/src/plugin_runtime/package.rs`: outline; 229–290, 523–568; exact call-site search at 291. Copy/commit/retention bodies unread.
- `crates/bitty-runtime/src/runtime/plugin.rs`: outline; 317–415.
- `bitty/justfile`: 1–65. Workspace manifest/lock search results were truncated; no full dependency audit.
- Documentation coverage is limited to the sections/searches listed above. Documentation repositories' HEADs were not recorded, so policy excerpts are local working-tree observations rather than commit-pinned research snapshots.

## Optimizations and non-finding hardening

1. **Queue admission cost:** `EventPipeline::publish` repeatedly recomputes aggregate usage and scans for oldest events. Cache per-plugin/global counters with invariant tests and a suitable eviction index. With Q queues and T matching targets, repeated full-queue scans can make admission at least O(TQ), apart from payload accounting; exact cost depends on unread helper bodies. Target O(T log Q) selection/update and O(Q) accounting state only after measurement.
2. **Avoid unnecessary copies:** Source is copied before compilation, Lua strings can be copied before byte-budget rejection, and local-content hashing concatenates all supplied bytes. Precheck lengths and use incremental hashing where the approved digest format permits. For B content bytes and F files, the current local helper uses O(B + F) auxiliary storage plus path sorting; streaming can remove the O(B) concatenation allocation.
3. **Representation clarity:** Reject invalid UTF-8 where strings are identifiers, or explicitly retain byte strings. Rename/reset documentation and validated-input types should make metric reset, raw byte ownership, and trust evidence unmistakable.
4. **Quota accounting:** Count host registration storage and decoder temporary buffers separately from live Lua heap and logical image payload lengths. Kitty's 320 MB policy is not automatically a 320 MB peak-RSS guarantee; vector capacity and overlapping decode buffers need measurement.
5. **Maintain policy parity:** Consolidate range checking and capability grammar where feasible, preserve intentionally distinct metadata/decode limits, and test their interfaces rather than assuming matching constant values imply matching semantics.

## Uncertainty and planned gaps

- No Rust tests were executed, no baseline failures were reproduced, and no cargo build/check/clippy/fmt gate was run. The task is report-only and disallows heavy builds, fetching, and reproduction work. Existing `.targets/` was neither trusted as fresh evidence nor modified.
- No claims about race reliability, worst-case memory, sandbox escape, arbitrary execution, or cross-platform confidentiality were validated dynamically.
- Need a future authorized review of real package build/archive formats, lock/canonical digest binding, transport and registry trust, consent persistence, store atomicity, concurrent local install, and rollback capability symmetry.
- Need upstream-version-pinned review of Piccolo raw table/metatable behavior, standard-library native callback budgeting, and GC accounting. No inference from standard Lua alone substitutes for this implementation.
- Need end-to-end plugin dispatch tests showing that queue budgets, VM scheduling, revocation, cached service grants, timers, and failure attribution apply to the actual executable runtime, not only the headless policy host.
- Need complete platform opener/clipboard forwarding and Windows ACL audits. File-approval UI behavior, subprocess descendants/reapers, Git repository configuration, and path-handle acquisition are not accepted as fully reviewed.
- Need actual rich decode peak-memory tests and fixture review, resource acquisition API tests, scene/placement/render tracing, and a separate assessment of the Sept 16 broad rich-feature status claims.
- Suggested tests above are defensive unit/property test goals, not executable vulnerability reproductions. No repair is included because source modification was expressly excluded.

## Checks and delivery evidence

- Initial and pre-report source checks: `git diff --exit-code` and `git diff --cached --exit-code` passed; source HEAD remained `06bc1f45995fd297a3f7324688bc81b0598ecf67`, with only pre-existing `?? .targets/` in status.
- Report lint uses the installed `markdownlint-cli2` v0.23.1 / markdownlint v0.41.1 and the research configuration, scoped with `--no-globs` to this report. This avoids the configured whole-repository glob and any on-demand installation.
- Product typecheck/lint/test gates are not applicable to the sole Markdown change and were not run; source correctness is not certified by report lint.
- Post-write verification: report lint selected exactly one file and returned exit 0 with zero issues. Both repositories' tracked and staged diffs remained empty; both HEADs and coarse untracked status matched the baseline. Only this authorized report was written by this review; no unrelated report or index was changed.
