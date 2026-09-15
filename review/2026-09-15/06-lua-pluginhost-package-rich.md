# Bitty Extension-Layer Crates Security Review: Lua / Plugin Host / Packaging & Distribution / Rich Text

- Scope (relative to workspace root):
  - `bitty/crates/bitty-lua/` (Lua bindings: VM budgets, sandbox, host bridge, config evaluation)
  - `bitty/crates/bitty-plugin-host/` (plugin host: registry, capabilities, authorization, event pipeline, install verification, git allowlist)
  - `bitty/crates/bitty-package/` (packaging/distribution: manifest, integrity chain, trust, sources, lockfiles, resolution, activation)
  - `bitty/crates/bitty-rich/` (rich text/multimedia: loader path policy, background decoding, kitty, image, hyperlink, clipboard, composer)
- Method: read-only review. Read each `Cargo.toml` and `lib.rs` first; for Lua/host focused on sandbox boundaries, API exposure, crash isolation, and resource quotas/timeouts; for package looked at manifest validation and signatures/integrity; for rich looked at oversized-input/malformed-input handling. Used `outline` → `symbol`/`read` slices and `deps`/`grep` edge tracing, without running code or modifying sources.
- Conclusion at a glance: Lua's instruction/wall-clock/memory budgets and `require` sandbox are largely solid, and the host bridge has reentrancy guards with marshalling caps; plugin-host's closed capability set, no wildcards, and hash-bound authorization are well done; package's 7-stage chain is complete; rich generally performs boundary checks before allocation in decoding. But there are reachable security-boundary gaps in compile-time quotas, bridge timeout semantics, unbounded registry growth, V-C signature authenticity, compatibility evaluation, source URL validation, file TOCTOU, and intercept fail-open handling, detailed below.

## Defect List

Severity: P0 (directly breaks the trust chain/sandbox) / P1 (high: privilege escalation, DoS, or integrity bypass reachable from untrusted input) / P2 (medium-low: needs specific conditions or has limited impact). `SEC` marks security class.

### bitty-lua

- P1 (SEC) `bitty/crates/bitty-lua/src/lib.rs` :: `drive_chunk` / `drive_stashed` —— public `execute` inputs have no length cap, and compile time is excluded from the wall-clock budget
  - Phenomenon: `drive_chunk(code: &str)` directly calls `Closure::load`, and the `Instant::now()` in `drive_stashed` explicitly starts after chunk load per its comment. `execute` / `execute_bounded` / `call_function` are all `pub`, and this layer enforces no `code.len()` cap.
  - Trigger: any caller that directly invokes `LuaVm::execute` passes an oversized chunk (e.g. a multi-MB single-line Lua chunk); the parse/allocation time of `Closure::load` is not constrained by `wall_budget_ms`, blocking the host thread for a long time.
  - Evidence: `drive_chunk` in `lib.rs` has no length check; the `drive_stashed` comment says “Execution wall-clock starts after chunk load … Compile input sizes are bounded at call sites (config: 64 KiB / 2048 lines; plugin events: 8 KiB), so excluding load cannot hide unbounded work”. That bound lives only in the comment and is not enforced in this crate. Any future caller that forgets to cap will be exposed.
  - Related: `config::ConfigEval::eval_config` also goes through `drive_chunk`, and its 64 KiB constraint is not visible in this crate.

- P1 (SEC) `bitty/crates/bitty-lua/src/host.rs` :: `BridgeState::bounded` —— post-hoc timeout, side effects already applied
  - Phenomenon: `bounded` first executes `f()` and only then checks `elapsed > deadline_ms`; on timeout it returns `E_TIMEOUT`, but host side effects (`store_set`, `notify_show`, etc.) are already committed, with no rollback/compensation documented.
  - Trigger: when a host implementation is occasionally slow (scheduling jitter, lock contention) and the Lua side relies on return values for consistency, a “written but judged timed-out” divergence appears.
  - Evidence: the `bounded` implementation at `host.rs:460-468`; the `HostServices` documentation says “bridge deadline-checks each call and fails closed with E_TIMEOUT”.

