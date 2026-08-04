---
name: domaincraft-core
description: Use when working on a DomainCraft project — modeling a domain, writing or editing domain.yaml, or attaching business logic to generated entities. Front-loads the domain-model contract and tells you what you can and cannot touch. Use when the user mentions domain.yaml, entities, fields, relations, features, auth, permissions, or the domaincraft CLI.
license: MIT
---

# DomainCraft — Core (domain.yaml)

You are the engineer who designs the domain model in `domain.yaml` (the single source of truth) and attaches business logic to the resulting code. You do **not** write CRUD, migrations, DTOs, or boilerplate — the tool generates that for you and hands you *production-ready* code.

## ENFORCED RULES — these are mandatory, not advice

If you cannot follow one, stop and ask the user. Do not improvise around a rule.

1. **Fixed pipeline, one step at a time:** write `domain.yaml` → `domaincraft validate` (must print `✓ Schema valid`) → `domaincraft generate` → implement logic in custom files → run the project's own benchmark. No machine exploration or extra reading between steps.
2. **Run the project's own artifact, never tool-discovery.** If the user says "run the benchmark from the project root", execute the artifact that already exists there (e.g. `node benchmark.js`), in that folder. Do not run `k6`, `winget`, `choco`, `scoop`, `node --version`, `dotnet --list-sdks`, `docker ps` to figure out *how* to do it. If the target command's binary is missing, ask the user.
3. **Ask instead of self-deciding.** Missing credentials, an unclear step, or a guardrail you can't meet → ask the user.

## Pain it removes

- No need to hand-write CRUD endpoints, repositories, DB migrations, auth, or DTOs.
- No need to keep the model and the generators in sync — `domain.yaml` describes everything.
- No fear of renames or refactors: the tool migrates files for you.
- All boilerplate, data layer, DB schema, and build wiring is on the tool.

## The full set of what you can use (JSON spec)

The authoritative list of allowed keys, field types, features, `on_delete` values, and shapes is in the JSON Schema bundled with this skill. **Read it before working** so you don't invent constructs that don't exist.

- `domain.schema.json` — in this same folder (`domaincraft-core/domain.schema.json`). Read this file directly; it ships with the skill, so it works even when the user only has the `domaincraft` binary and no source repo.

### Model constructs (summary — details in the schema)

- **Where keys live.** Only `project:` and `auth:` are nested blocks. `database`, `api_style`, `entities`, and `enums` are **top-level — siblings of `project:`, NOT inside it**. Nesting `database`/`api_style` under `project` fails with `yaml: unmarshal errors: ... database not found in type parser.ProjectConfig`.
- **Field definition strings** — every field in `fields:` is a string: `<type> [modifiers]`, e.g. `string [required, max:255]`. Types: `string`, `text`, `int`, `bigint`, `float`, `decimal`, `boolean`, `date`, `datetime`, `uuid`, `json`, `jsonb`, `array(Type)`, `enum(Name)`.
- **Relations are fields, not a separate key — there is no `relations:` block.** Declare a relation as a field whose type is `relation(Target)`:
  - `orderId: relation(Order) [required, on_delete:cascade]` — many-to-one (FK).
  - `parentId: relation(Category) [optional, on_delete:set_null]` — `set_null` requires `optional`.
  - `tags: relation(Tag) [many]` — many-to-many.
  - `profile: relation(Profile) [optional, unique]` — one-to-one.
  - `on_delete` values: `cascade`, `restrict`, `set_null`, `no_action`.
   - `relation(Target)` already creates the FK column (e.g. `OrderId`) and any inverse side — do **not** add a separate scalar `XxxId` field for it, and **never repeat a field name**. Declaring `buyerId` twice (once `uuid`, once `relation(...)`) is invalid.
- **field modifiers:** `required`, `optional`, `unique`, `hidden`, `primary`, `email`, `url`, `ipv4`, `many`, `min:N`, `max:N`, `gte:N`, `gt:N`, `lte:N`, `lt:N`, `regex:"..."`, `default:...`.
- **`features:`** per entity — `audit`, `audit_log`, `soft_delete`, `optimistic_lock`, `event_sourced`, `cacheable` (auto-add `createdAt`, `updatedAt`, `createdBy`, `updatedBy`, `deletedAt`, `version`).
- **`auth:`** — `type` (`jwt`, `none`), `entity`, `roles`, `endpoints` (`login`, `register`, `me`). With `type: jwt` the auth entity needs `email` and `password` fields. *How the JWT endpoints and token are actually generated is bridge-specific — see the bridge skill.*
- **`permissions:`** per entity — `read`/`create`/`update`/`delete` role arrays, tokens `*` (everyone) and `@Owner` (record owner).
- **`seed:`** — starter data; also `indexes`, `old_name`, `cache`, event-sourcing.

Roles (other than `*`, `@Owner`) must be declared in `auth.roles`. The auth entity must have `email` and `password` fields.

**Roles on the auth entity:** declare a field named `role` on the auth entity (e.g. `role: string [required, default:"User"]` or `role: enum(Role) [required]`). Each bridge decides how that field maps to authorization (JWT claims, attributes, policies) — see the bridge skill for the language-specific mechanism.

### Rules every model must satisfy (fix these or validation fails)

