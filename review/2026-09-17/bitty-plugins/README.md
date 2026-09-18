# Plugin ecosystem review — independent second pass

Date: 2026-09-17. Findings retain their original IDs. **Verified** below means
independently traced in current source and relevant contracts, not reproduced,
shipped, or exhaustively tested. Report-level second-pass ledgers supersede
original severity/coverage wording; original test results belong to the first pass.

## Reports

- [SDK and template](01-sdk-and-template.md): package entry, mock fidelity,
  schema/conformance limitations, generator and author workflow.
- [Registry and distribution](02-registry-and-distribution.md): registry/store
  metadata and validation, local installer transactions, trust boundaries.
- [Independent plugins](03-independent-plugins.md): Activity, Palette,
  Statusline, File Manager, Git Panel, and Wheel; targeted host cross-checks.
- [Documentation alignment](04-documentation-alignment.md): sampled manifest-hash
  contract, panel extraction gates, Git-tool ownership, and historical corrections.
  This follow-up does not independently renew every earlier finding.

## Verified priorities

### P1 — local package-store transactions

- **PLUG-REG-007:** same-version semantic manifest changes with unchanged Lua
  can commit a new index hash while retaining the old manifest. Resolution
  rejects that mismatch; this is availability, not execution of unapproved code.
- **PLUG-REG-010:** rename-error fallback attempts to delete the old index.
  Successful deletion followed by failed/interrupted replacement loses it;
  not every rename error causes loss. Platform fault behavior was not exercised.
- **PLUG-REG-011:** unprotected read-modify-write transactions and a shared
  temporary index permit lost updates when CLI processes overlap. No race run.

### P2 — strongest independently verified issues

- **PLUG-SDK-002, PLUG-SDK-003, PLUG-SDK-004, PLUG-SDK-005,
  PLUG-SDK-008:** missing documented package import entry, late UI registration,
  suspended mock dispatch, timer cancellation ordering, and shared mock values.
  These are package/mock defects, not evidence of host security bypasses.
- **PLUG-REG-001, PLUG-REG-002, PLUG-REG-003, PLUG-REG-005,
  PLUG-REG-006, PLUG-REG-008, PLUG-REG-009:** raw optional-type validation loss,
  browser shape mismatch, stale metadata provenance, checkout-vs-gitlink checks,
  raw/canonical digest contradiction, reinstall retention loss, and disabled
  package re-enablement. REG-001's complete bounds matrix was not revalidated.
- **PLUG-APP-003, PLUG-APP-004, PLUG-APP-005, PLUG-APP-008:** incomplete
  Activity purge, stale result roots, pane-cwd/repository-root conflation, and
  Statusline's missing aggregate scene-text limit. Git result-path findings do
  not establish actual subprocess cwd: the current bridge ignores Lua's second
  spawn argument.

### Corrected dispositions — do not carry forward the original severity

| ID           | Independent disposition                                                                                                                                                                                                 |
| ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| PLUG-SDK-001 | P1 withdrawn; P2 layout/RFC drift. Actual loader chooses `lua/` and accepts nested `<id suffix>/init.lua`; root-file absence does not block entry discovery. Generated activation remains untested.                     |
| PLUG-SDK-006 | Acknowledged P2 conformance coverage gap, explicitly documented, not a new implementation defect or host enforcement claim.                                                                                             |
| PLUG-APP-001 | P1 withdrawn; P2 timer contract/integration mismatch. SDK/RFC require activation-only creation, but the Rust bridge accepts later creation; real timer delivery remains unverified.                                     |
| PLUG-APP-002 | P1 withdrawn; conditional P2 read-error recovery risk. Disk store load precedes Lua and aborts activation on error; normal `get` reads memory. Deadline errors remain possible, but loss/recovery was not demonstrated. |
| PLUG-APP-007 | Withdrawn as current-host defect; mock-contract/test gap. Production backend converts non-completed results to errors before returning; mock `status` is not its contract.                                              |

PLUG-APP-009's P3 `pcall` result-handling behavior was also source-confirmed;
its false-return path was not executed. No IDs were renumbered or silently removed.

## Documentation-alignment follow-up

[Report 04](04-documentation-alignment.md) adds sampled static documentation/data-contract checks, not a renewed source audit or product execution.

| ID           | Qualified priority and verified fact                                                                                                                                                                                                                                                        |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| PLUG-DOC-001 | P2: schema calls `manifest_hash` canonical H-B while the procedure and sampled Activity record use raw manifest bytes. Corroborates PLUG-REG-006's documentation portion, not a second runtime defect or publisher-authentication proof. No digest-to-uninitialized-pin comparison was run. |
| PLUG-DOC-002 | P2: extraction gates still cite a draft Panel Runtime pre-study although the generic RFC is accepted. Public plugin registration/mount surfaces remain unresolved; generic acceptance does not deliver independent panel presentation.                                                      |
| PLUG-DOC-003 | P2: accepted Git-tool docs cite removed bundled owners rather than the current host authorizer. The sampled dispatch seam does not prove the complete installation-time tools contract.                                                                                                     |

