# Phase 0 Research: Birds of Colombia Catalog

**Feature**: `001-birds-catalog-api` | **Date**: 2026-08-26 | **Plan**: [plan.md](./plan.md)

The user's input fixed the stack, so there were no `NEEDS CLARIFICATION` markers to resolve
in Technical Context. The open questions were instead the *consequences* of that stack —
places where a default behaviour of .NET, PostgreSQL or Swashbuckle would quietly violate a
requirement in the spec. Those are what this document records.

Two findings (R-001, R-004) are cases where the obvious implementation fails a stated
requirement. They are the reason this file exists.

---

## R-001 — UUID version: v4, not v7

**Decision**: Server-assigned **UUIDv4** via `Guid.NewGuid()`, stored in a PostgreSQL
`uuid` column. Never client-supplied; a client-provided `id` on create is ignored.

**Rationale**: FR-013 requires that an identifier "is not guessable or inferable from
creation order". UUIDv7 places a 48-bit Unix-millisecond timestamp in its most significant
bits, so sorting two v7 values recovers their creation order exactly — it fails FR-013 by
construction, not by accident. UUIDv4 is 122 bits of randomness with no ordering signal.

This overrides the constitution's data-hygiene DEFAULT of "UUIDv7/ULID". The constitution
defines DEFAULT as "the choice absent a documented reason", overridable with one stated
sentence; FR-013 is that reason, and this is that sentence. The rule's actual intent —
"NEVER sequential integers across a boundary" — is fully honoured.

**Alternatives considered**:

- *UUIDv7 (`Guid.CreateVersion7()`, available since .NET 9)* — rejected: leaks creation
  order, contradicting FR-013. Its advantage is index locality: random v4 keys scatter
  B-tree inserts and cause page splits at high write volume. The spec asserts no write
  volume, and the demonstration dataset is five rows, so the cost is theoretical here. If
  write volume ever makes it real, FR-013 must be renegotiated *first*.
- *ULID* — same time-ordering objection as v7, plus it is not a native PostgreSQL type.
- *Database-generated (`gen_random_uuid()` in a column DEFAULT)* — rejected as the primary
  mechanism: the id would not be known to the service layer until after the `INSERT`
  returns, and generating it in C# keeps the domain object valid from construction. The
  column default is still declared as a defence-in-depth backstop.

---

## R-002 — Bird responses embed family and ecosystem in full

**Decision**: `GET /api/aves` and `GET /api/ave/{ave_id}` return each bird with its
`familia` and `ecosistema` as complete nested objects (`id` + `nombre` + `orden` /
`zona_geografica`), not as bare id references. Writes go the other way: `POST`/`PUT` accept
`familia_id` and `ecosistema_id` scalars.

**Rationale**: SC-001 requires that a consumer obtains a bird's complete classification —
"names, family with its order, ecosystem with its zone — from this catalog alone". With
bare foreign keys, satisfying SC-001 costs three round-trips per bird and pushes the join
onto every client. FR-003's "its family association and its ecosystem association" is
satisfied either way, so SC-001 is the deciding requirement.

The asymmetry between read and write shapes is deliberate: a write names a reference, a
read resolves it.

**Alternatives considered**:

- *Bare `familia_id` / `ecosistema_id` on reads* — rejected: forces N+1 client round-trips
  and fails SC-001's "from this catalog alone" as a single-request guarantee.
- *An `?expand=` query parameter* — rejected: the brief's operation inventory has no query
  parameters, and the spec's Assumptions place filtering out of scope. Adding one would be
  inventing unspecified behaviour.

**Consequence for the repository layer**: bird reads are a single `JOIN` across all three
tables mapped with Dapper's multi-mapping (`splitOn`), not three sequential queries.

---

## R-003 — Error model: RFC 7807, and the specific status code per rule

**Decision**: Every failure returns `application/problem+json`. The mapping from the spec's
rules to status codes:

