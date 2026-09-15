# bitty-devtools Read-Only Code Review Report

## Scope

- Repository: `bitty-devtools` (standalone repository; does not own the core debug protocol, only implements the consumer-side client)
- TS side: `src/*.ts`, about 15 files (`auth`, `automation`, `campaign`, `protocol`, `redaction`, `transport`, `client`, `inspection`, `tracing`, `control`, `cli`, `bounds`, `panel-runtime`, `compat-matrix`, `index`)
- Rust side: `crates/devtools-client/` (`auth.rs`, `transport.rs`, `protocol.rs`, `redaction.rs`, `bounds.rs`, `inspection.rs`, `tracing.rs`, `control.rs`, `compat.rs`, `lib.rs`)
- Engineering surface: `tests/` (14 `bun:test` files), `package.json`, `tsconfig.json`, `Cargo.toml`, `justfile`

## Method

- Reading order: `package.json` → `src/index.ts` → `src/protocol.ts` → `src/transport.ts` → `src/redaction.ts` → `src/auth.ts`, then fan out to `automation.ts`, `campaign.ts`, `client.ts`, `inspection.ts`, `tracing.ts`, `control.ts`, `cli.ts`, `bounds.ts`.
- Compared each module against its Rust-side namesake, focusing on: authentication/authorization semantics, sensitive-field coverage, reconnect and error handling, concurrency and idempotency, and TS/Rust type and constant consistency.
- Keyword searches: `reconnect|retry|backoff|idempot|nonce|bearer|token|secret|password|api[_-]?key|authoriz`, `drainExpired|revoke|nextRequestId|Date.now`, confirming negative capabilities (no reconnect, no idempotency keys, no automatic expiry cleanup).
- Read-only: no source code was modified; only this report file was added.

---

## Defect List

Convention: severity `P0` (blocking/exploitable) / `P1` (high-priority defect) / `P2` (medium-low priority); `SEC` marks security class.

### D01 P1 (SEC) `src/auth.ts` `ChildTokenStore.verify` / `crates/devtools-client/src/auth.rs` `ChildTokenStore::verify` echoes token cleartext in error messages

- Phenomenon: error messages for unknown or expired child tokens interpolate the caller-supplied `tokenStr` verbatim into exception text, carrying tokens into logs/call stacks/`ctl` stderr.
- Trigger: any request carrying a forged or expired `token` hits the `verify()` failure branch.
- Evidence:
  - `src/auth.ts:298`: `` `unknown child token '${tokenStr}'` ``
  - `src/auth.ts:304`: `` `child token '${tokenStr}' expired` ``
  - `crates/devtools-client/src/auth.rs:305`, `308`: same semantics with `format!("unknown child token '{token_str}'")` / `expired` branches.
  - Conflicts with the repository's own security requirements of “minimal collection, typed redaction, transmit after preview”.

### D02 P1 (SEC) `src/redaction.ts` `redactValue` value heuristic is dead code, `SENSITIVE_PATTERNS` coverage insufficient

- Phenomenon: long base64/hex-shaped high-entropy values under non-sensitive field names are never redacted; common names such as `bearer`, `private_key`, `passwd`, `credential`, and `passphrase` slip through.
- Trigger: when the field is named `bearer` (all three requests in `automation.ts` use that field name), `privateKey`, `passwd`, or `credentials`, `isSensitiveField()` returns `false` and `redactValue()` returns the original value directly.
- Evidence:
  - `src/redaction.ts:12-19`: only `password|secret|token|api[_-]?key|authorization|cookie`, without `bearer|private[_-]?key|passwd|pwd|credential|passphrase`; `api[_-]?key` requires the `api` prefix, so `private_key` never matches.
  - `src/redaction.ts:28-36`: the comment says “redact only sensitive fields”, both branches `return value`, so value-shape checks have no effect.
  - `crates/devtools-client/src/redaction.rs:21-26`: likewise returns directly after field-name judgment, with no value-shape fallback.
  - `src/automation.ts:97,117,155`: the `bearer: string` field name happens to be outside the sensitive table.