- P1 (SEC) `bitty/crates/bitty-lua/src/host.rs` :: `BridgeState::bounded_spawn` —— `process.spawn` fully skips bridge-layer timeouts
  - Phenomenon: the comment admits it “skips the post-hoc cheap-call deadline”, keeping only the reentrancy guard; the timeout for long calls is “enforced by the runtime (default 5s, max 30s, kill and reap)”. The default `HostServices::process_spawn` in this crate directly returns `E_SPAWN_UNAVAILABLE`; the real timeout lives in `bitty-runtime` (not reviewed here). This layer has no observable timeout/cancellation handle.
  - Trigger: when a downstream `HostServices` implementation forgets to kill/reap or sets the timeout too large, the VM thread is occupied indefinitely; the RC-1/RC-2 budgets of `call_function` also do not advance while host execution runs inside the bridge.
  - Evidence: `bounded_spawn` at `host.rs:483-489`; the `HostServices::process_spawn` documentation.

- P1 (SEC) `bitty/crates/bitty-lua/src/host.rs` :: `RegistrationCapture` + `build_bitty_root` —— unbounded registry, no length caps on `id/title/description/kind`
  - Phenomenon: `commands/events/timers` are unbounded `Vec`s; `command.register`'s `id/title` only pass `required_string` non-empty checks, `description` defaults to the empty string; `events.subscribe`'s `kind` accepts any string; `timers.create`'s `delay_ms` accepts any `u64`. A single `init.lua` can register a huge number of entries or huge strings within the 10M instruction budget, exhausting host memory.
  - Trigger: a malicious/runaway plugin loops over `bitty.commands.register` / `bitty.timers.create` in `init.lua`.
  - Evidence: `RegistrationCapture` at `host.rs:397-406`; parameter extraction in the three `Callback::from_fn` closures inside `build_bitty_root`; `next_timer_handle` is unbounded (`+= 1`).
  - Note: the `store/notify` paths have `MarshallingLimits`, but the registration path has none.

- P1 (SEC) `bitty/crates/bitty-lua/src/host.rs` :: `readonly_table` —— read-only relies only on `__newindex`, without sealing `rawset`/`rawget`/metatable paths
  - Phenomenon: `readonly_table` implements read-only via an empty proxy + `__index` forwarding + erroring `__newindex` + `__metatable=false`. This review found no code removing `rawset`/`rawget`/`debug.getmetatable`, and whether `Lua::core()` includes `rawget/rawset` by default is not explicitly tightened in this crate.
  - Trigger: if `rawset(bitty, ...)` or `debug.getmetatable` is available, a plugin can tamper with the `bitty` namespace or bypass the read-only proxy.
  - Evidence: the `readonly_table` implementation at `host.rs:1071-1103`; `lib.rs:384-389` only declares “no raw metatable access to host objects”, with no corresponding removal/test evidence. Rated as an unproven boundary gap that must be falsified by testing (see test gaps).

- P2 (SEC) `bitty/crates/bitty-lua/src/lib.rs` :: `LuaVm::reset` —— clears state but not the heap, easily misused causing resident memory
  - Phenomenon: the `reset` documentation itself admits it “does not reset the Lua heap (a new Lua would be required … caller should drop and recreate for generation N+1)”, but the method name and signature strongly suggest full reclamation.
  - Trigger: when the host calls `reset` instead of rebuilding the VM on generation switch, the suspended generation's heap (including content a malicious plugin allocated up to the 32 MiB cap) stays resident, and the new generation keeps accumulating.
  - Evidence: `lib.rs:502-508`.

- P2 (SEC) `bitty/crates/bitty-lua/src/host.rs` :: `from_lua_inner` —— `String::from_utf8_lossy` masks malicious bytes and creates key collisions
  - Phenomenon: Lua-string to `LuaValue::String` conversion uses lossy conversion, replacing non-UTF-8 bytes with `U+FFFD`. Two distinct byte strings can map to the same `String`, confusing `store`/`settings` key deduplication and auth-key comparison; and error messages are deliberately truncated (`BridgeError` never echoes untrusted content), so callers cannot distinguish them.
  - Trigger: a plugin passes keys/values containing invalid UTF-8.
  - Evidence: `host.rs:154-207`; the `BridgeError` documentation “never echoes untrusted content”. Recommend rejecting non-UTF-8 at least for keys (see suggestions).

