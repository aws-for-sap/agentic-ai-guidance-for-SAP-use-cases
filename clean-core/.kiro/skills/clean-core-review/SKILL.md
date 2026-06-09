---
name: clean-core-review
description: "Clean Core remediation: narrative retrospective after clean-core-fix. Reads each object's result.json + findings.json (inline fix: blocks), groups by outcome × review/failure category, identifies cross-object patterns, surfaces exemption + reviewer candidates. Produces RETRO.md with operator action items. Read-only. Use when the user wants to review what remediator did, find exemption candidates, or prepare a retrospective."
---

# clean-core-review: narrative retrospective

Reads per-object results from clean-core-fix and produces an actionable retrospective. Read-only.

This is purely a read-back of what the remediator wrote: `outcome` per object, `fix.status` per finding, and the `review_category` / `failure_category` taxonomies. The review makes no SAP decisions of its own.

## Inputs

- `<SID>/<PACKAGE>`, e.g., `S4H/ZFINANCE`
- `--reassess`: re-run `abapctl clean-core assess` to verify package-level delta

## Prerequisites

1. `clean-core/{SID}/{pkg}/fix-context/.context/<obj>.result.json` exist (run the clean-core-fix skill first).
2. `clean-core/{SID}/{pkg}/fix-context/.context/<obj>.findings.json` exist with terminal `fix:` blocks (prep + fix populated these).
3. `clean-core/{SID}/{pkg}/reports/SUMMARY.md` exists (baseline from assess).
4. `clean-core/{SID}/{pkg}/fix-context/RUN.md` exists (fix skill's run summary; optional but cross-reference).

## Steps

### Step 1: Load all artifacts

Walk `clean-core/${SID}/${PACKAGE}/fix-context/.context/` for every `*.result.json`. For each object:

1. Read `<obj>.result.json`: outcome, level_before/after, counts, gate, sap_state, review_breakdown, operator_action.
2. Read sibling `<obj>.findings.json`: full finding list with terminal `fix:` blocks. Per-finding narrative (`fix.elaboration`, `fix.successor`, `fix.successor_source`, `fix.review_category`, `fix.failure_category`) lives here, NOT in result.json.

Also load:
- Baseline `reports/SUMMARY.md` for pre-fix counts.
- `fix-context/manifest.json` for transport, object inventory, file path map (`name`/`type`/`findingsFile`/`baselineFile`).
- `fix-context/RUN.md` if present (fix skill's per-object outcome line is a useful cross-check).

**Rule:** result.json gives you the rollup (counts, outcome, level delta, gate). findings.json[].fix gives you the per-finding narrative. Don't try to pull per-finding successor / elaboration / category from result.json. It does not duplicate that data. The pointer is the sibling-path convention (lowercased basename), not a stored URI.

**Schema-compliance check on load:** flag and surface (don't silently remap) any of:
- `result.json.schema_version != 1` (integer)
- `result.json.outcome` not one of `success | partial | rolled_back | failed`
- `result.json` missing required fields: `object`, `object_type`, `outcome`, `level_before`, `level_after`, `counts`, `gate`, `sap_state`, `operator_action`
- `findings.json` finding without a terminal `fix.status` ∈ `{fixed, needs_review, failed}` despite a result.json sibling existing
- `fix.status=fixed` with no `successor` or no `successor_source`
- `fix.status=needs_review` with no `review_category` or no `elaboration`
- `fix.status=failed` with no `failure_category` or no `elaboration`
- `fix.status=in_progress` lingering after the run finished. Remediator should have closed every claimed item; an `in_progress` survivor means the remediator crashed mid-write and the next run will trigger all-or-nothing recovery

Report schema-noncompliant objects in RETRO narrative as **"schema violation: remediator needs re-run"**. Do not synthesise missing data.

### Step 2: Outcome matrix

Valid `outcome` values (per clean-core-remediator schema): `success | partial | rolled_back | failed`.

Cross-tabulate outcome against the dominant per-finding category. For `success` and `partial`, use `review_category` as the breakdown axis (which review reasons remain after remediator's best effort). For `rolled_back` and `failed`, use `failure_category` (why remediator or the gate aborted).

| Outcome      | Total | Notes |
|--------------|-------|-------|
| success      | N     | all findings `fixed`, gate passed |
| partial      | N     | mix of `fixed` + `needs_review`; gate passed |
| rolled_back  | N     | gate failed, baseline restored, items downgraded to `failed/gate_regression` |
| failed       | N     | non-gate failure (`source_put_failed`, `transport_closed`, `sap_unreachable`, `rollback_incomplete`) |

Then split by category: totals come from re-walking each object's `findings.json[].fix`:

**Review categories** (across all `needs_review` findings):

| review_category | Count |
|---|---|
| no_successor_found | N |
| probe_failed | N |
| probe_unavailable | N |
| cross_object_change_required | N |
| column_unmappable | N |
| semantic_drift | N |
| researched_unverified | N |

`semantic_drift` = field coverage was fine but the CDS successor auto-filters (client/validity/auth/draft) differently from the raw access, a row-set difference the remediator wouldn't ship. `researched_unverified` = the remediator found a plausible Tier-3 candidate but couldn't confirm it's released + fully equivalent, so it surfaced it as a lead rather than applying it. Both are operator-review items, not defects.

**Failure categories** (across all `failed` findings):

| failure_category | Count |
|---|---|
| gate_regression | N |
| source_put_failed | N |
| transport_closed | N |
| sap_unreachable | N |
| rollback_incomplete | N |

Break out `success` into two narrative buckets (not the matrix): **Applied** (`counts.fixed > 0`) and **Untouched** (`counts.fixed = 0` AND `counts.total = 0`, i.e. no findings the remediator needed to touch). Both count as `success` in the matrix. The interesting cell is `counts.fixed = 0` AND `counts.needs_review > 0`. That's `partial`, not `success`.

### Step 3: Level delta (package aggregate)

Before counts from `SUMMARY.md`. After counts computed by walking results:
- For objects with a result.json, `after_level` from `result.json.level_after`.
- For objects NOT in results (not C/D in the first place), `after_level = before_level`.

Aggregate to A/B/C/D counts. Present as:

```
Before: A=X  B=Y  C=Z  D=W  (total findings P1/P2/P3)
After:  A=X' B=Y' C=Z' D=W'
Delta:  <summary of promotions, e.g., "C→B: 29 objects; D→C: 2 objects; 3 untouched (needs_review couldn't resolve)">`
```

If `--reassess`, re-run `abapctl clean-core assess` and compare computed (`level_after` from result.json) vs live ATC. Flag discrepancies. They imply post-fix drift outside this run (e.g. operator manually fixed something, or another transport touched the package).

### Step 4: Cross-object pattern analysis

**By successor popularity** (walk all `fix.status=fixed` items across findings.json files):

```
I_SALESDOCUMENT:        16 fixes — 16 success, source: sap_github (15), release_state_lookup (1)
CL_BCS_MAIL_MESSAGE:     2 fixes — 2 success, source: sap_github
I_JournalEntryItem:      8 fixes — 8 success, source: release_state_lookup (Tier 2 — enrichment gap?)
```

A high `release_state_lookup` rate for a single successor signals an enrichment opportunity: assess didn't carry the successor over from sap_github / atc_tag, but the live CDS view did. Worth promoting to the static catalog.

**By review_category** (across `needs_review` items):

```
no_successor_found:     7 findings across 5 objects — exemption candidates
probe_failed:           3 findings — schema mismatches; KB gotcha?
column_unmappable:      2 findings — CDS column drift; needs operator review
cross_object_change_required: 1 finding — remediator correctly stayed in scope
probe_unavailable:      1 finding — concept-level successor (release_state category=C)
```

**By failure_category** (across `failed` items, scoped to non-gate-regression; gate failures roll up at outcome level):

```
source_put_failed:      0
transport_closed:       1 (transport released mid-run; resume after operator opens new TR)
sap_unreachable:        0
rollback_incomplete:    0
```

**Repeated review/failure modes:** if 3+ objects show the same `review_category` with the same successor/predecessor pair, surface once with the object list.

### Step 5: Reviewer-recommended fixes

Any `fix.status=fixed` item where human visual verification before transport release adds value:

- **Tier-3 / researched successors** (`fix.successor_source=researched`): **highest priority.** The remediator proposed this successor itself; SAP never asserted it. It cleared the released + coverage + semantic checks, but it's the least-vetted source. Always verify before release.
- **Tier-2 successors** (`fix.successor_source=release_state_lookup`): late binding, discovered live rather than pre-vetted in sap_github / atc_tag. Recommend a glance.
- **Objects with outcome=partial**: the operator already has open `needs_review` items; review the `fixed` siblings while reading the same file.
- **Objects with outcome=rolled_back**: remediator pushed, gate failed, baseline restored. The diff is interesting forensically (what remediator tried, why ATC regressed). Read findings.json + look at git history for the source vs baseline.

```markdown
## Fixes Recommended for Review

| Object | Finding | Successor | Source | Why review |
|--------|---------|-----------|--------|------------|
| ZORDERS | #2 | Z_RELEASED_VIEW | researched | Tier-3: agent-found, not SAP-asserted; verify released + equivalent |
| ZPAYMENTS | #4 | I_JournalEntryItem | release_state_lookup | Tier-2: verify CDS field mapping |
| ZBILLING | #2 | CL_BCS_MAIL_MESSAGE | sap_github | object outcome=partial; sibling has needs_review |

Read the source diff (workspace) and `findings.json[].fix.elaboration` for context.
```

**ABAP Unit regression tests (audit artifacts).** Note any object whose result.json has `gate.unit_test_added=true`: the remediator wrote a test pinning the migrated read-path behavior, shipped in the transport. These are the strongest audit evidence in the run: list them so the operator knows which fixes carry a persisted regression net. Read-path fixes on declarative objects (CDS/BDEF) can't host a test and fall back to ephemeral verification. Flag those for a closer manual look.

### Step 6: Exemption candidates

Objects where one or more findings landed at `fix.status=needs_review` with a `review_category` that signals "no code-level fix path exists": these are exemption candidates the operator should triage:

- `no_successor_found`: cascade exhausted (sap_github → release_state_lookup), no concrete or concept-level successor.
- `probe_unavailable`: release-state returned a concept-level successor (category `C`), no concrete API to migrate to.
- `cross_object_change_required`: remediator found a successor but applying it would break callers outside this object's scope.

The other two (`probe_failed`, `column_unmappable`) are usually KB amendment candidates, not exemption candidates. They signal remediator needs better mapping, not that the predecessor is unfixable.

Within `no_successor_found`, split the narrative into two sub-groups; the `fix.elaboration` text tells them apart (the remediator writes "Structural dead-end" for the first):
- **Structural (DDIC table writes)**: `UPDATE`/`MODIFY`/`INSERT`/`DELETE` on a DDIC table. CDS is read-only, so no CDS successor can ever exist. These are the genuine Level-D blockers. Operator action: exempt, or hand-redesign to a released BAPI/RAP BO if one exists for the area. Bulk-exemptable. Don't ask the operator to investigate each one; the dead-end is structural, not a lookup miss.
- **Incidental (reads/calls with no successor)**: bare `SELECT`, non-released FM/class with nothing in the catalogs. May have a successor the catalog lacks; worth a glance before exempting. (Note: bare-`SELECT` reads are Level C / non-blocking, lower urgency than the structural writes.)

For each exemption-candidate finding, surface:
- Object + findingNumber
- Predecessor (what the finding was about, typically `refObjectName` on the finding)
- `fix.review_category` + a structural/incidental tag (derived from the elaboration) + `fix.elaboration` (verbatim; remediator's narrative is the exemption justification draft)
- Marker ID: the `finding.quickfix` field, formatted `atc:<itemid>,<index>` (e.g. `atc:02F3121AA6451FD196A685D7FA956000,2633`). That string is the input to `abapctl check atc-exempt-proposal`. (Field is `quickfix`, NOT `quickfixInfo`; the latter does not exist on the finding.)

```markdown
## Exemption Candidates

| Object | # | Predecessor | Kind | Reason | Marker ID |
|--------|---|-------------|------|--------|-----------|
| ZSBOOK_DML | 1 | SBOOK (write) | structural | DDIC table write: CDS read-only, no write successor; exempt or BAPI/RAP redesign | <markerId> |
| ZFGDATAGEN1 | 7 | CL_FQMC_ACTIVE | incidental | internal SAP framework class, no public successor | <markerId> |
| ZINVOICE | 3 | API_NO_SUCC concept | incidental | probe_unavailable: release-state successor is category=C (concept), no concrete API | <markerId> |

To propose an exemption (per finding):
    abapctl check atc-exempt-proposal <markerId> -c ${CONNECTION}

Review the proposal, fill justification (paste the elaboration above), then:
    abapctl check atc-exempt <markerId> -c ${CONNECTION} --reason OTHR --justification "..."
```

### Step 7: KB amendment candidates

Proposals derived from patterns across results:

- **`column_unmappable` clusters**: same predecessor → successor pair, multiple objects, all column-drift failures: KB needs a column-rename rule for that pair.
- **`probe_failed` clusters**: successor returned by cascade but remediator's probe fails consistently: KB needs a gotcha entry (e.g. parameter rename, mandatory annotation).
- **High `release_state_lookup` rate**: assess's static catalog is missing entries that the live CDS view has. Promote those predecessor → successor pairs into sap_github / cc-kb.
- **`no_successor_found` predecessors seen 3+ times**: candidate for a sap_github contribution upstream.

```markdown
## Proposed KB Amendments (operator reviews; not auto-applied)

- `cc-kb/successors.md`: add `BAPI_PO_GETDETAIL → BAPI_PO_GETDETAIL1` — promoted from release_state_lookup hits in 4 objects this run.
- `cc-kb/gotchas.md`: `CL_BCS_MAIL_MESSAGE` always needs TRY/CATCH cx_bcs — seen as `probe_failed` in 2 objects before remediator learned the pattern.
- `cc-kb/column-maps.md`: `VBAK.VBELN → I_SalesOrder.SalesOrder` — column_unmappable in 3 objects this run.
```

### Step 8: Write RETRO.md

At `clean-core/${SID}/${PACKAGE}/reports/RETRO.md`:

1. Headline metrics (outcome matrix, level delta, review/failure category breakdowns)
2. Narrative (prose, covering what worked, what didn't, anomalies)
3. Reviewer-recommended fixes (Step 5)
4. Exemption candidates (Step 6)
5. Top failures with per-object diagnosis (any `outcome ∈ {rolled_back, failed}`; read the sibling findings.json and result.json.gate.elaboration)
6. KB amendment candidates (Step 7)
7. Operator action items (numbered, concrete; paste from `result.json.operator_action` per object, dedupe)

### Step 9: Print summary

```
clean-core-review ${SID}/${PACKAGE} complete.

Headline:
  Before: A=... B=... C=... D=... (N objects, P1/P2/P3 findings)
  After:  ...
  Delta:  ...

Outcomes:
  success:     X
  partial:     X  (review needs_review items)
  rolled_back: X  (gate regressed, baseline restored)
  failed:      X  (non-gate failure)

Findings (across all objects):
  fixed:        X
  needs_review: X
  failed:       X

Top categories:
  needs_review.no_successor_found:    K  (exemption candidates)
  needs_review.probe_failed:          K  (KB candidates)
  failed.gate_regression:             K  (rolled_back objects)

Operator actions:
  (1) Review K reviewer-recommended fixes (Tier-2 successors, partial outcomes)
  (2) File exemptions for K no_successor_found / probe_unavailable findings (marker IDs in RETRO.md)
  (3) Investigate K rolled_back objects (gate.elaboration in result.json)
  (4) Consider Q proposed KB amendments

Full report: clean-core/${SID}/${PACKAGE}/reports/RETRO.md
```

## Safety

- Read-only. No SAP writes (unless `--reassess` runs `abapctl clean-core assess`, also read-only).
- Does not modify result.json, findings.json, manifest.json, or any other artifact.
- KB amendment candidates are proposals; operator decides whether to apply.

## What this skill does NOT do

- Does not file exemptions (operator's call)
- Does not modify the KB (surfaces candidates only)
- Does not re-run failed fixes (that's re-invoking the clean-core-fix skill with `--only ...`)
- Does not produce cross-package analytics (single package scope)
- Does not synthesise result.json or findings.json (only remediator writes them; this skill only reads)
