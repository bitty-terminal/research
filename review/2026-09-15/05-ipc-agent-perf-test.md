# Bitty Support-Layer Crates Review Report: IPC / Agent / Perf / Test-Support / Compat-Lab

Scope:

- `bitty/crates/bitty-ipc/` (inter-process communication, including frames, channels, transports, wire, auth, rate limiting, bridging, MCP, snapshots, execution, tool distribution, devtools, ctl)
- `bitty/crates/bitty-agent/` (generic Agent protocol: identity, messages, observations, side queues, sessions, tool vocabulary)
- `bitty/crates/bitty-perf/` (performance baselines: startup, latency, idle)
- `bitty/crates/bitty-test-support/` (live-PTY gating)
- `bitty/crates/bitty-compat-lab/` (compatibility lab: matrix, comparators, reporting)

Method: read-only review. Fully read each `Cargo.toml` and `lib.rs`; for IPC focused on protocol version negotiation, frame format and length prefixes, timeouts and reconnects and authentication, and DoS surfaces (oversized messages, slow consumers, slow producers); for perf checked measurement methodology; for test-support checked whether fixtures mask real behavior; for compat-lab checked coverage-matrix completeness and evidence chain. No source code was modified; only this report was produced.

---

## Defect List

### P0

No directly remotely exploitable P0. The two closest candidates (trusted identity after fd passing, client `BITTY_SOCKET` phishing) require a local multi-user machine or control of the local environment, so they are tentatively rated P1 (SEC), see below.

### P1

#### P1-1 Transport layer trusts attestation instead of per-connection `SO_PEERCRED` re-verification; fd passing can impersonate identity SEC

- Path + symbol: `bitty/crates/bitty-ipc/src/devtools/serve.rs::transport_attested_peer`, `bitty/crates/bitty-ipc/src/auth.rs::VerifiedPeer::attested`
- Phenomenon: `transport_attested_peer(runtime_uid)` directly returns `VerifiedPeer::attested(runtime_uid)` without checking any peer credential; the comment admits that “per-connection `SO_PEERCRED` re-verification needs nightly or `unsafe` and is out of scope for this slice”.
- Trigger: on a multi-user local machine, a legitimate client of the `0600` socket passes its authenticated fd via `SCM_RIGHTS` to a process of another UID; the server keeps serving under the original `VerifiedPeer`, and the new holder bypasses `verify_peer_uid`.
- Evidence: in `serve.rs`, `transport_attested_peer` is `#[must_use] pub fn ... -> VerifiedPeer { VerifiedPeer::attested(runtime_uid) }`; in `auth.rs`, `attested(_runtime_uid)` ignores its argument, and a unit test asserts `attested == verify_peer_for_connection(good)`, making marks from the two sources indistinguishable.

#### P1-2 Client `BITTY_SOCKET` pointing anywhere sends requests in cleartext to an attacker listener SEC

- Path + symbol: `bitty/crates/bitty-ipc/src/devtools/serve.rs::resolve_socket_path`, `SocketEnv::from_process_env`
- Phenomenon: any non-empty `BITTY_SOCKET` is trusted verbatim and returned, checking only NUL and a length cap, without verifying ownership and mode; the client `connect`s to that path and sends sensitive payloads such as `terminal.text` with no server-identity verification.
- Trigger: an attacker controlling the victim process environment (e.g. shell startup files, SSH forwarding, crash-dump replay) places a listening socket in a writable directory and induces `bitty ctl` to connect to it.
- Evidence: the first branch of `resolve_socket_path` is `if let Some(sock) = bitty_socket { if !sock.is_empty() { ... return Ok(sock.to_string()); } }`; the `SocketEnv` documentation calls itself “an identifier, not a credential”, but the client side performs no peer verification.

#### P1-3 `RateLimiter` ignores sustained rate and enforces only the burst cap, doubling the effective limit versus the RFC

