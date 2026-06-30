# abapctl: flag catalog and non-obvious behaviors

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
| `object search <query>` | `[--type T] [--package P] [--max-results N]` | read-only; **default cap 50**. Exactly-50 results means probably truncated; raise `--max-results` or narrow with `--type`/`--package` |
| `object tree <pkg>` | none | read-only |
| `object path <obj>` | none | read-only |
| `object types` | none | read-only |
| `object inactive` | none | read-only |
| `object history <obj>` | `[--include T]` | read-only |
| `object activate <obj1> <obj2>...` | `[--dry-run]` | mutating, idempotent |
| `object delete <obj>` | `[--transport NR] [-y] [--dry-run]` | **destructive** |

### code (all read-only, all 1-based positions)

| Command | Required | Optional |
|---|---|---|
| `code definition <obj>` | `--line N --col M` | `[--end-col M] [--implementation] [--include T]` |
| `code element-info <obj>` | `--line N --col M` | `[--include T]` |
| `code references <obj>` | none | `[--line N --col M] [--include T]` |
| `code snippets <obj>` | none | `[--line N --col M] [--include T]` |
| `code complete <obj>` | `--line N --col M` | `[--include T] [--source <path>] [--prefix <text>] [--kind keyword\|method\|type\|all] [--max N]` |

`code complete` notes:
- Compiler-authoritative: same endpoint Eclipse uses for Ctrl+Space.
- `--prefix <text>` splices draft text at cursor and lets the server narrow by it. Best AI-agent hallucination guard.
- `--source <path>` sends a local file as the "current buffer" instead of the SAP-active source; validates unsaved code.
- `--kind` filters: `keyword`, `method`, `type`, or `all` (default).

### refactor (read-only by default)

Mutating only when an action flag is passed.

| Command | Evaluate (read-only) | Execute (mutating; `--transport` typically required) |
|---|---|---|
| `refactor quickfix <obj>` | `--line N --col M [--include]` | `+ --apply <index>` |
| `refactor rename <obj>` | `--line N --col M [--end-col M] [--include]` | `+ --new-name <name> [--transport NR]` |
| `refactor extract-method <obj>` | `--start-line --start-col --end-line --end-col [--include]` | `+ --name <method> [--transport NR]` |
| `refactor move <obj1> <obj2>...` | none | `+ --package <target> [--transport NR]` |

### source

| Command | Required | Optional | Safety |
|---|---|---|---|
| `source get <obj>` | none | `[--include T] [--adt-type PROG/P\|...]` | read-only |
| `source put <obj>` | `--file <path>` | `[--include T] [--adt-type ...] [--transport NR] [--no-activate] [-y] [--dry-run]` | **destructive** |
| `source push <obj1> <obj2>...` | none | `[--dir <path>] [--transport NR] [--no-activate] [-y] [--dry-run]` | **destructive** |
| `source format` | (stdin) | none | read-only |
| `source format-settings` | none | `[--style <s>] [--indentation true\|false]` | read-only with no flags; mutating with flags (system-level) |

`--include` valid for class methods only: `definitions` \| `implementations` \| `testclasses`.

`--adt-type` resolves type ambiguity when the same name is shared (e.g. PROG/P vs FUGR/FF). Values: `PROG/P`, `CLAS/OC`, `INTF/OI`, `FUGR/F`, `FUGR/FF`, `DDLS/DF`, etc.

`source push --dir <path>` reads `<OBJNAME>.abap` files from that directory.

### text-elements (translatable strings)

For PROG / CLAS / FUGR. Three categories: `symbols` (TEXT-001), `selections` (selection-screen labels), `headings` (LISTHEADER + COLUMNHEADER_1..4).

| Command | Required | Optional | Safety |
|---|---|---|---|
| `text-elements get <obj>` | none | `[--category symbols\|selections\|headings] [--adt-type PROG/P\|CLAS/OC\|FUGR/F]` | read-only |
| `text-elements put <obj>` | `--category K --file <path>` | `[--adt-type T] [--transport NR] [--activate] [--dry-run]` | **destructive** |

**Diverges from `source put`:**
- `--category` REQUIRED on put (no default; silent wrong-category writes too easy).
- `--file` REQUIRED on put.
- **No auto-activation.** Pass `--activate` only if you actually need to.
- Lock target is the textelements URI (`/sap/bc/adt/textelements/{programs|classes|functiongroups}/{name}`), NOT the parent. Handled internally; visible in logs.