### D03 P2 (SEC) `crates/devtools-client/src/redaction.rs` `redact_preview` redaction judgment disagrees with TS

- Phenomenon: Rust judges redaction via `out == "[REDACTED]"`; a non-sensitive field whose original text happens to be `"[REDACTED]"` is mislabeled, and the boundary where redacted output coincidentally equals the same value is also ambiguous. The TS `out !== text` is more accurate.
- Trigger: `field="preview"` with `text=="[REDACTED]"`, or future changes to value-redaction policy.
- Evidence:
  - `crates/devtools-client/src/redaction.rs:37-39`: dead `let _redacted = ...` variable + `was_redacted = out == "[REDACTED]"`.
  - `src/redaction.ts:52`: `wasRedacted = out !== text`.

### D04 P2 (SEC) `src/auth.ts` `ChildTokenStore.insert` overwriting the same token renews it indefinitely

- Phenomenon: when `isNew` is false it directly `set`-overwrites, checking neither capacity nor TTL monotonicity. A caller holding the same `token` string can repeatedly `insert` to refresh `createdAtMs`, bypassing the 60 s TTL.
- Trigger: reusing an already-leaked/already-issued `token` string with a fresh `createdAtMs` to re-`insert`.
- Evidence: `src/auth.ts:277-285`; Rust `crates/devtools-client/src/auth.rs:286-295` shares the same logic.

### D05 P1 (SEC) `src/auth.ts` `ChildTokenStore` expiry cleanup never triggers automatically, denying service after filling to 64

- Phenomenon: `drainExpired()` exists but has no production caller in the whole repository (only test callers); expired entries stay resident up to the `MAX_CHILD_TOKENS=64` cap, after which every new `insert` throws `LimitExceeded`.
- Trigger: long runs + natural expiry accumulation; an attacker can deliberately create 64 short-lived tokens to squat the table.
- Evidence: searching `drainExpired` hits only the definition at `src/auth.ts:315` and `tests/auth.test.ts:133`; `src/client.ts`, `src/transport.ts`, and `src/campaign.ts` never call it.

### D06 P2 (SEC) `src/automation.ts` `validateBearer` has no minimum-entropy requirement

- Phenomenon: `bearer` only validates `1..128` length and `^[A-Za-z0-9_-]+$`; a single character `"a"` is legal, inconsistent with the strength assumption of a “consent-issued per-session bearer”.
- Trigger: if the server trusts client-side shape checks, or weak test bearers flow into production.
- Evidence: `src/automation.ts:489-497`; 32-hex in tests is only convention (`tests/automation.test.ts:86`), with no lower-bound constraint.

### D07 [P1] `src/transport.ts` `IpcTransport.connect` / `crates/devtools-client/src/transport.rs` `IpcTransport::connect` uses cumulative request count as the connection cap

- Phenomenon: in `checkConnectionCap(this.requests)`, `requests` only grows; after 16 requests the next `connect()` (including reconnects) always throws `ConnectionLimit`, contradicting the “16 concurrent connections, shed newest” semantics.
- Trigger: the same `IpcTransport` instance sends >=16 requests, then `disconnect()`s and `connect()`s again; inevitable in long sessions.
- Evidence:
  - `src/transport.ts:500`: `checkConnectionCap(this.requests)`; `src/transport.ts:554`: `this.requests += 1`, while `disconnect()` never resets it (`508-510` is only a `clear()` stub).
  - `crates/devtools-client/src/transport.rs:563`: `check_connection_cap(self.requests)` has the same problem.

### D08 [P1] `src/transport.ts` `IpcTransport.request` reads a single frame with no RC-10 reassembly, asymmetric with `encodeRequest` fragmentation

