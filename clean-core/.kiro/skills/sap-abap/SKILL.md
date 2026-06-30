---
name: sap-abap
description: "Bring SAP ABAP development into the agentic era via the abapctl CLI: headless, JSON-first access to SAP's ADT development API and OData runtime services. Use it to read, navigate, assess, refactor, and write ABAP code; run ATC and unit checks; manage transport requests; assess and remediate objects toward Clean Core; and query business data through published OData services the way a Fiori app sees it. Trigger when the user mentions SAP, ABAP, ADT, ATC, clean core, transports, CDS, DDIC, RAP/behavior definitions, classes/programs/function groups, packages, ENHO, text-elements, release state, code navigation/completion, refactoring (rename/extract/move), workspace sync, data preview/SQL, ABAP Unit, OData/Fiori services, or service bindings (publish/test), or when the user asks to inspect, navigate, refactor, run, or modify ABAP code, or query live business data, against a live SAP system."
argument-hint: "[optional: what you want to do, e.g. 'assess ZFINANCE', 'rename method in ZCL_FOO', 'pull workspace']"
compatibility: "Requires the abapctl CLI on PATH and a live SAP system with ADT enabled, reachable via a configured connection in .abapctl.json (host/client/credentials); run `abapctl init` to create one. OData features additionally need published Gateway services."
---

# Working with SAP ABAP via abapctl

`abapctl` is the CLI you'll use for SAP's ADT development API and its OData runtime services. Reach for it whenever you need to read, navigate, assess, refactor, or write ABAP objects on a live SAP system, or query the business data a published service serves. It handles auth, CSRF, lock/unlock, activation, retries, and session reuse.

This `SKILL.md` is the **mental model + decision aid**: keep it loaded. Companion file **`reference.md`** is the **flag catalog and non-obvious behaviors**: read it when picking flags.

## First: get the live contract — never assume a command exists

This skill is a *map*, not the territory. The CLI evolves (new command groups, new flags) and this
file can lag. **Before you act, ground yourself in the live contract on the machine you're running:**

1. **`abapctl tools list --json`** — the authoritative machine-readable contract: every command with
   its parameters (positional/required/optional), and `readOnly` / `destructive` / `idempotent`
   annotations. **Run this first** to discover the real command set and to decide safety. Treat the
   `destructive` flag as binding regardless of what this skill says.
2. **`abapctl <group> --help`** and **`abapctl <group> <cmd> --help`** — exact flags, choices, and
   defaults for a specific command, when you need detail beyond the contract.
3. Only then consult the tables below as a shortcut for *which* command fits an intent.

**Do not invent command names, flags, or values from memory or from this skill alone.** If a command
or flag isn't in `tools list --json` / `--help` on this system, it does not exist here — adapt, don't
guess. If `tools list` shows a command this skill doesn't mention, trust `tools list`.

## Pick the right command (intent → command)

Scan this for the right command, then confirm its flags via `--help` / `tools list --json`. If your
intent isn't here, `abapctl tools list --json` is the complete set (~126 commands).

