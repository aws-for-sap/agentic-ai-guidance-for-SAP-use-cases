---
name: sap-abap
description: "Work with SAP ABAP objects on a live system via the abapctl CLI (headless, scriptable access to SAP's ADT API). Use to read, navigate, assess, refactor, or write ABAP code. Trigger when the user mentions SAP, ABAP, ADT, ATC, abapctl, clean core, transports, CDS, DDIC, RAP/behavior definitions, classes/programs/function groups, packages, ENHO, text-elements, release state, code navigation/completion, refactoring (rename/extract/move), workspace sync, data preview/SQL, ABAP Unit, or service bindings, or asks to inspect, navigate, refactor, run, or modify ABAP code against a live system."
argument-hint: "[optional: what you want to do, e.g. 'assess ZFINANCE', 'rename method in ZCL_FOO', 'pull workspace']"
---

# Working with SAP ABAP via abapctl

`abapctl` is the CLI you'll use for SAP's ADT REST API. Reach for it whenever you need to read, navigate, assess, refactor, or write ABAP objects on a live SAP system. It handles auth, CSRF, lock/unlock, activation, retries, and session reuse.

This `SKILL.md` is the **mental model + decision aid** (keep it loaded). Companion file **`reference.md`** is the **flag catalog and behaviors** (read it when picking flags). Source of truth: `abapctl --help`, `abapctl <group> --help`, `abapctl tools list --json`.

## Pick the right command (intent → command)

Scan this first. If your intent isn't here, fall back to `abapctl tools list --json`.

