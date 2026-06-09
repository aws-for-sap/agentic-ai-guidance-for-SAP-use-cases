# abapctl: flag catalog and behaviors

Companion to `SKILL.md`. Read this when you need to pick flags or you hit a non-obvious behavior. Every command supports `--json`. Most mutating commands support `--dry-run`. Position arguments are 1-based.

> **First-time agents**: read SKILL.md. The intent → command table there is faster than this catalog.

---

## Global flags

| Flag | Effect |
|---|---|
| `--json` | Structured JSON to stdout (use this on every call) |
| `--verbose` | Debug-level console output |
| `--log-file [path]` | Write log to file (default `.abapctl/logs/...`) |
| `--log-level <level>` | File log level: `debug` \| `info` \| `warning` \| `error` |
| `--session-file <path>` | Persist CSRF + cookies across invocations (or set `defaults.session_file`) |

## Exit codes

| Code | Meaning | Per-command examples |
|---|---|---|
| 0 | Clean success | `check atc` level A/B; `workspace status` clean; `clean-core assess` no C/D |
| 1 | Partial (data) | `check atc` C/D findings; `workspace diff` has diffs; `workspace push` some failed; `clean-core assess` C/D objects exist; `clean-core apply` any failed |
| 2 | Hard error | Auth, config, SAP fault, capability not available |

## Tool annotations

`abapctl tools list --json` returns annotations per command:

| Annotation | Meaning |
|---|---|
| `readOnly` | No SAP state change |
| `destructive` | Deletes or irreversibly mutates; needs `-y` and confirmation |
| `idempotent` | Safe to run multiple times with same effect |

`--dry-run` flips destructive/mutating to read-only-preview. Check per-command tables below for which commands support it.

---

## Command catalog

`-c, --connection <name>` selects the connection profile (omitted = `defaults.connection`). Required for SAP-facing commands.

### Standalone

| Command | Description | Safety |
|---|---|---|
| `init [--force] [--template]` | Create or extend `.abapctl.json`. Interactive in TTY; `--template` writes placeholder for headless agents | mutating, idempotent |
| `system-check [object] [--connection-only\|--read\|--write]` | Validate connectivity / read access / write access | read-only |
| `run --source <path>` | Execute ABAP statements; return text output | **mutating** (treat as code execution) |
| `discover [--raw]` | List ADT services on target system | read-only |
| `recipes [name] [--tags <csv>]` | List multi-step workflow guides | read-only |
| `tools list` | Print all commands with annotations | read-only |
| `tools coverage [--category X] [--unimplemented-only]` | Probe live discovery vs registered ENDPOINTS | read-only |

### config

| Command | Description |
|---|---|
| `config show` | Print active connection + all connections |
| `config set-connection <name>` | Switch active connection |
| `config add-connection [name]` | Interactively add a connection (writes back to `.abapctl.json`) |

### object

| Command | Flags | Safety |
|---|---|---|
| `object info <obj>` | `[--version active\|inactive]` | read-only |
| `object search <query>` | `[--type T] [--package P] [--max-results N]` | read-only |
| `object tree <pkg>` | — | read-only |
| `object path <obj>` | — | read-only |
| `object types` | — | read-only |
| `object inactive` | — | read-only |
| `object history <obj>` | `[--include T]` | read-only |
| `object activate <obj1> <obj2>...` | `[--dry-run]` | mutating, idempotent |
| `object delete <obj>` | `[--transport NR] [-y] [--dry-run]` | **destructive** |

### code (all read-only, all 1-based positions)

| Command | Required | Optional |
|---|---|---|
| `code definition <obj>` | `--line N --col M` | `[--end-col M] [--implementation] [--include T]` |
| `code element-info <obj>` | `--line N --col M` | `[--include T]` |
| `code references <obj>` | — | `[--line N --col M] [--include T]` |
| `code snippets <obj>` | — | `[--line N --col M] [--include T]` |
| `code complete <obj>` | `--line N --col M` | `[--include T] [--source <path>] [--prefix <text>] [--kind keyword\|method\|type\|all] [--max N]` |