| Intent | Command | Notes |
|---|---|---|
| **Browse / understand** | | |
| Find an object by name pattern | `object search <pattern>` | wildcards, `--type` filter. **Default cap 50** (`--max-results N` to raise). If you get *exactly* 50 back, assume more exist and narrow/raise |
| What is this object, where does it live | `object info <name>` | metadata + includes |
| What's in this package | `object tree <pkg>` or `package list` | tree = objects, list = packages |
| Who depends on this symbol | `code references <obj>` + `code snippets <obj>` | where-used + context |
| Read a RAP behavior definition | `bdef get <name>` | metadata + source; `bdef listinterfaces <name>` for assigned BO interfaces |
| CDS annotation defs / entity fields / DDL source / test deps | `cds annotations` / `cds element-info <e>` / `cds repository-access <e>` / `cds test-dependencies <e>` | all read-only |
| What is at line X col Y | `code element-info <obj> --line N --col M` | type, docs, components |
| Jump to definition | `code definition <obj> --line N --col M` | navigate target |
| What can I type here (compiler-authoritative) | `code complete <obj> --line N --col M [--prefix <text>]` | hallucination guard |
| What ENHO is active on this base object | `enho on <obj>` | decoded source + position |
| Inventory ENHOs in system | `enho list [--customer-only] [--package X]` | migration-risk pile |
| **Read source** | | |
| Whole class incl. method bodies | `source get ZCL_FOO` | **no `--include`** — returns the full global class (DEFINITION + IMPLEMENTATION + all methods) |
| A class's LOCAL helper include | `source get ZCL_FOO --include implementations` | CCIMP/CCDEF/CCAU — local classes only, NOT the global methods; usually a near-empty stub. See "Class includes" below |
| Program / interface / FUGR include / DDLS body | `source get <name>` | single source, no `--include` |
| Translatable strings (TEXT-001, headings) | `text-elements get <obj> --category symbols\|selections\|headings` | divergent semantics, see below |
| Revision history | `object history <obj>` | |
| **Run checks** | | |
| Syntax of the active version | `check syntax <obj>` | |
| Syntax of unsaved local file | `check syntax <obj> --source ./edit.abap` | |
| ATC on one object | `check atc <obj>` | exit 1 = findings |
| ATC + per-finding fix metadata | `check atc-inspect <obj> -v <variant> [--filter any\|automatic\|manual\|pseudo\|ai\|all]` | read-only |
| Apply an ATC finding's quick fix | `check atc-fix <obj> -v <variant> [--apply [index]]` | preview by default; `--apply` writes (**destructive**). For cursor-position fixes use `refactor quickfix` |
| ABAP Unit | `check unit <obj>` | `--junit` for CI |
| CDS DDL syntax | `check cds-syntax <obj>` | |
| Propose / request an ATC exemption | `check atc-exempt-proposal <markerId>` / `check atc-exempt <markerId> --reason FPOS\|OTHR --justification "..."` | proposal read-only; exempt mutating |
| **Preview data** | | |
| One DDIC table | `data query <table> [--rows N] [--where "..."]` | |
| One CDS view | `data cds <view> [--rows N]` | |
| Arbitrary SQL | `data sql --query "SELECT ..."` or `--source file.sql` | freestyle |
| **Consume OData runtime (Gateway, the *consumer's* view, orthogonal to `data`)** | | |
| List the live OData service catalog | `odata list [-f <search>] [--v2-only\|--v4-only]` | V2+V4; per-catalog failure → warning; both gated → CAPABILITY_NOT_AVAILABLE (older stacks) |
| Inspect a service (entity sets, keys, props) | `odata metadata <service> [--raw]` | `<service>` = catalog name or full `/sap/opu/odata...` path |
| Query a published service's data | `odata query <service> <entityset> [--filter --select --top --skip --orderby --expand --count]` or `--key "<raw OData key>"` | the *consumer's* view; `$filter` syntax is SAP's (service decides what's filterable) |
| Publish / unpublish a service binding | `service publish <name>` / `service unpublish <name>` | `--protocol v2\|v4` (auto-detects from binding when omitted); publish before `service test` |
| Read a binding (protocol, published state, OData URL) | `service binding-details <name>` | read-only |
| **Did my publish actually work?** | `service test <binding> [--runtime-url <path>] [--protocol v2\|v4] [--rows N]` | resolve runtime URL → $metadata → smoke-query → PASS/FAIL stage-named verdict |
| **DDIC deep metadata** | | |
| Table fields (parsed DDL) | `ddic table <name>` | not on older/classic stacks |
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
| Create with body in one call | `create <type> <name> --package PKG --source ./body.abap` | 18 of 21 types accept `--source`; CLAS/FUGR accept *repeated* `--source` for multi-include |
| Create domain (typed) | `create domain Z_DOM --data-type CHAR --length 10 [--fixed-value A:B:Active]` | typed flags or `--source` (XOR) |
| Create data element (typed) | `create data-element Z_DTEL --domain Z_DOM --label-short ... --label-long ...` | typed flags or `--source` (XOR) |
| **Transport** | | |
| Does this object need a transport? Which one? | `transport info <obj>` | object → requirements |
| Open new transport | `transport create --description "..."` | |
| List my transports | `transport list` | |
| Detail of a specific transport NR | `transport get <NR>` | **NR → tasks/objects (NOT `info`)** |
| Release / delete a transport | `transport release <NR> -y` / `transport delete <NR> -y` | destructive |
| Delete a transport holding a stale lock | `transport delete <NR> --recursive -y` | `-r` strips the transport's objects (ADT removeobject) first, then deletes |
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
| Regenerate a package's SUMMARY.md | `clean-core report <SID/PKG>` | local only, from existing state |
| Cross-package summary | `clean-core executive` | |
| **Successor lookup** | | |
| Is `CL_GUI_ALV_GRID` deprecated? Successor? | `release-state lookup CL_GUI_ALV_GRID` | live CDS view; S/4 only |
| **System** | | |
| First-time config | `init --template` (agents) / `init` (humans) | then edit `.abapctl.json` |
| Add another connection | `config add-connection <name>` | interactive |
| What's the active connection | `config show` | |
| What ADT services does this system expose | `discover` / `tools coverage` | capability probe |
| SAP system facts (kernel, DB, OS, SID) | `system info` | universal — works on every release incl. classic stacks |
| Installed software components | `system components [-f <substr>]` | release / support-package / patch level |
| Run arbitrary ABAP | `run --source ./snippet.abap` | **mutating, treat as code execution** |
| Health check | `system-check [obj]` | `--connection-only` / `--read` / `--write` |
| **Operational diagnostics (`monitor`)** | | |
| ABAP runtime errors (ST22 short dumps) | `monitor dumps [id] [--from <d> --to <d>]` | list, or show one dump by id; read-only |
| System log (SM21) | `monitor syslog [--from <d> --to <d> --limit N --user U]` | **executes ABAP via classrun**; default window = today |
| Background jobs (SM37) | `monitor jobs [name] [count] [--status ... --user U --log]` | list/step-detail read-only; `--log` **executes ABAP** |
| Which monitoring feeds exist | `monitor list-feeds` | read-only capability probe |