- Path + symbol: `bitty/crates/bitty-ipc/src/limits.rs::RateLimiter::check`
- Phenomenon: the `limit_per_sec` field is stored but unused (`let _ = self.limit_per_sec;`); only `timestamps.len() >= burst` is checked; with `RC9_REQ_PER_SEC=100` and `RC9_BURST_PER_SEC=200`, the effective sustained rate is 200 req/s.
- Trigger: any endpoint capacity-planned at 100 req/s still passes everything when sending continuously at 150 req/s after a burst.
- Evidence: the comment inside `check()` in `limits.rs` says “for simplicity burst is the effective cap”; the unit test `rate_limiter_burst_and_window` verifies only burst semantics.

#### P1-4 Request timeout clears only the pending table, not the queue; expired requests are still executed by the peer

- Path + symbol: `bitty/crates/bitty-ipc/src/channel.rs::IpcEndpoint::drain_expired`, `send_request`
- Phenomenon: `drain_expired` removes expired ids from `pending`, but the same `IpcRequest` already queued in `requests: BoundedChannel` can still be taken out by `recv_request` and dispatched; timeouts do not cancel queued work.
- Trigger: under queue backlog plus a small `timeout_ms` (e.g. 1 s), the receiver still takes out and executes an already-expired request after the timeout, while the sender has already treated it as a timeout failure, producing a “sender thinks cancelled, receiver already executed” inconsistency.
- Evidence: `send_request` does both `requests.try_send(request.clone())` and `pending.insert(...)`, while `drain_expired` only touches `pending`.

#### P1-5 `BridgeClient::answer` enqueues before correlating; unknown ids can fill the response queue and cause DoS

- Path + symbol: `bitty/crates/bitty-ipc/src/bridge.rs::BridgeClient::answer`
- Phenomenon: for an unknown `id`, `send_response` still succeeds in enqueueing and only returns `false`; an attacker forging many responses with unknown ids can fill the default 64-deep response queue, causing legitimate responses to fail with `ChannelFull`.
- Trigger: a peer able to inject forged responses into the bridge end (or a compromised provider) continuously sends unknown-id payloads.
- Evidence: inside `answer`, `self.endpoint.send_response(response)?; Ok(self.endpoint.complete(id))`, with no “drop without enqueueing if unknown” branch; the unit tests in the same file cover only zero ids and oversized payloads, not unknown-id queue occupation.

#### P1-6 Child token appears in cleartext in error strings; logging leaks secrets SEC

- Path + symbol: `bitty/crates/bitty-ipc/src/auth.rs::ChildTokenStore::verify`
- Phenomenon: `unknown child token '{token_str}'` and `child token '{token_str}' expired` interpolate the opaque token verbatim into `Unauthenticated.reason`; the error enters logs and diagnostics via `Display`.
- Trigger: any verification-failure path that gets logged leaks the token; a replay attacker can directly reuse an unexpired token.
- Evidence: two `format!("... '{token_str}' ...")` occurrences in `verify()` in `auth.rs`; the `tool.rs` side already redacts tool arguments, but there is no redaction here.

#### P1-7 `AgentMessage::is_untrusted_content` always returns `false`, misleading callers that make trust decisions

- Path + symbol: `bitty/crates/bitty-agent/src/message.rs::AgentMessage::is_untrusted_content`
- Phenomenon: the body directly returns `false`, with a comment saying “no content sniffing here; policy will be centralized later”; if a caller writes `if msg.is_untrusted_content()`, terminal-origin user messages are forever treated as trusted instructions.
- Trigger: any host that concatenates terminal echo into `AgentMessage.content` and relies on this helper for `T-10` marking.
- Evidence: the function in `message.rs` is implemented as three comment lines plus `false`; the real trust marking exists only in `AgentObservation::is_untrusted_surface`, so the two layers have split semantics.

#### P1-8 Performance unit tests substitute loose ceilings for real budgets, so regressions can hide under green lights

- Path + symbol: `bitty/crates/bitty-perf/src/latency.rs::HEADLESS_WALL_CLOCK_CEILING_MS`, `measure_latency`, `measure_latency_with_pty_echo`
- Phenomenon: the real budgets are p50 8 ms / p99 15 ms, but the unit tests relax them by environment to p50 30/60/80 ms and p99 120/150 ms; when `ps` is unavailable, `meets_cpu_budget` directly returns `true`; when `cat` is unavailable, `measure_latency_with_pty_echo` silently falls back to a synthetic echo model, reporting the same `LatencyReport` under an inconsistent metric definition.
- Trigger: under loaded CI or in environments without PTY/display, p50 degrading to 40 ms still stays green; performance regressions surface only in `benches/latency_real.rs` and Tier 1 real hardware, while `just check` does not mandate running benches.
- Evidence: the unit-test comment in `latency.rs` explicitly says “the real budget is gated by benches, not by this unit test”; in `idle.rs`, `meets_cpu_budget` returns `true` when `sampled_cpu_pct` is `None`.

