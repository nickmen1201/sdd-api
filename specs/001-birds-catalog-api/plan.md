# Implementation Plan: Birds of Colombia Catalog

**Branch**: `001-birds-catalog-api` | **Date**: 2026-08-26 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `specs/001-birds-catalog-api/spec.md`, original brief `formulations_birds_api_20260820_.md`

## Summary

A read/write REST catalog over three entities — Bird, Family, Ecosystem — exposing the
brief's 17 operations. Delivered as a .NET 10 Web API in three projects enforcing the
brief's mandated layering (Controller → Service → Repository, with Model and Context),
Dapper over PostgreSQL, server-assigned UUID keys, and an OpenAPI 3.1 contract that is
hand-authored and frozen *before* code, then verified against the running service by a
Swashbuckle drift-check plus Schemathesis. App and database run under Docker Compose.

The design's two load-bearing decisions, both traced to the spec rather than to habit:

- **UUIDv4, not UUIDv7.** FR-013 requires identifiers that are *not inferable from
  creation order*; v7 embeds a millisecond timestamp and is therefore ordered by
  construction. This overrides the constitution's UUIDv7/ULID **DEFAULT** — see
  [Complexity Tracking](#complexity-tracking).
- **Bird responses embed the full family and ecosystem objects.** SC-001 requires a
  consumer to obtain "family with its order, ecosystem with its zone" from this catalog
  alone; a bare foreign key would force a second and third round-trip.

## Technical Context

**Language/Version**: C# / .NET 10 (LTS), target framework `net10.0`, nullable reference
types and implicit usings enabled.

**Primary Dependencies**: ASP.NET Core (controller-based), Dapper, Npgsql,
Swashbuckle.AspNetCore (OpenAPI 3.1 emission + Swagger UI). Each justified in
[research.md](./research.md).

**Storage**: PostgreSQL 17, UTF-8 encoded. Schema applied from versioned SQL in
`db/init/`; no ORM and no migration framework (research.md R-006).

**Testing**: xUnit + `WebApplicationFactory` for integration, Testcontainers.PostgreSql
for an ephemeral database per run, Schemathesis for contract conformance against the
frozen `contracts/openapi.yaml`.

**Target Platform**: Linux container (`mcr.microsoft.com/dotnet/aspnet:10.0`), orchestrated
by Docker Compose alongside `postgres:17`.

**Project Type**: Web service (REST API). No frontend and no UI, per spec Assumptions.

**Performance Goals**: None asserted. The spec states no volume, latency or availability
targets and explicitly declines to invent them; the correctness criteria SC-001..SC-007
govern instead.

**Constraints**: Non-ASCII text must round-trip byte-for-byte (SC-007). A malformed
identifier must be distinguishable from an absent one (FR-015) — this constrains route
binding, see research.md R-004. Removal of a referenced family or ecosystem must be
refused, never cascaded (FR-018).

**Scale/Scope**: 3 entities, 17 endpoints, a demonstration dataset of 5 birds (SC-003).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-checked after Phase 1 design.*

Source: `constitution.md` at repository root (v1.0.0, 2026-07-24).

> **Repo hygiene issue — non-blocking.** `.specify/memory/constitution.md` is still the
> unfilled Spec-Kit placeholder. The real constitution lives at the repository root, and
> that is the file this gate was evaluated against. The two should be reconciled so that
> tooling and humans read the same document.
>
> **Profile assumption.** `ACTIVE PROFILES` has no box checked. This is a REST API, so the
> **SWE** profile is assumed active and its rules are enforced below. Data Eng and DS/ML
> are treated as inactive.

| Rule | Gate | Verdict |
|---|---|---|
| Intent flows downhill (spec → plan → contract → tasks → code) | Contract authored in this phase, from the spec; no behavior introduced here that the spec does not carry | **PASS** — one spec discrepancy raised for upstream correction rather than patched downstream (below) |
| Reproducible from a clean checkout | `docker compose up`, pinned `postgres:17` and SDK images, `packages.lock.json` committed, `global.json` pinning the SDK band | **PASS** |
| Artifacts live in the repo; one logical change = one commit | Spec, contract, plan and code move together; Conventional Commits | **PASS** |
| Secrets never in the repo | Connection string from environment; Compose supplies a local-only dev password; no credential committed | **PASS** |
| Data hygiene — surrogate IDs UUIDv7/ULID, never sequential | UUID**v4**, server-assigned, never sequential | **PASS with documented override** — v7 is time-ordered and contradicts FR-013; overriding a DEFAULT is permitted with a stated reason. Logged in Complexity Tracking. |
| Data hygiene — UTF-8 throughout | UTF-8 database; `UnicodeRanges.All` JSON encoder so accents emit literally | **PASS** |
| Data hygiene — timestamps UTC + ISO 8601 | No entity in the spec carries a timestamp; none invented | **N/A** |
| Automate the checkable | Spectral lints the contract; `dotnet format`, analyzers, nullable-as-error; Schemathesis in CI; red CI blocks merge | **PASS** |
| New dependencies justified in one line | research.md, Dependency justifications | **PASS** |
| **SWE** — contract-first, lintable, frozen; no endpoint absent from it | `contracts/openapi.yaml` hand-authored in Phase 1 *before* any code; Swashbuckle output diffed against it in CI | **PASS** — see Complexity Tracking for how code-first Swashbuckle is reconciled with contract-first |
| **SWE** — every operation declares all outcomes; RFC 7807 | Every operation enumerates its full status set; all errors are `application/problem+json` | **PASS** |
| **SWE** — dependencies point inward; DTOs at the edge | Domain models and repository interfaces in `Application`; Dapper confined to `Infrastructure`; wire DTOs confined to `Api` | **PASS** |
| **SWE** — strict static typing on public interfaces; contract tests validate running code | Nullable reference types enforced as errors; Schemathesis exercises the running service against the frozen contract | **PASS** |

**Result: gate PASSES.** No unjustified violations. Two entries carry documented
justifications, both recorded in Complexity Tracking.

### Post-design re-evaluation (after Phase 1)

Re-checked against the artifacts actually produced. The gate still passes, and the design
phase tightened three entries rather than loosening any:

| Rule | What Phase 1 changed | Verdict |
|---|---|---|
| Contract-first, frozen, no endpoint absent from it | `contracts/openapi.yaml` exists and was authored before any code. Machine-verified: **17 operations (5 aves / 6 familias / 6 ecosistemas)**, exactly matching the brief's inventory; no duplicate `operationId`; every `$ref` resolves. | **PASS — stronger** |
| Every operation declares all outcomes | Verified per operation. Reads declare 200/400/404, creates 201/400/409(/422), updates 200/400/404/409(/422), deletes 204/400/404(/409). Bare list operations declare only 200 — correct, as an empty catalog is a `200` with `[]`, never an error. | **PASS** |
| RFC 7807 on every error | All six reusable error responses are `application/problem+json`; validation failures extend it with `errors`. | **PASS** |
| Dependencies point inward; DTOs at the edge | data-model.md separates the domain (`Bird`, English, scalar FKs) from the read model (`BirdWithClassification`) from the wire DTO (`Ave`, Spanish, nested). Three distinct shapes, no persistence type on the wire. | **PASS — stronger** |
| Contract tests validate running code against the contract file | quickstart.md wires Schemathesis to the frozen file and adds a Swashbuckle drift check, making "no endpoint in code that isn't in the contract" mechanically enforced rather than asserted. | **PASS — stronger** |
| Reproducible from a clean checkout | quickstart.md's six-step sequence runs from `docker compose down -v`. One schema artifact serves both Compose and the tests. | **PASS** |
| Data hygiene — UUIDs | UUIDv4 throughout the contract, `readOnly` on every `id`. Override still the documented one below. | **PASS with the same override** |

One design decision was corrected during this phase: an initial draft served `PUT /api/aves`
*and* `PUT /api/ave`, which contradicted R-011's decision to reproduce the brief's inventory
without aliases and would have made the count 18. The alias was removed.

### Spec discrepancy raised upstream (not patched here)

**SC-002 undercounts the operation inventory.** It reads "All 16 operations from the brief
(5 birds, 6 families, 5 ecosystems)". The brief lists **six** ecosystem operations — list,
get-by-id, birds-by-id, add, update, delete — so the true total is **17**. The spec's own
Gherkin already covers all six (FR-005, FR-007, FR-012), so this is an arithmetic slip in
the prose enumeration, not a scope gap.

The contract implements all 17. Per the constitution's *intent flows downhill*, the fix
belongs in the spec: SC-002 should read "All 17 operations (5 birds, 6 families,
6 ecosystems)". Flagged here rather than silently absorbed.

