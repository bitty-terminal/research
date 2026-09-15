# Platform-layer crates read-only review: PTY / platform abstraction / config system

- Scope:
  - `bitty/crates/bitty-pty/` (PTY abstraction: spawn, lifecycle, resize, read/write pumps)
  - `bitty/crates/bitty-platform/` (platform abstraction: event loop, window, DPI, surface, clipboard, keyboard encoding, URL)
  - `bitty/crates/bitty-config/` (config system: plan, validation, merge, migration, reload, trust, file loading)
- Method: Read-only, no source changes. Read each crate's `Cargo.toml`, `src/lib.rs` entry and module docs, browsed all module symbols with `ctxctl outline`, then verified key symbols with `ctxctl read` slices:
  - PTY: `Pty` lifecycle and `Drop`, `kill`/`shutdown`/`wait`/`wait_timeout`, `take_reader`/`take_writer`, `resize`/`size`, `foreground_job`, Unix/Windows backend symmetry, reader-pump backpressure and EIO mapping, writer semantics, `PtyError` classification.
  - Platform: `cfg` conditional-compilation completeness, `App::run` no-display degradation, `Registry`, `WindowConfig`/`sanitize_opacity`, DPI conversion, `SurfaceTarget` lifetime, `Clipboard` Wayland/X11/headless, `encode_key_event_with_modifiers`, `validate_url`/`validate_file_url`.
  - Config: schema validation and defaults, hot-reload `classify_field`/`diff`/`reconcile_live`/`fallback_builtin`, merge priority and policy non-overridability, migration stubs, project-trust allowlist, Lua file boundary and error hints.
- Focus: resource leaks (fd/handle/thread/child process), races, uncovered platform differences, missing config validation, hard-coded values, swallowed errors.

## Defect list

Note: paths are all repo-relative; symbol names are public or `pub(crate)` symbols; "evidence" summarizes source behavior, not restated conclusions.

### P1-1 `Drop` blocks unboundedly and swallows errors; the `wait_timeout` fix does not cover the destructor path

- Location: `bitty/crates/bitty-pty/src/pty.rs`, `impl Drop for Pty`
- Symptom: The destructor performs `kill` and then calls unbounded `wait`, discarding both results with `let _`. If the backend `wait` hangs, the destructing thread hangs; the caller gets no timeout and no observable error.
- Trigger: Child already wedged, slow ConPTY reclamation, or accidentally dropping `Pty` on the event-loop/render thread.
- Evidence: A comment in the same file admits unbounded `wait` once stuck an entire Windows CI test binary for ~23 minutes, which is why the polling `wait_timeout` was added; yet `Drop` still takes the unbounded `wait` path instead of reusing the bounded path.

### P1-2 `recv_timeout`/`try_recv` fold EOF and I/O errors into the same `None`

- Location: `bitty/crates/bitty-pty/src/reader.rs`, `PtyReader::recv_timeout`, `PtyReader::try_recv`
- Symptom: `Disconnected` is mapped to `Ok(None)`, indistinguishable from clean EOF; `try_recv` maps both `Empty` and `Disconnected` to `None`. A caller seeing `None` mistakes it for "data fully read" and must call `join` to learn whether the pump errored, but the docs do not mandate that order.
- Trigger: Pump exits early on a kernel read error, or the receiver is dropped early causing `BrokenPipe`, while the consumer continues with EOF logic.
- Evidence: A unit test for a pump ending in `BrokenPipe` exists, but the `recv_timeout` docs describe `Disconnected` as "I/O errors see `join`" while returning the same `Ok(None)` as EOF.

### P1-3 `PtyWriter` docs claim drop writes EOT, but the code has no `Drop` impl

- Location: `bitty/crates/bitty-pty/src/writer.rs`, `struct PtyWriter`
- Symptom: The docs say dropping writes a newline plus the terminal EOF character before closing the descriptor, and graceful shutdown relies on it; the type actually has no `Drop` and only forwards `write`/`write_all`/`flush`, so whether EOF is sent depends entirely on the upstream handle's drop behavior.
- Trigger: The documented graceful path "drop the writer, then `wait`"; `cat`-style programs may never receive the expected EOF and block forever, forcing the caller to escalate to `kill`.
- Evidence: The file contains only `new` plus `io::Write` forwarding, with no `Drop`; the shutdown-semantics chapter in `pty.rs` still references the EOT assumption.