- P2 (SEC) `bitty/crates/bitty-lua/src/host.rs` :: `resolve_module_source` —— `is_file` → `canonicalize` → `metadata` → `read_to_string` is non-atomic TOCTOU
  - Phenomenon: it first selects candidates via `root.join(...).is_file()`, then `canonicalize`s, checks the 1 MiB cap via `metadata`, and finally `read_to_string`s. The file can be swapped between the four steps (same-directory writers, symlink flips), so the `metadata` length conclusion can disagree with the actually read content; it does not use `O_NOFOLLOW` + fd pinning.
  - Trigger: when the plugin root can be concurrently modified by writers other than the plugin itself (multi-plugin shared roots, external sync tools, malicious build scripts).
  - Evidence: `host.rs:1207-1260`. `canonical.starts_with(root)` is itself correct but does not close the check-to-use window.
  - Note: the `..`/leading-dot/charset checks in `validate_module_name` are in place and not in this finding.

- P2 (SEC) `bitty/crates/bitty-lua/src/stdlib.rs` :: `install_os` —— `PROCESS_START` global cross-VM timing side channel
  - Phenomenon: `os.clock` uses a process-level `OnceLock<Instant>`, so all plugin VMs share the same baseline. One plugin can probe scheduling load of other plugins/the host via `os.clock()`, breaking the isolation intuition of “same source and budget implies deterministic execution”.
  - Trigger: two mutually untrusted co-resident plugins sampling `os.clock()` simultaneously.
  - Evidence: `stdlib.rs:40,60-72`. `os.time/os.date` reading system time is an accepted baseline; this finding concerns only the global baseline of `clock`.

- [P2] `bitty/crates/bitty-lua/src/config.rs` :: `capture_views_table` —— `views` selector table has no entry cap
  - Phenomenon: the comment explicitly says “the `views` table adds no whole-table cap”, capturing each item with `capture_table(depth=2)`. Entry count is only indirectly constrained by RC-1/RC-2, so a huge `views = { a={...}, b={...}, ... }` can construct an oversized `ConfigData` for downstream within budget.
  - Trigger: a malicious user config or a tampered `init.lua` returning tens of thousands of selectors.
  - Evidence: `config.rs:686-713`. Top-level/nested tables have `MAX_CONFIG_TOP_KEYS/NESTED_KEYS/KEYMAPS`, with only this spot excepted.

### bitty-plugin-host

- P1 (SEC) `bitty/crates/bitty-plugin-host/src/manifest.rs` :: `FilesystemRequest::validate` —— path-pattern validation too weak, permissions over-broad
  - Phenomenon: rejects only empty strings, `>512` bytes, control characters/whitespace, and a total of 8 KiB. `/etc/**`, `~/.ssh/**`, `C:\Windows\**`, `../**`, absolute paths, and symlink patterns all pass. Real directory confinement is fully deferred to the execution layer (runtime, not reviewed here).
  - Trigger: a user mistakenly grants a `fs.read/fs.write` capability containing a broad pattern, letting a malicious plugin legitimately read sensitive paths.
  - Evidence: `manifest.rs:348-392`. By contrast, `bitty-rich`'s `loader.rs` has a five-step check for forbidden prefixes/regular files/symlink escapes, with no counterpart here.

- P1 (SEC) `bitty/crates/bitty-plugin-host/src/event.rs` :: `should_proceed` —— intercept timeout fail-open, veto can be bypassed by timeout
  - Phenomenon: `should_proceed(_, timed_out=true)` always returns `true`, documented as “Timeouts and errors are treated as abstention”. Among the four intercept points (only 4 in accepted v1), if any is a security intercept (e.g. outbound-request or dangerous-command confirmation), a plugin can get the action released by timing out/crashing.
  - Trigger: an intercept handler times out or returns an error while the decision should have been veto.
  - Evidence: `event.rs:1447-1465`; the `lib.rs` table “veto-wins, fail-open”. This is an accepted contract, but the review must record it: fail-open and security intercepts are mutually exclusive, and at least high-risk actions should be fail-closed (see suggestions).

- P1 (SEC) `bitty/crates/bitty-plugin-host/src/tools.rs` :: `is_allowed_git_args` —— allowlist has residual bypass surface
  - Phenomenon (a): `-c` is only matched exactly as `arg == "-c"`. Glued `-cfoo=bar` forms, the long `--config` option (if the installed git supports it), and `-C <path>` (changes working directory, equivalent to repo escape) are all unrejected. Although the `branch` arm forbids `-c/-C/-f/-u/-t` etc., the `status/diff/log/show/rev-parse/ls-files` arms do not forbid `-C`.
  - Phenomenon (b): `--paginate/--no-pager` and pager-related flags are not rejected. If spawn inherits `GIT_PAGER`/`PAGER` from the environment or `core.pager` is not pinned, `git log/show/diff` can launch an external pager (`less` etc.), exceeding the “read-only observation” assumption.
  - Trigger: a plugin controls the `args` array (`process.spawn` shape validation only constrains length/characters, not semantics, see previous section), and the host passes `args` verbatim to `Command`.
  - Evidence: `tools.rs:108-241`; `GIT_ALLOWED_SUBCOMMANDS` includes verbs such as `show/rev-parse/ls-files` that can read arbitrary objects by design (consistent with read-only positioning, but asymmetric with the `--output` ban: `--output` is banned while the `-o` short form was not reviewed — if any verb supports `-o` file writing, it slips through).