`code complete` notes:
- Compiler-authoritative: same endpoint Eclipse uses for Ctrl+Space.
- `--prefix <text>` splices draft text at cursor and lets the server narrow by it. Best AI-agent hallucination guard.
- `--source <path>` sends a local file as the "current buffer" instead of the SAP-active source, to validate unsaved code.
- `--kind` filters: `keyword`, `method`, `type`, or `all` (default).

### refactor (read-only by default)

Mutating only when an action flag is passed.

| Command | Evaluate (read-only) | Execute (mutating; `--transport` typically required) |
|---|---|---|
| `refactor quickfix <obj>` | `--line N --col M [--include]` | `+ --apply <index>` |
| `refactor rename <obj>` | `--line N --col M [--end-col M] [--include]` | `+ --new-name <name> [--transport NR]` |
| `refactor extract-method <obj>` | `--start-line --start-col --end-line --end-col [--include]` | `+ --name <method> [--transport NR]` |
| `refactor move <obj1> <obj2>...` | — | `+ --package <target> [--transport NR]` |

### source

| Command | Required | Optional | Safety |
|---|---|---|---|
| `source get <obj>` | — | `[--include T] [--adt-type PROG/P\|...]` | read-only |
| `source put <obj>` | `--file <path>` | `[--include T] [--adt-type ...] [--transport NR] [--no-activate] [-y] [--dry-run]` | **destructive** |
| `source push <obj1> <obj2>...` | — | `[--dir <path>] [--transport NR] [--no-activate] [-y] [--dry-run]` | **destructive** |
| `source format` | (stdin) | — | read-only |
| `source format-settings` | — | `[--style <s>] [--indentation true\|false]` | read-only with no flags; mutating with flags (system-level) |

`--include` valid for class methods only: `definitions` \| `implementations` \| `testclasses`.

`--adt-type` resolves type ambiguity when the same name is shared (e.g. PROG/P vs FUGR/FF). Values: `PROG/P`, `CLAS/OC`, `INTF/OI`, `FUGR/F`, `FUGR/FF`, `DDLS/DF`, etc.

`source push --dir <path>` reads `<OBJNAME>.abap` files from that directory.

### text-elements (translatable strings)

For PROG / CLAS / FUGR. Three categories: `symbols` (TEXT-001), `selections` (selection-screen labels), `headings` (LISTHEADER + COLUMNHEADER_1..4).

| Command | Required | Optional | Safety |
|---|---|---|---|
| `text-elements get <obj>` | — | `[--category symbols\|selections\|headings] [--adt-type PROG/P\|CLAS/OC\|FUGR/F]` | read-only |
| `text-elements put <obj>` | `--category K --file <path>` | `[--adt-type T] [--transport NR] [--activate] [--dry-run]` | **destructive** |

**Diverges from `source put`:**
- `--category` REQUIRED on put (no default, because silent wrong-category writes are too easy).
- `--file` REQUIRED on put.
- **No auto-activation.** Pass `--activate` only if you actually need to.
- Lock target is the textelements URI (`/sap/bc/adt/textelements/{programs|classes|functiongroups}/{name}`), NOT the parent. Handled internally (visible in logs).

Plaintext format (round-trippable):
```
KEY=VALUE
KEY2=VALUE2
```
- symbols: 3-char keys (e.g. `001=Header`); optional `@MaxLength:N` directive.
- selections: 30-char label limit; optional `@DDICReference:NAME`.
- headings: fixed key set (LISTHEADER, COLUMNHEADER_1..COLUMNHEADER_4).

EC6: `text-elements get` returns empty array (endpoint family genuinely absent; convention, not bug); `text-elements put` returns 404.

### data (all read-only)