Plaintext format (round-trippable):
```
KEY=VALUE
KEY2=VALUE2
```
- symbols: 3-char keys (e.g. `001=Header`); optional `@MaxLength:N` directive.
- selections: 30-char label limit; optional `@DDICReference:NAME`.
- headings: fixed key set (LISTHEADER, COLUMNHEADER_1..COLUMNHEADER_4).

On older/classic stacks: `text-elements get` returns empty array (endpoint family genuinely absent: convention, not bug); `text-elements put` returns 404.

### data (all read-only)

| Command | Required | Optional |
|---|---|---|
| `data query <table>` | none | `[--rows N] [--where <clause>]` |
| `data cds <view>` | none | `[--rows N]` |
| `data sql` | `--query <sql>` OR `--source <path>` | `[--rows N]` |

`data *` is the **design-time** preview (ADT datapreview). For the runtime consumer's view over the Gateway, use `odata *` below (orthogonal).

### odata (consume OData runtime services, all read-only)

The *consumer's* view over the Gateway (`/sap/opu/odata`, `/sap/opu/odata4`), distinct from `data *` (design-time ADT preview).

| Command | Required | Optional |
|---|---|---|
| `odata list` | none | `[-f <search>] [--v2-only\|--v4-only]` |
| `odata metadata <service>` | none | `[--raw]` |
| `odata query <service> <entityset>` | none | `[--filter F] [--select S] [--top N] [--skip N] [--orderby O] [--expand E] [--count] [--key "<raw OData key>"]` |

- `odata list`: live V2+V4 catalog. Per-catalog failure → warning (not fatal); both 403/404 → `CAPABILITY_NOT_AVAILABLE` (older stacks).
- `<service>`: a catalog technical name, or a full `/sap/opu/odata...` path verbatim.
- `--key` is a single-entity read: allows `--select`/`--expand`, rejects collection-only options.
- `$filter` syntax is SAP's; the service decides what's filterable. V2/V4 divergence (`$count` / `$search`) handled internally; output normalized to `{rows, count?}`.
- SAP-aware error hints: 404 catalog → SICF; 404 service → `/IWFND/MAINT_SERVICE`; 403 → SU53 / S_SERVICE; 5xx → `/IWFND/ERROR_LOG` / ST22.

### check

| Command | Required | Optional | Safety |
|---|---|---|---|
| `check syntax <obj>` | none | `[--source <path>]` | read-only |
| `check cds-syntax <obj>` | none | `[--source <path>]` | read-only |
| `check atc <obj>` | none | `[-v <variant>]` | read-only |
| `check atc-inspect <obj>` | `-v <variant>` | `[--filter any\|automatic\|manual\|pseudo\|ai\|all] [--max-verdicts N]` | read-only |
| `check atc-fix <obj>` | `-v <variant>` | `[--apply [index]] [--no-activate] [--transport NR] [--max-verdicts N] [--dry-run]` | preview default; `--apply` is **destructive** |
| `check unit <obj>` | none | `[--junit]` | read-only |
| `check atc-variants` | none | none | read-only |
| `check atc-exempt-proposal <markerId>` | none | none | read-only |
| `check atc-exempt <markerId>` | `--reason FPOS\|OTHR --justification <text>` | `[--approver <user>]` | mutating |

`check atc-inspect` surfaces metadata `check atc` discards: per-finding documentation URL, fix-availability flags (`manual` / `automatic` / `pseudo` / `ai`), tags. Read-only; no fixes applied. `check atc-fix` is the *apply* counterpart — preview by default, `--apply [index]` writes (bare `--apply` = all non-conflicting fixes, `--apply N` = only the Nth). For cursor-position fixes use `refactor quickfix` instead. `--all` (bulk auto-quickfix across the worklist) is not yet implemented.

### system (SAP system facts)

| Command | Optional | Safety |
|---|---|---|
| `system info` | none | read-only |
| `system components` | `[-f\|--filter <substr>]` | read-only |

`system info` → kernel release/PL, DB system/release/schema, OS, app server, derived SID, unicode. `system components` → installed software components (release / support package / patch level / description); `-f` is a case-insensitive substring over name+description. Both are **universal** (work on every release incl. classic stacks); no capability gate. CRITICAL: `system info`'s `sapSystemId` is the kernel system *number*, not the SID — the real SID is best-effort `derivedSid` from the app-server name.