- Phenomenon: `encodeRequest()` splits logical frames larger than 256 KiB into multiple `Frame`s for sending, but `request()` does a single `recvIncoming()` followed by `decodeResponse`, so large-response/large-request round-trips always yield `InvalidFrame` or silent truncation. The code comment admits “L2 not fixed”.
- Trigger: `params` or `result` exceeding 256 KiB (e.g. the 600 KiB case in `tests/transport.test.ts:104-123` only asserts `encodeRequest` frame counts, never round-trips).
- Evidence: fragmented sends at `src/transport.ts:528-541`; single-frame receive at `src/transport.ts:572-590`; the `Framer` class exists but is not wired into `IpcTransport`.

### D09 [P1] Transport has no reconnect/backoff/idempotency: zero hits for `reconnect|retry|backoff`, `nextRequestId` grows without bound

- Phenomenon: after disconnect there is no automatic reconnect, no backoff, and no request deduplication; `InspectionClient` and `AutomationClient` `nextRequestId` only `+=1`, with no wrap-around and no idempotency keys, so resends re-execute (`synthesizeInput` is non-idempotent).
- Trigger: after `TransportClosed`, callers can only manually `connect()`; sequential probes in `campaign.ts` record `fail` directly on timeout, with no retry semantics.
- Evidence: repository-wide search for `reconnect|retry|backoff` has zero source hits; `src/inspection.ts:632,655-656`, `src/automation.ts:722,800-801`; all `rpc` use non-injectable `Date.now()` (`src/inspection.ts:659`, `src/automation.ts:804`).

### D10 P1 (SEC) `src/protocol.ts` `isValidMethodForScope` disagrees with Rust `is_valid_method_for_scope` authorization matrix

- Phenomenon: TS places the four CTX-0159 read-only methods (`getGridText`/`getInputRing`/`getModifiers`/`getFocus`) into the `inspect` base set, while Rust still uses the old 7+4+3 matrix, returning `false` on queries. Cross-language clients make opposite allow/deny decisions for the same scope.
- Trigger: a Rust client invoking the four introspection methods with `Inspect`.
- Evidence: the augmentation set at `src/protocol.ts:196-200`; `crates/devtools-client/src/protocol.rs:105-119` lacks these four methods.

### D11 [P1] TS `BOUNDS` vs Rust `bounds.rs` has large gaps, `compat` JSON structures mutually incompatible

- Phenomenon: Rust lacks `MAX_CONNECTIONS`, `MAX_PERSISTENT_ID_LEN`, the entire automation family (`MAX_SYNTH_*`, `MAX_TRAJECTORY_*`, `MAX_AUTOMATION_*`, `MAX_BEARER_TOKEN_CHARS`, etc.), and `PREVIEW_MAX_CHARS`; the TS `MatrixJson` in `compat` (full version/generated/bounds/entries) and the simplified string from Rust `generate_matrix_json()` do not parse each other.
- Trigger: any joint debugging depending on identical thresholds on both ends or mutual matrix-JSON parsing.
- Evidence: `src/bounds.ts:10-80` vs `crates/devtools-client/src/bounds.rs:5-41`; `src/compat-matrix.ts:146-240` vs `crates/devtools-client/src/compat.rs:129-158`. Rust additionally has no `automation.rs`/`campaign.rs`/`cli.rs` counterparts; the automation contract is implemented TS-side only.

### D12 [P2] `src/protocol.ts` `validateFrameBytes` uses dual byte/character rulers, Rust uses bytes only

- Phenomenon: TS first applies `assertStringBounded` (UTF-8 bytes), then re-compares the same cap with `trimmed.length` (UTF-16 code units); the two rulers diverge on multi-byte text. Rust judges only `raw.len()` bytes.
- Trigger: 1 MiB boundary frames containing CJK/emoji, where the two ends may split allow/deny.
- Evidence: `src/protocol.ts:78-85`; `crates/devtools-client/src/protocol.rs:68-76`.

### D13 [P1] `src/campaign.ts` `CTL_VERB_MATRIX` contains mutating operations but no scratch-instance enforcement gate, probes non-idempotent