| Situation | Spec rule | Status | Why this code |
|---|---|---|---|
| Malformed identifier in path | FR-015 | **400** | Syntactically invalid request |
| No record with that identifier | FR-015 | **404** | Well-formed, absent |
| Mandatory field missing | FR-016 | **400** | Body fails schema validation |
| Duplicate scientific name / family name / ecosystem name | FR-017 | **409** | Conflict with existing state |
| `familia_id` or `ecosistema_id` references nothing | FR-014 | **422** | Well-formed and schema-valid, but semantically unprocessable |
| Deleting a family or ecosystem still in use | FR-018 | **409** | Conflict with existing state |

**Rationale**: FR-015 turns on the 400/404 distinction being observable, so the split has to
be exact rather than approximate. 422 separates "your JSON is fine but points at something
that does not exist" (FR-014) from "your JSON is malformed" (400), which lets a client tell
a typo'd UUID from a deleted family. 409 covers both uniqueness (FR-017) and
referential-refusal (FR-018) because both are conflicts with the current state of the
catalog rather than defects in the request.

Every `ProblemDetails` carries a stable `type` URI, a human-readable `detail` naming the
offending field or value, and validation failures additionally carry ASP.NET's `errors`
dictionary. This satisfies SC-006: the caller learns what went wrong without reading logs.

**Alternatives considered**:

- *422 for missing mandatory fields* — rejected: ASP.NET's `[ApiController]` already emits
  400 with a validation `ProblemDetails` for model-binding failures, and fighting that
  default costs more than it returns.
- *409 for dangling references* — rejected: it would collide with FR-017's duplicate case
  and make the two indistinguishable without parsing prose.

---

## R-004 — Route binding: do **not** use the `:guid` route constraint

**Decision**: Declare identifier path parameters as `Guid` **without** the `{id:guid}` route
constraint — `[HttpGet("{ave_id}")] public async Task<IActionResult> Get(Guid ave_id)`.

**Rationale**: This is the single sharpest trap in the feature. FR-015 requires that a
malformed identifier be reported as invalid, "not that the record is missing". With
`{ave_id:guid}`, a non-UUID value **fails route matching**, so ASP.NET never reaches the
controller and returns **404** — precisely the confusion FR-015 exists to forbid. Removing
the constraint lets the route match, model binding then fails, and `[ApiController]`
converts that into a **400** validation `ProblemDetails` automatically.

The correct behaviour here is what the framework does when you use *less* of it.

**Alternatives considered**:

- *`{ave_id:guid}` plus a catch-all fallback route returning 400* — rejected: it works, but
  encodes a requirement in routing-table ordering where the next person will not look for
  it.
- *Binding as `string` and calling `Guid.TryParse` in each action* — rejected: correct but
  repeated 11 times, and it moves validation out of the layer that already does it.

**Verification**: an integration test asserts `GET /api/ave/not-a-uuid` → 400 and
`GET /api/ave/{unused-but-valid-uuid}` → 404. Both cases are in the contract.

---

## R-005 — Non-ASCII round-tripping

**Decision**: PostgreSQL database created with `UTF8` encoding. In ASP.NET, set
`JsonSerializerOptions.Encoder = JavaScriptEncoder.Create(UnicodeRanges.All)`.

**Rationale**: SC-007 requires accented and non-ASCII text to round-trip byte-for-byte, and
FR-019 names `Cóndor de los Andes`, `Páramo` and `Kuntur` explicitly. `System.Text.Json`
defaults to a strict encoder that escapes every non-ASCII character, so `Páramo` is emitted
as `Páramo`. That is *valid* JSON and decodes back to the same string, so it arguably
satisfies a lenient reading of SC-007 — but it is not byte-identical, it breaks naive
consumers, and it makes the Swagger UI unreadable. Configuring the encoder emits the
character literally.

Npgsql and PostgreSQL both handle UTF-8 natively; no client-encoding setting is needed
beyond creating the database as UTF8.