| Command | Required | Optional |
|---|---|---|
| `data query <table>` | — | `[--rows N] [--where <clause>]` |
| `data cds <view>` | — | `[--rows N]` |
| `data sql` | `--query <sql>` OR `--source <path>` | `[--rows N]` |

### check

| Command | Required | Optional | Safety |
|---|---|---|---|
| `check syntax <obj>` | — | `[--source <path>]` | read-only |
| `check cds-syntax <obj>` | — | `[--source <path>]` | read-only |
| `check atc <obj>` | — | `[-v <variant>]` | read-only |
| `check atc-quickfix <obj>` | `-v <variant>` | `[--filter any\|automatic\|manual\|pseudo\|ai\|all] [--max-verdicts N]` | read-only |
| `check unit <obj>` | — | `[--junit]` | read-only |
| `check atc-variants` | — | — | read-only |
| `check atc-exempt-proposal <markerId>` | — | — | read-only |
| `check atc-exempt <markerId>` | `--reason FPOS\|OTHR --justification <text>` | `[--approver <user>]` | mutating |

`check atc-quickfix` surfaces metadata `check atc` discards: per-finding documentation URL, fix-availability flags (`manual` / `automatic` / `pseudo` / `ai`), tags. Read-only: no fixes applied.

### transport

| Command | Description | Safety |
|---|---|---|
| `transport info <object>` | Object → requirements + open transports | read-only |
| `transport create --description <text>` | `[--ref <uri>] [--package P] [--dry-run]` | mutating |
| `transport list` | `[--user <name>]` | read-only |
| `transport get <number>` | NR → tasks + objects | read-only |
| `transport release <number>` | `[-y] [--ignore-locks] [--ignore-atc] [--dry-run]` | **destructive** |
| `transport delete <number>` | `[-y] [--dry-run]` | **destructive** |

**Don't confuse:** `info <object>` (object name → reqs) vs `get <NR>` (transport NR → detail).

### create

All: `create <type> <name> --package <pkg> [--description <text>] [--transport NR] [--dry-run] [-c <conn>]`. Mutating, not idempotent. `--dry-run` for safe preview.

**Standard types** (ABAP/CDS source-bearing) accept `--source <path>` + `--no-activate`:
- `class`, `interface`, `program`, `include`, `function-group`, `ddl-source`, `service-definition`, `access-control`, `metadata-extension`

**XML-bodied DDIC types** accept `--source <path>` for raw XML body + `--no-activate`:
- `message-class`, `auth-field`, `auth-object`, `domain`, `data-element`

**Multi-include** (repeated `--source`, filename suffix dispatches):

`create class`:
- `{name}.clas.abap` (main)
- `{name}.clas.definitions.abap`
- `{name}.clas.implementations.abap`
- `{name}.clas.macros.abap`
- `{name}.clas.testclasses.abap`

`create function-group`:
- `{name}.fugr.abap` (TOP)
- `{name}.fugr.{fmname}.func.abap` (function module)
- `{name}.fugr.{suffix}.reps.abap` (function group include)

Single-source CLAS convenience: 1 `--source` with non-matching filename → main + warning.

**Typed-flag types** accept typed flags OR `--source` (XOR; mutex error names the conflicting flags):

`create domain`:
- `--data-type <type>` (CHAR, NUMC, DEC, ...), `--length N`, `--decimals N`
- `--lowercase`, `--conversion-exit <fn>`
- `--value-table <name>` (mutually exclusive with `--fixed-value`)
- `--fixed-value <spec>` repeatable. Spec grammar (explicit empty segments):
  - `A` (low only)
  - `1:10:` (low + high)
  - `A::Active` (low + text)
  - `1:10:Range` (full)

`create data-element`:
- `--type domain | predefinedAbapType` (inferred from `--domain` or `--data-type` if omitted)
- `--domain <name>` XOR `--data-type <type> [--length N] [--decimals N]`
- `--label-short`, `--label-medium`, `--label-long`, `--label-heading` (10/20/40/55 char caps)
- `--search-help <name>`, `--search-help-parameter <name>`
- `--set-get-parameter <name>`