- P2 (SEC) `bitty/crates/bitty-plugin-host/src/grant.rs` :: `revoke` —— single-capability revocation leaves no denial marker, allowing repeated re-prompt loops
  - Phenomenon: only full revocation (`capability=None`) writes into the `denials` set; `revoke(id, Some(cap))` only removes from `granted` without a trace. The host comment says denials exist so “hostile packages cannot re-prompt in a loop”, but the single-capability path is unprotected.
  - Trigger: a malicious plugin repeatedly requests a just-revoked single high-risk capability, inducing the user to mis-click.
  - Evidence: `grant.rs:193-231`.

- [P2] `bitty/crates/bitty-plugin-host/src/host.rs` :: `activate` —— zero-capability plugins activate with zero authorization, and `activate_unchecked_for_test` coexists with the formal path
  - Phenomenon: when `required.is_empty()`, it directly calls `registry.activate`; the logic is correct but means a manifest-parsing bug that drops capabilities degrades into a zero-authorization plugin; `activate_unchecked_for_test` (`pub(crate)`) bypasses all grant checks, and would be a backdoor if its visibility were ever widened.
  - Evidence: `host.rs:271-358,366-368`. Recommend adding an observable audit event for zero-capability activation (see suggestions).

### bitty-package

- P0 (SEC) `bitty/crates/bitty-package/src/trust.rs` :: `verify_signature` / `stub_sign` —— V-C is a deterministic stub, not a real signature
  - Phenomenon: `signature_hex == SHA256(key_id || manifest || artifact)` with the first 64 hex chars duplicated to 128 hex counts as passing. Anyone who can compute SHA-256 can forge a “valid signature” for any `key_id`; `KeyStore` is only an in-memory map, with no public-key cryptography, no algorithm identifier, and no revocation distribution.
  - Trigger: an attacker holding the `key_id` string (public information) can forge signatures offline, leaving no substantive difference between V-C and V-A.
  - Evidence: `trust.rs:284-351,359-369`. The V-C test in `install.rs` (`trust_vc_signature_fail_closed_and_valid_passes`) only proves the stub is self-consistent, not forgery-resistant. V-C must be explicitly documented/versioned as unimplemented, otherwise downstream mistakes “signed” for “authenticated”.

- P1 (SEC) `bitty/crates/bitty-package/src/integrity.rs` :: `check_compatibility` —— only checks emptiness and containing `.`, without semver range evaluation
  - Phenomenon: when the manifest declares `compat.bitty/plugin_api`, the check passes as long as the host version is non-empty and contains `.`. `requires >=2.0` can still install on `0.6.0`.
  - Trigger: after breaking host changes, old plugins are still activated, or new plugins trigger unknown behavior on old hosts.
  - Evidence: the comment at `integrity.rs:330-365` (“Minimal check … full semver range evaluation is deferred to resolver”). `requirement.rs` already has `VersionReq::parse/matches`, but it is unused here.

- P1 (SEC) `bitty/crates/bitty-package/src/source.rs` :: `PackageSource::validate` —— registry/git URLs have no scheme/host validation
  - Phenomenon: registry checks only emptiness/length/whitespace; git adds only a `rev` length check. `file:///etc/x`, cleartext `http://`, `javascript:`, and `ssh://-oProxyCommand=...`-shaped URLs all pass, and `rev` allows any 256 bytes (without restricting to a hex/sha charset).
  - Trigger: when a lockfile/registry index is poisoned or a mirror is hijacked, the fetch stage pulls from an arbitrary source.
  - Evidence: `source.rs:69-138`. `has_registry_provenance` is true only for `Registry`, while the provenance semantics of `Git` sources hang in a comment.