### Spec open questions, as resolved for this plan

The spec closes with three open questions. None blocks the design; each is answered here
using the spec's own stated assumption, and each answer is cheap to reverse.

| Question | Resolution in this plan | Cost to change |
|---|---|---|
| 1 — Access control | Out of scope per spec Assumptions. No authn/authz; every endpoint is anonymous. | Additive: middleware plus a contract security scheme. |
| 2 — Whole-record vs partial correction | **Whole-record replacement.** The brief's `PUT /api/ave` carries no path id, so the id arrives in the body and the body is the complete record. Omitting an optional field clears it. | Additive: a `PATCH` alongside `PUT`. |
| 3 — Uniqueness case sensitivity | **Exact match**, per the spec's stated assumption. A plain `UNIQUE` constraint; `Páramo` and `paramo` coexist. | One line: swap to a `UNIQUE` index on `lower(nombre)`. |

## Project Structure

### Documentation (this feature)

```text
specs/001-birds-catalog-api/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/
│   └── openapi.yaml     # Phase 1 output — frozen, contract-first
├── checklists/
│   └── requirements.md
└── tasks.md             # Phase 2 — created by /speckit-tasks, NOT by this command
```

### Source Code (repository root)

```text
global.json                      # pins the .NET 10 SDK band
BirdsCatalog.sln
Directory.Build.props            # nullable, warnings-as-errors, analyzers, lockfiles

src/
├── BirdsCatalog.Api/            # Controller layer (brief: "Controller")
│   ├── Controllers/             #   AvesController, FamiliasController, EcosistemasController
│   ├── Contracts/               #   wire DTOs — Spanish field names, the language boundary
│   ├── Mapping/                 #   domain <-> DTO, explicit, no AutoMapper
│   ├── ErrorHandling/           #   exception -> RFC 7807 ProblemDetails
│   ├── OpenApi/                 #   Swashbuckle configuration and 3.1 emission
│   ├── Program.cs
│   └── Dockerfile
├── BirdsCatalog.Application/    # Service layer (brief: "Service") + domain (brief: "Model")
│   ├── Domain/                  #   Bird, Family, Ecosystem — English, persistence-free
│   ├── Abstractions/            #   IBirdRepository, IFamilyRepository, IEcosystemRepository
│   ├── Services/                #   business rules: FR-014, FR-016, FR-017, FR-018
│   └── Errors/                  #   NotFound / Conflict / DanglingReference / InUse
└── BirdsCatalog.Infrastructure/ # Repository layer (brief: "Repository") + connection (brief: "Context")
    ├── Persistence/             #   NpgsqlConnectionFactory — the brief's "Context" role
    ├── Repositories/            #   Dapper implementations
    └── Sql/                     #   SQL kept out of C# string soup

tests/
├── BirdsCatalog.UnitTests/         # service rules, mapping, in-memory fakes
├── BirdsCatalog.IntegrationTests/  # WebApplicationFactory + Testcontainers PostgreSQL
└── contract/
    ├── schemathesis.Dockerfile
    └── run-contract-tests.sh       # st run against contracts/openapi.yaml

db/
└── init/
    └── 001_schema.sql           # tables, UNIQUE constraints, RESTRICT foreign keys

scripts/
└── seed-demo-data.ps1           # SC-003: seeds the 5 birds THROUGH THE API, not via SQL

docker-compose.yml               # api + postgres, healthcheck-gated startup
```