| Intent | Command | Notes |
|---|---|---|
| **Browse / understand** | | |
| Find an object by name pattern | `object search <pattern>` | wildcards, `--type` filter |
| What is this object, where does it live | `object info <name>` | metadata + includes |
| What's in this package | `object tree <pkg>` or `package list` | tree = objects, list = packages |
| Who depends on this symbol | `code references <obj>` + `code snippets <obj>` | where-used + context |
| What is at line X col Y | `code element-info <obj> --line N --col M` | type, docs, components |
| Jump to definition | `code definition <obj> --line N --col M` | navigate target |
| What can I type here (compiler-authoritative) | `code complete <obj> --line N --col M [--prefix <text>]` | hallucination guard |
| What ENHO is active on this base object | `enho on <obj>` | decoded source + position |
| Inventory ENHOs in system | `enho list [--customer-only] [--package X]` | migration-risk pile |
| **Read source** | | |
| Class header only | `source get ZCL_FOO` | classes default to header |
| Class methods | `source get ZCL_FOO --include implementations` | also `definitions`, `testclasses` |
| Program / interface / FUGR include / DDLS body | `source get <name>` | single source (no `--include`) |
| Translatable strings (TEXT-001, headings) | `text-elements get <obj> --category symbols\|selections\|headings` | divergent semantics (see below) |
| Revision history | `object history <obj>` | |
| **Run checks** | | |
| Syntax of the active version | `check syntax <obj>` | |
| Syntax of unsaved local file | `check syntax <obj> --source ./edit.abap` | |
| ATC on one object | `check atc <obj>` | exit 1 = findings |
| ATC + per-finding fix metadata | `check atc-quickfix <obj> -v <variant> [--filter automatic\|manual\|pseudo\|ai]` | read-only |
| ABAP Unit | `check unit <obj>` | `--junit` for CI |
| CDS DDL syntax | `check cds-syntax <obj>` | |
| **Preview data** | | |
| One DDIC table | `data query <table> [--rows N] [--where "..."]` | |
| One CDS view | `data cds <view> [--rows N]` | |
| Arbitrary SQL | `data sql --query "SELECT ..."` or `--source file.sql` | freestyle |
| **DDIC deep metadata** | | |
| Table fields (parsed DDL) | `ddic table <name>` | not S/4 EC6 |
| Domain / data element / table type / lock object / type group / structure / view / table-settings | `ddic <kind> <name>` | all read-only |
| **Refactor (read-only by default)** | | |
| Quick fix list at cursor | `refactor quickfix <obj> --line N --col M` | + `--apply <idx>` to execute |
| Rename | `refactor rename <obj> --line N --col M` | + `--new-name X --transport NR` to execute |
| Extract method | `refactor extract-method <obj> --start-line ... --end-line ...` | + `--name X --transport NR` to execute |
| Move object(s) | `refactor move <obj1> <obj2>` | + `--package X --transport NR` to execute |
| **Write source / activate** | | |
| Push edited file to one object | `source put <obj> --file ./e.abap [--include ...] --transport NR -y` | |
| Push many files | `source push <obj1> <obj2> --dir ./src --transport NR -y` | sequential write, batch activate |
| Activate | `object activate <obj1> <obj2>` | batch implicit when multiple |
| Pretty-print | `source format` (stdin) / `source format-settings` | settings flag = mutating |
| Translatable strings | `text-elements put <obj> --category X --file ./e.txt --transport NR` | **--category REQUIRED, no auto-activate** |
| Delete | `object delete <obj> -y --transport NR` | |
| **Create new object (one-shot)** | `create <type> <name> --package PKG [--transport NR]` | 21 types |
| Create with body in one call | `create <type> <name> --package PKG --source ./body.abap` | 14 of 21 types accept `--source`; CLAS/FUGR accept *repeated* `--source` for multi-include |
| Create domain (typed) | `create domain Z_DOM --data-type CHAR --length 10 [--fixed-value A:B:Active]` | typed flags or `--source` (XOR) |
| Create data element (typed) | `create data-element Z_DTEL --domain Z_DOM --label-short ... --label-long ...` | typed flags or `--source` (XOR) |
| **Transport** | | |
| Does this object need a transport? Which one? | `transport info <obj>` | object → requirements |
| Open new transport | `transport create --description "..."` | |
| List my transports | `transport list` | |
| Detail of a specific transport NR | `transport get <NR>` | **NR → tasks/objects (NOT `info`)** |
| Release / delete a transport | `transport release <NR> -y` / `transport delete <NR> -y` | destructive |
| **Workspace (git-like)** | | |
| Clone | `workspace init <pkg> [--recursive --depth N]` | SID baked in from `-c` |
| Status / diff | `workspace status [SID/PKG]` / `workspace diff <SID/PKG>` | |
| Pull / push | `workspace pull <SID/PKG>` / `workspace push <SID/PKG> -y` | |
| Add / remove tracked | `workspace add <SID/PKG> <obj>` / `workspace remove <SID/PKG> <obj>` | |
| Re-discover new objects | `workspace refresh <SID/PKG>` | |
| Discard local | `workspace reset <SID/PKG> [--sap]` | `--sap` re-downloads |
| **Clean Core** | | |
| Assess package | `clean-core assess <pkg>` | takes bare pkg; creates SID dir |
| Prep / apply | `clean-core prep <SID/PKG>` / `clean-core apply <SID/PKG> -y` | take `SID/PKG` |
| Cross-package summary | `clean-core executive` | |
| **Successor lookup** | | |
| Is `CL_GUI_ALV_GRID` deprecated? Successor? | `release-state lookup CL_GUI_ALV_GRID` | live CDS view; S/4 only |
| **System** | | |
| First-time config | `init --template` (agents) / `init` (humans) | then edit `.abapctl.json` |
| Add another connection | `config add-connection <name>` | interactive |
| What's the active connection | `config show` | |
| What ADT services does this system expose | `discover` / `tools coverage` | capability probe |
| Run arbitrary ABAP | `run --source ./snippet.abap` | **mutating, treat as code execution** |
| Health check | `system-check [obj]` | `--connection-only` / `--read` / `--write` |

## Always do this