- P2 (SEC) `bitty/crates/bitty-package/src/source.rs` :: `digest_local_content` / `check_local_path_drift` —— local-path digests do not bind metadata
  - Phenomenon: the digest is only SHA-256 over `sort(path) + path + \0 + bytes + \n`, excluding file mode, executable bits, symlink targets, and owners. An attacker swapping a symlink to a sensitive file while keeping bytes identical (or exploiting case/normalization differences) can bypass drift detection; `ensure_no_promotion_without_chain` only guards the “declared registry” flag, not content confusion.
  - Evidence: `source.rs:158-205`.

- [P2] `bitty/crates/bitty-package/src/activation.rs` :: `rollback_per_plugin` —— named per-plugin rollback, actually a full switch
  - Phenomenon: the function only checks that the target generation contains the plugin, then calls full `activate(target)`. Callers assuming a single-plugin blast radius will underestimate rollback risk.
  - Evidence: the comment at `activation.rs:538-563` (“actual per-plugin merge is deferred to the resolver”).

- [P2] `bitty/crates/bitty-package/src/manifest.rs` :: `PackageManifest::validate` —— `raw_bytes_len` is caller-supplied, self-reported length
  - Phenomenon: the `MANIFEST_MAX_BYTES` check relies on the `raw_bytes_len` field, but that field is a `usize` filled in when constructing the struct, not a value actually measured by the parser. A malicious constructor can fill in a small value to bypass it.
  - Evidence: `manifest.rs:670-737`. `verify_manifest` directly trusts that field.

### bitty-rich

- P1 (SEC) `bitty/crates/bitty-rich/src/background.rs` :: `BackgroundStore::load` + `read_file` —— check-to-use TOCTOU, can bypass `BG_MAX_ENCODED_BYTES`
  - Phenomenon: `load` first compares `metadata.len()` against 4 MiB, then `read_file` re-`File::open`s and reads with `take(4MiB+1)`. The file can be swapped to a larger one between the two opens; although `read_file` has a second length check so allocation is not unbounded, `BackgroundKey{len, modified}` is generated from the first `metadata`, so cache identity can disagree with the actually decoded bytes (cache poisoning: small file occupies the key, large-file content stays resident).
  - Trigger: a trusted directory concurrently written by another process (a downloader displaying while downloading, a cloud-sync directory).
  - Evidence: `background.rs:936-984`; the `validate_resource_path` (`canonicalize`) in `loader.rs:259-378` → `metadata` in `load` → `open` in `read_file` touch the same path three times, non-atomically. The header-sniff→bounds→decode chain in `decode_background` itself is good; the problem is in the file-acquisition layer.
  - Same class: the `symlink_metadata` → `metadata` → `canonicalize` in `loader.rs:295-360` is likewise three non-atomic steps, but that function leans toward rejection, so its impact is smaller than `load`.

- P2 (SEC) `bitty/crates/bitty-rich/src/composer.rs` :: `resolve_editor` + `run_editor` —— editor program has no allowlist and no path validation
  - Phenomenon: `resolve_editor` returns `$VISUAL`/`$EDITOR` trimmed verbatim; `run_editor` runs `Command::new(program).arg(path)` with no shell splitting (good), but `program` can be any absolute path, relative path, or unexpanded `~` path. If workspace configuration can inject `EDITOR`, it gains one arbitrary-binary execution (inheriting stdio).
  - Trigger: a malicious repository's `.envrc`/workspace configuration persuades the user to `export EDITOR=/tmp/pwn`, or the config-parsing layer passes an untrusted field through to the composer.
  - Evidence: `composer.rs:1072-1085,1215-1260`. The `timeout.min(EDITOR_TIMEOUT_MAX)` and kill logic are correct and not in this finding.

- P2 (SEC) `bitty/crates/bitty-rich/src/composer.rs` :: `write_composer_temp` —— `.sh` suffix + no owner-only on Windows
  - Phenomenon: temp files are fixed as `bitty-composer-<pid>-<nanos>-<seq>.sh`; the comment says “spaces in either are data”, but the `.sh` suffix induces users/tools to double-click-execute; `restrict_owner_only` is an empty implementation on non-Unix, relying only on temp-dir + immediate unlink, so drafts are readable by other users on multi-user Windows machines.
  - Evidence: `composer.rs:1149-1212`. The Unix path `create_new + 0600` is correct.