### P1-4 EIO-to-EOF mapping covers Linux only; other Unix systems may misreport errors

- Location: `bitty/crates/bitty-pty/src/reader.rs`, `fn pump`, `fn linux_eio`
- Symptom: The branch treating master-side EIO after child exit as clean EOF is guarded by `cfg!(target_os = "linux")`; if macOS or other Unix systems likewise report "child exited and slave closed" via EIO, it is treated as an ordinary `Err` and thrown via `join`.
- Trigger: Draining output after child exit on macOS or other non-Linux Unix.
- Evidence: On non-Linux, `linux_eio` returns a never-matching sentinel, with a comment stating it "never matches off-site"; the EIO unit test also only compiles on Linux.

### P1-5 Pump thread-creation failure panics via bare `expect`

- Location: `bitty/crates/bitty-pty/src/reader.rs`, `PtyReader::spawn`
- Symptom: `std::thread::Builder::spawn` failure is handled with `expect`, so resource exhaustion or thread-count limits make the library panic instead of returning a recoverable error.
- Trigger: Spawning a PTY reader under fd/memory/thread-quota pressure.
- Evidence: The comment claims "default construction cannot fail", but that assumption breaks under sandboxes, container quotas, and fd exhaustion; all other spawn paths in the same crate return `Result`.

### P1-6 Windows `foreground_pgid` is always `None`, so `when_busy` close confirmation is always "not busy" on that platform

- Location: `bitty/crates/bitty-pty/src/platform/windows.rs`, `fn process_group_leader`; `bitty/crates/bitty-pty/src/pty.rs`, `fn foreground_job`
- Symptom: ConPTY exposes no foreground-process-group surface, so `foreground_job` on Windows is forever `None`; callers are told to treat `None` as "not busy", so `close_confirm = when_busy` degrades to "never confirm" on Windows.
- Trigger: Closing a view/window on Windows while a long foreground task runs.
- Evidence: The Windows backend comment explicitly requires callers to treat `None` as "cannot tell, not busy"; the `pty.rs` docs repeat the same fail-open convention. This is a design tradeoff, but the silent single-platform downgrade of data-loss protection belongs explicitly in the defect list rather than only in a doc note.

### P1-7 `/proc` name reading has a pid-reuse TOCTOU

- Location: `bitty/crates/bitty-pty/src/pty.rs`, `fn process_name`
- Symptom: It first reads the foreground pgid via `tcgetpgrp`, then reads the corresponding `comm` file; between the two steps the pid may exit and be reused, so the name read may belong to an unrelated new process.
- Trigger: Closing confirmation while short-lived foreground tasks start/stop at high frequency; a wrong name only affects the confirmation text, not the busy/idle decision.
- Evidence: The function fail-softs to `None` but never re-checks "is the foreground pgid still the same pid after reading"; non-Linux returns `None` directly, a behavior fork.

### P1-8 Clipboard silently truncates to 8192 bytes; callers cannot detect the data loss

- Location: `bitty/crates/bitty-platform/src/clipboard.rs`, `const CLIPBOARD_MAX_BYTES`, `fn truncate_to_bytes`, `fn set_text`, `fn set_primary`, `fn get_text`, `fn get_primary`
- Symptom: Both read and write directions truncate by bytes around the syscall and back off to a character boundary, returning success `Ok` with no "was truncated" signal. After a large copy/paste the user believes the full text reached the clipboard, but only the prefix did.
- Trigger: Copy/paste over 8 KiB; CJK/emoji trigger it more easily under byte truncation.
- Evidence: Unit tests only assert "bounded set succeeds" and "truncation respects character boundaries", with no "truncation needs an explicit signal" assertion; `set_text_lossy` additionally swallows system errors.

### P1-9 `Clipboard::new` and `clear` fail-soft drop diagnostics; `lossy` reads may return a stale buffer

- Location: `bitty/crates/bitty-platform/src/clipboard.rs`, `fn new`, `fn clear`, `fn get_text_lossy`, `fn get_primary_lossy`
- Symptom: `new` maps `arboard` creation failure to headless without keeping the cause; `clear` discards `try_clear` errors to preserve the historic signature; `lossy` reads fall back to the in-memory buffer on system failure, so the caller receives a stale value while believing it is fresh.
- Trigger: No display, permissions, Wayland compositor without primary support, etc.; headless CI stays green while real-device behavior differs.
- Evidence: The `Err(_)` arm of `new` carries no log/error; `clear` uses an explicit `let _`; `lossy` falls back via `unwrap_or_else`. `new_strict`/`try_clear` exist but the default path does not mandate them.