- Phenomenon: `workspace.new`, `view.split`, `terminal.send`, etc. expect `ok` in the default matrix, with comments requiring “must hit a scratch instance”, but `runCampaign`/`runLiveCampaign` never verify that the instance is scratch; the `focus`/`close` loop in `probeWorkspaceIdRoundTrip` has no per-step timeout, no concurrency bound, and no rollback, so re-runs shift the baseline.
- Trigger: accidentally running a live campaign against a production instance.
- Evidence: the matrix at `src/campaign.ts:513-633`; `runCampaign` at `src/campaign.ts:1375-1419`; the round-trip loop at `src/campaign.ts:862-949`; the live entry at `src/campaign.ts:1436-1456` only passes through `socketPath`.

### D14 [P2] `src/campaign.ts` `probeEnvelopeConformance` compares only `error.class`, not `code`

- Phenomenon: a `denied` row passes on `class="Denied"` alone, so codes in the same `EXIT_PERM` group such as `ScopeDenied/ScopeViolation/Unauthenticated/ForbiddenField` are misjudged as failures or missed, inconsistent with the code table in `expectedExitForError()`.
- Trigger: the server returns `class="Denied", code="Unauthenticated"` or vice versa.
- Evidence: `src/campaign.ts:757-776` compares only `expectedClassFor()`; the code table at `src/campaign.ts:61-105` looks at both `class` and `code`.

### D15 [P2] `src/client.ts` `connect` defaults `runtimeUid ?? 1000`, `resolveSocketPath` fallback path non-portable

- Phenomenon: when `runtimeUid` is not passed it assumes 1000; without `XDG_RUNTIME_DIR` it concatenates `/run/user/<uid>`, resolving to an unreachable path before failing closed on macOS/Windows or container-mapped UIDs.
- Trigger: developer machines that are non-Linux or UID != 1000 directly calling `new DevtoolsClient({socketPath})`, or CLI usage without `--socket/--instance`.
- Evidence: `src/client.ts:122`; the fallback at `src/auth.ts:185-188`; the no-instance `null` fail-closed at `src/cli.ts:295-297` is correct, but the error message does not mention the UID/platform assumption.

### D16 [P2] `src/tracing.ts` mixes billing and truncation units, `spoolPath` double standard

- Phenomenon: `startTrace` writes `"/tmp/bitty-traces/..."`, while `startTraceWithFilter` writes `"/run/user/1000/bitty/traces/..."` — two paths for the same session; `appendToTrace` bills with `data.length` (UTF-16) while `maxBytes` is a byte budget, so multi-byte text can exceed budget; `fetchTraceChunk`'s `continuation` mixes character offsets with byte totals via `offset + chunk.length < rec.bytes`.
- Trigger: trace payloads containing non-ASCII, paging by `offset`.
- Evidence: `src/tracing.ts:229,271,430-453,398-422`. Rust `crates/devtools-client/src/tracing.rs:400-424` bills with `.len()` bytes, opposite to TS behavior, widening the interop divergence.

### D17 [P2] `src/transport.ts` `RateLimiter` never uses `limitPerSec`, `Framer` discards legal prefixes on overflow

- Phenomenon: `check()` compares only `burst` (200); `limitPerSec` (100) is pure decoration, leaving the 100/s sustained quota nominal; `Framer.pushBytes` zeroes the entire buf on overflow, losing already-buffered legal prefixes with it.
- Trigger: sustained 150/s requests all pass; an attacker first stuffs garbage to trigger zeroing and destroy legitimate frames.
- Evidence: `src/transport.ts:213-222`; `src/transport.ts:147-153`. Rust `crates/devtools-client/src/transport.rs:238-251` (`let _ = self.limit_per_sec` proves non-use) and `167-175` share the same problem.

### D18 [P2] `src/inspection.ts` live/shadow dual semantics, `getSnapshotForTerminal` depends on server-unimplemented semantic snapshots

- Phenomenon: when connected it calls `rpc("bitty.debug/getSnapshot")`, and when the server actually returns a `runtime-stats` snapshot the client fails closed with `SemanticSnapshotUnavailable` (correct, but showing the contract break); when unconnected it falls back to locally synthesized `PanelRuntime` data, and the two paths have different field semantics, easily mistaken as equivalent.
- Trigger: calling `listPlugins/getPlugin` while the `DevtoolsClient` is unconnected yields stub data; live `getSnapshot` always throws unavailable.
- Evidence: `src/inspection.ts:680-713,916-956` (H1 comment admits the cross-repo gap); `src/client.ts:79-88,119-150` headless fallback semantics.