**Types with extra required flags:**
| Type | Extra |
|---|---|
| `function-module` | `--group <fugr>` |
| `function-group-include` | `--group <fugr>` |
| `service-binding` | `--service <srvd>` |
| `package` | `--description <text>` (required); `[--swcomp <name>] [--transport-layer <name>] [--package-type development\|structure\|main] [--super-package <name>] [--record-changes]` |

**Capability-gated:** DDIC create on EC6. The write capability check is content-type aware (rejects systems advertising the term but only v1+xml dialect). Modern systems (S4H/S4J/A4H) all green.

**Deferred:** `create annotation-definition` is shell-only. Test users get HTTP 403 on all 3 modern systems. Reserved for future authorized testing.

### package

| Command | Flags | Safety |
|---|---|---|
| `package list` | `[-p <prefix...>]` | read-only |
| `package exists <name>` | — | read-only |
| `package lookup <type>` | `[--filter <pattern>]` | read-only |

`package lookup` types: `transportlayers`, `softwarecomponents`, `applicationcomponents`, `translationrelevances`.

### cds (all read-only)

| Command | Flags |
|---|---|
| `cds annotations` | — |
| `cds element-info <entity>` | `[--follow-associations] [--no-extension-views] [--no-secondary-objects]` |
| `cds repository-access <entity>` | — |
| `cds test-dependencies <entity>` | `[--level hierarchy\|unit] [--associations]` |

### ddic (all read-only)

`ddic <subcommand> <name>` where subcommand is one of:

`table-settings`, `data-element`, `domain`, `table-type`, `lock-object`, `type-group`, `structure`, `view`, `table`

`ddic table` returns parsed DDL fields; not available on EC6.

### service

| Command | Flags | Safety |
|---|---|---|
| `service publish <name>` | `[--version V] [--dry-run]` | mutating, idempotent |
| `service unpublish <name>` | `[-y] [--version V] [--dry-run]` | **destructive**, idempotent |

### bdef

| Command | Flags | Safety |
|---|---|---|
| `bdef get <name>` | — | read-only |
| `bdef create <name>` | `--package P [--description T] [--implementation-type managed\|unmanaged] [--file <path>] [--transport NR] [--dry-run]` | mutating |

### enho (ECC → S/4HANA migration surface)

| Command | Description | Safety |
|---|---|---|
| `enho on <obj>` | List ENHOs active on a base object (decoded ABAP source + position + switch state) | read-only |
| `enho list` | System-wide / package-scoped ENHO inventory | read-only |

`enho on` flags: `[--type <adtType>] [--include T] [--no-decode]`.
- `--include` valid only for CLAS/OC.
- `--no-decode` keeps base64.

`enho list` flags: `[--type <kind>] [--customer-only] [--package <name>] [--max-results N]`.
- `--type` friendly values:
  - `source-code` (ENHO/XHH)
  - `badi-impl` (ENHO/XHB)
  - `enhancement` (ENHO/XH)
  - `enhancement-spot` (ENHS/XS)
  - `badi-spot` (ENHS/XSB)
- Default fanout (no `--type`) covers the 3 ENHO impl types; spots opt-in via explicit `--type`.
- `--customer-only` runs Z*/Y* prefix queries. Namespaced customers (`/ACME/*`) require `--package` or downstream JSON filtering.
- `--max-results N` is per search bucket, **NOT** a total cap.

### release-state

| Command | Description | Safety |
|---|---|---|
| `release-state lookup <object>` | Live CDS-view query against `I_APISWITHCLOUDDEVSUCCESSOR` | read-only |

Output shape per case:

| Case | `successors[]` | Plain text |
|---|---|---|
| Single object successor | 1 entry, `category: 'O'`, `name: 'CL_FOO'` | `→ CL_FOO (CLAS)` |
| Multiple object successors | N entries, all `category: 'O'` | one `→ ...` line per |
| Concept successor (no concrete class) | 1 entry, `category: 'C'`, `name: ''`, `conceptName: 'XCO Library'` | `→ concept: XCO Library` |
| Not in catalog | `[]`, `found: false` | `<NAME>: not found` |

Exit 0 for found AND not-found (lookup completed). Exit 2 if system lacks the view (EC6 → `CAPABILITY_NOT_AVAILABLE`).

### reference

| Command | Description |
|---|---|
| `reference update` | Download SAP API reference data from GitHub |

### clean-core

| Command | Target | Flags | Safety |
|---|---|---|---|
| `clean-core assess <pkg>` | bare pkg | `[-v <variant>] [--force]` | read-only (writes local reports) |
| `clean-core report <SID/PKG>` | SID/PKG | — | mutating (local only) |
| `clean-core executive` | — | `[--sid <sid>]` | mutating (local only) |
| `clean-core prep <SID/PKG>` | SID/PKG | — | mutating (local only) |
| `clean-core apply <SID/PKG>` | SID/PKG | `[-y] [--object <name>] [--dry-run]` | **destructive** |

`assess` takes a bare pkg because it CREATES the SID directory. `prep` / `apply` / `report` come after, so they take `SID/PKG`.

### workspace

Always `<package>` arg = `SID/PACKAGE`, except `init` (bare pkg → SID derived from `-c`).

| Command | Required | Optional | Safety |
|---|---|---|---|
| `workspace init <pkg>` | — | `[--source-dir <name>] [--transport NR] [--filter <pat>] [--type <csv>] [--recursive] [--depth N] [--no-download] [--concurrency N] [--force] [--dry-run]` | mutating |
| `workspace status [SID/PKG]` | — | — | read-only |
| `workspace list [SID/PKG]` | — | — | read-only |
| `workspace pull <SID/PKG> [obj...]` | — | `[--force] [--dry-run]` | mutating, idempotent |
| `workspace push <SID/PKG> [obj...]` | — | `[--force] [--no-activate] [--transport NR] [-y] [--dry-run]` | **destructive** |
| `workspace diff <SID/PKG> [obj]` | — | `[--sap] [--name-only]` | read-only |
| `workspace add <SID/PKG> <obj...>` | — | `[--concurrency N] [--dry-run]` | mutating, idempotent |
| `workspace remove <SID/PKG> <obj...>` | — | `[--delete] [-y] [--dry-run]` | mutating |
| `workspace reset <SID/PKG> [obj...]` | — | `[-y] [--sap] [--dry-run]` | **destructive**, idempotent |
| `workspace refresh <SID/PKG>` | — | `[--recursive] [--depth N] [--filter <pat>] [--type <csv>] [--concurrency N] [--force] [--dry-run]` | mutating |

`workspace init --recursive --depth N` warning: at depth >1, sideways package refs explode: a single `--depth 2` run can pull thousands of metadata-only objects. Prefer `--depth 1` per package, or invoke `init` per leaf package.

`workspace push` activation: enabled by default; `--no-activate` skips. `workspace pull` does not write to SAP, only to local disk.

`workspace diff --sap` compares local against live SAP source (re-download); without it, compares local against the baseline captured at last sync.

---

## Behaviors (the things that surprise you)

### Class includes
- `source get ZCL_FOO` → header only (no method bodies). To get bodies: `source get ZCL_FOO --include implementations`.
- `--include` valid values per command:
  - `source get/put`: `definitions` \| `implementations` \| `testclasses`
  - `code *`, `refactor *`, `object history`, `enho on`: above + `main`
- Other types (PROG, INTF, FUGR/FF, DDLS) ignore `--include` (single source).