**Alternatives considered**:

- *`JavaScriptEncoder.UnsafeRelaxedJsonEscaping`* — rejected: it also stops escaping
  HTML-sensitive characters (`<`, `>`, `&`), which is a needless XSS exposure for consumers
  that interpolate responses into markup. `UnicodeRanges.All` relaxes exactly the escaping
  SC-007 concerns and no more.
- *Leaving the default escaping* — rejected: fails a byte-for-byte reading of SC-007.

**Verification**: an integration test posts `Cóndor de los Andes` / `Páramo` / `Kuntur`,
then asserts the raw HTTP response bytes contain the literal UTF-8 sequences.

---

## R-006 — Schema management: versioned SQL, no migration framework

**Decision**: A single `db/init/001_schema.sql`, mounted into the PostgreSQL container's
`/docker-entrypoint-initdb.d/`. Integration tests apply the same file to their
Testcontainers instance, so one artifact defines the schema everywhere.

**Rationale**: "Reproducible from a clean checkout" is the constitutional requirement, and
one committed SQL file applied identically by Compose and by the test suite meets it with
zero dependencies. The schema is three tables with no planned evolution; a migration
framework would be machinery for a problem this feature does not have.

The known limitation is stated rather than hidden: `docker-entrypoint-initdb.d` runs **only
when the data volume is empty**. Changing the schema after first run requires
`docker compose down -v`. That is acceptable while the schema is being authored and
becomes unacceptable the moment real data exists.

**Alternatives considered**:

- *DbUp* — rejected for now, and named as the migration path. It is small, applies
  idempotent versioned scripts at startup, and fits Dapper. Adopt it the first time the
  schema changes with data present.
- *EF Core Migrations* — rejected: pulling in EF Core for migrations alone, on a project
  that deliberately uses Dapper, is a large dependency for a small job.
- *Flyway as a fourth container* — rejected: more orchestration than three tables warrant.

---

## R-007 — Enforcing uniqueness and referential refusal in the right place

**Decision**: Enforce **both** in the database and in the service layer, with distinct jobs.

- Database: `UNIQUE` on `aves.nombre_cientifico`, `familias.nombre`, `ecosistemas.nombre`;
  foreign keys declared `ON DELETE RESTRICT`.
- Service: pre-checks that produce the specific, human-readable message the spec demands.
- Repository: translates PostgreSQL SQLSTATE `23505` (unique violation) into the conflict
  error and `23503` (foreign key violation) into the dangling-reference error.

**Rationale**: A service-layer check alone is a race — two concurrent creates both read
"no duplicate", both insert, and the catalog ends with two `Vultur gryphus` rows, violating
FR-017. Only the database constraint is a real guarantee. But a bare constraint violation
surfaces as an opaque driver exception, which fails SC-006's requirement that the caller
learns what went wrong. So the pre-check exists for the *message* and the constraint exists
for the *guarantee*, and the SQLSTATE translation covers the narrow window between them.

`ON DELETE RESTRICT` is what makes FR-018 structural: deleting a family that still has birds
cannot succeed even if a future code path forgets to check. `CASCADE` would silently delete
birds, which FR-018 forbids in as many words ("no bird is deleted or left unclassified").

**Alternatives considered**:

- *Service-layer checks only* — rejected: racy, as above.
- *Database constraints only* — rejected: opaque messages, fails SC-006.
- *Serializable transactions instead of constraints* — rejected: a heavier concurrency
  control than a `UNIQUE` index, with retry handling the spec never asked for.

---

## R-008 — Swashbuckle emits OpenAPI 3.1; the frozen file stays the source of truth

**Decision**: Use `Swashbuckle.AspNetCore` configured to emit OpenAPI **3.1**, served at
`/swagger/v1/swagger.json` with Swagger UI at `/swagger`. A CI step fetches that document
from the running container and diffs it against `contracts/openapi.yaml`; any difference
fails the build.