### monitor (operational diagnostics)

| Command | Required | Optional | Safety |
|---|---|---|---|
| `monitor list-feeds` | none | none | read-only (capability probe) |
| `monitor dumps [id]` | none | `[--from <date>] [--to <date>]` | read-only |
| `monitor syslog` | none | `[--from <date>] [--to <date>] [--limit N] [--user <name>]` | **executes ABAP via classrun** (read-intent, but runs code) |
| `monitor jobs [jobname] [jobcount]` | none | `[--name <pat>] [--user <u>] [--status failed\|finished\|running\|scheduled\|ready] [--from <date>] [--to <date>] [--limit N] [--log]` | list/step-detail read-only (data sql); `--log` **executes ABAP via classrun** |

ST22 short dumps / SM21 system log / SM37 background jobs. `dumps`/`jobs` list-and-detail are read-only (data sql); `syslog` and `jobs --log` run ABAP via classrun, so treat them like `run`. Dates accept ISO (`2026-06-29`) or SAP (`YYYYMMDD[HHMMSS]`). `syslog`/`jobs` default window is **today in the client timezone** — pass `--from/--to` explicitly if the SAP system is in another timezone. `<jobcount>` is SAP's 8-char run id, taken from the list output.

### transport

| Command | Description | Safety |
|---|---|---|
| `transport info <object>` | Object → requirements + open transports | read-only |
| `transport create --description <text>` | `[--ref <uri>] [--package P] [--dry-run]` | mutating |
| `transport list` | `[--user <name>]` | read-only |
| `transport get <number>` | NR → tasks + objects | read-only |
| `transport release <number>` | `[-y] [--ignore-locks] [--ignore-atc] [--dry-run]` | **destructive** |
| `transport delete <number>` | `[-r\|--recursive] [-y] [--dry-run]` | **destructive** |

**Don't confuse:** `info <object>` (object name → reqs) vs `get <NR>` (transport NR → detail).

`transport delete --recursive` (`-r`) first removes the transport's objects (ADT removeobject) so a transport holding a stale lock on a deleted object can be cleaned up. Fail-fast: a failed removeobject aborts before the DELETE. Refuses only a known-released (`status==='R'`) transport.

### create

All: `create <type> <name> --package <pkg> [--description <text>] [--transport NR] [--dry-run] [-c <conn>]`. Mutating, not idempotent. `--dry-run` for safe preview.

**Standard types** (ABAP/CDS source-bearing, accept `--source <path>` + `--no-activate`):
- `class`, `interface`, `program`, `include`, `function-group`, `ddl-source`, `service-definition`, `access-control`, `metadata-extension`

**CDS-DDL data-definition types** (accept `--source <path>` with `define table`/`define structure` + `--no-activate`):
- `table` (TABL/DT, capability-gated on older stacks), `structure` (TABL/DS, works broadly)

**XML-bodied DDIC types** (accept `--source <path>` for raw XML body + `--no-activate`):
- `message-class`, `auth-field`, `auth-object`, `domain`, `data-element`

(18 of 21 types accept `--source`; `class`/`function-group` via repeated multi-include. The 3 without: `service-binding`, `package`, `annotation-definition`.)

**Multi-include** (repeated `--source`, filename suffix dispatches):

`create class`:
- `{name}.clas.abap`: main
- `{name}.clas.definitions.abap`
- `{name}.clas.implementations.abap`
- `{name}.clas.macros.abap`
- `{name}.clas.testclasses.abap`

`create function-group`:
- `{name}.fugr.abap`: TOP
- `{name}.fugr.{fmname}.func.abap`: function module
- `{name}.fugr.{suffix}.reps.abap`: function group include

Single-source CLAS convenience: 1 `--source` with non-matching filename → main + warning.

**Typed-flag types** (accept typed flags OR `--source`, XOR; mutex error names the conflicting flags):