### Transport disambiguation
- `transport info <object>`: object name → requirements + open transports.
- `transport get <NR>`: transport number → tasks/objects.
- Mixing them up returns `OBJECT_NOT_FOUND`.

### Stale CSRF on persisted session
- First **write** after the session expires returns 403 Forbidden. Reads still work because they don't need CSRF.
- Recovery: delete `.abapctl/.session.json` and retry once; abapctl re-logs in transparently.
- Looks like an auth bug or SAP outage but is just session staleness.

### `text-elements put` lock target
The lock target is the **textelements URI** (`/sap/bc/adt/textelements/{programs|classes|functiongroups}/{name}`), NOT the parent object URI. Locking the parent yields a valid handle but the PUT 423s. The CLI handles this internally; agents only need to know the symptom.

### Activation defaults
- `source put` / `source push` / `workspace push` / all `create *` (where source is provided) / `bdef create` activate by default. `--no-activate` to skip.
- `text-elements put` does NOT activate by default. Pass `--activate` to opt in.

### Capability gating
- `release-state lookup`: S/4 only (S4H/S4J/A4H). EC6 → `CAPABILITY_NOT_AVAILABLE`.
- `ddic table`: not available on EC6.
- `create annotation-definition`: shell-only (test users 403).
- DDIC create: EC6 capability-gated via content-type-aware check.
- `text-elements get` on EC6: returns empty array (endpoint family absent).
- Batch activation (`activation/runs`): S4H + A4H only.

### Position semantics
- `--line` and `--col` are **1-based**, matching ABAP editor. `--line 1 --col 1` = first char.
- Don't zero-index from JS/Python conventions.

### Output streams
- JSON → stdout. Progress and ANSI-colored retry messages → stderr.
- **Never `2>&1`** when piping to `jq` / a JSON parser.

---

## Workflow chains

### Understand → assess impact → change

```bash
abapctl code definition ZCL_FOO --line 10 --col 5 -c dev --json
abapctl code element-info ZCL_FOO --line 10 --col 5 -c dev --json
abapctl code references ZCL_FOO -c dev --json
abapctl code snippets ZCL_FOO -c dev --json
```

### Source round-trip

```bash
abapctl source get ZCL_FOO --include implementations -c dev > impl.abap
# edit
abapctl check syntax ZCL_FOO --source impl.abap -c dev --json
abapctl source put ZCL_FOO --file impl.abap --include implementations \
  --transport <NR> -c dev -y --json
abapctl check atc ZCL_FOO -c dev --json
```

### Safe refactoring

```bash
# evaluate (read-only)
abapctl refactor rename ZCL_FOO --line 10 --col 5 -c dev --json
# review, then execute (mutating)
abapctl refactor rename ZCL_FOO --line 10 --col 5 \
  --new-name NEW --transport DEVK900001 -c dev --json
```

### Clean Core pipeline

```bash
abapctl package list -c dev --json
abapctl clean-core assess ZFINANCE -c dev --json     # creates <SID>/ZFINANCE/
abapctl clean-core executive --json
abapctl clean-core prep S4H/ZFINANCE -c dev
# fix sources
abapctl clean-core apply S4H/ZFINANCE -c dev --dry-run --json
abapctl clean-core apply S4H/ZFINANCE -c dev -y --json
```

### Multi-include create (CLAS)

```bash
abapctl create class ZCL_X --package ZDEV --description "Calc" -c dev \
  --source ./out/ZCL_X.clas.abap \
  --source ./out/ZCL_X.clas.definitions.abap \
  --source ./out/ZCL_X.clas.implementations.abap \
  --source ./out/ZCL_X.clas.testclasses.abap \
  --transport <NR> --json
```

### Translatable strings round-trip

```bash
abapctl text-elements get ZPROG --category symbols -c dev > sym.txt
# edit sym.txt
abapctl text-elements put ZPROG --category symbols --file sym.txt \
  --transport <NR> -c dev --json
# Note: no auto-activation. Pass --activate if needed.
```