## Always do this

1. **Pass `-c <name>`** unless you know `defaults.connection` is what you want.
2. **Pass `--json`** on every call. JSON → stdout. Progress/colors → stderr. Don't `2>&1` if piping to a JSON parser.
3. **Persist sessions**: `--session-file .abapctl/.session.json` (or set `defaults.session_file`). Saves seconds per call.
4. **Probe capability before assuming**: `discover -c <c>` (top level) or `tools coverage -c <c>` (registry vs system). Some endpoints don't exist on every release (older/classic stacks expose fewer; modern S/4HANA more).

## Agent-context note (subagents)

If you're a fixer/applier subagent spawned with sealed context, your cwd may not be the project root. **Before running any `abapctl` command, `cd` to the directory containing `.abapctl.json`.** Connection config resolves from cwd; a wrong cwd reads as "connection not configured" and looks like a SAP outage. If `.abapctl.json` isn't found in any ancestor, abort; don't guess.

## Safety model

| Class | Examples | Default |
|---|---|---|
| **read-only** | `object info/search/tree/path/types/inactive/history`, all `code *`, `source get`, all `check *` (except `atc-exempt`/`atc-fix`), all `data *`, all `ddic *`, all `cds *`, `enho on/list`, `release-state lookup`, `text-elements get`, `workspace status/diff/list`, `transport info/list/get`, `package list/exists/lookup`, `monitor dumps/list-feeds`, `discover`, `tools *` | Run freely |
| **mutating** | `object activate`, all `create *`, `transport create`, `refactor <x> --<action-flag>`, `service publish`, `workspace init/pull/add/remove/refresh`, `bdef create`, `check atc-exempt`, `run`, `monitor syslog`, `monitor jobs --log`, `clean-core report/executive`, `source format-settings --<flag>` | Confirm with user; preview with `--dry-run` if available. **`monitor syslog` and `monitor jobs --log` execute ABAP via classrun** (read-intent but they run code — treat like `run`) |
| **destructive** | `source put/push`, `object delete`, `text-elements put`, `check atc-fix --apply`, `workspace push/reset`, `clean-core apply`, `transport release/delete`, `service unpublish` | **Always confirm.** Pass `-y` only after explicit user OK |

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

### Class includes: `--include` is for LOCAL classes, NOT the global methods
**The single most misunderstood flag. Read this before editing any class.**

- `source get ZCL_FOO` **with NO `--include`** returns the **full global class** — `CLASS x
  DEFINITION … ENDCLASS.` *and* `CLASS x IMPLEMENTATION … ENDCLASS.` with every method body. This is
  what you edit to change a method. (Verified: no-`--include` reads `…/source/main`.)
- `--include implementations` (CCIMP) / `--include definitions` (CCDEF) / `--include testclasses`
  (CCAU) / `--include macros` (CCMAC) address the **local helper** includes — `lcl_*` classes and
  local types only. On a typical class these are a near-empty comment stub.