- P2 (SEC) `bitty/crates/bitty-rich/src/kitty.rs` :: `KittyGraphicsStub` —— chunked path reaches 320 MiB in a single transfer and evicts normal entries
  - Phenomenon: `KITTY_LEDGER_MAX_BYTES = 320MB` is storage + in-flight total; `begin_chunk` only rejects `first.len() > cap`, and subsequent `append_chunk`s accumulate slice by slice up to the cap. A single remote transfer can legitimately occupy 320 MiB and kick out all old entries via `evict_to_fit` (FIFO). The single-shot path truncates at 4 KiB while the chunked path allows 320 MiB — an 80,000x gap — so a remote peer can bypass the “small image” assumption by choosing chunked encoding.
  - Trigger: a malicious peer sends an oversized image fragmented with `m=1`.
  - Evidence: the Bounds documentation at the top of `kitty.rs` and `begin_chunk/append_chunk/evict_to_fit`. The 16 MiB pixel cap in `kitty_decode.rs` and the 64 MiB decode cap in `image.rs` are decode-layer caps; the intake layer already holds 320 MiB of raw bytes before them.

- P2 (SEC) `bitty/crates/bitty-rich/src/hyperlink.rs` :: `is_safe_hyperlink_uri` —— `file:` passes straight to the OS handler, `https` validation delegated to an unreviewed module
  - Phenomenon: `file:` goes through `validate_file_url`, everything else through `validate_url` (both in `bitty-platform`, not reviewed here). The bundled corpus lists `file:///tmp/report.txt` as accepted, meaning remote PTY output `OSC 8;;file:///etc/passwd` renders as a clickable link that hands off to the OS opener on click. `hyperlink_info` only checks the length of `id_param`, not its charset.
  - Trigger: the user clicks a `file:` link constructed by a remote application.
  - Evidence: `hyperlink.rs:1-60` and the `adversarial_uri_corpus_is_rejected` / `supported_uri_corpus_is_accepted` tests. Rejecting `javascript:` etc. is good, but the `file:` surface is unconverged.

- [P2] `bitty/crates/bitty-rich/src/clipboard.rs` :: `handle_action` —— writes captured by default, forwarding decision fully delegated to callers
  - Phenomenon: this crate “always returns `WriteCaptured`, and the caller decides whether to forward to the platform”. If callers forward by default, a remote peer can write the clipboard without any user gesture (M1 should be gated opt-in). This layer has no policy enforcement point.
  - Evidence: the top documentation of `clipboard.rs` and `ClipboardPolicy` (`Gated` is only a type; `handle_action` does not read it). The read path's one-time token + constant-time comparison + evict cap are solidly implemented and not in this finding.

- [P2] `bitty/crates/bitty-rich/src/image.rs` :: `ImageStore::insert` —— pure-metadata admission, no pixel/compression consistency binding
  - Phenomenon: `insert(source, width, height, compressed_len, frame_count, generation)` performs bounds accounting and accumulates `total_bytes` using only caller-declared width/height/length, without holding pixels or verifying that `compressed_len` matches the dimensions. Bookkeeping bytes can decouple from real decoded bytes; `frame_count.clamp(1, 64)` silently clamps instead of rejecting, letting callers underestimate quotas via clamping.
  - Evidence: `image.rs:427-504`. The “validate-before-allocate” in `kitty_decode.rs:decode_raw/decode_png` is correct, but the `ImageStore` layer and decode layer are two separate cap sets (4096 vs 8192 edge length, 64MiB vs 64MiB), needing an alignment audit.

## Suggestions

