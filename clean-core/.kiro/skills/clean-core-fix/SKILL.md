---
name: clean-core-fix
description: "Clean Core remediation: spawn one clean-core-remediator agent per object in sealed context, in batches of N (default 3, max 4). Read each per-object result.json, aggregate counts, circuit-break on persistent failures, print run summary. Each object's findings.json is the per-object todo and the remediator decides per finding. Use when the user has run clean-core-prep and wants to apply fixes to ABAP objects."
---

# clean-core-fix: orchestrate per-object remediation

Spawn one `clean-core-remediator` agent per ABAP object that prep staged. Each remediator runs sealed (its own context, its own findings.json), decides per finding, and writes a single `result.json` per object. This skill batches spawns, aggregates results, and circuit-breaks if SAP looks unreachable.

This skill makes **no SAP decisions**. It does not classify findings, pick successors, or rewrite source. All of that is the remediator's job.

## Inputs

- `<SID>/<PACKAGE>`: must match what the clean-core-prep skill staged.
- `-c <connection>`: abapctl connection profile.
- `--parallel N`: concurrent remediator agents (default `3`, **max 4**; Kiro's `use_subagent` hard-caps at 4 simultaneous subagents). Each runs in its own sealed context.
- `--only OBJ1,OBJ2`: restrict to these objects.
- `--skip OBJ1,OBJ2`: exclude these objects.
- `--dry-run`: print the queue, do not spawn.

## Prerequisites

- `clean-core/<SID>/<PACKAGE>/fix-context/manifest.json` exists (run the clean-core-prep skill first).
- Manifest has top-level `transport` field stamped (prep does this in its Step 4). If missing → exit, tell operator to re-run prep with `--transport <nr>`.
- `.abapctl/reference/cc-kb/` populated (prep verified this; we re-check defensively).
- `abapctl` on PATH; cwd = `PROJECT_ROOT` (directory holding `.abapctl.json`). Remediator agents inherit this and rely on it.
- `clean-core-remediator` agent installed at `.kiro/agents/clean-core-remediator.json`.
- **Subagents enabled**: `kiro-cli settings chat.enableSubagent true` (this skill spawns clean-core-remediator agents via Kiro's `use_subagent` tool, which is gated behind this setting). The running agent needs `use_subagent` in its `tools`. The built-in `kiro_default` agent has it, and subagent spawning is allow-all by default (no `availableAgents` allowlist required). If you run a custom agent, ensure it lists `use_subagent`.

## Side effects

- Spawns `clean-core-remediator` agents. Each one mutates one object's `findings.json` (inline `fix:` blocks) and writes `<obj>.result.json`. Each remediator pushes source to SAP under the resolved transport, gates with syntax/ATC/unit, rolls back on regression.
- This skill writes **only** `clean-core/<SID>/<PACKAGE>/fix-context/RUN.md` (run summary) and prints a terminal summary. **Never synthesises or edits a result.json.**

## Steps

### Step 1: Validate

```bash
cd ${PROJECT_ROOT}
test -f clean-core/${SID}/${PACKAGE}/fix-context/manifest.json
```

Read manifest. Confirm `transport` field present and non-empty. If missing:

> Manifest has no transport. Run the clean-core-prep skill on `${SID}/${PACKAGE}` with `--transport <nr>` first.

Exit.

### Step 2: Build object queue

Iterate `manifest.objects[]`. Each entry carries `name`, `type`, `sourceFile`, `baselineFile`, `contextFile`, `findingsFile`. These are the file paths the remediator will read. Abapctl writes filenames sanitized + lowercased (e.g. `ZPAYMENTS` → `zpayments.prog.abap`, `/COMPANY/ZCL_X` → `(company)zcl_x.clas.abap`); the running agent must pass these resolved paths into the spawn prompt rather than letting the remediator derive them from `${OBJECT}`.

For each candidate:
1. Apply `--only` / `--skip` filters against `objects[].name`.
2. Read `${FIX_CONTEXT_DIR}/${findingsFile}`; count `fix.status` distribution:
   - Every item has a terminal `fix.status` (`fixed`, `needs_review`, or `failed`). **Already-processed**: skip silently unless operator explicitly listed it via `--only` (then re-queue; the remediator re-claims `needs_review`/`failed` items per its Step 0 rules).
   - Any item with no `fix` block, or `fix.status=in_progress`, or status absent. **Queue**.
3. Capture `OBJECT_TYPE` from the manifest entry's `type`.
4. Compute `RESULT_FILE` as `dirname(findingsFile)/${basename(findingsFile, '.findings.json')}.result.json` (sibling of findings, lowercased name preserved).

Order: alphabetical by object name. (Objects are independent: no inter-object ordering is needed.)

### Step 3: Dry-run (if `--dry-run`)

Print:

```
Queue (N objects):
  ZCANCEL_BILLING  (PROG/P)  14 findings  status: 14 pending
  ZPAYMENTS        (PROG/P)   3 findings  status: 2 fixed, 1 needs_review (re-queue)
  ...
Skipped (already complete): N
Skipped (--skip):          N
```

Exit. No spawns.

### Step 4: Execute in batches

Process the queue in chunks of `--parallel` (default 3, max 4). For each batch:

1. **Spawn N `clean-core-remediator` agents in parallel via the `use_subagent` tool**. One `use_subagent` call per object, all in a single batch so they run concurrently (Kiro runs up to 4 simultaneously and blocks until the batch finishes). For each, set:
   - `agent_name`: `"clean-core-remediator"` (the named config in `.kiro/agents/`)
   - `relevant_context`: the per-object variable block (below); this scopes the remediator to its one object
   - `query`: the literal instruction line (below)

   The `relevant_context` must contain, verbatim:

   ```
   WORKING DIRECTORY: Before running any abapctl command, cd ${PROJECT_ROOT} (the directory containing .abapctl.json). abapctl resolves connection config from cwd; a sub-directory cwd produces "connection not configured" errors that look like SAP outages. Do not rely on inherited cwd.

   OBJECT=${OBJECT}
   OBJECT_TYPE=${OBJECT_TYPE}
   SID=${SID}
   PACKAGE=${PACKAGE}
   CONNECTION=${CONNECTION}
   FIX_CONTEXT_DIR=clean-core/${SID}/${PACKAGE}/fix-context
   PROJECT_ROOT=${PROJECT_ROOT}
   SOURCE_FILE=${objects[i].sourceFile}
   BASELINE_FILE=${objects[i].baselineFile}
   FINDINGS_FILE=${objects[i].findingsFile}
   CONTEXT_FILE=${objects[i].contextFile}
   RESULT_FILE=${RESULT_FILE}
   ```

   The `query` is: `Read FIX_CONTEXT_DIR/manifest.json, FIX_CONTEXT_DIR/${FINDINGS_FILE}, and FIX_CONTEXT_DIR/${CONTEXT_FILE}. Follow your agent definition end-to-end.`

   The five file-path variables come from the manifest entry for this object; do **not** synthesize them from `${OBJECT}` (abapctl sanitizes + lowercases filenames, including encoding namespaces as `(NS)NAME`).

   `${PROJECT_ROOT}` is the directory containing `.abapctl.json` discovered by walking up from cwd. If not found → abort the run before spawning.

2. **Wait for the batch.** Each remediator prints one terminal line and exits with code 0 (success/partial) or 1 (rolled_back/failed).

3. **Read each object's `RESULT_FILE`** (the path passed in the spawn prompt, sibling of findings.json, lowercased), the authoritative outcome. If a remediator exited without writing a compliant result.json:
   - First failure for this object this run → re-spawn once (fresh context, same prompt; deletes `.abapctl/.session.json` first as a hygiene step).
   - Second failure → record the object as `failed/sap_unreachable` in the run summary. **Do not synthesize result.json.**

4. **Print one-line status per object** to the main conversation:

   ```
   [12/45] ZCANCEL_BILLING: success (fixed=14, needs_review=0, failed=0) C→B
   [13/45] ZPAYMENTS:       partial (fixed=11, needs_review=3, failed=0) C→C
   [14/45] ZBROKEN:         rolled_back (fixed=0, needs_review=0, failed=8) C→C
   ```

5. **Circuit-break check**. Track outcomes across the *whole run* (not just this batch):
   - If the last 3 results are `failed` outcome with `failure_category=sap_unreachable` → SAP is likely down. **Stop the run.** Do not spawn the next batch. Print:

     ```
     Circuit-break: 3 consecutive sap_unreachable results. Aborting before next batch.
     Investigate connectivity, then re-run clean-core-fix to resume.
     ```

   - Other patterns (mix of `partial`, `rolled_back`, `failed/gate_regression`, etc.) → continue. Individual gate failures are object-specific signals, not infrastructure ones.

### Step 5: Aggregate + write `RUN.md`

After the loop completes (normally or via circuit-break), tally outcomes by re-reading every object's `RESULT_FILE` (each one already known from Step 2's queue):

```
clean-core/${SID}/${PACKAGE}/fix-context/RUN.md
```

```markdown
# Run: ${SID}/${PACKAGE}

Started: <ISO>
Finished: <ISO>
Transport: <TR>
Connection: <conn>
Parallelism: <N>

## Outcomes

| Outcome      | Count |
|--------------|-------|
| success      | N     |
| partial      | N     |
| rolled_back  | N     |
| failed       | N     |
| (skipped: already complete) | N |

## Findings totals

|              | Total | Fixed | Needs review | Failed |
|--------------|-------|-------|--------------|--------|
| All objects  | N     | N     | N            | N      |

## Per-object

| Object | Outcome | level | fixed | needs_review | failed | operator_action |
|--------|---------|-------|-------|--------------|--------|-----------------|
| ZX     | success | C→B   | 14    | 0            | 0      | (none)          |
| ZY     | partial | C→C   | 11    | 3            | 0      | Review 3 items in findings.json |
```

`level` shown as `before→after` from each result.json's `level_before`/`level_after`.

### Step 6: Print terminal summary

```
clean-core-fix ${SID}/${PACKAGE} complete.

  success:     N
  partial:     N
  rolled_back: N
  failed:      N
  skipped:     N (already complete)

  Findings: <total> total, <fixed> fixed, <needs_review> needs review, <failed> failed.
  Transport: <TR>

Review per-object outcomes in fix-context/RUN.md and individual result.json files.
Then invoke the clean-core-review skill on ${SID}/${PACKAGE} for narrative + exemption suggestions.
```

Exit code: `0` if any object succeeded or partially succeeded; `1` if every queued object came back `failed` or `rolled_back`; `2` if validation failed before spawning any remediator (missing manifest, missing transport, project root lookup failed).

## What this skill does NOT do

- Does **not** read or write findings.json (only the remediator mutates `fix:` blocks).
- Does **not** read or write result.json (only the remediator writes them; this skill re-reads them to aggregate).
- Does **not** classify findings or pick successors. That's the remediator's per-object job.
- Does **not** rerun ATC at package level. Each remediator gates at object level. Package-level reassessment is the clean-core-assess skill again.
- Does **not** create or release transports. Operator owns lifecycle.
- Does **not** mutate manifest.json. Prep stamped the transport; this skill only reads.
- Does **not** pass pre-baked source content into retry spawns. Every spawn is a clean clean-core-remediator reading the on-disk artifacts itself.

## Operator controls

- **Resume**: re-invoke the same command. Step 2 skips objects whose findings have only terminal `fix.status` values. Re-queue an object via `--only OBJ` (the remediator re-claims `needs_review`/`failed` per its rules).
- **Force re-run a "complete" object**: pass `--only OBJ`. The remediator re-cascades and re-probes; a successful re-run can move `needs_review` items to `fixed`.
- **Pause**: Ctrl+C between batches stops cleanly. In-flight remediator agents will finish their current operation; this skill exits before the next batch.
- **Tune parallelism**: lower `--parallel` if SAP rate-limits or sessions thrash; raise on idle systems with high churn. Default 3 is safe across S4H/S4J/A4H.

## CLI reference

When you need `abapctl` syntax, consult the `sap-abap` skill (`.kiro/skills/sap-abap/`). This skill itself does not call `abapctl` directly. Only the remediator agents do.