**Structure Decision**: Three projects, not one and not four. The count is set by the
brief, which mandates the Controller/Service/Repository/Model/Context layering as a
deliverable rather than as an internal preference — so the layering must be visible in the
project graph, and the compiler must be able to enforce it. `Api → Application ←
Infrastructure` turns the constitution's *dependencies point inward* rule into a build
error rather than a code-review note: `Application` references neither of the others, so no
Dapper type and no `HttpContext` can reach the domain. Domain models sit inside
`Application` rather than in a fourth `Domain` project because at three entities the extra
assembly buys nothing that the folder boundary does not already give.

The brief's five layer *names* map onto this as: Controller → `Api/Controllers`,
Service → `Application/Services`, Repository → `Infrastructure/Repositories`,
Model → `Application/Domain`, Context → `Infrastructure/Persistence/NpgsqlConnectionFactory`.
The Context role is a connection factory rather than an EF-style `DbContext` because Dapper
has no unit of work; naming it `DbContext` would imply change tracking that does not exist.

## Complexity Tracking

| Violation | Why Needed | Simpler Alternative Rejected Because |
|---|---|---|
| **UUIDv4 instead of the constitution's UUIDv7/ULID DEFAULT** | FR-013 requires identifiers "not guessable or inferable from creation order". UUIDv7 embeds a millisecond timestamp in its high bits, so creation order is directly recoverable from any two ids — it fails the requirement by construction. The constitution marks this a DEFAULT, overridable with a documented reason; this is that reason. | UUIDv7 rejected because it violates FR-013. Its benefit is B-tree insert locality, which matters at write volumes this catalog does not have (5 demo rows, no stated throughput target). If volume ever justifies it, FR-013 must be revisited first — the ordering leak is the entire point of v7. |
| **Code-first Swashbuckle under a contract-first constitution** | The stack mandates Swashbuckle, which generates OpenAPI *from* code — the opposite direction to the SWE rule that the contract is "written and linted before code, and frozen". Reconciled by inverting the roles: `contracts/openapi.yaml` is hand-authored now, linted by Spectral, and is the single source of truth; Swashbuckle's runtime document is treated purely as an *observation of the code* and diffed against the frozen file in CI. Drift fails the build, and the fix is always to change the code or re-flow through the spec — never to regenerate the contract from the code. | Swashbuckle-as-source-of-truth rejected: it would make the contract a downstream artifact of the code, so no endpoint could ever be "absent from the contract" and the rule would be vacuous. Dropping Swashbuckle rejected: the user's stack specifies it, and it earns its place as the drift detector and Swagger UI host. |
| **Three projects rather than one** | The brief specifies the layered distribution as an explicit deliverable; project boundaries make the layering compiler-enforced. | A single project with folders rejected: nothing would stop a controller from taking an `NpgsqlConnection`, and the inward-dependency rule would rely on reviewer vigilance. |
| **Testcontainers as a test dependency** | Dapper's value is hand-written SQL, and that SQL is only meaningfully tested against a real PostgreSQL. Testcontainers gives an ephemeral, port-collision-free database per run, which is what "reproducible from a clean checkout" demands of the test suite. | A shared Compose-managed test database rejected: it carries state between runs and collides on developer machines, making failures order-dependent. In-memory fakes rejected at this layer: they test the fake, not the SQL. |