---

## Suggestions

### S01 (SEC) Uniformly redact error messages so tokens never enter exception text (effort S)

- Approach: change `ChildTokenStore.verify` (TS/Rust) `unknown/expired` errors to fixed text without token values (e.g. `unknown child token` + truncated hash/request ID), and similarly clean verbatim `scopedId` echoes in `scope mismatch`; add unit-test assertions that `AuthError` messages contain no token substring.
- Expected benefit: removes the most direct bearer-leak surface, passing a negative security-test gate.
- Effort: S.

### S02 (SEC) Complete the sensitive-field table and make value-shape fallback actually work (effort S)

- Approach: extend `SENSITIVE_PATTERNS` with `bearer|private[_-]?key|passwd|pwd|credential|passphrase|session[_-]?key` (case- and separator-insensitive), or switch to an explicit field allowlist + redact unknown `*token*` by default; fix the dead `redactValue` branch: at least label high-entropy long strings under non-sensitive fields as `truncated+redacted:false` with an entropy hint exposed in the `redactPreview` marker, or explicitly remove the heuristic to avoid misleading.
- Expected benefit: real secret fields such as `automation.bearer` are masked by default.
- Effort: S.

### S03 (SEC) Fix token-overwrite renewal and accumulation DoS together (effort M)

- Approach: on same-value `insert` overwrites, require monotonic `createdAtMs` with no TTL extension (or directly reject overwrites, requiring `revoke` first); call `drainExpired` from `ControlClient`/`IpcTransport` request entries or scheduled GC, and add `dropped/rejected` count observability to `LimitExceeded`.
- Expected benefit: TTL cannot be bypassed, and long sessions are not killed by expired accumulation.
- Effort: M.

### S04 Fix connection-count semantics and document the reconnect policy (effort M)

- Approach: change the `checkConnectionCap` argument in `connect()` to the true active-connection count (passed in by the owner), keeping `requests` only for rate/audit counting; clarify whether `disconnect()` resets; state in `campaign.ts`/README that “there is no automatic reconnect; callers own backoff”, or add an optional `reconnectPolicy` (fixed backoff + jitter + cap) without enabling it by default.
- Expected benefit: long-lived clients no longer self-lock after 16 requests; reconnect behavior becomes predictable and testable.
- Effort: M.

### S05 Wire in `Framer` reassembly to complete RC-10 large-frame round-trips (effort M)

- Approach: change `IpcTransport.request` to accumulate multiple `Frame`s via `Framer` and reassemble by length before `decodeResponse`; add 600 KiB-scale round-trip unit tests (one each for TS/Rust); assert `sendRequest` fragmentation and receive reassembly share the `MAX_FRAME_BYTES` constant.
- Expected benefit: large automation trajectories/frame snapshots no longer necessarily fail, with tests covering the real failure surface.
- Effort: M.

### S06 Unify the TS/Rust authorization matrix and single-source constants (effort M)

- Approach: extract the `isValidMethodForScope` method table (including the four CTX-0159 methods) into a two-repo shared contract snapshot test (golden list), requiring dual-repo co-changes on any addition/removal; fill the `BOUNDS` gaps or explicitly declare “Rust implements only a subset”, and pick one `compat` JSON as canonical (TS full structure recommended, with Rust changed to isomorphic serialization).
- Expected benefit: eliminates cross-language allow-on-one-side/deny-on-the-other and threshold divergence; joint-debug failures localize to versions rather than implicit constants.
- Effort: M.

### S07 Unify byte semantics: bill/truncate/validate all in UTF-8 bytes (effort S)