#### P1-9 Compatibility-matrix CI proves only self-consistency, never cross-terminal consistency; `SKIP` counts as green

- Path + symbol: `bitty/crates/bitty-compat-lab/src/matrix.rs::generate_matrix_json`, `bitty/crates/bitty-compat-lab/src/compare.rs::compare_all`, `bitty/crates/bitty-compat-lab/tests/compat_matrix.rs::matrix_differential_is_graceful`
- Phenomenon: `generate_matrix_json` unconditionally writes `"SKIP"` for the four terminals; `compare_all` returns an error containing `not found` when there is no `recording/references/bitty` dump, which the caller catches and directly `return`s (skips); `live_compat.rs` only runs when `BITTY_COMPAT_LIVE=1`, otherwise each case prints skip JSON and returns.
- Trigger: a default `cargo test` / CI all-green only means “self-replay determinism”; Ghostty/Kitty/WezTerm/Alacritty differences are never compared in gates.
- Evidence: `matrix.rs:258` hard-codes `SKIP` for four terminals; the differential case in `compat_matrix.rs` has `Err(e) if ... => { eprintln!("SKIP..."); return; }` as its first branch; `live_compat.rs:44` gates on `BITTY_COMPAT_LIVE==1`.

#### P1-10 Wire protocol has no version negotiation, two version fields are split, and upgrades break hard

- Path + symbol: `bitty/crates/bitty-ipc/src/wire.rs::validate_wire_version`, `bitty/crates/bitty-ipc/src/frame.rs` module documentation, `bitty/crates/bitty-ipc/src/devtools/json.rs::parse_request`
- Phenomenon: the generic envelope only accepts `v==1` (`WIRE_VERSION=1`), and the frame documentation explicitly says “No negotiation”; devtools has a separate string `version:"1.0"` (`DEVTOOLS_PROTOCOL_VERSION`), with no mapping between the two version schemes; there is no handshake negotiation and no capability advertisement, and `ping` only echoes the version.
- Trigger: upgrading either end to v2 yields wholesale `VersionMismatch` / `MissingVersion`, with no canary and no downgrade; mixed deployments of old and new `bitty` with `bitty-ai`/devtools break.
- Evidence: `wire.rs:75` errors on anything not equal to 1; `frame.rs:5` says “No negotiation, no compression”; `devtools.rs:71` has an independent version constant.

### P2

#### P2-1 `Framer` buffer upper bound disagrees between documentation and implementation; slow drip can hold a connection long-term

- Path + symbol: `bitty/crates/bitty-ipc/src/frame.rs::Framer::push_bytes`
- Phenomenon: the documentation says “at most one partial frame plus a 4-byte header”, while the implementation allows up to `MAX_BUFFERED_BYTES + MAX_FRAME_BYTES` (about 512 KiB + 8 B); the first branch `buf.len()+bytes.len() > MAX_BUFFERED && bytes.len() > MAX_BUFFERED` is a conjunction, so single large pushes and multiple small pushes follow different thresholds; `Framer` itself has no time dimension, so a 1-byte slow drip can occupy one connection slot indefinitely (a slow-consumer amplifier under the 16-connection cap).
- Trigger: an attacker slowly sends a legal prefix at 1 B per packet, or sustains a half-frame with slices just under the threshold; the `serve_connection` timeout depends on an external stream timeout, and pure headless multiplexers have no timeout.
- Evidence: two `if` branches with different thresholds starting at `frame.rs:191`; `MAX_BUFFERED_BYTES = MAX_FRAME_BYTES + 8`.

#### P2-2 `PeerCredentials::current` returns uid/gid 0 on non-Unix, easily mistaken for root