`create domain`:
- `--data-type <type>` (CHAR, NUMC, DEC, ...), `--length N`, `--decimals N`
- `--lowercase`, `--conversion-exit <fn>`
- `--value-table <name>` (mutually exclusive with `--fixed-value`)
- `--fixed-value <spec>` repeatable. Spec grammar (explicit empty segments):
  - `A`: low only
  - `1:10:`: low + high
  - `A::Active`: low + text
  - `1:10:Range`: full

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
| `service-binding` | `--service <srvd>` `[--protocol v2\|v4]` (default v2) `[--binding-category ui\|webapi]` (default by protocol; requires `--protocol`). **Auto-activates by default** so the binding is immediately publishable; `--no-activate` leaves it inactive (then `object activate <name>` before `service publish`, else publish fails "does not exist") |
| `package` | `--description <text>` (required); `[--swcomp <name>] [--transport-layer <name>] [--package-type development\|structure\|main] [--super-package <name>] [--record-changes]` |

**Capability-gated:** DDIC create on older/classic stacks; the write capability check is content-type aware (rejects systems advertising the term but only v1+xml dialect). Modern S/4HANA systems all green.

**Deferred:** `create annotation-definition` is shell-only; test users get HTTP 403 on all 3 modern systems. Reserved for future authorized testing.

### package

| Command | Flags | Safety |
|---|---|---|
| `package list` | `[-p <prefix...>]` | read-only |
| `package exists <name>` | none | read-only |
| `package lookup <type>` | `[--filter <pattern>]` | read-only |

`package lookup` types: `transportlayers`, `softwarecomponents`, `applicationcomponents`, `translationrelevances`.

### cds (all read-only)

| Command | Flags |
|---|---|
| `cds annotations` | none |
| `cds element-info <entity>` | `[--follow-associations] [--no-extension-views] [--no-secondary-objects]` |
| `cds repository-access <entity>` | none |
| `cds test-dependencies <entity>` | `[--level hierarchy\|unit] [--associations]` |

### ddic (all read-only)

`ddic <subcommand> <name>` where subcommand is one of:

`table-settings`, `data-element`, `domain`, `table-type`, `lock-object`, `type-group`, `structure`, `view`, `table`

`ddic table` returns parsed DDL fields; not available on older/classic stacks.

### service

| Command | Flags | Safety |
|---|---|---|
| `service publish <name>` | `[--protocol v2\|v4] [--version V] [--dry-run]` | mutating, idempotent |
| `service unpublish <name>` | `[--protocol v2\|v4] [-y] [--version V] [--dry-run]` | **destructive**, idempotent |
| `service binding-details <name>` | none | read-only |
| `service test <name>` | `[--runtime-url <path>] [--protocol v2\|v4] [--rows N]` | read-only |

`--protocol` routes to the v2/v4 publish endpoint; omitted = auto-detect from the binding's declared version. Distinct from `--version` (= serviceversion).

`service test` resolves a published binding to its runtime URL (`--runtime-url` override → V2 convention → catalog match → V4 convention), fetches `$metadata`, and smoke-queries the first entity set. Verdict `{published, protocol, runtimeUrl, entitySet?, rowCount?, ok, stage?}`. Exit: ok→0, not-published→1, not-found/failure→2. `binding.odataUrl` is an ADT path (406 at runtime); never used for the query; resolution is separate.

### bdef

| Command | Flags | Safety |
|---|---|---|
| `bdef get <name>` | none | read-only |
| `bdef create <name>` | `--package P [--description T] [--implementation-type managed\|unmanaged] [--file <path>] [--transport NR] [--dry-run]` | mutating |
| `bdef listinterfaces <name>` | none | read-only (assigned BO interfaces; usually empty; 404 → `[]` on older stacks) |

### enho (ECC → S/4HANA migration surface)

| Command | Description | Safety |
|---|---|---|
| `enho on <obj>` | List ENHOs active on a base object: decoded ABAP source + position + switch state | read-only |
| `enho list` | System-wide / package-scoped ENHO inventory | read-only |