- **Exactly one primary key — on EVERY entity.** Add `id: uuid [primary]` to each entity. Omitting it on any single entity fails the whole validation: `entity must have at least one primary key (add a field marked [primary]...)`.
- **Enums go in two places.** Declare the enum in the top-level `enums:` block AND reference it with the `enum(...)` wrapper: `status: enum(OrderStatus)`. A bare `OrderStatus` as a type is NOT valid — the `unknown type` error now explicitly says to use `enum(EnumName)`.
- **`min`/`max` are string length, not numeric bounds.** For numbers use `gte`/`gt`/`lte`/`lt`: `price: decimal [required, gte:0]`. Writing `min:0` on a `decimal` only produces a warning, but it's a bug in intent.
- **`on_delete:set_null` requires `optional`**: `relation(Target) [optional, on_delete:set_null]`.
- **Unused enums are just a warning** — declare only enums you actually reference in a field, or remove them.

### YAML gotchas

- **Quote `@Owner` in permissions.** YAML reserves `@`, so an unquoted `@Owner` fails to parse. Write `["@Owner"]` or `"@Owner"`.
- **No space after `:` inside a field definition string.** `default: 5` is read by YAML as a nested mapping. Write `default:5` (or `default:"pending"` for quoted string values).

### Minimal model example

```yaml
project:
  name: MyApp
database: postgresql          # top-level — sibling of project, not nested
api_style: rest               # top-level — sibling of project, not nested
entities:
  OrderItem:
    fields:
      id: uuid [primary]      # every entity needs exactly one [primary]
  Order:
    fields:
      id: uuid [primary]
      number: string [required, unique]
      total: decimal [required, gte:0]
      items: relation(OrderItem) [many]   # FK column is created for you
```

## What is generated for you (do not touch)

All of the CRUD, migrations, DTOs, validation, DB schema, and auth are auto-generated files — **do not hand-edit them**; they are overwritten when `domain.yaml` changes.

## Your areas of responsibility

1. **Write `domain.yaml`** — model entities, relations, features, and permissions.
2. **Attach business logic to the ready CRUD** — work on top of generated scaffolds in custom files (`overwrite: false`): hooks, extension points, service implementations. The exact hook shapes are bridge-specific — see the bridge skill.
3. **Renames** — declare `old_name: <previous name>` on an entity so the tool renames custom files and warns about breaking changes.
4. **CLI commands:**
   - `domaincraft validate` — validate `domain.yaml`.
   - `domaincraft bridges` — list available bridges (C#, admin, Go, …).
   - `domaincraft generate --domain domain.yaml --bridge <id>` — generate code.
   - `--prune` — auto-clean orphaned files during refactoring (for CI).
   - `--migrate` — run DB migrations described by the bridge.

## How to work — follow this order, nothing else

Work in a strict pipeline. **One step at a time.** Do not jump ahead, do not batch steps:

1. **Write `domain.yaml`** — only the model, matching the spec the user gave (rule 2 above). Don't touch any other files yet.
2. **`domaincraft validate --domain domain.yaml`** — it must pass with `✓ Schema valid` before you continue.
3. **`domaincraft generate --domain domain.yaml --bridge <id> --output .`** — produces the code.
4. **Implement business logic** — only in `overwrite: false` custom files (see the bridge skill).
5. **Verify / run the benchmark** — run the project's own artifact (e.g. `node benchmark.js`) from the project root, if the task asked for it.

Before each step, **re-read the corresponding skill section** (this one for the model/CLI, the bridge skill for generated code) and confirm the current step matches it. If a step would violate a rule, stop and ask.

### Fixing validation errors

- Fix **exactly the reported error**, then re-run `validate`. One fix → one validate. Do not "also improve" the model or reformat unrelated fields in the same pass — that is how duplicate/conflicting fields sneak in.
- Re-read the error string and fix that specific cause; do not guess or rewrite large chunks.

### Do NOT (unprompted environment discovery)

Only run commands that advance the current step. Everything below is out of scope unless the user explicitly asks:

- When the task needs the system running (a DB, the API), **start the generated `docker compose up -d --build` immediately** — do not first inspect what's on the machine (`docker ps`, `docker --version`, `dotnet --list-sdks`, `Get-Service`, `Test-NetConnection`, `psql`/`pg_isready`, port scans). The compose file is the contract for how the stack runs; run it, don't verify the environment first.
- Do **not** probe the machine's toolchain (`node --version`, `dotnet --list-sdks`, `docker --version`, port scans). If you need capability info the project can't provide, ask the user. `--version`, `--help`, and `bridges` are for the CLI's own capabilities; they will not tell you what is installed.
- Do **not** read the whole generated tree or every generated file. Read only what a step requires (a custom file, a service/repository signature). The bridge skill lists what you need.
- Do **not** generate before validation passes.
- Do **not** add files, migrations, or infrastructure that the tool already generates.

### When a step depends on something the user hasn't provided

Stop and ask. Do not discover it yourself: DB credentials, an empty port, a bridge not listed, a missing SDK — ask the user, don't probe the machine.

## Golden rule

**Do not write or edit generated code.** Your value is the model in `domain.yaml` plus business logic on top of the ready CRUD. The tool owns all the boilerplate and migrations.