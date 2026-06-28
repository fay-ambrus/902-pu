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
- `User` (base: givenName, familyName, nickName)
  - → `Scout` (ecset code, OIDC-only login) → `ActiveScout` → `Leader` (has assignment or workgroup membership) → `FinancialWorkGroupMember` → `FinancialOfficer` (full or financial assignment in ECSET)
  - → `ExternalUser` (email, password) → `Accountant`
- `Family` (contains members; auto-detected by name + parent email, manually adjustable)
- Inheritance mapped via TPH (Table Per Hierarchy) in EF Core

### Organization
- `Unit` — self-referencing hierarchy (patrol/troop/group/work_group)
  - Has leaders, members, parent, children

### Finance
- `BankTransaction` — értéknap, partner neve, partner számlaszáma, Money (HUF only), közlemény, projekt (required), megjegyzés
- `CashTransaction` — értéknap, partner neve, Money, közlemény, projekt (required), megjegyzés
- `Category` — spending category for transactions (e.g. felszerelés, egyenruha, tagdíj). Used for reports/charts.
- `Invoice` (Számla) — kiállító, leírás, kiállító adószáma, Money, ki fizette ki, megjelenés (e-számla?), számla fájl, beérkezett-e a pénzügyeshez
- `CardHandover` (Bankkártya-átadás) — dátum, kitől, kihez. Tracks which user holds each team bank card.
- `PaymentRequest` (Fizetési kérelem) — kitől (projekt/user), kihez (projekt/user), létrehozó, Money, létrehozás időpontja, határidő, leírás, számla (optional), státusz (open/fulfilled/overdue)
- `Project` (FinancialProject) — név, leírás, pénzügyi felelős(ök), tranzakciók, költségvetés, beszámoló, kezdő/befejező dátum, státusz (létrehozva/terv beadva/terv elfogadva/projekt vége/beszámoló elfogadva), cserkészév
- `Budget` (Költségvetés) — költségvetési sorok, paraméterek, állapot (szerkesztés alatt/beadva/elfogadva), ellenőrző, díjak. Bevételi+kiadási oldal. Végső mérleg biztonsági tartalékkal és nélkül.
- `BudgetItem` (Költségvetési sor) — leírás, mennyiség (formula string), Money. Positive = bevétel, negative = kiadás.
- `BudgetParameter` — leírás, mennyiség. Mandatory parameter: biztonsági tartalék (%). Locked after budget submission.
- `FeeManagement` (Díjkezelés) — résztvevő kategóriák, résztvevők, résztvevőnkénti egyenleg, díjelemek. Creates PaymentRequests. Incoming transactions auto-matched to participants by közlemény.
- `StatementExpression` (Közlemény-kifejezés) — pattern matching rule that auto-assigns bank transactions to projects based on közlemény content.
- `ScoutYear` (Cserkészév) — fiscal year starting September 1. Every Project belongs to one.
- `BankAccount` — IBAN, linked GoCardless connection
- `Money` — owned type (amount + currency), embedded in parent tables

### Enums
- `Currency`: HUF, EUR, USD
- `TransactionStatus`: OPEN, FULFILLED, OVERDUE
- `ProjectStatus`: CREATED, PLAN_SUBMITTED, PLAN_ACCEPTED, PROJECT_ENDED, REPORT_ACCEPTED
- `BudgetStatus`: DRAFT, SUBMITTED, ACCEPTED
- `UnitType`: PATROL, TROOP, GROUP, WORK_GROUP

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