`enho on` flags: `[--adt-type <adtType>] [--include T] [--no-decode]`.
- `--adt-type` (e.g. `PROG/P`, `CLAS/OC`, `FUGR/F`) — auto-discovered if omitted. NOT `--type` (that's the `enho list` flag).
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

Exit 0 for found AND not-found (lookup completed). Exit 2 if system lacks the view (older/classic stacks → `CAPABILITY_NOT_AVAILABLE`).

### reference

| Command | Description |
|---|---|
| `reference update` | Download SAP API reference data from GitHub |

### clean-core

| Command | Target | Flags | Safety |
|---|---|---|---|
| `clean-core assess <pkg>` | bare pkg | `[-v <variant>] [--force]` | read-only (writes local reports) |
| `clean-core report <SID/PKG>` | SID/PKG | none | mutating (local only) |
| `clean-core executive` | none | `[--sid <sid>]` | mutating (local only) |
| `clean-core prep <SID/PKG>` | SID/PKG | none | mutating (local only) |
| `clean-core apply <SID/PKG>` | SID/PKG | `[-y] [--object <name>] [--dry-run]` | **destructive** |

`assess` takes a bare pkg because it CREATES the SID directory. `prep` / `apply` / `report` come after, so they take `SID/PKG`.

### workspace

Always `<package>` arg = `SID/PACKAGE`, except `init` (bare pkg → SID derived from `-c`).

| Command | Required | Optional | Safety |
|---|---|---|---|
| `workspace init <pkg>` | none | `[--source-dir <name>] [--transport NR] [--filter <pat>] [--type <csv>] [--recursive] [--depth N] [--no-download] [--concurrency N] [--force] [--dry-run]` | mutating |
| `workspace status [SID/PKG]` | none | none | read-only |
| `workspace list [SID/PKG]` | none | none | read-only |
| `workspace pull <SID/PKG> [obj...]` | none | `[--force] [--dry-run]` | mutating, idempotent |
| `workspace push <SID/PKG> [obj...]` | none | `[--force] [--no-activate] [--transport NR] [-y] [--dry-run]` | **destructive** |
| `workspace diff <SID/PKG> [obj]` | none | `[--sap] [--name-only]` | read-only |
| `workspace add <SID/PKG> <obj...>` | none | `[--concurrency N] [--dry-run]` | mutating, idempotent |
| `workspace remove <SID/PKG> <obj...>` | none | `[--delete] [-y] [--dry-run]` | mutating |
| `workspace reset <SID/PKG> [obj...]` | none | `[-y] [--sap] [--dry-run]` | **destructive**, idempotent |
| `workspace refresh <SID/PKG>` | none | `[--recursive] [--depth N] [--filter <pat>] [--type <csv>] [--concurrency N] [--force] [--dry-run]` | mutating |

`workspace init --recursive --depth N` warning: at depth >1, sideways package refs explode (we've seen 9806 metadata-only objects from one `--depth 2` run). Prefer `--depth 1` per package, or invoke `init` per leaf package.

`workspace push` activation: enabled by default; `--no-activate` skips. `workspace pull` does not write to SAP, only to local disk.

`workspace diff --sap` compares local against live SAP source (re-download); without it, compares local against the baseline captured at last sync.

---

## Non-obvious behaviors (the things that consume cycles)

### Class includes — `--include` is for LOCAL classes, NOT the global methods
- `source get ZCL_FOO` **with NO `--include`** → the **full global class**: `CLASS x DEFINITION … ENDCLASS.` *and* `CLASS x IMPLEMENTATION … ENDCLASS.` with every method body (reads `…/source/main`). **This is what you edit to change a method.**
- `--include implementations` (CCIMP) / `definitions` (CCDEF) / `testclasses` (CCAU) / `macros` (CCMAC) → the **local helper** include only (`lcl_*` classes, local types). On a typical class this is a near-empty comment stub, **not** the global methods.
- **Footgun:** PUTting the global `CLASS x IMPLEMENTATION` block via `--include implementations` creates a duplicate → activation fails `"CLASS … IMPLEMENTATION" may only occur once`, and the poisoned include **persists** (later correct main pushes keep failing until it's cleared back to its stub). To edit a global method, round-trip the **full main source** (no `--include`). Use `--include` only when the file really contains local helper classes. (Full analysis: `docs/source-put-include-ccimp-probe-2026-06-29.md`.)
- `--include` valid values per command:
  - `source get/put`: `definitions` \| `implementations` \| `testclasses` (+ `macros`)
  - `code *`, `refactor *`, `object history`, `enho on`: above + `main`
- Other types (PROG, INTF, FUGR/FF, DDLS) ignore `--include` (single source).

### Transport disambiguation
- `transport info <object>`: object name → requirements + open transports.
- `transport get <NR>`: transport number → tasks/objects.
- Mixing them up returns `OBJECT_NOT_FOUND`, a common cycle-burner.

### Stale CSRF on persisted session
- First **write** after the session expires returns 403 Forbidden. Reads still work because they don't need CSRF.
- Recovery: delete `.abapctl/.session.json` and retry once. SapClient re-logs in transparently.
- Looks like an auth bug or SAP outage but is just session staleness.

### `text-elements put` lock target
The lock target is the **textelements URI** (`/sap/bc/adt/textelements/{programs|classes|functiongroups}/{name}`), NOT the parent object URI. Locking the parent yields a valid handle but the PUT 423s. The CLI handles this internally; agents only need to know the symptom.

### Activation defaults
- `source put` / `source push` / `workspace push` / all `create *` (where source is provided) / `bdef create` activate by default. `--no-activate` to skip.
- `text-elements put` does NOT activate by default. Pass `--activate` to opt in.

### Capability gating
- `release-state lookup`: modern S/4HANA only. Older/classic stacks → `CAPABILITY_NOT_AVAILABLE`.
- `ddic table`: not available on older/classic stacks.
- `create annotation-definition`: shell-only (test users 403).
- DDIC create: capability-gated on older/classic stacks via a content-type-aware check.
- `text-elements get` on older/classic stacks: returns empty array (endpoint family absent).
- Batch activation (`activation/runs`): modern S/4HANA only.

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
# NO --include: full global class incl. method bodies. The normal way to edit a class.
abapctl source get ZCL_FOO -c dev > zcl_foo.clas.abap
# edit
abapctl check syntax ZCL_FOO --source zcl_foo.clas.abap -c dev --json
abapctl source put ZCL_FOO --file zcl_foo.clas.abap --transport <NR> -c dev -y --json
abapctl check atc ZCL_FOO -c dev --json
# Add --include only to edit LOCAL helper classes (CCIMP/CCDEF/CCAU), never the global methods.
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
abapctl clean-core prep <SID>/ZFINANCE -c dev
# fix sources
abapctl clean-core apply <SID>/ZFINANCE -c dev --dry-run --json
abapctl clean-core apply <SID>/ZFINANCE -c dev -y --json
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
abapctl workspace status <SID>/ZFIN --json
abapctl workspace diff <SID>/ZFIN --json
abapctl workspace push <SID>/ZFIN -c dev --dry-run --json
abapctl workspace push <SID>/ZFIN -c dev -y --json
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

**Success payloads are heterogeneous; do NOT assume a uniform `{ok:true}` envelope.** Each command returns its own shape, and several return a **bare array** (no wrapper object): `object search`, `object types`, `object inactive`, `object history`, `cds annotations`, `package list`, and others. Branch on the command, not on a shared `ok` field. The **error** envelope, by contrast, IS uniform (see below): detect failure by the presence of `error` (and the non-zero exit code), not by the absence of `ok:true`.

**Success patterns** (examples; shapes vary per command):

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
    },
    "prod": {
      "host": "sap-prod.example.com",
      "port": 44300,
      "client": "100",
      "secure": true,
      "username": "DEVELOPER",
      "password_aws_secret": "your-secret-name",
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

`-c <name>` selects the connection; `defaults.connection` is used when omitted.

**Password resolution** (precedence): `password_aws_secret` (fetched from AWS Secrets Manager via the `aws` CLI) wins over `password_env` (env var name, default `ABAPCTL_PASSWORD`). Per connection, set one:
- `password_env`: name of the env var holding the password (e.g. `SAP_DEV_PASSWORD`).
- `password_aws_secret`: Secrets Manager secret id. The SecretString is used verbatim if plain; if JSON, a single-key object uses that value, and a multi-key object needs `password_aws_key` to pick the field. AWS region/profile/credentials come from the `aws` CLI's own config; a configured-but-failing AWS fetch errors out (never falls back to the env var).

Directory layout:
- `.abapctl/` (fixed): `reference/` (`sap-api-reference/`, `cc-kb/`), `logs/`, `.session.json`. Like `.git/`; don't put project files here.
- `workspace_dir` (`"abap"`): ABAP source at `{SID}/{PACKAGE}/`; manifests at `.manifests/{SID}/{PACKAGE}/`.
- `clean_core.output_dir` (`"clean-core"`): assessment output at `{SID}/{PACKAGE}/`.