- Path + symbol: `bitty/crates/bitty-ipc/src/auth.rs::PeerCredentials::current`
- Phenomenon: when the real uid cannot be obtained without `unsafe`, it silently fills in 0; if a caller treats it as peer credentials and compares it against `expected_uid`, it may unexpectedly pass on a service running as root.
- Trigger: runtime glue code calls `current()` for convenience instead of extracting the value from platform `getuid`/`SO_PEERCRED`.
- Evidence: the `current()` implementation in `auth.rs` uses `uid: 0, gid: 0`, with a comment telling Unix to provide values externally, but the function signature carries no `#[cfg]` or `deprecated` marker.

#### P2-3 `validate_no_ambient_auth` is an approximate scan that can be bypassed and can misfire

- Path + symbol: `bitty/crates/bitty-ipc/src/wire.rs::validate_no_ambient_auth`
- Phenomenon: it only does literal `auth/scope/role` comparison against depth-1 quoted strings; a value string such as `{"foo":"scope"}` at depth 1 is likewise falsely rejected; escaped `"\u0073cope"` and case variants like `" Auth"` bypass the early rejection (the server-side `authorize_method` still provides defense in depth, but the early gate can be bypassed).
- Trigger: constructing an envelope with an escaped key name to pass `validate_request_envelope`, depending on whether the server consistently enforces authorization everywhere.
- Evidence: the hand-written scan starting at `wire.rs:209` compares before decoding `\uXXXX`; `key_start` does not distinguish keys from values.

#### P2-4 `verify_unix_endpoint` is a pure function that does not check symlinks; standalone reuse can be fooled by directory replacement

- Path + symbol: `bitty/crates/bitty-ipc/src/auth.rs::verify_unix_endpoint`
- Phenomenon: it only compares mode and owner numbers; symlink-rejection logic lives in `devtools/serve.rs::attestation_for` (`symlink_metadata`), so the two are separated; downstream code that directly reuses the `auth` module gets no symlink protection.
- Trigger: on a multi-user machine where the socket parent directory is replaced with a symlink, the pure-function check still passes.
- Evidence: that function in `auth.rs` makes no `symlink_metadata` call; `serve.rs` implements it separately with a comment marked `CR-IPC-01`.

#### P2-5 Transport stub `try_send_wire_bytes` has unfaithful semantics and inconsistent double bounds

- Path + symbol: `bitty/crates/bitty-ipc/src/transport.rs::StdioTransportStub::try_send_wire_bytes`
- Phenomenon: it first validates against `MAX_FRAME_BYTES+4`, then re-validates with `Frame::new(wire)` against `MAX_FRAME_BYTES`; the interval `MAX_FRAME+1..MAX_FRAME+4` passes first and fails later; the comment admits it is “not protocol-faithful, test visibility only”.
- Trigger: unit tests based on this helper pass without implying real-pipe behavior passes.
- Evidence: the implementation and comment starting at `transport.rs:201`.

#### P2-6 MCP/channel timeouts and capacities are inconsistent in three places; no reconnect and no heartbeat

- Path + symbol: `bitty/crates/bitty-ipc/src/mcp.rs::McpClientStub`, `bitty/crates/bitty-ipc/src/channel.rs::DEFAULT_REQUEST_TIMEOUT_MS`, `bitty/crates/bitty-ipc/src/transport.rs::StdioTransportStub::close`
- Phenomenon: MCP defaults to 10 s with a config-determined cap, the generic channel defaults to 5 s with a 30 s cap, and pending caps are 32 vs 64; after `close`, `clear` does not reopen, and there are no `reconnect`, `backoff`, or heartbeat paths; `forward_to` silently leaves residue when the peer is full, returning no backpressure signal.
- Trigger: after peer crash, pipe-full, or long unresponsiveness, callers can only rebuild the object, with no graceful recovery path.
- Evidence: constants at `mcp.rs:32` and `channel.rs:44`; the `clear` comment at `transport.rs:126` says it “does not clear the closed flag”.

#### P2-7 `serve_connection` has no total-duration cap for slow requests; clock rollback can wedge the rate limiter