### P1-10 `collect_diagnostics` misses several field groups, diverging from `ConfigPlan::validate`

- Location: `bitty/crates/bitty-config/src/validation.rs`, `fn collect_diagnostics`
- Symptom: Batch diagnostics only collect font/window/terminal/layout/decoration/scrollbar/mouse/appearance/keymaps/plugins/extends/schema/undeclared, missing selection, close_confirm, mod_key, views, etc. that `plan.rs validate` does check. Incremental diagnostics under-report, so the `config check` full path and the editor-diagnostics path fork in results.
- Trigger: Project or user layer errs only on a missed field.
- Evidence: Item-by-item comparison of the per-field `if let Some(v)` list against the `validate` list in `plan.rs` shows the gaps; existing unit tests only cover the font/window/undeclared trio.

### P1-11 Project trust has no path normalization; empty hash plus in-memory state make the binding bypassable or re-asked every launch

- Location: `bitty/crates/bitty-config/src/trust.rs`, `struct TrustRecord`, `struct TrustStore`, `fn matches`, `fn is_stale`, `fn check_trust`
- Symptom: `canonical_path` uses plain string equality with no normalization, trailing-slash stripping, `..` normalization, symlink resolution, or Windows case normalization; `a/proj` vs `a/proj/` and different spellings of the same directory count as different principals. `content_hash` is an opaque string with no non-empty constraint, so empty matches empty. Storage is memory-only with the persistent location an open item, so restarts forget everything.
- Trigger: Accessing the same project directory under different spellings; hash computation missing or empty; re-entering the confirmation flow on every launch.
- Evidence: `matches`/`is_stale` are pure `==`/`!=` string comparisons; `TrustStore` has no file backend; the module docs admit DB location, rename invalidation, expiry, and UX are all open.

### P1-12 Config root only probes XDG/HOME; Windows known folders are missing

- Location: `bitty/crates/bitty-config/src/file.rs`, `fn config_dir_with_env`, `fn profile_file_path_with_env`
- Symptom: `config_dir` only accepts `XDG_CONFIG_HOME` or `HOME`, returning `None` when both are empty. On Windows `HOME` is usually unset and `APPDATA`/`LOCALAPPDATA` are never queried, so default probing directly has no root.
- Trigger: Standard Windows environments without explicit `HOME`/`XDG_CONFIG_HOME`.
- Evidence: Both the comment and implementation of the function list only those two variables; the live wrapper of `profile_file_path` likewise reads only those two variables. The type-level font fallback chain instead has four branches for Linux/macOS/Windows, showing platform-branch awareness exists but was not carried over to the path layer.

### P1-13 `terminal.shell` validation is too loose; NUL and other spawn-time errors surface misclassified as `Upstream`

- Location: `bitty/crates/bitty-config/src/types.rs`, `impl TerminalConfig::validate`; `bitty/crates/bitty-pty/src/builder.rs`, `fn validate`; `bitty/crates/bitty-pty/src/error.rs`, `enum PtyError`
- Symptom: Shell validation only checks trimmed non-emptiness, length, and control characters; it does not reject NUL or constrain absolute/bare-name/space-containing forms. A shell containing NUL passes the config layer and only fails at the PTY layer as `Upstream("...must not contain NUL...")`, mislabeling the error kind as upstream failure rather than parameter-validation failure, so `config check` and spawn errors diverge in wording.
- Trigger: `terminal = { shell = "..." }` carrying NUL or a "whole command line with arguments" string.
- Evidence: `TerminalConfig::validate` has no NUL branch; NUL checks for program/args in `builder.rs` return `Upstream` instead of precise variants such as `InvalidEnvVar`/`InvalidCwd`; the same file also reuses `Upstream` for argument-count/total-byte and environment-entry overlimits.

### P2-1 `PtyError::flatten_upstream` drops the source chain; diagnostics keep only one line of text

