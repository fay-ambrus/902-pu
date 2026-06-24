# 902-pu Project Steering

## Overview

A financial reporting/planning system for a Hungarian scouting organization. ASP.NET Core Web API backend with SQLite persistence via Entity Framework Core.

## Tech Stack

- **.NET 10** (net10.0)
- **ASP.NET Core Web API** with Controllers
- **Entity Framework Core** with SQLite (`app.db`)
- **ASP.NET Core Identity** for local username/password auth
- **OpenID Connect** for external auth via ecset.cserkesz.hu
- **OpenAPI** spec served at `/openapi/v1.json` in dev mode
- Root namespace: `_902_pu` (because C# identifiers can't start with a digit)

## Authentication

Two login methods:

1. **Local login** — ASP.NET Core Identity (username/password). No public registration (closed system).
2. **OIDC login** — Authorization Code flow against `https://ecset.cserkesz.hu/o`
   - Client type: Confidential
   - Algorithm: RS256
   - Scopes requested: `openid profile cserkesz`
   - Redirect URI: `https://localhost:7247/signin-oidc`
   - Post-logout redirect: `https://localhost:7247/signout-callback-oidc`

### OIDC Provider Claims (ecset.cserkesz.hu)

The `cserkesz` scope provides:
- `csapat` — troop URL (e.g. `mcssz/9999`)
- `jogviszony` — membership status (e.g. `felnott`, `mukodo`)
- `kepesitesek` — list of qualifications (objects with `kepesites` and `datum`)
- `megbizatasok` — list of active assignments (objects with `megbizatas` and `egyseg_slug`)

The `profile` scope provides: `name`, `given_name`, `family_name`, `nickname`, `preferred_username`, `email`

We only need `csapat` and `megbizatasok` from the OIDC claims.

### Secrets

Stored in .NET User Secrets (not in source):
- `Oidc:ClientId`
- `Oidc:ClientSecret`

`appsettings.json` only contains the non-secret authority URL.

## Domain Model

Defined in `class_diagram.puml`. Key entities:

### People
- `Person` (base) → `Scout` → `Administrator`
- `Person` → `Accountant`
- `Family` (contains members)
- Inheritance mapped via TPH (Table Per Hierarchy) in EF Core

### Organization
- `Unit` — self-referencing hierarchy (patrol/troop/group/work_group)
  - Has leaders, members, parent, children

### Finance
- `Transaction` — payer/payee (ITransactable), amount (Money), status, method, dates
- `BankAccount` — holds transactions
- `FinancialProject` — links a plan + report + transactions
- `FinancialPlan` — contains PlanItems and PlanParameters
- `PlanItem` — name, unitAmount (Money), quantity (formula string), quantityResolved
- `PlanParameter` — name/value pairs referenced in formulas
- `TransactionPartner` — external party (name + tax number)
- `Money` — owned type (amount + currency), embedded in parent tables

### ITransactable Interface
Implemented by: `Scout`, `FinancialProject`, `TransactionPartner`
Note: EF Core can't directly map polymorphic navigation across unrelated hierarchies. Transaction uses raw FK integers (`PayerId`/`PayeeId`).

### Enums
- `Currency`: HUF, EUR, USD
- `TransactionStatus`: CREATED, FULFILLED, OVERDUE, PENDING
- `PaymentMethod`: CASH, BANK_TRANSFER, CARD
- `MembershipStatus`: ACTIVE, INACTIVE, ALUMNI, CANDIDATE
- `UnitType`: PATROL, TROOP, GROUP, WORK_GROUP

### Events
- `Event` — name, dates, organizers (Scouts), units, participants (Persons), linked FinancialProject

## Financial Plan Design Decisions

- **Parameters** are named values (e.g. `participants = 25`) stored at the plan level
- **PlanItem quantity** is stored as a formula string (e.g. `"participants * days"`) evaluated on the frontend
- **Frontend evaluates formulas** using `expr-eval` (JS library, ~5KB)
- **Backend stores both** the formula string and the resolved numeric result
- Backend is formula-agnostic — treats quantity as an opaque string
- If backend evaluation is ever needed, use `DynamicExpresso` (C# NuGet) — same syntax as expr-eval

## API Structure

- Controllers in `Controllers/` folder
- `[ApiController]` + `[Route("api/[controller]")]` convention
- Auth endpoints: `POST /api/auth/login`, `GET /api/auth/login-oidc`, `POST /api/auth/logout`, `GET /api/auth/me`

## Development

- Run with HTTPS: `dotnet watch run --launch-profile https`
- Dev certificate trusted via `dotnet dev-certs https --trust`
- HTTPS port: 7247, HTTP port: 5217
- OpenAPI spec: `https://localhost:7247/openapi/v1.json`

## Project Structure

```
902-pu/
├── .vscode/
│   └── settings.json          (lualatex build config)
├── spec/
│   ├── diagrams/
│   │   ├── users-class-diagram.puml
│   │   └── users-class-diagram.png
│   ├── csapat_logo_lila.png   (header/watermark logo)
│   └── specification.tex      (main spec document, LuaLaTeX)
├── STEERING.md
├── .gitignore
└── LICENSE
```

## Specification Document

- Written in LaTeX, compiled with **LuaLaTeX** (for Sora font via fontspec)
- Brand color: `#604696` (wosm-lila), grey: `#aea8a6`
- Header: logo + vertical rule + troop name (italic, grey)
- Footer: org info (left), page number (right)
- Background watermark: faded logo via tikz opacity
- Diagrams: PlantUML with `skinparam backgroundColor transparent` and `skinparam defaultFontName Sora`
- VS Code: LaTeX Workshop configured for lualatex in `.vscode/settings.json`