- Path + symbol: `bitty/crates/bitty-ipc/src/devtools/serve.rs::serve_connection`, `bitty/crates/bitty-ipc/src/limits.rs::RateLimiter::evict_old`
- Phenomenon: each `read_exact` relies on an external stream timeout, with no “single-request total duration” cap; `RateLimiter` uses `now_ms.saturating_sub(front)`, so a clock rollback never evicts, and once the limiter fills it rejects forever.
- Trigger: slow drips that stay within the stream timeout to stay alive; callers passing a wall clock instead of a monotonic clock.
- Evidence: no total timer inside the loop in `serve.rs`; the eviction condition at `limits.rs:128`.

#### P2-8 `ctl` cross-thread queue has no priority; head-of-line blocking can time out healthy requests

- Path + symbol: `bitty/crates/bitty-ipc/src/ctl.rs::enqueue_control_and_wait`, `MAX_QUEUED_CONTROLS`
- Phenomenon: `CTL_TIMEOUT=5 s` is shared by both ends, and the 64-deep queue drains serially, so tail requests time out while queued; the rejection path already has “auth failures are not enqueued” (with a regression test), but head-of-line blocking among healthy requests is unresolved.
- Trigger: bursting 64 `sendInput`/`spawn` requests while the main thread applies each one longer than 5 s in total.
- Evidence: the queue and `recv_timeout(CTL_TIMEOUT)` implementation in `ctl.rs`.

#### P2-9 Agent sessions are too lenient about tool-call placement; side queues silently drop counts

- Path + symbol: `bitty/crates/bitty-agent/src/session.rs::AgentSession::push_message`, `bitty/crates/bitty-agent/src/queue.rs::SideQueue::push`
- Phenomenon: non-Assistant turns carrying `tool_calls` are structurally admitted (the comment says this keeps tests flexible); `SideQueue::push` silently drops the oldest when full, incrementing only a `dropped` counter that requires active polling to see, so observation gaps under terminal floods have no explicit signal.
- Trigger: abnormal or malicious hosts interleaving user turns with tool calls; high-frequency `TerminalOutput` flooding the 64-deep side queue.
- Evidence: the state-machine comment in `session.rs` and the no-backpressure implementation in `queue.rs`.

#### P2-10 Tool redaction is both over-broad and memory-resident SEC

- Path + symbol: `bitty/crates/bitty-agent/src/tool.rs::is_sensitive_key`, `scrub_text`, `ToolCall::arguments`
- Phenomenon: the trailing `l.contains("token")` in `is_sensitive_key` falsely redacts legitimate keys such as `tokenizer` and `tokens_used`; redaction applies only to log/IPC views, while the original text remains in `arguments`/`content` heap memory with no `zeroize`, so a heap dump leaks secrets.
- Trigger: tool arguments that count token usage get rewritten and break business logic; a process crash dump gets read.
- Evidence: the trailing wildcard in the sensitive-key function in `tool.rs`; the design storing originals in `ToolCall`/`ToolResult` and redacting only in `Debug`.

#### P2-11 Startup and idle probes are affected by process-global state and single-point sampling

- Path + symbol: `bitty/crates/bitty-perf/src/startup.rs::probe_winit_availability`, `probe_wgpu_availability`, `bitty/crates/bitty-perf/src/idle.rs::sample_self_cpu_pct`
- Phenomenon: `winit::EventLoop` can only be created once per process, so a second probe in the same process always yields `RecreationAttempt` (although classified as `Unavailable`, real-hardware re-measurement is order-dependent); the `wgpu` probe was observed at 10.2 s (per comment), far above the intuitive 5 s cap; idle CPU is a single `ps` sample after `sleep 200ms`, not a 10-minute mean.
- Trigger: the same test process probing a headless window first and then a real window; slow Windows driver paths; on Windows without `ps`, `None` directly counts as pass.
- Evidence: the classification comments for `RecreationAttempt`/`any_thread` and the relaxed 15 s upper bound in `startup.rs`; the single-`ps` implementation in `idle.rs`.

#### P2-12 Compatibility layer uses a hand-written JSON parser and a fixed 80x24 assumption