- Location: `bitty/crates/bitty-pty/src/error.rs`, `fn flatten_upstream`, `impl std::error::Error for PtyError`
- Symptom: Upstream errors keep only their `Display` text, and `source()` returns `None` for `Upstream`. The `io` passthrough keeps its source chain — inconsistent behavior; triaging resize/spawn failures loses errno/call context.
- Trigger: Any `openpty`/`resize`/`get_size`/`spawn_command` failure.
- Evidence: `source` returns `Some` only for `Io`; ADR-0004's "types must not leak" requires keeping text but does not require dropping the cause — a boxed `#[source]` or an extra `raw_os_error` field satisfies both.

### P2-2 Fingerprint stripping has a scan-vs-exec race; non-UTF-8 keys match via lossy conversion

- Location: `bitty/crates/bitty-pty/src/platform/unix.rs`, `bitty/crates/bitty-pty/src/platform/windows.rs`, `fn open_pty_and_spawn`; `bitty/crates/bitty-pty/src/builder.rs`, `fn should_strip_graphics_fingerprint`
- Symptom: First iterate `std::env::vars_os` removing key by key with `env_remove`, then re-apply an exact-key pass "belt-and-braces". The comment admits markers appearing between scan and exec can slip through; the remedy covers only exact keys, not prefix classes. Non-UTF-8 keys are prefix-matched via `to_string_lossy`, which can misclassify.
- Trigger: Concurrent process-environment mutation during spawn; or suspicious keys inherited from a non-UTF-8 locale.
- Evidence: The double-loop structure and the verbatim comment; the Windows and Unix copies are fully duplicated, so fixing one side easily misses the other.

### P2-3 `to_size` pixel width/height are always 0; DPI/window pixel size never reaches the PTY

- Location: `bitty/crates/bitty-pty/src/platform/unix.rs`, `bitty/crates/bitty-pty/src/platform/windows.rs`, `fn to_size`
- Symptom: `PtySize`'s `pixel_width`/`pixel_height` are always 0; platform-side `PhysicalSize`/scale changes only travel the cols/rows channel. Programs depending on pixel size (reading the `TIOCGWINSZ` pixel fields after `SIGWINCH`) always see 0.
- Trigger: HiDPI scale switch or font-size change followed by only `resize(cols, rows)`.
- Evidence: Both platforms' `to_size` share the same literal; the public `Pty::resize`/`size` API likewise has no pixel parameter. The docs say "the kernel updates the window size and delivers SIGWINCH", but the pixel fields are never updated.

### P2-4 Windows `convert_status` comment says signal is always `None`, but the code still forwards the upstream value

- Location: `bitty/crates/bitty-pty/src/platform/windows.rs`, `fn convert_status`
- Symptom: The comment says Windows has no POSIX signals so it is always `None`, but the implementation forwards `status.signal().map(...)`. If upstream ever returns `Some`, the public-API promise breaks.
- Trigger: Upstream ConPTY status-mapping change.
- Evidence: The Unix and Windows `convert_status` bodies are nearly isomorphic apart from the comment; Windows is not written as a literal `signal: None`.

### P2-5 URL handler hard-codes system absolute paths; preflight only runs on Linux and only checks `is_file`

- Location: `bitty/crates/bitty-platform/src/url.rs`, `fn url_handler`, `fn url_dispatch`, `fn handler_available`, `fn spawn_url_handler`
- Symptom: Handlers are three hard-coded absolute paths per platform; Linux preflights existence with `Path::is_file` (without checking the executable bit), while Windows/macOS spawn with no preflight. On NixOS, minimal containers, or customized systems the path may not exist, yielding `UrlLaunch`, while tests only cover the "missing handler errors first" Linux branch.
- Trigger: Target machine lacks the absolute-path handler, or has it on PATH but not at the absolute path.
- Evidence: Three literal branches in `url_handler`; `handler_available` uses only `is_file`; the availability guard in `spawn_url_handler` sits inside `cfg!(target_os = "linux")`.

### P2-6 `wl-copy` uses a bare PATH name, the background reaper writes stderr, and the 2-second semantics are fragile

