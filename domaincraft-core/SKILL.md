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

- **types (primitives/field types):** `string`, `int`, `decimal`, `bool`, `datetime`, `date`, `uuid`, `json`, `array(...)`, enum references.
- **field modifiers:** `required`, `unique`, `hidden`, `max:N`, `min:N`, `default:...`, `index`, type wrappers.
- **`entities:`** — name → entity with `fields`, `relations`, `features`, `permissions`, `seed`, `indexes`, `old_name`.
- **`relations:`** — `on_delete` (`cascade`, `restrict`, `set_null`, `no_action`), `many`, `set_null`.
- **`features:`** per entity — `audit`, `audit_log`, `soft_delete`, `optimistic_lock` (auto-add `createdAt`, `updatedAt`, `createdBy`, `updatedBy`, `deletedAt`, `version`).
- **`auth:`** — `type` (`jwt`, `none`), `entity`, `roles`, `endpoints` (`login`, `register`, `me`).
- **`permissions:`** per entity — role-based `read/write/update/delete`, tokens `*` (everyone) and `@Owner` (record owner).
- **`seed:`** — starter data; also `indexes`, `cache`, event-sourcing.

Roles (other than `*`, `@Owner`) must be declared in `auth.roles`. The auth entity must have `email` and `password` fields.

### Minimal model example

```yaml
entities:
  Order:
    fields:
      number: string [required, unique]
      total: decimal [required, min:0]
    relations:
      items: { from: OrderItem, on_delete: cascade }
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