| #   | Suggested approach                                                                                                                                                                                                                                                                                                             | Expected benefit                                                                    | Effort                                     |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------- | ------------------------------------------ |
| 1   | Lua: add a unified `MAX_CHUNK_BYTES` (e.g. 64 KiB) + `MAX_CHUNK_LINES` (e.g. 2048) at `drive_chunk` / `execute` / `eval_config` entries, failing directly with `Failed` without entering `Closure::load`; move the wall-clock start earlier or set a separate small budget for compilation and count it in `instructions_used` | Close compile-time DoS so callers no longer rely on comment discipline              | S                                          |
| 2   | Lua: cap `RegistrationCapture` (e.g. 64/64/64 for commands/events/timers) + length caps on registered fields (e.g. 128/256/128 bytes for id/title/kind), returning `E_LIMIT` when exceeded; fail closed before `next_timer_handle` overflow                                                                                    | Close init.lua unbounded registration exhausting host memory                        | S                                          |
| 3   | Lua: change side-effecting bridge calls like `store_set` to “check timing/pre-check first, mark indeterminate state on timeout” or provide idempotency keys; at minimum carry `maybe_applied=true` in `E_TIMEOUT` errors so Lua/hosts can reconcile                                                                            | Eliminate “written but judged timed-out” split-brain                                | M                                          |
| 4   | Lua: add configurable caps for `process.spawn` at the bridge layer (concurrency, cumulative per-VM count, single argv total bytes partly present), rejecting directly when exceeded; lower the runtime 5s/30s contract into this crate's trait docs + `#[cfg(test)]` timeout harness                                           | Even poor `HostServices` implementations cannot hang the VM thread                  | M                                          |
| 5   | Lua: falsify with tests whether `rawset(bitty)/debug.getmetatable(bitty)` is reachable; if reachable, explicitly remove `rawset/rawget/debug` in `install_retained_stdlib` or add `__metatable` + proxy-freeze double protection for the `bitty` table                                                                         | Close read-only proxy bypasses, aligning sandbox claims with implementation         | S (verify) / M (fix)                       |
| 6   | Lua: in `from_lua_inner`, reject non-UTF-8 for keys (at least keys) instead of lossy conversion; or uniformly return `E_VALUE_ENCODING`                                                                                                                                                                                        | Eliminate key collisions and unauditable silent substitution                        | S                                          |
| 7   | Lua: change `resolve_module_source` to fd-pinned reads (`open(O_NOFOLLOW)` → `fstat` → `read`), or at least re-verify length and `starts_with` after `read_to_string`; rename `reset` to `reset_metrics` or split `reset`/`recreate` with a `#[must_use]` warning in docs                                                      | Close the TOCTOU window and prevent misuse                                          | M                                          |
| 8   | Lua: change `os.clock` to a per-VM monotonic origin (initialized with `LuaVm::new`) instead of process-global; or remove `clock` from the retained stdlib                                                                                                                                                                      | Close the cross-plugin timing side channel                                          | S                                          |
| 9   | Lua: add an entry cap to the `views` table (e.g. 64) or reuse the top-level cap; keep the existing 64 KiB output cap on `string.format`                                                                                                                                                                                        | Prevent oversized config objects from reaching downstream                           | S                                          |
| 10  | Host: tighten `FilesystemRequest::validate` with allowlist syntax (anchored to a relative root, forbid absolute paths/parent dirs/`~` expansion, forbid `/proc /sys /dev`, require patterns to start from allowed roots), sharing the same prefix table with `bitty-rich/loader.rs`                                            | Move privilege escalation from execution time to install time; mis-grants fail fast | M                                          |
| 11  | Host: tiered intercept policy — keep fail-open for ordinary observed events, switch high-risk intercepts (network egress, command execution, data exfiltration) to fail-closed + count timeouts as violations and fuse the handler; add a `critical: bool` parameter to `should_proceed`                                       | Timeouts no longer auto-release dangerous actions                                   | M                                          |
| 12  | Host: complete `is_allowed_git_args`: globally reject `-C/--config/--config-env` (exact + prefix), reject pager surfaces like `--paginate/--no-replace-all`, audit the meaning of short `-o` across allowed verbs; pin `GIT_PAGER=cat` + clear `PAGER` + `--no-pager` at the spawn layer, with unit coverage                   | Close git-composed escapes and pager invocation                                     | S                                          |
| 13  | Host: write denials (or introduce a “re-grant cooldown”) on single-capability `revoke` as well, unifying re-prompt throttling                                                                                                                                                                                                  | Close looping authorization inducement                                              | S                                          |
| 14  | Packaging: for V-C either switch to real signatures (ed25519 + algorithm identifier + key rotation/revocation distribution), or downgrade the type/docs to `StubSigned` and refuse it as a production trust basis in `verify_install`                                                                                          | Callers no longer mistake the stub for authentication                               | L (real signatures) / S (downgrade + docs) |
| 15  | Packaging: reuse `requirement::VersionReq::parse/matches` in `check_compatibility` for real range evaluation; keep fail-closed when the host version is missing but the manifest requires one, and additionally fail closed on malformed host versions                                                                         | Incompatible combinations cannot install/activate                                   | S                                          |
| 16  | Packaging: add a scheme allowlist to `PackageSource::validate` (registry only `https:`, git only `https:/ssh:` with `-oProxyCommand`-shaped rev/url characters forbidden), restrict `rev` to `[0-9a-f]`/tag-safe charsets; have the parser measure `raw_bytes_len` instead of self-reporting                                   | Close poisoned-lockfile fetching from arbitrary sources                             | S                                          |
| 17  | Packaging: bind file mode/symlink targets (at least `is_symlink + target`) into local digests, normalize input paths in `digest_local_content` (reject `..`, absolute paths)                                                                                                                                                   | Close drift bypasses                                                                | M                                          |
| 18  | Rich text: change `BackgroundStore::load` to a single `open` followed by `fstat` + `read` on the same fd, generating `BackgroundKey` from the hash/length of actually read bytes rather than the first `metadata`; add a post-`File::open` `is_allowed(canonical)` re-check between `validate_resource_path` and `load`        | Close file TOCTOU and cache poisoning                                               | M                                          |
| 19  | Rich text: validate `resolve_editor` (forbid shell metacharacters and empties, require absolute paths to exist as regular files, or provide an allowlist config option); drop the `.sh` suffix on temp files (use `.txt`/no suffix), add ACLs on Windows or document the multi-user risk                                       | Close EDITOR injection and draft leaks                                              | S                                          |
| 20  | Rich text: cap single kitty chunked transfers (e.g. 16 MiB, aligned with the decode layer) + audit total ledger quotas against decode quotas together; do not activate `file:` links by default (require second confirmation or only allow `https/http/mailto` clicks), add charset validation to `id_param`                   | Converge remote-reachable maximum memory and click surface                          | M                                          |

