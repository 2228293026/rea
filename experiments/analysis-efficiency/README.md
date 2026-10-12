# Analysis efficiency measurements

Tracks the experiments for [#2016](https://github.com/morluto/rea/issues/2016).
The production CLI/MCP contracts and defaults remain the compatibility baseline.
Downloaded artifacts, runtime copies, exported Evidence, transcripts and raw
measurements belong outside the checkout.

## Reproduce a measurement

Use the repository's pinned Node/npm and locked dependencies. Build each selected
revision, then copy its `dist/`, `bridge/`, `scripts/`, `skills/`, `package.json`
and `package-lock.json` to a separate runtime directory before editing production
sources. That copy needs the matching locked dependencies. The initial frozen
baseline is `42b0ed126df7d5b908ac369715378d5e519051b7`; this study does not
silently substitute a newer `main` or the previous Code Mode experiment.

The public artifacts and acquisition digests are in `corpus.json`. Supply their
actual local paths and an optional JSON object of explicit provider environment
settings. Do not add credentials to that file. Provider settings are inherited
by the child, but the report persists only the selected settings' digest. It
does not serialize the ambient environment.

```sh
export NODE_OPTIONS=--max-old-space-size=512
export UV_THREADPOOL_SIZE=1
export GOMAXPROCS=1
export GOMEMLIMIT=2GiB
python3 experiments/analysis-efficiency/resource-run.py -- \
  node experiments/analysis-efficiency/measure.mjs \
  --runtime-root /absolute/frozen-runtime \
  --revision 42b0ed126df7d5b908ac369715378d5e519051b7 \
  --artifact obsidian.asar --path /absolute/obsidian-1.14.4.asar \
  --providers /absolute/selected-providers.json \
  --output /absolute/new-measurement-directory
```

The wrapper requires Linux cgroup v2 and a working user systemd manager. It owns
a unique transient scope, restricts the whole process tree to one CPU, caps
memory at 4 GiB with no swap and 192 tasks, and samples memory/tasks/CPU every
100 ms. It stops its own scope above 3 GiB or below 4 GiB host available memory;
startup requires 8 GiB available. Cleanup failures retain the owned scope's
identity. Run heavy builds, providers and model trials serially. Sampled peaks
can miss short spikes; cgroup limit events are separately reported.

An actual Codex + Chromium trial exceeded the initial 192-task allowance and
was stopped automatically below 1 GiB sampled memory, with no OOM. The isolated
browser model retry uses `resource-run.py --task-ceiling 256 -- COMMAND...` to
account for both process trees; memory, swap and CPU controls are unchanged.
Keep the original aborted run and cleanup receipt. This is experiment capacity,
not a REA defect or a successful browser analysis.

## What the numbers mean

`summary.json` records the selected artifact's before/after SHA-256, runtime
file-table digest, dispatcher and lockfile digests, runner source digest,
Node/SDK versions, catalog sizes, caller arguments and producer Evidence IDs.
`runtime-files.json` identifies the exact compiled files measured. Actual
provider settings and runtime files must stay fixed during a comparison.

Every tool call measures three intervals at the public SDK Transport boundary:

- Client work before `send`, including first-use schema preparation.
- Transport/server roundtrip, including framing, I/O, server work and client
  JSON parsing. This interval is **not** a Ghidra-only or provider-only timer.
- Client work after the parsed response reaches `onmessage`.

The SDK call's total excludes the runner's subsequent encoding and disk writes.
Result byte counts measure compact JSON of the parsed client result, not raw
wire bytes or tokens. Progress notifications retain their actual arrival times;
they do not imply more precise internal engine timings than the producer emits.
Scope CPU time and aggregate process-tree memory come from the wrapper's final
JSON line. Fresh processes have cold application caches; host page caches are
not globally flushed.

`--delivery summary` is an explicit candidate workflow, initially for artifact
inventory. It must be supported by the selected server's advertised schema.
The default `complete` runs the existing calls unchanged. Compare the same
analyst facts and complete source Evidence before crediting a smaller response.
Projection can reduce delivery bytes without reducing upstream analysis time.
For SQLite WASM, summary delivery reads the completed retained result through
the summary and exports views after preserving the original transport error.
`--javascript-question selected` analyzes only the exact Obsidian sources
`package.json` and `main.js`, keeps inventory identity, and reads each observed
`main.js` module by its exact node ID. Package metadata is not a module selector.

Compare paired runs with the same runner source and provider settings:

```sh
node experiments/analysis-efficiency/compare.mjs \
  --left /absolute/complete-run --right /absolute/summary-run \
  --output /absolute/new-comparison.json
```

This checks all selected analyst facts, including inventory source IDs, exact
path records, manifest, counts, completeness, contradictions, coverage,
unexplored branches and limitations. It excludes only delivery observation IDs
and the named recovery-path description. Different settings, changed facts or
failed workflows make the comparator fail. Binwalk's per-invocation region UUIDs
are separately preserved with both original values and JSON paths; only those
opaque lookup values are excluded from physical-region fact equality. Original
producer replies are never rewritten. Selected JavaScript comparisons likewise
preserve both values of the two named parent-heap-dependent byte budgets before
excluding their values from fact equality; fixed limits and coverage rows remain
checked. Counts include extra view calls and
observed tool errors. Use repeated runs with alternating order before claiming
a distribution or a speedup; one pair is only one observation.

For a completed WASM result exceeding the MCP response budget, the runner keeps
the original error, exports retained Evidence through the public tool, and
authenticates the exact reported record using the selected runtime's parser.
The report names this recovery; it does not relabel the oversized inline reply
as successful delivery.

## Initial observations and exclusions

The frozen baseline has 2,326,609 bytes of complete catalog JSON, of which
494,207 bytes describe names, descriptions and input schemas. One Obsidian
trial spent about 21.9 s before the first request was sent, then about 0.77 s
in that request's transport/server roundtrip. Other cold runs differed
substantially. These single observations identify a measurement boundary;
they do not establish a speedup or a distribution.

The initial matrix exercised shipped Obsidian, NewPipe, ShareX, OpenWrt, SQLite
WASM and jq artifacts. SQLite produced a complete result larger than the
transport frame and requires an explicit recovery run. A first jq attempt hit
the SDK's default 60 s timeout because the experimental runner used a superseded
three-argument call signature. SDK 2.3.1 accepts request options as argument two;
the corrected jq run completed real Ghidra analysis in about 265 s. That earlier
driver timeout is excluded from REA regression claims. Provisional runs with
different runner source digests are not one paired statistical cohort.

No cross-session engine database cache, default analyzer changes, external
plugin loading, or broad plugin SDK is enabled by this experiment.

## Obsidian delivery checkpoint

The [recorded pairs](results/obsidian-20261012.json) compare frozen baseline
`42b0ed12` and candidate `f1106181` in complete → summary and summary → complete
order using the same runner source, bytes and provider settings. Both preserve
every selected fact and the atomic inventory Evidence ID.

| Measurement                                         |        Complete | Summary + exact path view |
| --------------------------------------------------- | --------------: | ------------------------: |
| Tool-result compact JSON, including lifecycle calls |   937,404 bytes |              16,322 bytes |
| Tool calls per question                             |               3 |                         4 |
| Complete catalog compact JSON                       | 2,326,609 bytes |           2,370,505 bytes |
| Workflow elapsed, first pair                        |         11.04 s |                   10.23 s |
| Workflow elapsed, second pair                       |         10.57 s |                   10.18 s |

Reply JSON falls by about 98.3%; catalog JSON grows by about 1.9%. The two elapsed
observations do not establish a statistical speedup. Inventory coverage and
upstream byte acquisition are unchanged. The candidate remains opt-in; existing
requests continue to deliver the complete inspection. Each run verifies the
selected artifact's before/after digest and reports no observed OOM or task
limit events. Source typing and projection improve selected delivery without
introducing an engine database cache or an external plugin protocol.

## Task-scoped analyzer experiment

[The recorded jq pair](results/jq-presets-20261012.json) explicitly disables
`Decompiler Switch Analysis` through the existing `REA_GHIDRA_DISABLED_ANALYZERS`
setting and compares it with the unchanged Ghidra default. Both use the same
frozen runtime and real jq bytes, one CPU, a fresh ephemeral database and the
same entry/main question. Completed main dossiers exceed the MCP frame in both
runs; existing retained native views recover the selected facts, preserving the
observed transport errors in the reports.

One disabled/default pair takes 174.77/276.23 seconds. Entry and main pseudocode,
main strings and its block count agree. However, callee `0x40daa0` has 604
recovered body bytes in eight ranges by default, and 164 bytes in four ranges
with the same 646-byte span after disabling switch analysis. Equal top-level
counts do not establish call-graph equivalence. **Keep the production default.**
Use the existing control only for a caller-selected task whose required facts
have been checked; no general fast preset is introduced. A separate real
coreutils/Ghidra probe rejects a nonexistent analyzer before bridge readiness,
preserves the named failure and closes the session. Source bytes remain unchanged.

`--schemas selected` is a separate SDK 2.3.1 client experiment. It passes the exact
live named `tools/list` definition to the SDK's public `callTool` option, retaining
output validation. It does not change REA or the default runner mode. Selected
definitions compile per call rather than preparing the entire cached catalog;
report this client effect separately from engine analysis or model behavior.

## Real Codex CLI trials

`agent-trial.py` accepts an explicit frozen runtime and revision, the same
question templates in `questions/`, and optional explicit provider settings.
It fingerprints the actual runtime, runs pinned local Codex CLI with a private
temporary profile, copies only the authentication file with mode 0600, disables
shell/web/multi-agent tools, and removes copied profile state afterward. Raw
events and answers remain outside Git. A zero exit code is not a factual grade.
Review the original tool calls, failed requests, result limitations and answer.

The optional `--enabled-tools FILE` supplies a list of exact names to the
temporary Codex MCP configuration's `enabled_tools`, as described in the
[official MCP configuration](https://learn.chatgpt.com/docs/extend/mcp?surface=cli).
It leaves the REA catalog and other clients intact. Initial Obsidian trials
showed parameter-correction attempts and different delivery choices even with
the same question; filtering alone has not established a cost or quality win.
Requested model aliases are recorded without inventing a resolved model identity.

The browser question uses `--authority browser` to permit only its explicitly
selected public scenario in a fresh, owned headless profile. Browser observations
and cleanup are graded separately from static artifact analysis.

## Real artifact compatibility checkpoint

[Six frozen baseline/pilot pairs](results/real-artifacts-20261012.json) exercise
Obsidian, NewPipe, ShareX, OpenWrt, SQLite WASM and published historical WETH code
through public MCP tools. All selected facts agree under the stated comparison
rules. Obsidian complete delivery and atomic inventory Evidence identity remain
unchanged. ShareX native-boundary claims remain scoped to the selected DLL;
NewPipe retains a null DEX descriptor rather than inventing one.

SQLite's complete 15,382,128-code-unit WAT exceeds the MCP response budget in both
revisions. The control recovers complete Evidence through file export; the pilot
reads summary and four exact export forms inline, without running WABT again.
Inline byte totals exclude file exports, so they do not establish a size win.
OpenWrt comparison preserves both sets of upstream-generated region UUIDs before
comparing offsets, extents, signatures, coverage and limitations. WETH is a
published historical example; a live RPC request returned HTTP 403, so this is
not live-chain verification. Every pair records unchanged target bytes and zero
observed OOM/task-limit events. The Linux x64 results do not establish behavior
for IDA, Hopper, macOS, Windows or Harmony.

## Same-question model and selected-source follow-up

[Two actual SQLite model trials](results/model-wasm-20261012.json) preserve all
12 requested facts. The question, developer instruction, provider settings and
tool allowlist hashes match. A description-only follow-up supplies the exact
summary kind and its fields. Calls change from 42 to 3, workflow time from
292.64 to 63.79 seconds, and reported input tokens from 304,698 to 74,387. The
follow-up reads no unnecessary WAT pages. Both keep the original completed
analysis's transport error and authenticate the retained source/view identities.
This is one stochastic model pair, not a statistical or general causal claim.

[The selected JavaScript pair](results/obsidian-selected-20261012.json) preserves
all 24 observed `main.js` module identities and the exact two-source selection.
The strict comparison initially differs on parent budget values derived from
observed heap availability; both values and paths remain recorded separately.
Other coverage facts, per-family rows, fixed worker limits, omitted/unknown
counts, statistics and Electron observations agree. Coverage remains partial,
with 36 unselected relevant files and 384 semantic unknowns. These static facts
do not establish runtime behavior. The comparator does not rewrite Evidence.

[Ten real CLI trial reviews](results/model-domains-20261012.json) cover native,
managed, Android, firmware, Electron/JavaScript, WASM, EVM and browser tasks.
The selected JavaScript model preserves core facts and partial coverage but
omits full budget/family provenance. The browser producer captures complete
DOM/accessibility/URL evidence and terminates its owned process; direct model
answers lose two accessibility metadata fields to client context clipping and
report them unknown. A requested under-development Codex code-mode configuration
still produces a direct MCP call in the actual trace, so no executor/composition
win is credited. Profile cleanup, original errors, actual calls, usage, facts and
scope limits remain independently reviewed. Enabling a configuration flag does
not prove that a capability was used.

Discovered REA defects are recorded separately for team repair at the user's
request. This follow-up performs no additional REA bug fixes.

## Nonempty managed boundaries and final verification

[A second shipped managed library](results/sharex-helpers-20261012.json) checks
11 ModuleRefs, 147 PInvoke declarations and 225 native implementations from
ShareX HelpersLib 19.0.2. All complete normalized artifact and native-boundary
fields match between baseline and pilot, including source Evidence identities.
This covers nonempty declarations beyond the main ShareX DLL's empty import
tables. It does not verify native exports, execute the assembly or resolve CLR
dependencies.

[Verification receipts](results/verification-20261012.json) include 42 real
WABT CLI/MCP cases and the complete real SQLite CLI JSON: the unchanged default
returns the same original source Evidence ID, normalized result and raw result
as the baseline exported MCP record. Source bytes are unchanged. The seven-file
contract/composition run passes 51/52 assertions and hits the existing 20-second
presentation-test deadline while sharing one CPU. That same unmodified assertion
passes alone in 10.835 seconds. The original combined failure and isolated pass
remain recorded; no test deadline or REA bug was patched. Build, type/lint/module
checks and generated-document checks pass. No full `check:pr` or unverified
provider/platform coverage is implied.

## Actual old Evidence migration

[The migration receipt](results/migration-20261012.json) imports the original
baseline SQLite Evidence bundle into the pilot runtime, reads a typed summary
and four export forms, and exports the retained bundle again. The complete old
WASM record is deeply equal after import/export, with its original authenticated
Evidence ID. The original baseline Obsidian complete inspection also works as
inline input to the new inventory summary, retaining the old parent and atomic
inventory identities and manifest. Original input files remain byte-identical.

The runtime explicitly selects an absent WABT directory and requests no producer
operation. These views consume existing Evidence without another provider run;
this is portable-result reuse through existing import/export, not an engine
database cache. The five-call workflow completes in 5.458 seconds with a sampled
aggregate peak of 700,542,976 bytes and no observed limit or OOM events.

To repeat it with genuine old public outputs:

```sh
python3 experiments/analysis-efficiency/resource-run.py -- \
  node experiments/analysis-efficiency/verify-migration.mjs \
  --runtime-root /absolute/frozen-pilot-runtime \
  --revision 259e99dd906290189217cf0fb1a19769bcf87101 \
  --bundle /absolute/baseline-exported-evidence.json \
  --inspection-reply /absolute/baseline-complete-inspection-reply.json \
  --output /absolute/new-migration-directory
```
