---
name: domaincraft-core
description: Use when working on a DomainCraft project — modeling a domain, writing or editing domain.yaml, or attaching business logic to generated entities. Use when the user mentions domain.yaml, entities, fields, relations, features, auth, permissions, or the domaincraft CLI. Loads the domain-model contract and the fixed workflow.
license: MIT
---

# DomainCraft — Core (domain.yaml)

You are the engineer who designs the domain model in `domain.yaml` (the single source of truth) and attaches business logic to the resulting code. You do **not** write CRUD, migrations, DTOs, or boilerplate — the tool generates that for you and hands you *production-ready* code.

## ENFORCED RULES — mandatory, not advice

If you cannot follow one, stop and ask the user. Do not improvise around a rule.

1. **Fixed pipeline, one step at a time:** write `domain.yaml` → `domaincraft validate` (must print `✓ Schema valid`) → `domaincraft generate` → implement logic in custom files → run the project's own benchmark. No machine exploration or extra reading between steps.
2. **Run the project's own artifact, never tool-discovery.** If the user says "run the benchmark from the project root", execute the artifact that already exists there (e.g. `node benchmark.js`), in that folder. Do not run `k6`, `winget`, `choco`, `scoop`, `node --version`, `dotnet --list-sdks`, `docker ps` to figure out *how* to do it. If the target command's binary is missing, ask the user.
3. **Ask instead of self-deciding.** Missing credentials, an unclear step, or a guardrail you can't meet → ask the user.

Follow the rule-1 pipeline in order: write `domain.yaml` → `validate` → `generate` → implement logic only in `overwrite: false` custom files → run the project's own artifact. Before each step, re-read the matching section (this skill for the model/CLI, the bridge skill for generated code). If a step would violate a rule, stop and ask.

### Fixing validation errors

- Fix **exactly the reported error**, then re-run `validate`. One fix → one validate. Do not "also improve" the model in the same pass — that is how duplicate/conflicting fields sneak in.

### Do NOT (unprompted environment discovery)

Only run commands that advance the current step:

- When the task needs the system running (a DB, the API), **start the generated `docker compose up -d --build` immediately** — no pre-checks of what's installed or running. The compose file is the contract for how the stack runs.
- Do **not** probe the machine (`node --version`, `dotnet --list-sdks`, `docker --version`, port scans). `--version`, `--help`, and `bridges` tell you about the CLI, not the machine. If you need info the project can't provide, ask the user.
- Do **not** read the whole generated tree. Read only what a step requires (a custom file, a service/repository signature). The bridge skill lists what you need.
- Do **not** generate before validation passes.
- Do **not** add files, migrations, or infrastructure the tool already generates.

## Golden rule

**Do not write or edit generated code.** Your value is the model in `domain.yaml` plus business logic on top of the ready CRUD. The tool owns all the boilerplate and migrations.

## Pain it removes

- No hand-written CRUD endpoints, repositories, DB migrations, auth, or DTOs.
- No need to keep the model and generators in sync — `domain.yaml` describes everything.
- Renames/refactors are safe: the tool migrates files for you.

## Your areas of responsibility

1. **Write `domain.yaml`** — model entities, relations, features, and permissions.
2. **Attach business logic to the ready CRUD** — in custom files (`overwrite: false`): hooks, extension points, service implementations. Exact hook shapes are bridge-specific — see the bridge skill.
3. **Renames** — declare `old_name: <previous name>` on an entity so the tool renames custom files and warns about breaking changes.
4. **CLI commands:** `domaincraft validate` · `domaincraft generate --domain domain.yaml --bridge <id>` · `domaincraft bridges` · `--prune` (auto-clean orphaned files, CI) · `--migrate` (run bridge-declared DB migrations) · `--admin` (also generate an admin panel, written to `<output>/admin/`).

## Reference — the model contract

The authoritative list of allowed keys, field types, features, `on_delete` values, and shapes is in `domain.schema.json` (same folder as this skill). **Consult it only when you're unsure a construct exists** — the summary below covers the common cases; you do not need to read the whole schema upfront.

### Model constructs

- **Where keys live.** Only `project:` and `auth:` are nested blocks. `database`, `api_style`, `entities`, and `enums` are **top-level — siblings of `project:`, NOT inside it**. Nesting `database`/`api_style` under `project` fails with `yaml: unmarshal errors: ... database not found in type parser.ProjectConfig`.
- **Field definition strings** — every field in `fields:` is a string: `<type> [modifiers]`, e.g. `string [required, max:255]`. Types: `string`, `text`, `int`, `bigint`, `float`, `decimal`, `boolean`, `date`, `datetime`, `uuid`, `json`, `jsonb`, `array(Type)`, `enum(Name)`.
- **Relations are fields, not a separate key — there is no `relations:` block.** Declare a relation as a field whose type is `relation(Target)`:
  - `orderId: relation(Order) [required, on_delete:cascade]` — many-to-one (FK).
  - `parentId: relation(Category) [optional, on_delete:set_null]` — `set_null` requires `optional`.
  - `tags: relation(Tag) [many]` — many-to-many.
  - `profile: relation(Profile) [optional, unique]` — one-to-one.
  - `on_delete` values: `cascade`, `restrict`, `set_null`, `no_action`.
  - `relation(Target)` already creates the FK column (e.g. `OrderId`) and any inverse side — do **not** add a separate scalar `XxxId` field for it, and **never repeat a field name**. Declaring `buyerId` twice (once `uuid`, once `relation(...)`) is invalid.
- **field modifiers:** `required`, `optional`, `unique`, `hidden`, `readonly`, `primary`, `email`, `url`, `ipv4`, `many`, `min:N`, `max:N`, `gte:N`, `gt:N`, `lte:N`, `lt:N`, `regex:"..."`, `default:...`.
  - **`hidden` vs `readonly`:** `hidden` (`internalNotes: string [hidden, optional]`) excludes a field from API **responses** (it stays client-settable via requests). `readonly` (`balance: decimal [required, readonly, gte:0]`) does the **opposite** — it is persisted and returned in responses but **excluded from create/update/patch requests** and from request→entity mapping, so it is server-owned (a read-only value the client must not write). Pick `readonly` for computed/ledger values the client must read but never set; pick `hidden` for values the client must never read.
- **`features:`** per entity — `audit`, `audit_log`, `soft_delete`, `optimistic_lock`, `event_sourced`, `cacheable` (auto-add `createdAt`, `updatedAt`, `createdBy`, `updatedBy`, `deletedAt`, `version`).
- **`auth:`** — `type` (`jwt`, `none`), `entity`, `roles`, `endpoints` (`login`, `register`, `me`, `setup`). With `type: jwt` the auth entity needs `email` and `password` fields. `setup` bootstraps the first user with the first declared role (typically `Admin`) and only succeeds while no account exists — it is the production bootstrap path. *How the JWT endpoints and token are actually generated is bridge-specific — see the bridge skill.*
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

### What is generated for you (do not touch)

All of the CRUD, migrations, DTOs, validation, DB schema, and auth are auto-generated files — **do not hand-edit them**; they are overwritten when `domain.yaml` changes. This is the Golden rule above applied to concrete files.

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
