# Kiro Agents for SAP Clean Core

We introduced the [ABAP Accelerator CLI](https://github.com/aws-for-sap/Automate-SAP-development-workflows-using-ABAP-CTL) as a headless primitive for SAP ABAP development workflows. This repository builds one such workflow on top of it: assessing and remediating an ABAP package for **SAP Clean Core**, classified by SAP's compliance levels A through D ([Clean Core whitepaper](https://www.sap.com/documents/2024/09/20aece06-d87e-0010-bca6-c68f7e60039b.html)).

It runs in [Kiro CLI](https://cli.kiro.dev) as an assess → prep → fix → review workflow. Start with assessment alone to see where a system stands today, or carry on through the fix step, which remediates findings one object at a time, in parallel, saving every change to an SAP transport number you provide.

```
This workflow:  assess → prep → fix → review   (Kiro skills + agent)
              │
              ▼
abapctl:   the ABAP Accelerator CLI primitive
              │  HTTPS / ADT REST
              ▼
SAP:       your ECC / S/4HANA system
```

## Quick start

Once the [prerequisites](#prerequisites) are met and the workflow is [installed](#install), run Kiro from your project and ask in plain language:

```bash
cd <your-project>     # the directory with .abapctl.json
kiro-cli              # the default agent loads the skills on demand
```

> assess ZFINANCE on connection s4j

Kiro runs the assessment, shows you the A/B/C/D breakdown, and points you at the next step. Continue through prep, fix, and review as you acknowledge each one. [The workflow](#the-workflow) below describes what each step does.

## The workflow

Four skills, one per step. You drive each in plain language; the matching skill loads on demand, runs the `abapctl` commands, and reports back. By default you run one step at a time and acknowledge each before the next. The steps are independent skills, so you can also ask the agent to chain them end-to-end (assess through review) for a hands-off run.

### 1. Assess

> assess ZFINANCE on connection s4h

Discovers every object in the package, runs ATC under the cloud-readiness variant, and classifies each object into one of SAP's four Clean Core levels. The level is the object's worst finding.

| Level | ATC findings | Meaning | Action |
|-------|--------------|---------|--------|
| **A** | none | Fully clean | Cloud-ready, nothing to do |
| **B** | P3 (info) | Uses documented extension points | Acceptable, low upgrade risk |
| **C** | P2 (warnings) | Uses internal or undocumented APIs | Plan migration; verify before upgrade |
| **D** | P1 (errors) | Non-released APIs, modifications, blocked patterns | Must remediate (cloud blocker) |

The skill prints the A/B/C/D distribution and the top remediation targets, and writes a `SUMMARY.md` plus per-object findings. It is read-only against SAP and safe to repeat. For a cross-package roll-up, ask for the executive summary (`abapctl clean-core executive`).

### 2. Prep

> prep S4H/ZFINANCE for fixing

For every C and D object, downloads the current source, saves a read-only **baseline** copy for rollback, and stages a per-finding work list. It also resolves the transport: it validates a transport you name (or reuses one already stamped in the manifest), and prompts you if none is set. It never creates or releases a transport. The resolved transport is stamped into the manifest so the fix step writes every change through it.

### 3. Fix

> fix S4H/ZFINANCE

Spawns `clean-core-remediator` agents in parallel (default 3, max 4, the Kiro `use_subagent` cap), one per object, each in a sealed context that sees only its assigned object. For each finding, the remediator:

1. **Cascades for a released successor**, stopping at the first hit:
   - **ATC tags on the finding**: the `Successors:` metadata SAP ships when it has already classified the deprecated API.
   - **SAP released-API catalog**: the reference JSONs populated by `abapctl reference update`.
   - **Live release-state lookup**: `abapctl release-state lookup <api>` queries `I_APISWithCloudDevSuccessor` on the target system directly, always current to its release.
2. **Probes the successor** for compatibility (CDS column coverage, method/FM signature) and, for table-to-CDS read swaps, checks for semantic drift (client/validity/auth filters that change the row set).
3. **Rewrites** the finding in the working source: minimum diameter, no wrappers, no method extraction, no unrelated cleanup.
4. **Pushes** the object under the resolved transport, then **gates** it: `abapctl check syntax` + `check atc` + any unit tests. The gate passes only if the Clean Core level does not regress.
5. **Rolls back to baseline** if the gate fails.

When a fix is not provably safe and behavior-equivalent, the remediator defers it for human review. If three objects in a row fail with SAP unreachable, the run circuit-breaks and tells you to check connectivity.

### 4. Review

> review S4H/ZFINANCE

Reads every per-object result and writes a retrospective (`RETRO.md`): outcome breakdown, **reviewer-recommended fixes** (late-binding successors, partial outcomes, rolled-back objects), **exemption candidates** (findings with no code-level fix path), and **knowledge-base amendment proposals** (patterns worth promoting into the catalog). Read-only: it surfaces candidates; you file exemptions and edit the knowledge base.

## Prerequisites

- **[ABAP Accelerator CLI (`abapctl`)](https://github.com/aws-for-sap/Automate-SAP-development-workflows-using-ABAP-CTL)** on PATH (`abapctl --version`). Use the latest release.
- **[Kiro CLI](https://cli.kiro.dev)** installed.
- **Subagents enabled:** `kiro-cli settings chat.enableSubagent true`. The fix step spawns remediators via Kiro's `use_subagent` tool, which is gated behind this setting. The built-in `kiro_default` agent already has the tool; a custom agent must list `use_subagent` in its `tools`.
- **A configured `.abapctl.json`** in your project with at least one connection, and SAP's released-API catalog populated (`abapctl reference update`).

## Install

Clone the guidance repository and copy this workflow's `.kiro/` tree (skills + agent) and `cc-kb/` knowledge base into your project.

```bash
git clone https://github.com/aws-for-sap/agentic-ai-guidance-for-SAP-use-cases
cd agentic-ai-guidance-for-SAP-use-cases/clean-core   # this workflow's folder

cp -r .kiro/agents/* <project>/.kiro/agents/
cp -r .kiro/skills/* <project>/.kiro/skills/
mkdir -p <project>/.abapctl/reference/cc-kb
cp -r cc-kb/* <project>/.abapctl/reference/cc-kb/
```

The skills and agent can also live globally in `~/.kiro/`; the `cc-kb/` knowledge base stays per-project under `.abapctl/reference/cc-kb/`, since that is where the remediator reads it.

## What's in the workflow

```
.kiro/
├── agents/
│   ├── clean-core-remediator.json          # the worker agent (config)
│   └── clean-core-remediator.prompt.md     # the worker agent (system prompt)
└── skills/
    ├── sap-abap/                # abapctl command reference the agents read
    ├── clean-core-assess/       # ATC + level classification
    ├── clean-core-prep/         # download source, baselines, findings; resolve transport
    ├── clean-core-fix/          # spawn remediators in parallel
    └── clean-core-review/       # retrospective + exemption candidates

cc-kb/                           # remediation knowledge base (copied to .abapctl/reference/cc-kb/)
```

The four skills load into Kiro's `kiro_default` agent (or your own); `clean-core-remediator` is the only agent shipped. The `cc-kb/` knowledge base is style and idiom guidance, not a successor list (successors come from SAP, via the fix-step cascade).

## Where work is stored

Each run writes its artifacts under `clean-core/{SID}/{PACKAGE}/` in your project, separate from your source:

```
clean-core/{SID}/{PACKAGE}/
├── reports/
│   ├── SUMMARY.md          # assess output (human-readable)
│   └── RETRO.md            # review output
├── state/
│   └── progress.json       # per-object level + finding count
└── fix-context/
    ├── manifest.json       # object inventory, paths, the stamped transport
    ├── <obj>.<type>.abap   # working source the remediator edits
    ├── .baseline/          # read-only rollback copies
    └── .context/
        ├── <obj>.findings.json   # per-finding work list + fix state
        ├── <obj>.context.md      # operator-friendly summary
        └── <obj>.result.json     # per-object outcome
```

The baseline copies are what the gate rolls back to. The findings and result files are what the review step reads to build the retrospective. All writes are atomic.

## Extend the workflow

The four steps are independent skills over stable, file-based contracts, so you can build on any layer.

- **Run any step on its own.** Assess is a read-only analysis. Prep, fix, and review each stand alone given the artifacts the previous step wrote, so you can stop, inspect, and resume at any boundary.
- **Build a UI, dashboard, or progress tracker.** `progress.json`, `findings.json`, and `result.json` are stable schemas (documented in the `clean-core-remediator` prompt). Read them to render live run progress, drive a dashboard, or aggregate `result.json` across packages for portfolio reporting.
- **Resume and re-run.** Assess is idempotent (`--force` to start fresh). Fix skips objects already at a terminal state and re-queues a single object with `--only`; a crashed remediator recovers from its baseline on the next run. Re-invoke the same command from CI or a scheduled job to continue where it stopped.
- **Reuse the worker agent.** `clean-core-remediator` takes one object plus its findings and runs the full cascade → probe → apply → gate → rollback loop in a sealed context. Drive it from your own orchestrator (a pre-merge gate, a single-object fix, a different batching strategy) without the fix skill.
- **Tune the run.** `--parallel` sets concurrency, `--only` / `--skip` scope the object set, and `--dry-run` previews the queue before anything spawns.
- **Edit the knowledge base.** `cc-kb/` is plain markdown the remediator reads for style and idiom guidance. Add your team's conventions or domain patterns, or pull SAP's published references into it (see [`cc-kb/README.md`](cc-kb/README.md)); the review step proposes amendments you can fold back in.
- **Use the primitive directly.** Every SAP interaction is an `abapctl` command. For anything the workflow does not cover (other object types, custom checks, code navigation), call `abapctl` directly or write a new skill against it.

## References

- [ABAP Accelerator CLI (`abapctl`)](https://github.com/aws-for-sap/Automate-SAP-development-workflows-using-ABAP-CTL): the headless primitive this workflow is built on.
- [SAP Clean Core Extensibility whitepaper](https://www.sap.com/documents/2024/09/20aece06-d87e-0010-bca6-c68f7e60039b.html): the four compliance levels and the cloud-readiness model.
- [SAP-samples/abap-cheat-sheets](https://github.com/SAP-samples/abap-cheat-sheets): ABAP syntax patterns and modern examples (Apache 2.0).
- [SAP/abap-atc-cr-cv-s4hc](https://github.com/SAP/abap-atc-cr-cv-s4hc): the cloudification library, fetched by `abapctl reference update` (Apache 2.0).
- [SAP/styleguides](https://github.com/SAP/styleguides): Clean ABAP style guidance (Apache 2.0).

## License

MIT-0 (MIT No Attribution).