- Location: `bitty/crates/bitty-platform/src/clipboard.rs`, `fn wl_copy_primary`, `fn wl_copy_spawn`, `fn spawn_wl_copy_reaper`, `fn reap_wl_copy`
- Symptom: Primary synchronously prefers PATH-resolved `wl-copy --primary` — payload over stdin is correct, but the program name is not pinned to an absolute path, so PATH hijacking can capture clipboard text. The background thread reports failure via `eprintln!`, i.e. the library writes unstructured warnings to stderr. The `WL_COPY_WAIT` 2s / `POLL` 10ms pair assumes the parent exits quickly; if some implementation does not fork-daemonize, the timeout kill clears freshly written primary content early.
- Trigger: Wayland session, missing/hijacked/behaviorally different `wl-copy`, wedged compositor.
- Evidence: `Command::new(program)` is parameterized by design, with the comment admitting "parameterized for testability"; the failure arm does `eprintln!("warning: ...")`; the `reap` timeout arm does `kill`+`wait`.

### P2-7 `SurfaceTarget` lifetime is a docs-only constraint with no compile-time enforcement

- Location: `bitty/crates/bitty-platform/src/surface.rs`, `struct SurfaceTarget`, `fn with_raw_handles`
- Symptom: A GPU surface must be released before the last window/target clone, otherwise the raw handle dangles; the obligation is stated only in docs, with no ownership enforcement across crates. A slightly careless future renderer causes UAF (the `wgpu`-side creation is `unsafe`).
- Trigger: Render surface cache outliving the window; abnormal multi-window close order.
- Evidence: Module docs: "Nothing enforces this across crates yet; the future renderer/GpuContext slice owns that obligation".

### P2-8 DPI error-variant misuse and silent normalization

- Location: `bitty/crates/bitty-platform/src/dpi.rs`, `struct LogicalPixel`, `struct ScaleFactor`, `fn new_sanitized`
- Symptom: `LogicalPixel::new` returns `InvalidScaleFactor` for NaN/inf, conflating pixel and scale semantics so callers cannot tell "illegal size" from "illegal scale". `WindowHandle::scale_factor` and the surface origin both unify illegal compositor values to 1.0 via `new_sanitized`, silently rendering at the wrong size with no log.
- Trigger: Compositor returning 0/negative/NaN scale; illegal logical-pixel input.
- Evidence: The error-construction site in `LogicalPixel::new`; the two `new_sanitized` call sites in `app.rs` and `surface.rs`.

### P2-9 Keyboard encoding drops shift semantics, `Ctrl+digit` is ambiguous, and `MAX_ENCODED_LEN` is unenforced

- Location: `bitty/crates/bitty-platform/src/keyboard.rs`, `fn encode_key_event_with_modifiers`, `fn ctrl_control_byte`, `const MAX_ENCODED_LEN`
- Symptom: `shift` has no legacy effect (docs say so explicitly), so `Shift+Tab` and `Tab` are both `\t` and back-tab is inexpressible; `super` is ignored outright. `Ctrl+digit/punct` with no C0 mapping falls back to bare text, indistinguishable from not pressing Ctrl. `MAX_ENCODED_LEN` 8 is only a comment ("Fn/CSI are small") with no assert/truncation/test binding the overlong path.
- Trigger: Terminal apps relying on back-tab, CSI modifier encoding, `Ctrl+5`, etc.
- Evidence: The modifier-semantics chapter lists each fallback above; `encoded_len_bound` exists but the production path never enforces it.

### P2-10 IME preedit/commit silently truncate; window title is unbounded and relies on caller sanitization

- Location: `bitty/crates/bitty-platform/src/event.rs`, `fn map_ime`; `bitty/crates/bitty-platform/src/app.rs`, `fn set_title`
- Symptom: Preedit over 128 chars and commit over 256 chars/1024 bytes are silently truncated, so long CJK compositions lose their tail with no error. `set_title` passes the string straight to `winit`, while the docs require the caller to sanitize and bound it first — but the type imposes no length/control-character constraint.
- Trigger: Long preedit strings; untrusted PTY output written straight to the title bar.
- Evidence: The hand-written truncation loop in `map_ime`; `set_title` has no validation and pushes responsibility to an "app-side sanitizer" in docs.

### P2-11 Unknown theme fallback writes stderr; library behavior forks from validation wording