**Rationale**: The constitution requires the contract to precede code and be frozen, while
Swashbuckle generates the contract *from* code. Rather than pick a winner, the two are given
different jobs: the hand-authored file is the source of truth, and Swashbuckle's output is
an *observation of the implementation* used to detect drift. This is strictly stronger than
either alone — it is what mechanically enforces "no endpoint exists in code that isn't in
the contract", which a hand-written file cannot enforce on its own.

**Version pinning is required, not optional.** OpenAPI 3.1 emission is version-dependent
across the Swashbuckle 7.x/8.x/9.x line, and .NET 10 templates no longer include Swashbuckle
by default (the built-in `Microsoft.AspNetCore.OpenApi` took that slot). The first
implementation task therefore pins an exact Swashbuckle version and **asserts** that the
emitted document begins `openapi: 3.1.` — this is verified in the build, not assumed here.

**Fallback if that assertion fails**: switch the emitter to the built-in
`Microsoft.AspNetCore.OpenApi` package, which targets OpenAPI 3.1 on .NET 10, and keep
Swashbuckle solely for Swagger UI. The frozen contract, the drift check and every consumer
are unaffected, because the source of truth never depended on the generator. This risk is
contained by design.

**Alternatives considered**:

- *Generate the contract from code and commit the output* — rejected: makes the contract a
  downstream artifact, so the "no endpoint absent from the contract" rule becomes
  unfalsifiable.
- *Hand-authored contract with no drift check* — rejected: nothing then stops code and
  contract from diverging, which is the failure mode the SWE profile targets.

---

## R-009 — Contract testing with Schemathesis

**Decision**: Run Schemathesis from its own container against the frozen
`contracts/openapi.yaml`, targeting the live Compose service. Enable
`status_code_conformance`, `response_schema_conformance`, `content_type_conformance` and
`not_a_server_error`. Seed fixed for determinism.

**Rationale**: This is the constitution's "contract tests validate the running code against
the contract file", satisfied literally. Pointing Schemathesis at the frozen file rather
than at Swashbuckle's live document is what makes the test meaningful — testing generated
output against the code that generated it proves nothing.

Two consequences shape the contract itself:

- Every operation must enumerate **every** status code it can return, or
  `status_code_conformance` fails on legitimate behaviour. This is the same discipline the
  constitution already requires ("every operation declares all outcomes, errors included"),
  so the tool enforces a rule that already applied.
- Schemathesis generates hostile values against declared types, including non-UUID strings
  for `format: uuid` path parameters. Those must produce a documented **400** — which is
  exactly R-004's behaviour. Schemathesis will therefore catch the `:guid` constraint trap
  automatically if it is ever reintroduced.

Fixed determinism note: a seed is pinned so a failing run is reproducible, per the
constitution's "seeds fixed where determinism is feasible".

**Alternatives considered**:

- *Dredd* — rejected: less actively maintained, weaker OpenAPI 3.1 support.
- *Hand-written contract tests only* — rejected: they test the cases the author thought of,
  which is the set least likely to contain the bug.

---

## R-010 — Language boundary: Spanish on the wire, English in the domain

**Decision**: Routes, JSON field names and database columns use the brief's Spanish
vocabulary (`/api/aves`, `nombre_cientifico`, `zona_geografica`). C# domain types use
English (`Bird`, `Family`, `Ecosystem`). Translation happens exactly once, in the `Api`
project's explicit mapping code.