### Workspace (git-like)

```bash
abapctl workspace init ZFIN -c dev --recursive --depth 1 --json
abapctl workspace status S4H/ZFIN --json
abapctl workspace diff S4H/ZFIN --json
abapctl workspace push S4H/ZFIN -c dev --dry-run --json
abapctl workspace push S4H/ZFIN -c dev -y --json
```

### Runtime verification

```bash
cat > /tmp/verify.abap <<'ABAP'
DATA(d) = cl_abap_typedescr=>describe_by_name( 'ZCL_FOO' ).
out->write( d->get_relative_name( ) ).
ABAP
abapctl run --source /tmp/verify.abap -c dev --json
```

---

## JSON output contract

All commands with `--json` return structured JSON to stdout.

**Success patterns:**

```jsonc
// check atc
{ "object": "ZFOO", "ok": false, "level": "D", "findings": [...] }

// check syntax
{ "object": "ZFOO", "ok": true, "messages": [] }

// check unit
{ "object": "ZFOO", "ok": true, "summary": { "total": 5, "passed": 5, "failed": 0 }, "tests": [...] }

// system-check
{ "ok": true, "checks": [{ "name": "connection", "status": "pass" }, ...] }

// run (ok)
{ "object": "...", "ok": true, "output": "..." }
// run (runtime error)
{ "object": "...", "ok": false, "output": "", "error": "Runtime error in line 5" }

// workspace push
{ "pushed": 3, "failed": 0, "skipped": 1, "interrupted": false }

// release-state lookup
{ "predecessor": "CL_GUI_ALV_GRID", "found": true,
  "successors": [{ "category": "O", "name": "CL_SALV_TABLE", "type": "CLAS" }],
  "releaseState": "Use system internally", "source": "I_APISWITHCLOUDDEVSUCCESSOR",
  "warnings": [] }
```

**Error pattern** (non-zero exit):

```jsonc
{ "ok": false, "error": { "code": "SAP_HTTP_ERROR", "message": "401 Unauthorized" } }
```

**Exit code semantics by command group:**

| Command | Exit 0 | Exit 1 | Exit 2 |
|---|---|---|---|
| `check atc` | Level A/B | Level C/D (findings) | SAP error |
| `check unit` | All pass | Test failures | SAP error |
| `workspace status` | No modifications | Modified/missing files | Error |
| `workspace diff` | No differences | Differences found | Error |
| `workspace push` | All pushed | Any failed/interrupted | Error |
| `clean-core assess` | All A/B | C/D findings | Error |
| `clean-core apply` | All applied | Any failed | Error |
| `release-state lookup` | Lookup completed (found OR not found) | (N/A) | Capability not available |

---

## Configuration

Lives in `.abapctl.json` at project root. Create with `abapctl init` (interactive) or `abapctl init --template` (headless). Inspect with `abapctl config show --json`.

```jsonc
{
  "connections": {
    "dev": {
      "host": "sap-host.example.com",
      "port": 44300,
      "client": "100",
      "secure": true,
      "username": "DEVELOPER",
      "password_env": "SAP_DEV_PASSWORD",
      "language": "EN"
    }
  },
  "defaults": {
    "connection": "dev",
    "workspace_dir": "abap",
    "session_file": ".abapctl/.session.json"
  }
}
```

`-c <name>` selects the connection; `defaults.connection` is used when omitted. Passwords come from env vars named in `password_env`.

Directory layout:
- `.abapctl/` (fixed): `reference/` (`sap-api-reference/`, `cc-kb/`), `logs/`, `.session.json`. Like `.git/` (don't put project files here).
- `workspace_dir` (`"abap"`): ABAP source at `{SID}/{PACKAGE}/`; manifests at `.manifests/{SID}/{PACKAGE}/`.
- `clean_core.output_dir` (`"clean-core"`): assessment output at `{SID}/{PACKAGE}/`.