- Path + symbol: `bitty/crates/bitty-compat-lab/src/compare.rs::parse_snapshot_json`, `EXPECTED_WIDTH`, `EXPECTED_HEIGHT`, `bitty/crates/bitty-compat-lab/src/matrix.rs::MATRIX`
- Phenomenon: the comparator hand-extracts quotes/numbers/booleans instead of using `serde_json`; generic key names such as `cursor.row` are found by whole-text search via `find_key_value_start`, so identical strings inside `text` payloads can be mis-extracted; geometry unconditionally asserts 80x24, so real geometry changes in resize/DPI corpora are never compared; the 14-row `matrix.rs` and the ~40-row `report.rs` dual matrices coexist with no single source.
- Trigger: snapshots containing `"row"` text, non-80x24 corpora, and matrix-vs-report evolution skew.
- Evidence: the hand-written parser starting at `compare.rs:160`; the `EXPECTED_WIDTH/HEIGHT` constants; `report.rs::ROWS` and `matrix.rs::MATRIX` maintained independently.

---

## Suggestions

1. Add a per-connection re-verification path for `transport_attested_peer` (SEC, suggested approach: switch to `peer_credentials_unix_socket` once stable, and use a small reviewed `unsafe getsockopt(SO_PEERCRED)` shim at platform seams in the interim, re-verifying before every privileged action; carry a connection nonce in `VerifiedPeer` so `attested` and re-verified marks are not interchangeable; expected benefit: closes fd-passing impersonation; effort M)
2. Add server-identity verification to client socket discovery (SEC, suggested approach: after connecting, the client verifies the peer uid/pipe SID, or by default refuses `BITTY_SOCKET` pointing outside owner directories with a warning; document that the environment variable is only a hint and untrusted; expected benefit: prevents local phishing listeners; effort S)
3. Fix `RateLimiter` sustained rate (suggested approach: implement a sliding-window dual threshold: reject when exceeding 100 within 1 s, with burst 200 allowed only via a short-window token bucket; add sustained-rate unit tests; expected benefit: restores RFC RC-9 semantics; effort S)
4. Cancel queued requests on timeout (suggested approach: `drain_expired` also removes already-expired items from the `requests` queue, or the receiver filters expired items in `recv_request` with a counter; expected benefit: eliminates the “sender timed out, receiver still executes” split; effort S)
5. Change `BridgeClient::answer` from enqueue-then-correlate for unknown ids to correlate-first (suggested approach: pre-check with `complete`, and if `pending` has no such id, drop directly with a counter without occupying the response queue; expected benefit: closes forged-response DoS; effort S; SEC side benefit)
6. Redact child-token errors (SEC, suggested approach: `verify` failure reasons return only categories such as `unknown/expired/scope-mismatch` plus a truncated hash, never the original text; store-side index tokens by hash; expected benefit: logs no longer leak secrets; effort S)
7. Clarify or remove `is_untrusted_content` semantics (suggested approach: either delete the stub or change it to an explicit source-based mark such as `from_terminal_output: bool`; update documentation and call examples accordingly; expected benefit: eliminates `T-10` misjudgment; effort S)
8. Tighten performance gates into two-layer assertions (suggested approach: keep loose anti-flake assertions in unit tests but rename them `*_smoke`, enforce the true 8/15 ms budgets in `cargo bench` and Tier 1 evidence; mark `path: synthetic-fallback` in the report when `measure_latency_with_pty_echo` falls back so incomparable evidence is explicit; treat `ps`-unavailable as `inconclusive` rather than pass; expected benefit: regressions no longer hide; effort M)
9. Introduce minimal real comparison for compatibility differentials (suggested approach: check in a minimal deterministic reference dump or run `live_compat` + `compare_all` in a `BITTY_COMPAT_LIVE` nightly job, keeping CI self-consistent by default but requiring `reference_compared>0` nightly; merge `MATRIX` and `ROWS` into a single source or add a consistency test; expected benefit: the matrix lives up to its name; effort M)
10. Wire-protocol version roadmap (suggested approach: freeze `v1` semantics, add a `negotiateVersion`/`ping` capability-advertisement document, specify unknown-version behavior as “echo maximum supported + close” to avoid silent upgrades; unify `wire v` and `devtools version` naming; expected benefit: upgrades can be canaried; effort M)
11. Add a time dimension to `Framer` and align documentation (SEC, suggested approach: change documentation to “at most about two frames”, or tighten the implementation to a single frame plus header; add single-request total duration and minimum rate at the connection layer (`read_timeout` already exists, add rate control); document `now_ms` as a monotonic clock and constrain it; expected benefit: slow-drip cost becomes predictable; effort S)
12. Precise tool redaction plus memory hygiene (SEC, suggested approach: match `token` only on whole words or delimiter boundaries, allow `tokenizer/tokens_used`, and add unit tests; evaluate `zeroize` or prompt clearing of originals, or at least document heap-dump risk; expected benefit: fewer false positives and a smaller leak surface; effort S)
13. Replace the comparator with `serde_json` or formalize the hand-written parser (suggested approach: if keeping zero dependencies, add escape/`\u` and key-scope unit tests plus fuzzing, otherwise switch to `serde_json`; assert geometry per-corpus declarations instead of global 80x24; expected benefit: eliminates mis-parsing; effort M)
14. Gate `PeerCredentials::current` with `#[cfg]` or rename to `insecure_placeholder` (SEC, suggested approach: make it unavailable at compile time on non-Unix platforms or explicitly `unavailable` at runtime, preventing misuse; expected benefit: prevents mistaking it for root; effort S)