- Approach: unify TS-side `appendToTrace`, `fetchTraceChunk`, and `validateFrameBytes` on `TextEncoder` byte lengths; remove the character second comparison in `validateFrameBytes` or state two thresholds explicitly; add TS/Rust cross-check tests (including emoji/CJK boundary frames).
- Expected benefit: boundary frames and trace quotas agree on both ends, no longer split allow/deny.
- Effort: S.

### S08 Make `RateLimiter` live up to its name or rename it honestly (effort S)

- Approach: either implement a 100/s sustained + 200 burst dual bucket with a sliding window, or rename/docs-change to “burst-only 200/s” and fix the RC-9 comment; change `Framer` overflow to keep fully parsed frames and drop only the incomplete tail with a counter.
- Expected benefit: rate-limit semantics agree with documentation, burst behavior explainable.
- Effort: S.

### S09 Campaign live safety gate: scratch-instance assertion + probe timeouts/idempotency (effort M)

- Approach: add `requireScratchInstance` to `runLiveCampaign` (instance-name allowlist/explicit `--i-know-this-is-scratch`); add per-step timeouts to `probeWorkspaceIdRoundTrip`/`probeTerminalSpawnObservability` with second confirmation for default-off `closeWorkspaces`; validate `class+code` together in `probeEnvelopeConformance`, sharing one judgment function with `expectedExitForError`.
- Expected benefit: the blast radius of accidentally hitting production instances converges, with failures attributed to codes rather than classes.
- Effort: M.

### S10 Clarify dual-path semantics, bearer minimum length, and injectable clocks (effort S)

- Approach: document and annotate in types that “offline synthesized data is never equivalent to live responses”; add a minimum length to `validateBearer` (e.g. 16) with a comment that entropy requirements are ultimately enforced server-side; make `nowMs` in `rpc` injectable (defaulting to `Date.now()`), likewise `atMs` in `ControlClient.audit`, for deterministic testing.
- Expected benefit: less mistaking stub data for production evidence, with unit-test coverage of rates and audit times.
- Effort: S.

---

## Test Gaps

- Reconnect and backoff: no disconnect-reconnect, backoff-jitter, or idempotent-replay tests; post-`TransportClosed` behavior is only “throw”, with no recovery-path tests.
- Large-frame round-trips: only `encodeRequest` fragment-count assertions, no `Framer`-reassembled `decodeResponse` round-trips, no 1 MiB boundary multi-byte frame cross-checks.
- Redaction negatives: no field-name tests for `bearer/private_key/passwd/credential`, no behavior tests for high-entropy values under non-sensitive fields, no “error messages contain no token” assertions.
- Token lifecycle: no tests for overwrite renewal, concurrent filling to 64, recovery via `drainExpired` after hitting the cap, or TTL boundaries (`59_999/60_000` only has positive cases, no clock-rollback cases).
- Rate-limit sustained: only burst cases (5/200), no 100/s sustained pressure, no multi-window sliding, no `limitPerSec` effectiveness tests (because it is ineffective).
- Concurrency and idempotency: `automation`/`campaign` are all sequential calls, with no interleaved concurrent `synthesizeInput`, no `nextRequestId` wrap-around, no `BITTY_CTL_ELEVATE` race tests.
- Cross-language consistency: no TS/Rust shared goldens (scope matrix, `BOUNDS`, `compat` JSON, `shortInstanceHash` beyond auth) tests; missing `compat` JSON mutual parsing.
- Live and destructive: `runLiveCampaign` is never covered by `bun test` (by design), but the live behaviors of scratch gates, timeout sentinel `EXIT_TIMEOUT`, and output truncation `MAX_CAMPAIGN_OUTPUT_BYTES` have no integration-exercise records.
- Platform-specific: macOS `SUN_LEN` for `resolveSocketPath`, Windows pipe ACLs, and `XDG_RUNTIME_DIR`-missing fallbacks have only headless assertions, with no real multi-platform CI evidence presented in this repository.
- Observability: silent drops in `sendDropOldest`/`pushAudit` have no count-alarm tests; post-full-256 `auditLog` behavior is only implicitly covered.

---

Date: 2026-09-15