1. **Pass `-c <name>`** unless you know `defaults.connection` is what you want.
2. **Pass `--json`** on every call. JSON → stdout. Progress/colors → stderr. Don't `2>&1` if piping to a JSON parser.
3. **Persist sessions**: `--session-file .abapctl/.session.json` (or set `defaults.session_file`). Saves seconds per call.
4. **Probe capability before assuming**: `discover -c <c>` (top level) or `tools coverage -c <c>` (registry vs system). Some endpoints don't exist on every release (EC6 lacks lots; S/4 has more).

## Agent-context note (subagents)

If you're a remediator/applier subagent spawned with sealed context, your cwd may not be the project root. **Before running any `abapctl` command, `cd` to the directory containing `.abapctl.json`.** Connection config resolves from cwd; a wrong cwd reads as "connection not configured" and looks like a SAP outage. If `.abapctl.json` isn't found in any ancestor, abort. Don't guess.

## Safety model

| Class | Examples | Default |
|---|---|---|
| **read-only** | `object info/search/tree/path/types/inactive/history`, all `code *`, `source get`, all `check *` (except `atc-exempt`), all `data *`, all `ddic *`, all `cds *`, `enho on/list`, `release-state lookup`, `text-elements get`, `workspace status/diff/list`, `transport info/list/get`, `package list/exists/lookup`, `discover`, `tools *` | Run freely |
| **mutating** | `object activate`, all `create *`, `transport create`, `refactor <x> --<action-flag>`, `service publish`, `workspace init/pull/add/remove/refresh`, `bdef create`, `check atc-exempt`, `run`, `source format-settings --<flag>` | Confirm with user; preview with `--dry-run` if available |
| **destructive** | `source put/push`, `object delete`, `text-elements put`, `workspace push/reset`, `clean-core apply`, `transport release/delete`, `service unpublish` | **Always confirm.** Pass `-y` only after explicit user OK |

Rules:

- **Refactoring is dual-mode.** `refactor rename/extract-method/move/quickfix` without an action flag is read-only evaluation. Becomes mutating only with `--new-name`, `--name`, `--package`, or `--apply`. Always evaluate first, show the plan, then execute.
- **`--dry-run` converts mutating → read-only.** Available on all `create *`, `source put/push`, `object activate/delete`, `text-elements put`, `transport create/release/delete`, `service publish/unpublish`, `clean-core apply`, all `workspace` writes, `bdef create`. Use it to preview.
- **Transports**: writes on managed packages need `--transport NR`. `transport info <object>` (object name → requirements + open transports) ≠ `transport get <NR>` (transport number → tasks/objects). Local packages (`$*`) skip transport.
- **`run` executes arbitrary ABAP.** Treat as fully mutating. Review the source before invoking; never auto-run.

## Exit codes are signal

| Code | Meaning | Action |
|---|---|---|
| `0` | Clean success | Continue |
| `1` | **Partial**: findings exist, some objects failed, diffs present | Parse JSON; report per-object outcome |
| `2` | Hard error (auth, config, SAP fault) | Stop; surface `error.code` / `error.message` to user |

`check atc` exit 1 = findings, not crash. `workspace diff` exit 1 = diffs exist. Treat exit 1 as **data**.

## Non-obvious rules that trip up agents

These are the things that consume cycles when you guess.

### Class includes: `--include` is mandatory for method bodies
- `source get ZCL_FOO` returns header only. To get bodies: `--include implementations`.
- Valid values per command:
  - `source get/put`: `definitions` | `implementations` | `testclasses`
  - `code *`, `refactor *`, `object history`, `enho on`: above + `main`
- Other types (PROG, INTF, FUGR/FF, DDLS) ignore `--include`. Single source.

### Multi-include `create class` and `create function-group`: repeated `--source`
Filename suffix dispatches to the right slot. Don't pass JSON metadata or wrap the calls. The CLI does it.

CLAS slots:
- `{name}.clas.abap`: main
- `{name}.clas.definitions.abap`
- `{name}.clas.implementations.abap`
- `{name}.clas.macros.abap`
- `{name}.clas.testclasses.abap`

FUGR members:
- `{name}.fugr.abap`: TOP
- `{name}.fugr.{fmname}.func.abap`: function module (FUGR/FF)
- `{name}.fugr.{suffix}.reps.abap`: function group include (FUGR/I)