- **To edit a global method: round-trip the full main source (NO `--include`)** — `source get ZCL_FOO`
  → edit → `source put ZCL_FOO --file …` (no `--include`).
- **Footgun (breaks activation):** writing the global `CLASS x IMPLEMENTATION` block into
  `--include implementations` creates a *duplicate* `IMPLEMENTATION` → activation fails with
  `"CLASS … IMPLEMENTATION" may only occur once`, and the bad include **persists**, so even a later
  correct main push keeps failing until the include is cleared back to its stub. Only use
  `--include` when the file genuinely contains *local* helper classes.
- Valid values per command:
  - `source get/put`: `definitions` | `implementations` | `testclasses` (+ `macros`)
  - `code *`, `refactor *`, `object history`, `enho on`: above + `main`
- Other types (PROG, INTF, FUGR/FF, DDLS) ignore `--include` (single source).

### Multi-include `create class` and `create function-group`: repeated `--source`
Filename suffix dispatches to the right slot. Don't pass JSON metadata or wrap the calls; the CLI does it.

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
- `--category` is **REQUIRED** (`symbols` | `selections` | `headings`). No default; silent wrong-category writes too easy.
- `--file` REQUIRED.
- **No auto-activation.** Use `--activate` to opt in. (Probe-confirmed: text-elements don't need activation; this is intentional.)
- **Lock target is the textelements URI**, not the parent object URI; handled internally. Just don't be surprised if you see `/sap/bc/adt/textelements/programs/...` in logs.
- Plaintext format: `KEY=VALUE` per line. Symbols: 3-char keys; selections: 30-char limit; headings: fixed key set (LISTHEADER, COLUMNHEADER_1..4).

### `create <type> --source <path>` covers 18 of 21 types
Single `--source`, hard-coded channel per command:
- **source-main** (text/plain → `{root}/source/main`, 9): `program`, `interface`, `include`, `function-module`, `function-group-include`, `ddl-source`, `service-definition`, `access-control`, `metadata-extension`
- **root-xml** (application/* → root URL, 5): `message-class`, `auth-field`, `auth-object`, `domain`, `data-element`
- **CDS-DDL** (`define table`/`define structure` → `source/main`, 2): `table` (TABL/DT), `structure` (TABL/DS)

`create class` and `create function-group` use *repeated* `--source` (multi-include, see above) (+2).

The 3 that do NOT accept `--source`: `service-binding` (use `--service`), `package` (metadata only), `annotation-definition` (shell-only, deferred).

`create domain` and `create data-element` accept either typed flags OR `--source` (XOR; mutex error names the conflicting flags).

`function-module` and `function-group-include` require `--group <fugr>`.

`create annotation-definition` is shell-only (deferred; test users get HTTP 403).

### Positions are 1-based
`--line 1 --col 1` = first character of first line. Matches the ABAP editor. Don't zero-index.

### SID/PACKAGE for workspace and clean-core
- `workspace init ZFIN -c dev` (bare pkg in) → afterward addressed as `<SID>/ZFIN`. SID comes from `connections.<name>.sid`.
- `clean-core assess ZFIN -c dev` takes bare pkg (creates the SID dir).
- `clean-core prep|apply|report` and all `workspace pull|push|status|diff|...` take `SID/PKG`.
- `workspace list` (no arg) prints `SID/PACKAGE` of every workspace.

### Stable-source conventions
- `source format` reads stdin only; pipe in, don't pass a filename.
- `source format-settings` is read-only without flags (prints), mutating with `--style`/`--indentation` (writes system-level formatter settings; get user OK).
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
# NO --include: gets/puts the whole global class incl. method bodies. This is the
# normal way to edit a class method. Only add --include for LOCAL helper classes.
abapctl source get ZCL_FOO -c dev > zcl_foo.clas.abap
# edit zcl_foo.clas.abap
abapctl check syntax ZCL_FOO --source zcl_foo.clas.abap -c dev --json
abapctl source put ZCL_FOO --file zcl_foo.clas.abap --transport <NR> -c dev -y --json
abapctl check atc ZCL_FOO -c dev --json
```

**Safe refactor (evaluate → review → execute):**
```bash
abapctl refactor rename ZCL_FOO --line 42 --col 12 -c dev --json
# show output, get user approval
abapctl refactor rename ZCL_FOO --line 42 --col 12 \
  --new-name NEW --transport <NR> -c dev --json
```

**Publish → verify → consume an OData service** (the loop-closer):
```bash
abapctl service publish ZUI_FOO_SB -c dev --json          # make the binding a runtime service
abapctl service test ZUI_FOO_SB -c dev --json             # PASS: published + resolves + serves data?
# FAIL: verdict names the stage: resolve (pass --runtime-url) / metadata (MAINT_SERVICE) / query (S_SERVICE, ERROR_LOG)
abapctl odata query ZUI_FOO_SB Suppliers --top 5 -c dev   # then query the data a real consumer sees
```

**Build a service from scratch** (CDS view → service → binding → publish → test). CRITICAL ordering:
```bash
abapctl create ddl-source ZCC_V --package Z001 --transport <NR> --description "..." --source view.ddls -c dev
abapctl create service-definition ZCC_SD --package Z001 --transport <NR> --description "..." --source def.srvd -c dev
abapctl create service-binding ZCC_SB --package Z001 --transport <NR> --description "..." --service ZCC_SD --protocol v2 -c dev
# create service-binding AUTO-ACTIVATES by default. Only if you passed --no-activate must you run
# `object activate ZCC_SB` before publish (publish fails "does not exist" against an inactive binding).
abapctl service publish ZCC_SB -c dev
abapctl service test ZCC_SB -c dev --json
# Gotchas (live-proven): create ddl-source/service-definition REQUIRE --description (400 otherwise);
# the package must be a real dev package, not a structure package (409 "cannot contain development objects");
# publish needs the underlying CDS view to be OData-exposable or it fails at the SADL layer (CX_SADL_ASSERT), a CDS-modeling issue, not a tooling one.
```

**Successor lookup for a deprecated API** (live fallback when findings.json enrichment is missing):
```bash
abapctl release-state lookup CL_GUI_ALV_GRID -c dev --json
# { successors: [{ category: 'O', name: 'CL_SALV_TABLE', ... }], found: true }
# category 'O' = swap class A for class B
# category 'C' = use the named concept (e.g. "XCO Library")
# modern S/4HANA only; older/classic stacks return CAPABILITY_NOT_AVAILABLE
```

**ECC → S/4HANA migration assessment** (ENHO surface):
```bash
abapctl enho list --customer-only --type source-code -c <conn> --json
abapctl enho on SAPMV45A --adt-type PROG/P -c <conn> --json
# enho list  --type:     source-code, badi-impl, enhancement, enhancement-spot, badi-spot (default fanout = 3 impl types)
# enho on    --adt-type: PROG/P, CLAS/OC, FUGR/F (auto-discovered if omitted) — NOT --type
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
| 0 | A | Clean | none |
| 3 | B | Info only | none |
| 2 | C | Warnings | Optional |
| 1 | D | Errors | **Must fix**: blocks cloud |

## Failure decoder

| Symptom | Cause | Fix |
|---|---|---|
| 401/403 on first call | Password env var not exported in this shell | Re-export `$SAP_*_PASSWORD`; retry |
| 403 only on writes (read works) | Stale CSRF in persisted session | Delete `.abapctl/.session.json`; retry once |
| HTTP 423 / "object is locked" | Someone holds the lock; or `text-elements put` against parent URI | `object info <name> --json` to see holder. Don't force-unlock; surface to user |
| `transport info <NR>` returns OBJECT_NOT_FOUND | Wrong verb: `info` takes object name, not transport NR | Use `transport get <NR>` |
| "capability not available" / 404 on a registered command | Endpoint not on this release | `discover -c <c> --json`; or `tools coverage -c <c> --json` |
| Recursive `workspace init --depth N` pulled tens of thousands of objects | Sideways package refs explode at depth >1 | Use `init` per leaf package, or `--depth 1` |

## Don't

- Don't invent command names, flags, or enum values; verify against `tools list --json` (the contract) or `--help`. The CLI evolves and this skill can lag — the live contract wins.
- Don't pass `-y` silently on destructive commands. Confirm with the user first.
- Don't hardcode hosts, passwords, transports, or SIDs; read from config.
- Don't parse non-JSON stdout with regex; add `--json` and parse the object.
- Don't zero-index `--line` / `--col`.
- Don't `2>&1` when piping to `jq`; progress goes to stderr.
- Don't pre-flight a transport before a write; `source put` resolves it. (`transport get <NR>` to *check* a known NR is fine; don't *create* one speculatively.)