## Test Gaps

- Missing Lua sandbox-escape negative tests: no cases for `rawset(bitty, ...)`, `debug.getmetatable(bitty)`, `string.dump`, coroutine cross-VM interference, `require("...")` traversal (`..`, absolute paths, case, overlong names, 1 MiB boundary ±1); `tests/host_bridge.rs` and `tests/measurement_lua.rs` cover only budgets and the bridge happy path.
- Missing budget-adversarial tests: oversized single-chunk compile time, cumulative memory across consecutive `execute` calls, whether `reset` releases the heap, whether `store` stays consistent after a slow-host `E_TIMEOUT`, host-side rate limiting for infinite `process.spawn` loops — all without cases.
- Missing registry-pressure tests: tens of thousands of `commands.register` / `timers.create` calls, oversized `title` (MB-scale), `next_timer_handle` boundaries — no cap tests.
- Missing host grant/event tests: whether re-requesting after `revoke(Some)` re-prompts, activation auditing for zero-capability plugins, observability of earliest-event loss under `DropOldest`, expectations of `should_proceed(timed_out)` for high-risk actions (currently asserted as fail-open, missing fail-closed tiered cases), violation counting and fuse thresholds for intercept handler errors/timeouts.
- Missing git allowlist negative tests: no cases for `-C`, `--config`, glued `-cK=V`, pagers, short `-o` behavior across verbs, `GIT_PAGER` pollution; existing `tools.rs` tests lean toward the `branch` verb.
- Missing packaging-trust tests: V-C forgery test (a stub signature computed with a different key should be rejected — currently it must pass; missing a regression pinning “must fail”), `compat` range nourishment (`>=2.0` on `0.6.0` should fail), malicious `registry`/`git` URLs (`file:`, `http:`, revs with spaces/controls), `raw_bytes_len` self-reported-small bypass, `rollback_per_plugin` blast-radius assertions.
- Missing rich-text file-layer tests: TOCTOU race tests for concurrent replacement in `load` (changing the file between metadata and read), symlink flips, `~/..` escapes, cache poisoning from `BackgroundKey` vs actually read bytes mismatch, `read_file` ±1 around the 4 MiB boundary.
- Missing decode-layer tests: malformed PNGs (IHDR lying about large dimensions, tRNS/16-bit/interlaced Adam7 bombs), `width*height*4` overflow boundaries, JPEG progressive-coefficient memory (when `progressive_jpeg_charge` underestimates), animated WebP frames (should reject), `decode_raw` off-by-one lengths, alignment tests for the inconsistent dual caps in `kitty_decode` vs `image`.
- Missing interaction-surface tests: post-click behavior of `file:` links (only the presentation layer was reviewed here; OS handlers are outside), default forwarding policy of clipboard callers, `EDITOR` pointing at malicious binaries/paths with spaces, Windows temp-file permissions, DoS demonstrations of the kitty 320 MiB ledger evicting normal entries.

---

- Date: 2026-09-15
- Coverage: `bitty-lua`, `bitty-plugin-host`, `bitty-package`, `bitty-rich`
- Note: this report is read-only review without source changes; all paths are repository-relative, with no absolute paths used.
