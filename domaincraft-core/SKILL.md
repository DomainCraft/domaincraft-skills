---
name: domaincraft-core
description: Use when working on a DomainCraft project — modeling a domain, writing or editing domain.yaml, or attaching business logic to generated entities. Front-loads the domain-model contract and tells you what you can and cannot touch. Use when the user mentions domain.yaml, entities, fields, relations, features, auth, permissions, or the domaincraft CLI.
license: MIT
---

# DomainCraft — Core (domain.yaml)

You are the engineer who designs the domain model in `domain.yaml` (the single source of truth) and attaches business logic to the resulting code. You do **not** write CRUD, migrations, DTOs, or boilerplate — the tool generates that for you and hands you *production-ready* code.

## Pain it removes

- No need to hand-write CRUD endpoints, repositories, DB migrations, auth, or DTOs.
- No need to keep the model and the generators in sync — `domain.yaml` describes everything.
- No fear of renames or refactors: the tool migrates files for you.
- All boilerplate, data layer, DB schema, and build wiring is on the tool.

## The full set of what you can use (JSON spec)

The authoritative list of allowed keys, field types, features, `on_delete` values, and shapes is in the JSON Schema bundled with this skill. **Read it before working** so you don't invent constructs that don't exist.

- `domain.schema.json` — in this same folder (`domaincraft-core/domain.schema.json`). Read this file directly; it ships with the skill, so it works even when the user only has the `domaincraft` binary and no source repo.

### Model constructs (summary — details in the schema)

- **top-level keys:** `project`, `database`, `api_style`, `auth`, `entities`, `enums` (details in the schema).
- **Field definition strings** — every field in `fields:` is a string: `<type> [modifiers]`, e.g. `string [required, max:255]`. Types: `string`, `text`, `int`, `bigint`, `float`, `decimal`, `boolean`, `date`, `datetime`, `uuid`, `json`, `jsonb`, `array(Type)`, `enum(Name)`.
- **Relations are fields, not a separate key — there is no `relations:` block.** Declare a relation as a field whose type is `relation(Target)`:
  - `orderId: relation(Order) [required, on_delete:cascade]` — many-to-one (FK).
  - `parentId: relation(Category) [optional, on_delete:set_null]` — `set_null` requires `optional`.
  - `tags: relation(Tag) [many]` — many-to-many.
  - `profile: relation(Profile) [optional, unique]` — one-to-one.
  - `on_delete` values: `cascade`, `restrict`, `set_null`, `no_action`.
- **field modifiers:** `required`, `optional`, `unique`, `hidden`, `primary`, `email`, `url`, `ipv4`, `many`, `min:N`, `max:N`, `gte:N`, `gt:N`, `lte:N`, `lt:N`, `regex:"..."`, `default:...`.
- **`features:`** per entity — `audit`, `audit_log`, `soft_delete`, `optimistic_lock`, `event_sourced`, `cacheable` (auto-add `createdAt`, `updatedAt`, `createdBy`, `updatedBy`, `deletedAt`, `version`).
- **`auth:`** — `type` (`jwt`, `none`), `entity`, `roles`, `endpoints` (`login`, `register`, `me`). With `type: jwt` the csharp bridge generates the `/login`, `/register`, `/me` endpoints and JWT token issuance; the auth entity needs `email` and `password` fields.
- **`permissions:`** per entity — `read`/`create`/`update`/`delete` role arrays, tokens `*` (everyone) and `@Owner` (record owner).
- **`seed:`** — starter data; also `indexes`, `old_name`, `cache`, event-sourcing.

Roles (other than `*`, `@Owner`) must be declared in `auth.roles`. The auth entity must have `email` and `password` fields.

### Rules every model must satisfy (fix these or validation fails)

- **Exactly one primary key.** Every entity needs one field marked `[primary]` (e.g. `id: uuid [primary]`). Omitting it → `entity must have at least one primary key (add a field marked [primary]...)`.
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
entities:
  OrderItem:
    fields:
      id: uuid [primary]
  Order:
    fields:
      id: uuid [primary]
      number: string [required, unique]
      total: decimal [required, gte:0]
      items: relation(OrderItem) [many]
```

## What is generated for you (do not touch)

All of the CRUD, migrations, DTOs, validation, DB schema, and auth are auto-generated files — **do not hand-edit them**; they are overwritten when `domain.yaml` changes.

## Your areas of responsibility

1. **Write `domain.yaml`** — model entities, relations, features, and permissions.
2. **Attach business logic to the ready CRUD** — work on top of generated scaffolds in custom files (`overwrite: false`): hooks, partial methods, service implementations.
3. **Renames** — declare `old_name: <previous name>` on an entity so the tool renames custom files and warns about breaking changes.
4. **CLI commands:**
   - `domaincraft new` — create a project.
   - `domaincraft validate` — validate `domain.yaml`.
   - `domaincraft bridges` — list available bridges (C#, admin, Go, …).
   - `domaincraft generate --domain domain.yaml --bridge <id>` — generate code.
   - `--prune` — auto-clean orphaned files during refactoring (for CI).
   - `--migrate` — run DB migrations described by the bridge.

## How to work

1. First `validate` — make sure the model is clean.
2. Then `generate` — you get ready CRUD.
3. Write business logic in custom files (the `overwrite: false` zones), not in generated ones.
4. When the model changes, re-run `generate`; generated files refresh, your custom code survives.

## Golden rule

**Do not write or edit generated code.** Your value is the model in `domain.yaml` plus business logic on top of the ready CRUD. The tool owns all the boilerplate and migrations.