- Location: `bitty/crates/bitty-config/src/theme.rs`, `fn resolve_theme`, `fn resolve_theme_with_status`, `fn normalize_theme_name`
- Symptom: The pure `resolve_theme_with_status` distinguishing Default/Named/FallbackUnknown is good; but the startup-path `resolve_theme` directly calls `eprintln!`, i.e. the library writes user-visible logs to stderr. An overlong theme name still returns `Some(lowercased)` in normalize and finally falls back to default, while the validation layer should fail closed on overlong input — the two paths disagree on the same input.
- Trigger: Misspelled or overlong theme in config.
- Evidence: `eprintln!("bitty: unknown theme ...")`; the overlong branch of `normalize_theme_name` does not return `None`.

### P2-12 Reload diff compares formatted strings; float precision and field coverage may drift

- Location: `bitty/crates/bitty-config/src/reload.rs`, `fn diff`, `fn classify_field`, `fn reconcile_live`
- Symptom: `font.size` is compared after two-decimal formatting and `line_height`/`letter_spacing`/`opacity` after three-decimal formatting, so changes smaller than half the last place count as no diff. `diff` only validates `new`, assuming `old` is legal; if `old` was already corrupt via external write, the comparison baseline is untrustworthy. The Live/Restart/Rejected tables in `classify_field` vs `merge_class_for` in `merge.rs` vs the merge-attribution table are each hand-written, so new fields easily drift across all three.
- Trigger: Fine-tuning font size/line height; old valid config corrupted by external write; new schema field added.
- Evidence: `push_if_changed` is all post-`format!` string comparison; `diff` starts with only `new.validate()`; the file-header "Drift note" admits the table is a draft slated to move.

### P2-13 Background-image path only gets syntax validation; `~`/absolute form and roots-membership checks stop at this crate

- Location: `bitty/crates/bitty-config/src/types.rs`, `fn validate_background_image_path`, `fn validate_background_image_roots`
- Symptom: Only non-emptiness, length, NUL, and absolute-or-`~`-leading form are checked; existence, normalization, approved-root membership, regular-file-ness, and decoding are all declared as the `bitty-rich` loading pipeline's job. The project layer may declare `views`/`decoration` background images, so if the loading-side allowlist has a gap, a project file can steer reads toward any absolute path.
- Trigger: Project config carrying `background_image` pointing at a sensitive path.
- Evidence: Both functions' docs explicitly list "this crate never opens files", and the project-allowlist comment in `trust.rs` records the risk as presentation-only and lets it through.

### P2-14 File loading has a TOCTOU window between double size checks; `cwd` only rejects empty

- Location: `bitty/crates/bitty-config/src/file.rs`, `fn load_layer_with_kind`; `bitty/crates/bitty-pty/src/builder.rs`, `fn validate`
- Symptom: First `metadata.len` is judged against 64 KiB, then `content.len` is re-judged after `read_to_string`; a concurrent rewrite/swap can change the file between the two checks — the second check catches growth, but the semantics are still non-atomic. Line-count/single-line-byte limits are re-checked inside `parse_lua_config`, so the loading and parsing layers each check once — duplicated logic. PTY `cwd` only rejects the empty string without checking existence/directory-ness, pushing the error down into upstream spawn text.
- Trigger: Config file concurrently written; `cwd` pointing at a nonexistent path.
- Evidence: `metadata` and `read_to_string` are two separate syscalls; the `cwd` branch of `validate` only checks `is_empty`.

## Suggestions

