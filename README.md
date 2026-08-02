# DomainCraft Skills

Agent skills for working with **DomainCraft** — a domain-driven code generator. Describe your domain once in `domain.yaml`, get *production-ready* code, and focus on the business logic instead of boilerplate.

This repo ships two cooperating skills:

| Skill | Folder | What it covers |
|-------|--------|----------------|
| **DomainCraft Core** | `domaincraft-core/` | Modeling the domain in `domain.yaml` — entities, fields, relations, features, auth, permissions; the CLI workflow; the full JSON spec of what you can use. |
| **DomainCraft C# Bridge** | `domaincraft-csharp/` | Working with the generated ASP.NET Core REST API (EF Core + PostgreSQL + JWT): which files are generated for you and where your business logic goes. |

## Overview

DomainCraft turns a declarative `domain.yaml` model into a working backend with clean architecture — CRUD endpoints, repositories, DTOs, database migrations, auth, and validation. It is a **bridge-based** system: each target language/framework is a pluggable template pack. The first bridge generates a C# ASP.NET Core REST API.

These skills are written for both humans and AI code assistants. Their job is to make clear:

- What DomainCraft does for you (the pain it removes).
- What you are responsible for (the model and the business logic on top of the generated CRUD).
- **What not to touch** — the auto-generated files that get overwritten on every `generate`.

So instead of re-reading hundreds of generated lines, an assistant immediately knows what it works with and how to work with it.

## What pain it removes

- Hand-writing CRUD endpoints, repositories, DTOs, migrations, and auth.
- Keeping a model and its generators in sync.
- Fearing renames and refactors — the tool migrates and renames files for you.
- Reading/maintaining boilerplate that is regenerated constantly.

## Golden rule

**Do not write or edit generated code.** Your value is the model in `domain.yaml` and the business logic on top of the ready CRUD. DomainCraft owns all the boilerplate and migrations.