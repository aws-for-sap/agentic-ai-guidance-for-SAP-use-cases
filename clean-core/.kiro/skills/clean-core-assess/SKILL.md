---
name: clean-core-assess
description: "Clean Core remediation: assess a package's ABAP objects against an ATC clean-core variant (default `CLEAN_CORE`; override with -v). Runs `abapctl clean-core assess`, parses level distribution (A/B/C/D), flags top remediation targets. Read-only. First step of the remediation workflow. Use when the user asks to assess, classify, or measure clean-core posture of an ABAP package."
---

# clean-core-assess: run ATC, classify levels

Runs the Clean Core assessment against an ABAP package. Read-only. Produces the ATC output that `clean-core-prep` consumes.

## Inputs

- `<PACKAGE>` (e.g., `ZDPR_DATAGEN`)
- `-c <connection>` (abapctl connection name; SID derived from connection)
- `-v <variant>`: ATC check variant. **Optional.** Omit to use the default (`clean_core.atc_variant` in `.abapctl.json`, ships as `CLEAN_CORE`). Pass `-v <name>` only when the user names a specific variant (e.g. a customer-defined `Z_CLEAN_CORE`). List available variants with `abapctl check atc-variants`.
- `--force`: re-run assess even if output already exists

## Prerequisites

- `abapctl` on PATH
- Connection reachable
- A connection profile defined in `.abapctl.json`

## Steps

### Step 1: Resolve SID + check for existing assessment

Resolve the SID for the given connection. `abapctl config show --json` returns the config object; pick the SID from the specific connection you're using:

```bash
SID=$(abapctl config show --json | jq -r ".connections.\"${CONNECTION}\".sid")
```

If `SID` is empty or `null`, the connection isn't configured. Print an error and exit.

Check whether an assessment already exists:

```bash
EXISTING="clean-core/${SID}/${PACKAGE}/reports/SUMMARY.md"
if [[ -f "${EXISTING}" && -z "${FORCE}" ]]; then
  echo "Assessment exists at clean-core/${SID}/${PACKAGE}/. Use --force to re-run."
  # Jump to Step 3 (summarize the existing output).
  SKIP_ASSESS=1
fi
```

### Step 2: Run assess (if SKIP_ASSESS is unset)

```bash
abapctl clean-core assess ${PACKAGE} -c ${CONNECTION}
# If the user named a specific variant, append it:
#   abapctl clean-core assess ${PACKAGE} -c ${CONNECTION} -v ${VARIANT}
```

This runs ATC against every object in the package (including sub-package objects) under the resolved variant (`-v` if given, else `clean_core.atc_variant` from config, else the built-in `CLEAN_CORE`), classifies each into A/B/C/D, and writes output to `clean-core/${SID}/${PACKAGE}/`.

Expected duration: ~3–10 min per 100 objects depending on system load.

### Step 3: Summarize output

Read `clean-core/${SID}/${PACKAGE}/reports/SUMMARY.md`. Parse:
- Level distribution (counts per A/B/C/D)
- Total findings by priority (P1/P2/P3)
- Top 5 objects by finding count
- Top 5 APIs referenced (from "Most Referenced APIs" section)

Print to main conversation:

```
Package: ${PACKAGE} (${SID})
Objects: ${N} assessed

Clean Core Level Distribution:
  A (fully clean):     ${Na} (${pct}%)
  B (pragmatic):       ${Nb} (${pct}%)
  C (plan migration):  ${Nc} (${pct}%)  — target for clean-core-prep → fix
  D (blocked):         ${Nd} (${pct}%)  — cloud migration blockers

Findings: ${P1} P1 / ${P2} P2 / ${P3} P3

Top C/D Objects by Finding Count:
  ZEXAMPLE_HEAVY     (FUGR/F, D): 78 findings
  ZEXAMPLE_REPORT    (PROG/P, D): 45 findings
  ...

Top Referenced APIs (legacy):
  <SAP_TABLE>   (55×) → <released CDS successor, if any>
  <LEGACY_FM>   (34×) → <released class/API successor, if any>
  ...

Output: clean-core/${SID}/${PACKAGE}/

Next: invoke the clean-core-prep skill on ${SID}/${PACKAGE} to stage the workspace for fixing.
```

## Safety

- Read-only. No SAP writes, no locks, no transports.
- Does not modify source files.

## What this skill does NOT do

- Does not propose fixes (that's the clean-core-fix skill, which spawns clean-core-remediator agents)
- Does not stage the fix workspace (that's the clean-core-prep skill)
- Does not run against all packages (cross-package is `abapctl clean-core executive`)