| ID   | Suggested fix                                                                                                                                                                                                                                                                              | Expected benefit                                                                                   | Effort |
| ---- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------- | ------ |
| S-1  | Change `Pty::Drop` to "kill + bounded reclaim via `wait_timeout` + error-report hook"; document that dropping on the event thread is forbidden, and provide `shutdown_timeout` for explicit out-of-destructor calls where needed                                                           | Eliminates destructor hangs; zombie reclamation can no longer stall the event loop                 | M      |
| S-2  | Keep `Err(Disconnected)` for `recv_timeout` on `Disconnected`, or add a `RecvOutcome` enum; make `try_recv` distinguish `Empty`/`Disconnected`; handle errors before EOF at call sites                                                                                                     | Eradicates EOF/error confusion; pump errors no longer silently become EOF                          | S      |
| S-3  | Add an explicit `Drop` for `PtyWriter` or fix the docs: either implement EOT writing, or declare "graceful close requires the caller to explicitly write the EOF byte; upstream drop only closes the fd" plus unit tests                                                                   | Close semantics match implementation; `cat`-style cases stop being flaky                           | S      |
| S-4  | Extend EIO mapping to "EIO on Unix generally means EOF" or expand by a `raw_os_error == EIO` constant table to macOS; keep the raw error plus `raw_os_error` on unknown platforms                                                                                                          | macOS drain stops misreporting; consistent behavior across Unix                                    | S      |
| S-5  | Return `Result` from `PtyReader::spawn`, converting thread-creation failure into `PtyError::Io` or a new `SpawnReader` variant                                                                                                                                                             | Recoverable under resource pressure; no more in-library panic                                      | S      |
| S-6  | Add an explicit unknown-busy tri-state to Windows close confirmation: keep `foreground_job` as `None`, but have the upper `CloseConfirm` layer take the conservative `always`-style prompt branch when busyness is unknowable, or annotate the platform difference in UI copy (Bliss)      | Closes the Windows fail-open data-loss window                                                      | M      |
| S-7  | In `process_name`, re-verify `foreground_pgid` is still the same pid after reading `comm`; discard the name or the whole job snapshot on mismatch                                                                                                                                          | Eliminates wrong confirmation text from pid reuse                                                  | S      |
| S-8  | Change clipboard overlimit from silent truncation to "truncate and return `Truncated(usize)`/dedicated error", or at least provide `try_set_text_strict`; keep the `lossy` family but document the stale semantics                                                                         | Large pastes no longer silently lose their tail; users can tell                                    | S      |
| S-9  | Keep the owned text of the original `arboard` error in `Clipboard::new` (new field or log callback), mark `clear` `#[deprecated]` pointing at `try_clear`; add an observable marker when `lossy` reads fall back                                                                           | Headless-vs-device divergence becomes locatable; green CI with forked behavior becomes diagnosable | S      |
| S-10 | Resolve `wl-copy` to an absolute path or `which`-check before launch and record the resolution; hand reaper errors back to the caller via callback/return instead of `eprintln!`; document the fork-daemonize assumption and assert "parent exits quickly" for non-forking implementations | PATH-hijack surface converges; library stops writing stderr directly                               | M      |
| S-11 | Share one field table between `collect_diagnostics` and `ConfigPlan::validate` (extract a field accessor or macro); add the missing selection/close_confirm/mod_key/views checks plus regression unit tests                                                                                | Editor diagnostics agree with `config check`; nothing under-reported                               | S      |
| S-12 | Add NUL rejection and shape hints to shell validation (bare name vs absolute path vs argument-containing string); move PTY overlimit errors from `Upstream` to precise `Invalid*` variants while keeping the `source` chain                                                                | Fail closed at config time; spawn errors point straight at the field                               | S      |
| S-13 | Normalize trust paths uniformly (trailing-slash stripping, `..` normalization, symlink resolution, Windows case normalization); require non-empty `content_hash` with algorithm/length constraints; until persistence lands, document "forgets on restart"                                 | Project-trust binding cannot be bypassed; UX expectations are explicit                             | M      |
| S-14 | Add Windows known-folder (`APPDATA`/`LOCALAPPDATA`) branches to the config root plus a unit-test matrix; the type layer already has a four-platform font chain — align the path layer                                                                                                      | Windows default launch no longer has no root                                                       | S      |
| S-15 | Add pixel parameters to `to_size` or a new `resize_px`, passing platform `PhysicalSize`+scale straight into PTY `pixel_width`/`pixel_height`; keep the public API cols/rows compatible                                                                                                     | `TIOCGWINSZ` pixel fields become correct under HiDPI; terminal apps stop reading 0 for layout      | M      |
| S-16 | Give `SurfaceTarget` a lifetime or a `GpuSurfaceGuard` RAII so "surface released before window" is expressed in ownership                                                                                                                                                                  | Cross-crate dangling handles get caught by the compiler                                            | M      |
| S-17 | Add the `Shift+Tab` back-tab and CSI modifier encoding tables for keyboard, or document them as unsupported with explicit rejection by the upper keymap; bind `MAX_ENCODED_LEN` with a `debug_assert`/unit test                                                                            | Complete modifier semantics; overlong encodings surface early                                      | M      |
| S-18 | Change IME truncation to return a truncation marker or error; add a centralized sanitizer for title input (length + control characters) enforced inside `set_title` instead of relying on callers                                                                                          | Long CJK no longer loses its tail; title-bar injection surface converges                           | S      |
| S-19 | Remove in-library `eprintln!` from theme fallback and uniformly return `ThemeResolution` for the binary layer to decide logging/exit code; unify overlong theme names with the validation layer as fail-closed                                                                             | Embeddable library without side effects; validation and runtime agree                              | S      |
| S-20 | Compare reload-diff floats with bit-exact/tolerance-explicit comparison instead of fixed-width formatting; validate both old and new ends in `diff`; extract the `classify`/`merge_class`/attribution trio into a single-source table plus coverage unit tests                             | Fine-tuning never loses a diff; new fields never drift                                             | M      |