Single-source CLAS convenience: one `--source` with non-matching filename → treated as main + warning.

### Ambiguous object names: `--adt-type`
Some Z names exist as multiple types (e.g. `ZFOO` as both PROG/P and FUGR/FF). Pass `--adt-type PROG/P` (or `CLAS/OC`, `FUGR/FF`, etc.) to disambiguate. Available on `source get`, `source put`, `text-elements get`, `text-elements put`.

### `text-elements put` diverges from `source put`
- `--category` is **REQUIRED** (`symbols` | `selections` | `headings`). No default. Silent wrong-category writes too easy.
- `--file` REQUIRED.
- **No auto-activation.** Use `--activate` to opt in. (Probe-confirmed: text-elements don't need activation; this is intentional.)
- **Lock target is the textelements URI**, not the parent object URI. Handled internally; just don't be surprised if you see `/sap/bc/adt/textelements/programs/...` in logs.
- Plaintext format: `KEY=VALUE` per line. Symbols: 3-char keys; selections: 30-char limit; headings: fixed key set (LISTHEADER, COLUMNHEADER_1..4).

### `create <type> --source <path>` covers 14 of 21 types
Two channels, hard-coded per command:
- **source-main** (text/plain → `{root}/source/main`): `program`, `interface`, `include`, `function-module`, `function-group-include`, `ddl-source`, `service-definition`, `access-control`, `metadata-extension`
- **root-xml** (application/* → root URL): `message-class`, `auth-field`, `auth-object`, `domain`, `data-element`

`create class` and `create function-group` use repeated `--source` (multi-include, see above).

`create domain` and `create data-element` accept either typed flags OR `--source` (XOR, mutex error names the conflicting flags).

`function-module` and `function-group-include` require `--group <fugr>`.

`create annotation-definition` is shell-only (deferred, test users get HTTP 403).

### Positions are 1-based
`--line 1 --col 1` = first character of first line. Matches the ABAP editor. Don't zero-index.

### SID/PACKAGE for workspace and clean-core
- `workspace init ZFIN -c dev` (bare pkg in) → afterward addressed as `<SID>/ZFIN`. SID comes from `connections.<name>.sid`.
- `clean-core assess ZFIN -c dev` takes bare pkg (creates the SID dir).
- `clean-core prep|apply|report` and all `workspace pull|push|status|diff|...` take `SID/PKG`.
- `workspace list` (no arg) prints `SID/PACKAGE` of every workspace.

### Stable-source conventions
- `source format` reads stdin only. Pipe in; don't pass a filename.
- `source format-settings` is read-only without flags (prints), mutating with `--style`/`--indentation` (writes system-level formatter settings, get user OK).
- Passwords come from env vars named via `password_env`. Auth fail on first call usually means env var not exported in current shell.

## Common workflows

Short list. For richer recipes use `abapctl recipes` and `abapctl recipes <name> --json`.

**Understand before changing:**
```bash
abapctl code definition ZCL_FOO --line 42 --col 12 -c dev --json
abapctl code element-info ZCL_FOO --line 42 --col 12 -c dev --json
abapctl code references ZCL_FOO -c dev --json
abapctl code snippets ZCL_FOO -c dev --json
```

**Source round-trip (download → edit locally → check → upload → verify):**
```bash
abapctl source get ZCL_FOO --include implementations -c dev > impl.abap
# edit impl.abap
abapctl check syntax ZCL_FOO --source impl.abap -c dev --json
abapctl source put ZCL_FOO --file impl.abap --include implementations \
  --transport <NR> -c dev -y --json
abapctl check atc ZCL_FOO -c dev --json
```

**Safe refactor (evaluate → review → execute):**
```bash
abapctl refactor rename ZCL_FOO --line 42 --col 12 -c dev --json
# show output, get user approval
abapctl refactor rename ZCL_FOO --line 42 --col 12 \
  --new-name NEW --transport <NR> -c dev --json
```

**Successor lookup for a deprecated API** (live fallback when findings.json enrichment is missing):
```bash
abapctl release-state lookup CL_GUI_ALV_GRID -c dev --json
# { successors: [{ category: 'O', name: 'CL_SALV_TABLE', ... }], found: true }
# category 'O' = swap class A for class B
# category 'C' = use the named concept (e.g. "XCO Library")
# S/4 only (S4H/S4J/A4H); EC6 returns CAPABILITY_NOT_AVAILABLE
```

**ECC → S/4HANA migration assessment** (ENHO surface):
```bash
abapctl enho list --customer-only --type source-code -c ec6 --json
abapctl enho on SAPMV45A --type PROG/P -c ec6 --json
# Friendly --type: source-code, badi-impl, enhancement, enhancement-spot, badi-spot
# Default fanout (no --type) covers the 3 ENHO impl types.
```

**Clean core assessment:**
```bash
abapctl package list -c dev --json
abapctl clean-core assess ZFINANCE -c dev --json     # creates <SID>/ZFINANCE/
abapctl clean-core prep <SID>/ZFINANCE -c dev
# edit fix-context/*.abap
abapctl clean-core apply <SID>/ZFINANCE -c dev --dry-run --json
abapctl clean-core apply <SID>/ZFINANCE -c dev -y --json
```

**Workspace (git-like):**
```bash
abapctl workspace init ZFIN -c dev --recursive --json
abapctl workspace status <SID>/ZFIN --json
abapctl workspace diff <SID>/ZFIN --json
abapctl workspace push <SID>/ZFIN -c dev --dry-run --json
abapctl workspace push <SID>/ZFIN -c dev -y --json
```

**Runtime verification (ask the live system a question):**
```bash
cat > /tmp/verify.abap <<'ABAP'
DATA(desc) = CAST cl_abap_classdescr( cl_abap_typedescr=>describe_by_name( 'ZCL_FOO' ) ).
LOOP AT desc->methods INTO DATA(m). out->write( m-name ). ENDLOOP.
ABAP
abapctl run --source /tmp/verify.abap -c dev --json
```

## Clean Core levels

Object level = worst finding across the object.

| ATC priority | Level | Meaning | Action |
|---|---|---|---|
| 0 | A | Clean | — |
| 3 | B | Info only | — |
| 2 | C | Warnings | Optional |
| 1 | D | Errors | **Must fix**: blocks cloud |

## Failure decoder

| Symptom | Cause | Fix |
|---|---|---|
| 401/403 on first call | Password env var not exported in this shell | Re-export `$SAP_*_PASSWORD`; retry |
| 403 only on writes (read works) | Stale CSRF in persisted session | Delete `.abapctl/.session.json`; retry once |
| HTTP 423 / "object is locked" | Someone holds the lock; or `text-elements put` against parent URI | `object info <name> --json` to see holder. Don't force-unlock. Surface to user |
| `transport info <NR>` returns OBJECT_NOT_FOUND | Wrong verb: `info` takes object name, not transport NR | Use `transport get <NR>` |
| "capability not available" / 404 on a registered command | Endpoint not on this release | `discover -c <c> --json`; or `tools coverage -c <c> --json` |
| Recursive `workspace init --depth N` pulled tens of thousands of objects | Sideways package refs explode at depth >1 | Use `init` per leaf package, or `--depth 1` |

## Don't

- Don't invent command names. Verify with `--help` or `tools list --json`. The CLI evolves.
- Don't pass `-y` silently on destructive commands. Confirm with the user first.
- Don't hardcode hosts, passwords, transports, or SIDs. Read from config.
- Don't parse non-JSON stdout with regex. Add `--json` and parse the object.
- Don't zero-index `--line` / `--col`.
- Don't `2>&1` when piping to `jq`. Progress goes to stderr.
- Don't pre-flight a transport before a write. `source put` resolves it. (`transport get <NR>` to *check* a known NR is fine; don't *create* one speculatively.)