Shared routing and research 032/038 capture drift remain TERM-DOC-003/005 in
[Terminal 08](../bitty-terminal/08-documentation-alignment.md), not duplicate
plugin findings. Report 04 supersedes selected September 16 split/capture claims:
independent packages and candidate provider/history documents exist, but deferred
presentation and future features are not delivered evidence. Advisory integrity
and mock trust do not prove a live signing bypass; acknowledged resolver/SDK/range
convergence remains a cross-owner contract question. Existing corrected SDK,
timer, recovery, and mock dispositions above still apply.

## Verification scope and revision drift

Read workspace/research guidance and the scoped SDK, template, registry, docs,
Activity, Statusline, Git Panel, File Manager, and host AGENTS; loaded
`ctxctl-core`. MCP outlines rejected workspace paths; repo-local `ctxctl outline`
worked, followed by targeted source slices. No nested agent or CarryCtx was used.

- SDK advanced from first-pass `3e9ebb5` to
  `84d41ae96d6b733fcc57be6274935cef12c6590b`: one dependency-only commit changing
  TypeScript 5.7.2 to 7.0.2 in `package.json`/`bun.lock`. Reviewed source bodies
  are unchanged; prior typecheck/test success does not validate the new pin.
- Host remains `06bc1f45995fd297a3f7324688bc81b0598ecf67`; registry remains
  `95bb0bf610885e7d4464cc3056d9318c82c3e4d7`; docs remain
  `9fdbcd02af835d23868a7c15c5ce12f1efcd093a`. Template and all six plugin HEADs
  match the report baselines. Source trees are clean except the pre-existing
  untracked host `.targets/`; research already had the untracked campaign.
- Registry has eight uninitialized gitlinks and only Wheel initialized. Pins
  were read, not fetched or checked against all file bodies. Mounted docs pin
  is 34 commits behind the consulted sibling docs. Sibling HEAD is not pin proof.
- Docs evidence pages explicitly disclaim independently verified/shipped
  compatibility. Git Panel's older host-enforcement status does not override
  current `HostToolsAuthorizer` and spawn-error implementation.

This pass traced entry selection, Activity load/save/purge/timers, selected SDK
mock behavior, registry normalization/metadata/trust display, local package
install/update/index/retention, and the Git spawn result/cwd seam. It was not a
second full audit of every file listed by the first-pass reviewers.

## Real coverage and test limits

**No product tests ran in this second pass.** First-pass reports record 53 selected
SDK/template tests, 82 registry tests, and 806 Lua assertions, plus their own
blocked gates. Those are distinct units and historical executions, not a combined
coverage percentage or integration proof. No generated package was activated;
no timer scheduler, transient-store recovery, fault/race, real Git subprocess,
UI/browser, deployed store, signature chain, or cross-platform behavior was run.

Not independently revalidated in full: **PLUG-SDK-007, PLUG-SDK-009–012;
PLUG-REG-004, PLUG-REG-012–014; PLUG-APP-006, PLUG-APP-010**. Historical review
matrices, most tests, optimizations, supply-chain/dependency contents, host
capability enforcement, and remaining first-pass risk sections are not blanket
endorsed. Palette/Wheel behavior was not retraced; deliberate stubs and planned
features were not promoted to defects.

Registry integrity is advisory. Raw manifest hashes are not canonical H-B or
publisher authentication; the generator emits unsigned/unverified, not verified.
The package trust model is a deliberate non-cryptographic stub, and the actual
local CLI rejects remote sources. No end-to-end registry authentication exploit
is established. Security observations remain risk/remediation only, with no
payloads, exploit execution, or reproduction code.

Report 04 uses the same registry/host/plugin-docs baselines above and terminal
docs `7947fb3`. It samples registry declarations, Activity metadata, documentation,
and host catalog/spawn/trust seams; SDK/template/independent-plugin bodies were
not rereviewed. No product tests, pin-hash comparison, signature-chain check,
storefront execution, or full per-plugin documentation audit was performed.

## Files and checks

The original second pass authored only `01-sdk-and-template.md`,
`02-registry-and-distribution.md`, `03-independent-plugins.md`, and this `README.md`.
Report 04 was authored separately. This indexing follow-up edits only the campaign
and three product READMEs; all reports and the root research index remain untouched.
No source edits, commits, installs, network access, generated artifacts, or
workflow state changes. Report tests and coverage prose were corrected rather
than running suites that could write outside the allowed scope.

Installed `markdownlint-cli2` 0.23.1 is the Markdown-only quality gate; it is run
from `research` with `--no-globs`, without fixes or network. The original pass
used its four owned paths; this follow-up targets only the campaign and three
product READMEs. Final lint and read-back results are reported at completion;
Markdown lint is not product-test or implementation evidence.