## Test gaps

- PTY:
  - Real `SIGWINCH` delivery missing: existing resize unit tests only assert `size()` readback, never proving the foreground process group received the signal; needs a child with a signal handler end-to-end.
  - No unit test that the child sees EOF after writer drop covering the documented graceful path; the `cat` echo case never asserts "dropping the writer exits".
  - No bounded-`Drop` test: no fault-injection test that "destruction still finishes in bounded time when `wait` hangs after kill".
  - Non-Linux EIO: `linux_eio_is_mapped_to_eof` only compiles on Linux; macOS has no corresponding corpus.
  - Error text for thread-creation failure, repeated `try_clone_reader`, and double `take_writer` is covered, but double `take_reader` and writer/reader interleave order are thinly covered.
  - Windows `spawn_windows` covers spawn/resize/read-write/kill, but ConPTY busy/idle, `tty_name=None`, and `signal=None` platform-difference assertions are thin.
- Platform:
  - `headless_run` accepts both "Ok with display, DisplayUnavailable without" so regression sensitivity is low; peers must run it once in each environment or degradation-logic regressions stay green.
  - `gui-tests` are off by default and never run in CI; window creation, resize/redraw, raw handles, and DPI refresh have only local evidence; the most recent local run environment and commit must be recorded explicitly.
  - URL has only a validation corpus with no real-spawn integration (`open_url`/`open_file_url` are `allow(dead_code)`); negative tests for missing handlers, PATH differences, and `file` capability isolation are thin.
  - Clipboard is mostly headless-buffer based; Wayland primary `wl-copy` real-device sync, `arboard` fallback, and truncation observability have no integration assertions.
  - Keyboard lacks boundary unit tests for `Shift+Tab`, full upper/lower `Ctrl` tables, `Alt+Ctrl` combos, and overlong long-text/emoji output; `MAX_ENCODED_LEN` has no binding test.
  - Long IME string truncation, title sanitization, and post-illegal-scale-normalization render size have only unit assertions, with no contract test on "is truncation perceivable".
  - Zero-size surface skip and DPI-switch recompute have unit tests, but the "surface released before window" lifetime has no compile-time or runtime test.
- Config:
  - Hot reload has only pure `diff`/`reconcile` unit tests, with no file-watcher, atomic-write (rename vs truncate), debounce, or keep-previous-on-failure end-to-end; `should_retain_previous` only judges rejected, not restart-required stickiness.
  - Non-short-circuit `collect_diagnostics` has a unit test, but the missing fields mean the "report all errors together" promise does not hold for selection/close_confirm/mod_key/views.
  - Migration has only the empty 0→1 stub unit test, with no regression for a second real rename/reshape version; `migrate_rename_field_stub` is an identity stub with no production caller.
  - Missing-profile fail-closed and illegal-name fail-closed have unit tests, but Windows rootless, explicit `--config` missing, symlink, and concurrent-rewrite TOCTOU have no integration.
  - Trust has only in-memory unit tests (hash change means stale, deny untrusted, etc.) with no cases for normalization (trailing slash/`..`/symlink/case), empty hash, persistence, or rename invalidation.
  - Background-image roots membership, decoding, and dimensions live on the `bitty-rich` side; the config side is syntax-only; negative tests for project-layer `background_image` pointing at sensitive paths are missing.
  - Theme-fallback stderr side effects, the `normalize` overlong branch, and CLI-overlay attribution repair (attribution backfill in `resolve_effective_full`) have only scattered unit tests, lacking an explicit matrix of "CLI-covered sibling fields still attribute to baseline".

Date: 2026-09-15

Covered crates: `bitty-pty`, `bitty-platform`, `bitty-config`