---

## Test Gaps

- IPC oversized messages: the 256 KiB single-frame cap, 1 MiB logical cap, and `CHUNK_CEILING` all have unit tests, but there is no end-to-end case for “declared length huge + body tiny” prefix poisoning on a real `serve_connection` (only at `Framer` level); missing multi-frame AAABBB concatenation and interleaved half-packet fuzz corpora.
- IPC slow consumers: `send_drop_oldest` only has count unit tests, `forward_to` peer-full residue has no backpressure-signal assertion; `serve_connection` slow-drip and slow-read have no time-gated unit tests; `RateLimiter` clock rollback and out-of-order `now_ms` have no cases.
- IPC timeouts and reconnects: `drain_expired` queue linkage, `CTL_TIMEOUT` head-of-line blocking, non-reopenable-after-`close`, and retry after partial moves in `forward_to` are all uncovered; MCP has no disconnect-reconnect, heartbeat, or backoff tests.
- IPC authentication: the `SO_PEERCRED` real path, `LOCAL_PEERCRED`, and `GetNamedPipeClientProcessId` are all outside headless with no integration evidence; symlink rejection exists only in `serve.rs`, not in the pure `auth` reuse path; `BITTY_SOCKET` phishing has no client-side case; child-token redaction has no log assertion.
- Perf methodology: missing comparison data between the echo model and real `cat` echo; unquantified deviation between single-point `ps` and 10-minute means; no fixed-order case for `winit` second-probe order dependence; bench true budgets are not in the default gate.
- Test-support: under `BITTY_TEST_FORCE_NO_PTY=1` everything skips yet stays green, with no “skip count” report; POSIX program assumptions under stacked `#[cfg(unix)]` gates have no Windows ConPTY counterpart cases; `pty_supported` conflates backend existence with specific-program existence.
- Agent: the tool-call-placement leniency branch has no “production should reject” counter-example unit test; constant-false `is_untrusted_content` has no caller-contract test; `dropped` on side-queue overflow is not automatically surfaced, missing an end-to-end gap-awareness case; redaction false positives (`tokenizer`) and heap residue have no cases.
- Compat-lab: zero coverage of real four-terminal differentials (constant `SKIP`); `live_compat` requires manual `BITTY_COMPAT_LIVE=1` and depends on local `tmux/nvim/fzf/htop/chafa`, so CI never runs it; the hand-written JSON parser has no fuzzing; `MATRIX` vs `ROWS` dual sources have no consistency gate; six categories are explicit `Gap` (`tmux DCS`, `real ssh`, `IME composition`, `interactive mouse`, `OS clipboard round-trip`, `shell integration installers`), with `kitty 5522` as `Partial`.

---

Date: 2026-09-15

Coverage: `bitty-ipc`, `bitty-agent`, `bitty-perf`, `bitty-test-support`, `bitty-compat-lab`
