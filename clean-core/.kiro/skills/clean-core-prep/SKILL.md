---
name: clean-core-prep
description: "Clean Core remediation: stage a workspace for the clean-core-fix skill. Verifies the assess output exists and the KB is populated, runs `abapctl clean-core prep` to download source + baseline + enriched findings, resolves the transport (flag, reuse, or prompt), and prints prep-summary.md. Pure orchestration (no successor lookups, no fix planning, no schemas). Decisions are owned by the per-object clean-core-remediator agent. Use when the user has run an assess and wants to stage objects for fixing."
---

# clean-core-prep: stage workspace for fixing

Reads assess output, runs `abapctl clean-core prep` to download source + baseline + enriched findings, resolves a transport, and prints a summary the operator can read before invoking the clean-core-fix skill.

This skill **does no successor lookups, no classification, no fix planning**. Those are the remediator agent's job. Prep's role is purely mechanical: verify preconditions, invoke the CLI, prompt for transport, summarise.


## Inputs

- `<SID>/<PACKAGE>`: e.g. `S4H/ZFINANCE`. SID must match the connection's SID.
- `-c <connection>`: abapctl connection profile.
- `--transport <nr>`: pin transport. If omitted, skill reuses (from prior `manifest.json.transport`) or prompts.
- `--force`: delete existing `fix-context/` and re-prep from scratch. Operator-only escape hatch.

## Prerequisites

- `clean-core/<SID>/<PACKAGE>/reports/SUMMARY.md` exists (run the clean-core-assess skill first).
- `.abapctl/reference/cc-kb/` exists with `.md` files (the remediator reads patterns from here; ships with this kit; copy it in per the install step).
- `abapctl` on PATH; connection's SID matches `<SID>`.

## Side effects

- Calls `abapctl clean-core prep` (downloads source + baseline + writes findings.json, all read-only against SAP).
- Calls `abapctl transport get <nr>` to validate a reused or prompted transport.
- Stamps the resolved transport into `fix-context/manifest.json` (adds top-level `transport` field) so subsequent invocations don't re-prompt and the remediator reads from a single source.
- Writes `fix-context/prep-summary.md` for the operator.
- May prompt operator to create a transport. **Never creates one autonomously.**

No source writes to SAP.

## Steps

### Step 1: Verify assess output

```bash
test -f clean-core/${SID}/${PACKAGE}/reports/SUMMARY.md
```

If missing:

> Run the clean-core-assess skill on `${PACKAGE}` (connection `${CONNECTION}`) first.

Exit.

### Step 2: Verify KB

`abapctl clean-core prep` requires `.abapctl/reference/cc-kb/` to contain at least one `.md` file. Pre-check upfront, since its native error is cryptic:

```bash
if ! ls .abapctl/reference/cc-kb/*.md >/dev/null 2>&1; then
  echo "ERROR: .abapctl/reference/cc-kb/ is missing or has no .md files."
  echo "Install the cc-kb that ships with this kit:"
  echo "    mkdir -p .abapctl/reference/cc-kb && cp -r <kit>/cc-kb/* .abapctl/reference/cc-kb/"
  echo "(abapctl reference update fetches SAP's JSON catalog, NOT the cc-kb markdown.)"
  exit 1
fi
```

### Step 3: Run `abapctl clean-core prep` (idempotent)

```bash
test -d clean-core/${SID}/${PACKAGE}/fix-context && [ -z "${FORCE}" ] || \
  abapctl clean-core prep ${SID}/${PACKAGE} -c ${CONNECTION}
```

If `--force`, delete `fix-context/` first (preserve a copy of the prior `manifest.json.transport` value if present; re-validate later).

The CLI writes:
- `fix-context/<obj>.abap`: main source for each D/C object
- `fix-context/.baseline/<obj>.abap`: read-only rollback copy
- `fix-context/.context/<obj>.findings.json`: findings, **already enriched with sap_github successors and any ATC-tagged successors** (this happened at assess time)
- `fix-context/manifest.json`: prep manifest with kbDir + paths + sub-includes (this skill stamps `transport` into it in Step 4)

Sub-include files (`{fg}.fugr.{fm}.abap`, `{cls}.clas.{slot}.abap`) are written too, when findings reference them.

### Step 4: Resolve transport, stamp into manifest.json

Order:

1. **`--transport <nr>` flag**: use it. Validate via `abapctl transport get <nr> -c ${CONNECTION}`. If closed/missing → tell operator, exit.
2. **`manifest.json` already has `transport`** (prior prep run): validate with `abapctl transport get`. If still open → use. If closed → tell operator the prior transport closed and prompt for new one.
3. **No flag, no prior value**: prompt:

   > No transport specified for `${PACKAGE}`. Create one with:
   >
   >     abapctl transport create --package ${PACKAGE} --description "Clean Core ${PACKAGE}" -c ${CONNECTION}
   >
   > Paste the transport number here, or re-invoke with `--transport <nr>`.

   Never create autonomously. Operator owns transport lifecycle.

Once resolved, stamp it into `fix-context/manifest.json` by adding/updating a top-level `transport` field. Atomic write: read manifest → set `transport: "<TR>"` → write to `manifest.json.tmp` → rename. The remediator reads this field directly (no separate `.transport` sidecar).

### Step 5: Write `prep-summary.md`

Write `clean-core/${SID}/${PACKAGE}/fix-context/prep-summary.md`:

```markdown
# Prep Summary: ${PACKAGE} (${SID})

Generated: <ISO timestamp>
Transport: <TR>  (validated open via `abapctl transport get`)
KB: .abapctl/reference/cc-kb/  (<N> files)

## Objects ready to fix

| Level | Count | Total findings |
|-------|-------|----------------|
| D     | <n>   | <sum>          |
| C     | <n>   | <sum>          |

## Findings enrichment (assess-time)

| Source                        | Findings with successor |
|-------------------------------|------------------------|
| sap_github (static catalog)   | <n>                    |
| atc_finding_tag (ATC inline)  | <n>                    |
| **Residual (no successor)**   | **<n>**                |

The remediator will look up residuals at fix time, via `abapctl release-state lookup` (a thin wrapper over the live `I_APIsWithCloudDevSuccessor` CDS view) or KB research. Residual count is informational, not a blocker.

## Workspace

- Source:    fix-context/<obj>.abap          (<N> files)
- Baseline:  fix-context/.baseline/<obj>.abap (read-only — remediator never edits)
- Findings:  fix-context/.context/<obj>.findings.json
- Manifest:  fix-context/manifest.json

## Next

Dry-run first to see exactly what each clean-core-remediator would do, then execute (invoke the clean-core-fix skill):

    Dry-run:  clean-core-fix on ${SID}/${PACKAGE} with --dry-run
    Execute:  clean-core-fix on ${SID}/${PACKAGE}
```

To compute the table values:

- **Level + count + total findings**: read `clean-core/<SID>/<PKG>/state/progress.json` (already there from assess), grouping by `level`, count + sum of findings per object.
- **Enrichment counts**: walk `fix-context/.context/*.findings.json`. For each finding:
  - has `successor` populated → count under `sap_github` (assess put it there from the static catalog)
  - has inline ATC tag with successor name → count under `atc_finding_tag`
  - neither → count as residual
  - These three buckets are disjoint per finding.

No file writes beyond `prep-summary.md` and the `transport` field added to `manifest.json` in Step 4.

### Step 6: Print final terminal summary

```
clean-core-prep ${SID}/${PACKAGE} complete.

  D objects: <n>  (<x> findings)
  C objects: <n>  (<x> findings)
  Residuals (no assess-time successor): <n>  — remediator will look up live
  Transport: <nr>

Review fix-context/prep-summary.md, then invoke clean-core-fix on ${SID}/${PACKAGE} (start with --dry-run).
```

## What this skill does NOT do

- Does **not** classify findings, pick successors, or plan fixes. Those are the remediator's per-object decisions
- Does **not** query the successor catalog (`I_APIsWithCloudDevSuccessor`). The remediator does that live at fix time via `abapctl release-state lookup`
- Does **not** estimate target levels. The remediator reports the actual outcome in result.json
- Does **not** create transports. Operator owns lifecycle
- Does **not** modify SAP source

## CLI reference

When you need `abapctl` syntax, consult the `sap-abap` skill (`.kiro/skills/sap-abap/`). Examples used here:

- `abapctl clean-core prep <SID>/<PKG> -c <conn>`: the workhorse
- `abapctl transport get <nr> -c <conn>`: validate transport status
- `abapctl transport create --package <pkg> --description <text> -c <conn>`: operator runs this; skill never does