**Rationale**: The brief fixes the Spanish route paths and column names, and the spec names
the entities in English with the Spanish in parentheses. Both are honoured because the
constitution already mandates a DTO layer at the edge ("NEVER leak persistence models onto
the wire") — so the mapping code exists regardless, and the language boundary rides along at
no additional cost. Putting the boundary anywhere else would scatter Spanish identifiers
through business logic.

**Alternatives considered**:

- *Spanish throughout the C# code* — rejected: mixes languages inside logic and reads
  poorly against .NET naming conventions.
- *English on the wire* — rejected: contradicts the brief's explicit endpoint inventory.

---

## R-011 — Path naming keeps the brief's inconsistencies verbatim

**Decision**: Implement the brief's paths exactly as written, including
`GET /api/ave/{ave_id}` (singular) alongside `DELETE /api/aves/{ave_id}` (plural), and the
body-carried id on `PUT /api/ave` and `PUT /api/familias`.

**Rationale**: The brief is an acceptance document — it states that submissions with
integer ids "will not be accepted", so its inventory is being checked against. Normalising
the paths to a consistent plural would be more tasteful and would fail that check. The
inconsistency is recorded here so that it reads as a deliberate decision rather than a
transcription error by whoever reads the contract next.

**Alternatives considered**:

- *Normalise everything to plural* — rejected: diverges from the acceptance inventory.
- *Serve both, with the tidy paths as aliases* — rejected: doubles the surface area of the
  contract for cosmetic gain, and every alias is an endpoint that must be specified,
  tested and maintained.

---

## R-012 — Startup ordering under Compose

**Decision**: The `postgres` service declares a `pg_isready` healthcheck; the `api` service
declares `depends_on: { postgres: { condition: service_healthy } }`. The API additionally
exposes `/health`, which verifies it can open a connection.

**Rationale**: A plain `depends_on` waits only for the container to *start*, not for
PostgreSQL to accept connections, so the API loses the race and crashes on first query.
This is the most common way a working Compose setup fails on a colleague's slower machine —
and "reproducible from a clean checkout" means reproducible there too. The healthcheck
gate makes startup deterministic rather than lucky.

`/health` also gives the contract-test and seed scripts a reliable readiness signal to poll
instead of a fixed sleep.

**Alternatives considered**:

- *A `wait-for-it.sh` wrapper on the API entrypoint* — rejected: Compose healthchecks are
  native and need no script in the image.
- *Retry-on-connect in application startup* — worth having eventually, but it hides a
  misconfigured dependency behind a timeout rather than declaring it.

---

## Dependency justifications

The constitution requires one line per new dependency: why, and why not the standard
library.

| Dependency | Why | Why not stdlib / built-in |
|---|---|---|
| `Dapper` | Explicitly specified by the user; thin `IDbConnection` mapping over hand-written SQL | Raw ADO.NET works but hand-maps every column, which is boilerplate with no payoff |
| `Npgsql` | The PostgreSQL ADO.NET driver; there is no in-box alternative | No PostgreSQL provider ships with .NET |
| `Swashbuckle.AspNetCore` | Explicitly specified; serves Swagger UI and produces the runtime document used for the drift check | `Microsoft.AspNetCore.OpenApi` is the in-box alternative and is the documented fallback in R-008 |
| `Testcontainers.PostgreSql` | Ephemeral, isolated PostgreSQL per test run, so Dapper's SQL is tested against the real engine | No in-box equivalent; the alternative is a shared database with cross-run state |
| `xunit` | Test framework | No in-box test framework |
| `schemathesis` (container, not a NuGet reference) | Property-based contract conformance against the frozen contract | No .NET equivalent with comparable OpenAPI 3.1 fuzzing |

Every one of these is pinned to an exact version with `packages.lock.json` committed.

---

## Deliberately not researched

Out of scope per the spec's Assumptions, and therefore absent from the design rather than
deferred within it: search, filtering, sorting, pagination, bulk import, images, audio,
coordinates, conservation status, caching, rate limiting, authentication and authorisation.

Entity timestamps (`created_at` / `updated_at`) are also absent. The constitution's
timestamp hygiene rule governs timestamps that exist; no entity in the spec carries one, and
inventing them would be exactly the "unspecified behaviour" the constitution says to stop
and ask about. If audit history is wanted, it enters at the spec.
