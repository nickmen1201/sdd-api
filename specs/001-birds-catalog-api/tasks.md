---
description: "Task list for feature implementation"
---

# Tasks: Birds of Colombia Catalog

**Input**: Design documents from `/specs/001-birds-catalog-api/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md), [data-model.md](./data-model.md), [contracts/openapi.yaml](./contracts/openapi.yaml), [quickstart.md](./quickstart.md)

**Tests**: Test tasks ARE included. The constitution's SWE profile makes contract tests a
MUST ("Contract tests validate the running code against the contract file"), and plan.md
names the suite: xUnit + `WebApplicationFactory`, Testcontainers.PostgreSql, Schemathesis.

**Organization**: Grouped by the spec's `@P1..@P4` features, which partition the frozen
contract's **17 operations** exactly — 6 / 3 / 6 / 2 — so every phase is a shippable slice.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: `[US1]`..`[US4]`, mapping to the spec's `@P1`..`@P4` features
- Exact file paths are given in every task

## Path Conventions

Three-project layout per plan.md, *Source Code (repository root)*:
`src/BirdsCatalog.{Api,Application,Infrastructure}/`, `tests/`, `db/init/`, `scripts/`.
The dependency direction `Api → Application ← Infrastructure` is compiler-enforced —
`Application` references neither of the others.

## Requirement coverage map

The spec's unprioritised **Data integrity** feature is cross-cutting; its requirements are
distributed to the phase that owns the code path:

| Requirement | Phase |
|---|---|
| FR-013 (UUIDv4 ids), FR-019 (non-ASCII round-trip) | Phase 2 — Foundational |
| FR-001, FR-002, FR-003, FR-004, FR-005, FR-015 (read paths) | Phase 3 — US1 |
| FR-008, FR-009, FR-010, FR-014, FR-016, FR-017 (birds) | Phase 4 — US2 |
| FR-011, FR-012, FR-017 (vocabularies), FR-018 | Phase 5 — US3 |
| FR-006, FR-007 | Phase 6 — US4 |
| FR-020 / SC-003 (demonstration dataset) | Phase 7 — Polish |

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project skeleton, the compiler-enforced layering, and the toolchain that makes
the constitution's "automate the checkable" rule real.

- [ ] T001 Create `global.json` at repository root pinning the .NET 10 SDK band with `rollForward: latestFeature`
- [ ] T002 Create `BirdsCatalog.sln` and three projects — `src/BirdsCatalog.Api` (`webapi`, controller-based, not the minimal-API template), `src/BirdsCatalog.Application` (`classlib`), `src/BirdsCatalog.Infrastructure` (`classlib`) — wiring references `Api → Application`, `Api → Infrastructure`, `Infrastructure → Application`, and **no** reference out of `Application` (plan.md, Structure Decision)
- [ ] T003 Create `Directory.Build.props` at repository root enabling `Nullable=enable` with `TreatWarningsAsErrors=true`, `ImplicitUsings=enable`, .NET analyzers, and `RestorePackagesWithLockFile=true` so `packages.lock.json` is committed (constitution: reproducible from a clean checkout)
- [ ] T004 [P] Create xUnit test projects `tests/BirdsCatalog.UnitTests` and `tests/BirdsCatalog.IntegrationTests`, add both to `BirdsCatalog.sln`, and reference the projects under test
- [ ] T005 [P] Add `.editorconfig` at repository root and confirm `dotnet format --verify-no-changes` exits zero
- [ ] T006 [P] Add `.spectral.yaml` at repository root extending `spectral:oas`, and confirm `specs/001-birds-catalog-api/contracts/openapi.yaml` lints clean — the contract is frozen, so a lint failure is fixed in the contract now, before any code exists
- [ ] T007 [P] Add `.gitignore` and `.dockerignore` at repository root covering `bin/`, `obj/`, and local secrets

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Schema, container topology, domain types, error model, and the two encoder and
routing decisions the whole feature's correctness rests on.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

- [ ] T008 Create `db/init/001_schema.sql` reproducing the DDL in data-model.md verbatim — `pgcrypto`, tables `familias`/`ecosistemas`/`aves`, the three `UNIQUE` constraints, both foreign keys as **`ON DELETE RESTRICT`** (never `CASCADE` or `SET NULL` — FR-018 forbids both outcomes), and indexes `ix_aves_familia_id` / `ix_aves_ecosistema_id`
- [ ] T009 [P] Create `src/BirdsCatalog.Api/Dockerfile` as a multi-stage build on `mcr.microsoft.com/dotnet/sdk:10.0` then `mcr.microsoft.com/dotnet/aspnet:10.0`
- [ ] T010 Create `docker-compose.yml` at repository root with `postgres:17` (UTF-8 encoding, `db/init/` mounted at `/docker-entrypoint-initdb.d`, a `pg_isready` healthcheck) and the `api` service on port 8080 declaring `depends_on: condition: service_healthy` (research.md R-012); the dev password is supplied by Compose only, never committed elsewhere
- [ ] T011 [P] Create domain entities `Bird`, `Family`, `Ecosystem` in `src/BirdsCatalog.Application/Domain/` — English names, plain C# types, no persistence or ASP.NET attributes; `Bird` carries `FamilyId`/`EcosystemId` **scalars**, not object references (data-model.md, Domain model shape)
- [ ] T012 [P] Create the read model `BirdWithClassification` in `src/BirdsCatalog.Application/Domain/BirdWithClassification.cs` holding a `Bird` plus its resolved `Family` and `Ecosystem` (research.md R-002)
- [ ] T013 [P] Create domain error types `NotFoundError`, `ConflictError`, `DanglingReferenceError`, `InUseError` in `src/BirdsCatalog.Application/Errors/` — one per row of the response-code table in data-model.md
- [ ] T014 [P] Create repository interfaces `IBirdRepository`, `IFamilyRepository`, `IEcosystemRepository` in `src/BirdsCatalog.Application/Abstractions/`, covering read, write and dependent-count operations
- [ ] T015 Create `src/BirdsCatalog.Infrastructure/Persistence/NpgsqlConnectionFactory.cs` — the brief's "Context" role as a connection factory, not a `DbContext`, since Dapper has no unit of work — plus a `ServiceCollection` extension registering it and the repositories
- [ ] T016 Bind the connection string from the `ConnectionStrings__BirdsCatalog` environment variable in `src/BirdsCatalog.Api/Program.cs`, with no credential in `appsettings.json` (constitution: secrets never in the repo)
- [ ] T017 Create `src/BirdsCatalog.Api/ErrorHandling/ProblemDetailsExceptionHandler.cs` mapping every domain error to `application/problem+json` — `NotFoundError` to 404, `ConflictError` to 409, `DanglingReferenceError` to 422, model-binding failure to 400 with an `errors` object — matching the contract's `ProblemDetails` and `ValidationProblemDetails` schemas and its six reusable error responses (research.md R-003)
- [ ] T018 Configure `src/BirdsCatalog.Api/Program.cs` with `JavaScriptEncoder.Create(UnicodeRanges.All)` so accents emit literally rather than as `\u00xx` escapes (FR-019, SC-007, research.md R-005), and register controllers, DI and the `/health` endpoint
- [ ] T019 [P] Create `src/BirdsCatalog.Api/OpenApi/` Swashbuckle configuration emitting **OpenAPI 3.1** at `/swagger/v1/swagger.json` with Swagger UI at `/swagger`; the generated document is an observation of the code, never the source of truth (research.md R-008)
- [ ] T020 [P] Create the shared wire DTO `Identificador` in `src/BirdsCatalog.Api/Contracts/Identificador.cs` matching the contract's `Identificador` schema, returned by every `201 Created` alongside its `Location` header
- [ ] T021 Create `tests/BirdsCatalog.IntegrationTests/Fixtures/PostgresFixture.cs` — a Testcontainers.PostgreSql container applying `db/init/001_schema.sql`, wrapped by a `WebApplicationFactory` that overrides the connection string, plus per-test truncation so runs are order-independent
- [ ] T022 [P] Create in-memory repository fakes in `tests/BirdsCatalog.UnitTests/Fakes/` implementing the three abstractions from T014, for service-rule tests that must not touch a database

**Checkpoint**: `docker compose up --build -d` serves `/health`, `/swagger` renders, and
`dotnet test` runs green against an empty suite.

---

## Phase 3: User Story 1 - Consult the catalog (Priority: P1) 🎯 MVP

**Goal**: A consumer obtains a bird's complete classification — names, family with its
order, ecosystem with its zone — from one request against this catalog alone (SC-001).
Delivers the **6 read operations**: `listarAves`, `obtenerAve`, `listarFamilias`,
`obtenerFamilia`, `listarEcosistemas`, `obtenerEcosistema`.

**Independent Test**: Seed rows directly through the T021 fixture — not via the API, since
the write endpoints do not exist yet — then `GET /api/aves` and confirm each bird carries
`familia` **with `orden`** and `ecosistema` **with `zona_geografica`** as embedded objects.
An empty catalog returns `200` with `[]`. A malformed id returns `400`, an unknown id `404`.

### Tests for User Story 1 ⚠️

> Write these first and confirm they FAIL before implementing.

- [ ] T023 [P] [US1] Contract tests for `listarAves` and `obtenerAve` in `tests/BirdsCatalog.IntegrationTests/Aves/ReadAvesContractTests.cs` — assert the `200` bodies validate against the frozen contract's `Ave` schema, including nested `familia` and `ecosistema`
- [ ] T024 [P] [US1] Contract tests for `listarFamilias`/`obtenerFamilia` and `listarEcosistemas`/`obtenerEcosistema` in `tests/BirdsCatalog.IntegrationTests/Vocabularios/ReadVocabulariosContractTests.cs` (FR-004, FR-005)
- [ ] T025 [P] [US1] Integration test in `tests/BirdsCatalog.IntegrationTests/Aves/EmptyCatalogTests.cs` asserting `GET /api/aves` on an empty catalog returns `200` with `[]`, never an error
- [ ] T026 [P] [US1] Integration test in `tests/BirdsCatalog.IntegrationTests/Identifiers/MalformedVsMissingIdTests.cs` asserting `GET /api/ave/not-a-uuid` returns **400** and `GET /api/ave/{unused-but-valid-uuid}` returns **404**, for all three entities — the sharpest check in the feature (FR-015, quickstart Scenario 4)
- [ ] T027 [P] [US1] Integration test in `tests/BirdsCatalog.IntegrationTests/Encoding/NonAsciiRoundTripTests.cs` asserting on the **raw response bytes** that `Cóndor de los Andes`, `Páramo` and `Kuntur` appear literally — a test that deserialises before asserting passes while SC-007 fails (FR-019, quickstart Scenario 6)

### Implementation for User Story 1

- [ ] T028 [P] [US1] Create read DTOs `Ave`, `Familia`, `Ecosistema` in `src/BirdsCatalog.Api/Contracts/` — Spanish field names, `familia` and `ecosistema` as nested objects, matching the frozen contract exactly (research.md R-010)
- [ ] T029 [P] [US1] Create explicit domain-to-DTO mappers in `src/BirdsCatalog.Api/Mapping/` with no AutoMapper, projecting `BirdWithClassification` to `Ave`
- [ ] T030 [P] [US1] Add read SQL to `src/BirdsCatalog.Infrastructure/Sql/AvesSql.cs` — a single joined `SELECT` over `aves`, `familias` and `ecosistemas` serving both list and by-id
- [ ] T031 [US1] Implement the read half of `src/BirdsCatalog.Infrastructure/Repositories/BirdRepository.cs` using Dapper multi-mapping to populate `BirdWithClassification` in one round trip (depends on T030)
- [ ] T032 [P] [US1] Implement the read half of `src/BirdsCatalog.Infrastructure/Repositories/FamilyRepository.cs` and `EcosystemRepository.cs`, with their SQL in `src/BirdsCatalog.Infrastructure/Sql/`
- [ ] T033 [P] [US1] Implement read methods on `src/BirdsCatalog.Application/Services/BirdService.cs`, raising `NotFoundError` for an unknown id (FR-015)
- [ ] T034 [P] [US1] Implement read methods on `src/BirdsCatalog.Application/Services/FamilyService.cs` and `EcosystemService.cs` (FR-004, FR-005)
- [ ] T035 [US1] Implement `GET /api/aves` and `GET /api/ave/{ave_id}` in `src/BirdsCatalog.Api/Controllers/AvesController.cs` — declare the path parameter as `Guid` **without** the `{ave_id:guid}` route constraint, so a malformed value is a model-binding `400` and not a routing `404` (FR-015, research.md R-004)
- [ ] T036 [P] [US1] Implement `GET /api/familias` and `GET /api/familias/{familia_id}` in `src/BirdsCatalog.Api/Controllers/FamiliasController.cs`, applying the same no-`:guid` rule
- [ ] T037 [P] [US1] Implement `GET /api/ecosistemas` and `GET /api/ecosistemas/{ecosistema_id}` in `src/BirdsCatalog.Api/Controllers/EcosistemasController.cs`, applying the same no-`:guid` rule
- [ ] T038 [US1] Run `dotnet test` and confirm T023–T027 under `tests/BirdsCatalog.IntegrationTests/` now pass, and that `/swagger/v1/swagger.json` matches `specs/001-birds-catalog-api/contracts/openapi.yaml` for those six paths

**Checkpoint**: US1 is independently demonstrable — quickstart Scenarios 2, 4 and 6 pass.
This is the MVP.

---

## Phase 4: User Story 2 - Curate bird records (Priority: P2)

**Goal**: A maintainer registers, corrects and removes birds so the catalog stays current
rather than a frozen dataset. Delivers **3 operations**: `crearAve`, `actualizarAve`,
`eliminarAve`.

**Independent Test**: With a family and an ecosystem seeded via the fixture, `POST` a bird,
re-read it by the returned id, `PUT` a change, confirm no other field moved, then `DELETE`
it and confirm its family and ecosystem remain untouched.

### Tests for User Story 2 ⚠️

- [ ] T039 [P] [US2] Contract tests for `crearAve`, `actualizarAve` and `eliminarAve` in `tests/BirdsCatalog.IntegrationTests/Aves/WriteAvesContractTests.cs`, asserting every declared status — 201/400/409/422, 200/400/404/409/422, and 204/400/404
- [ ] T040 [P] [US2] Integration test in `tests/BirdsCatalog.IntegrationTests/Aves/BirdLifecycleTests.cs` covering register, retrieve, correct, retrieve, remove — asserting that a correction alters no untouched field and that removal leaves the family and ecosystem in place (FR-008, FR-009, FR-010)
- [ ] T041 [P] [US2] Integration test in `tests/BirdsCatalog.IntegrationTests/Aves/DanglingClassificationTests.cs` asserting `POST` and `PUT` with a `familia_id` or `ecosistema_id` matching nothing return **422** naming the unresolved reference (FR-014)
- [ ] T042 [P] [US2] Unit tests in `tests/BirdsCatalog.UnitTests/Services/BirdValidationTests.cs` over the T022 fakes: a missing `nombre_cientifico`, `nombre_comun`, `familia_id` or `ecosistema_id` is rejected with the field named; a missing `nombre_indigena` or `lengua_indigena` is accepted with the field left null (FR-016, both branches)
- [ ] T043 [P] [US2] Integration test in `tests/BirdsCatalog.IntegrationTests/Aves/ScientificNameUniquenessTests.cs` asserting a duplicate `nombre_cientifico` returns **409** naming the duplicate, while several birds sharing the same family *and* ecosystem are all accepted — sharing a classification is not a duplicate (FR-017)

### Implementation for User Story 2

- [ ] T044 [P] [US2] Create write DTOs `AveNueva` and `AveActualizada` in `src/BirdsCatalog.Api/Contracts/` matching the frozen contract's `required` sets; `AveActualizada` carries the id **in the body**, because the brief's `PUT /api/ave` has no path id (plan.md, open question 2 — whole-record replacement)
- [ ] T045 [US2] Add insert, update and delete SQL to `src/BirdsCatalog.Infrastructure/Sql/AvesSql.cs`
- [ ] T046 [US2] Implement the write half of `src/BirdsCatalog.Infrastructure/Repositories/BirdRepository.cs`, translating the `uq_aves_nombre_cientifico` and foreign-key violations raised by Npgsql into `ConflictError` and `DanglingReferenceError` — the database is the guarantee (depends on T045, research.md R-007)
- [ ] T047 [US2] Implement `Create`, `Update` and `Delete` on `src/BirdsCatalog.Application/Services/BirdService.cs`: assign the id with `Guid.NewGuid()` — UUIDv4, **not** v7, whose embedded timestamp would make creation order recoverable (FR-013, research.md R-001) — pre-check family and ecosystem existence for the FR-014 message, and pre-check scientific-name uniqueness for the FR-017 message
- [ ] T048 [US2] Add `POST /api/aves`, `PUT /api/ave` and `DELETE /api/aves/{ave_id}` to `src/BirdsCatalog.Api/Controllers/AvesController.cs`, returning `201` with a `Location` header and an `Identificador` body, and `204` on delete (depends on T035)
- [ ] T049 [US2] Configure `InvalidModelStateResponseFactory` in `src/BirdsCatalog.Api/Program.cs` so binding and `required` failures emit the contract's `ValidationProblemDetails` with a populated `errors` object (FR-016, SC-006)
- [ ] T050 [US2] Run `dotnet test` and confirm T039–T043 under `tests/BirdsCatalog.IntegrationTests/Aves/` and `tests/BirdsCatalog.UnitTests/Services/` pass, and that the bird rows of quickstart.md Scenario 5 behave as tabulated

**Checkpoint**: US1 and US2 both work independently. Birds are curatable end to end.

---

## Phase 5: User Story 3 - Maintain the reference vocabularies (Priority: P3)

**Goal**: A maintainer manages families and ecosystems so birds are classified against a
consistent standard vocabulary. Delivers **6 operations**: `crearFamilia`,
`actualizarFamilia`, `eliminarFamilia`, `crearEcosistema`, `actualizarEcosistema`,
`eliminarEcosistema`.

**Independent Test**: Register a family, correct it, confirm the correction is returned,
then remove it while nothing references it. Separately, with a bird classified under it,
confirm the removal is **refused** and that the bird is still present and still classified.

### Tests for User Story 3 ⚠️

- [ ] T051 [P] [US3] Contract tests for the three `familias` write operations in `tests/BirdsCatalog.IntegrationTests/Vocabularios/WriteFamiliasContractTests.cs`, asserting 201/400/409, 200/400/404/409 and 204/400/404/409
- [ ] T052 [P] [US3] Contract tests for the three `ecosistemas` write operations in `tests/BirdsCatalog.IntegrationTests/Vocabularios/WriteEcosistemasContractTests.cs`
- [ ] T053 [P] [US3] Integration test in `tests/BirdsCatalog.IntegrationTests/Vocabularios/FamilyLifecycleTests.cs` — register with name and taxonomic order, confirm it becomes available for classifying birds, correct it, then remove it (FR-011)
- [ ] T054 [P] [US3] Integration test in `tests/BirdsCatalog.IntegrationTests/Vocabularios/EcosystemLifecycleTests.cs` — the same shape with name and geographic zone (FR-012)
- [ ] T055 [P] [US3] Integration test in `tests/BirdsCatalog.IntegrationTests/Vocabularios/DeleteInUseRefusedTests.cs` asserting that deleting a family or ecosystem with dependents returns **409**, the entry is still registered, and **no bird is deleted or left unclassified** — re-list the dependents afterwards and assert the count is unchanged, since a `CASCADE` regression would return `204` and quietly empty the catalog (FR-018, quickstart Scenario 5)
- [ ] T056 [P] [US3] Integration test in `tests/BirdsCatalog.IntegrationTests/Vocabularios/DeleteOnceUnreferencedTests.cs` asserting the same delete returns **204** once the last dependent bird is removed (FR-018, second scenario outline)
- [ ] T057 [P] [US3] Integration test in `tests/BirdsCatalog.IntegrationTests/Vocabularios/VocabularyNameUniquenessTests.cs` asserting duplicate family and ecosystem names return **409**, and that `Páramo` and `paramo` coexist — uniqueness is exact-match per the spec's stated assumption (FR-017, data-model.md)

### Implementation for User Story 3

- [ ] T058 [P] [US3] Create write DTOs `FamiliaNueva` and `FamiliaActualizada` in `src/BirdsCatalog.Api/Contracts/`, with the id carried in the body on update per the brief's `PUT /api/familias`
- [ ] T059 [P] [US3] Create write DTOs `EcosistemaNuevo` and `EcosistemaActualizado` in `src/BirdsCatalog.Api/Contracts/`
- [ ] T060 [P] [US3] Add write SQL plus a dependent-count query (`SELECT count(*) FROM aves WHERE familia_id = @id`) to `src/BirdsCatalog.Infrastructure/Sql/FamiliasSql.cs`
- [ ] T061 [P] [US3] Add the equivalent write and dependent-count SQL to `src/BirdsCatalog.Infrastructure/Sql/EcosistemasSql.cs`
- [ ] T062 [US3] Implement the write half of `src/BirdsCatalog.Infrastructure/Repositories/FamilyRepository.cs`, translating `uq_familias_nombre` violations into `ConflictError` and the `ON DELETE RESTRICT` violation into `InUseError` (depends on T060)
- [ ] T063 [US3] Implement the write half of `src/BirdsCatalog.Infrastructure/Repositories/EcosystemRepository.cs` on the same pattern (depends on T061)
- [ ] T064 [US3] Implement `Create`, `Update` and `Delete` on `src/BirdsCatalog.Application/Services/FamilyService.cs` and `EcosystemService.cs`: UUIDv4 ids, a name-uniqueness pre-check for the message, and a dependent-count check producing the FR-018 refusal with its reason stated
- [ ] T065 [US3] Add `POST`, `PUT` and `DELETE` actions to `src/BirdsCatalog.Api/Controllers/FamiliasController.cs` and `src/BirdsCatalog.Api/Controllers/EcosistemasController.cs` (depends on T036, T037)
- [ ] T066 [US3] Run `dotnet test` and confirm T051–T057 under `tests/BirdsCatalog.IntegrationTests/Vocabularios/` pass, with particular attention to `DeleteInUseRefusedTests.cs` proving the FR-018 refusal leaves dependents intact

**Checkpoint**: US1 through US3 all work independently. All 15 single-entity operations are live.

---

## Phase 6: User Story 4 - Compare birds across families and ecosystems (Priority: P4)

**Goal**: A researcher retrieves the birds of a given family or ecosystem for comparative
analysis on centralized data. Delivers the final **2 operations**: `listarAvesPorFamilia`
and `listarAvesPorEcosistema`.

**Independent Test**: With several birds under one family and others elsewhere, request
that family's birds and confirm exactly those are returned and no others — 100% precision
and recall. A vocabulary entry with no birds returns `200` with `[]`; an unknown id
returns `404`.

### Tests for User Story 4 ⚠️

- [ ] T067 [P] [US4] Contract tests for `listarAvesPorFamilia` and `listarAvesPorEcosistema` in `tests/BirdsCatalog.IntegrationTests/CrossEntity/CrossEntityContractTests.cs`, asserting 200/400/404 and that each element validates against the `Ave` schema with its nested classification
- [ ] T068 [P] [US4] Integration test in `tests/BirdsCatalog.IntegrationTests/CrossEntity/ExactSubsetTests.cs` seeding birds inside and outside the target classification, asserting exactly the expected subset is returned for both entities (FR-006, FR-007, SC-004)
- [ ] T069 [P] [US4] Integration test in `tests/BirdsCatalog.IntegrationTests/CrossEntity/EmptyAndUnknownClassificationTests.cs` asserting an existing entry with no birds returns `200` with `[]` while an id matching no entry returns `404` — the empty case must never be an error (FR-015)

### Implementation for User Story 4

- [ ] T070 [P] [US4] Add `SELECT ... WHERE familia_id = @id` and `WHERE ecosistema_id = @id` joined queries to `src/BirdsCatalog.Infrastructure/Sql/AvesSql.cs`, backed by the `ix_aves_familia_id` and `ix_aves_ecosistema_id` indexes from T008
- [ ] T071 [US4] Add `ListByFamily` and `ListByEcosystem` to `src/BirdsCatalog.Infrastructure/Repositories/BirdRepository.cs`, reusing the T031 multi-mapping projection (depends on T070)
- [ ] T072 [US4] Add `ListBirdsByFamily` and `ListBirdsByEcosystem` to `src/BirdsCatalog.Application/Services/FamilyService.cs` and `EcosystemService.cs`, raising `NotFoundError` when the classification itself is absent — distinct from an existing classification that simply has no birds
- [ ] T073 [P] [US4] Add `GET /api/familias/{familia_id}/aves` to `src/BirdsCatalog.Api/Controllers/FamiliasController.cs` and `GET /api/ecosistemas/{ecosistema_id}/aves` to `src/BirdsCatalog.Api/Controllers/EcosistemasController.cs`
- [ ] T074 [US4] Run `dotnet test` and confirm T067–T069 under `tests/BirdsCatalog.IntegrationTests/CrossEntity/` pass, then confirm all **17** operations in `specs/001-birds-catalog-api/contracts/openapi.yaml` are implemented and no endpoint exists in `src/BirdsCatalog.Api/Controllers/` that is absent from it

**Checkpoint**: All four user stories independently functional; the contract is fully covered.

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: The validation harness that proves the feature against the spec, plus the
mechanical guards the constitution requires.

- [ ] T075 [P] Create `scripts/seed-demo-data.ps1` registering the 3 families, 3 ecosystems and 5 birds of quickstart.md **through the HTTP API, never via SQL** — SC-003 requires it — capturing each server-assigned id from the `201` response for the birds that follow; only `Vultur gryphus` carries the optional indigenous fields, so both FR-016 branches are exercised; exit non-zero if any bird is missing on re-read (FR-020)
- [ ] T076 [P] Create `scripts/check-contract-drift.ps1` fetching `/swagger/v1/swagger.json` from the running container, diffing it against `specs/001-birds-catalog-api/contracts/openapi.yaml`, and asserting the emitted document declares `openapi: 3.1.x` — a difference is a **failure, not a prompt to regenerate** (research.md R-008)
- [ ] T077 [P] Create `tests/contract/schemathesis.Dockerfile` and `tests/contract/run-contract-tests.sh` running Schemathesis against the **frozen** contract with checks `status_code_conformance`, `response_schema_conformance`, `content_type_conformance` and `not_a_server_error`, and a fixed seed (research.md R-009)
- [ ] T078 Add a `contract-tests` Compose profile to `docker-compose.yml` wiring the T077 container, runnable as `docker compose --profile contract-tests run --rm schemathesis`
- [ ] T079 [P] Add a CI workflow at `.github/workflows/ci.yml` running `dotnet format --verify-no-changes`, `dotnet build`, `dotnet test`, Spectral on the frozen contract, the T076 drift check and the T077 Schemathesis run — red CI blocks merge (constitution: automate the checkable)
- [ ] T080 Run the full six-step clean-checkout sequence from quickstart.md, *Full validation from a clean checkout*, and confirm every step exits zero
- [ ] T081 Walk quickstart.md Scenarios 1 through 6 manually and confirm each stated expectation, performing the FR-018 refused-delete check by hand at least once
- [ ] T082 [P] Update `README.md` at repository root with what the service is, how to run it, and links to spec.md, plan.md and quickstart.md
- [ ] T083 Confirm the constitution's inward-dependency rule still holds by checking that `src/BirdsCatalog.Application/BirdsCatalog.Application.csproj` carries no `ProjectReference` and no Dapper, Npgsql or ASP.NET package reference

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: no dependencies — start immediately
- **Foundational (Phase 2)**: depends on Setup — **blocks every user story**
- **US1 (Phase 3)**: depends on Foundational only
- **US2 (Phase 4)**: depends on Foundational; extends `AvesController.cs` after T035
- **US3 (Phase 5)**: depends on Foundational; extends `FamiliasController.cs` and `EcosistemasController.cs` after T036 and T037
- **US4 (Phase 6)**: depends on Foundational; reuses the T031 projection and both vocabulary controllers
- **Polish (Phase 7)**: T075 needs US2 **and** US3, since it creates vocabularies then birds; T076–T081 need all four stories for a full green run

### User Story Dependencies

The four stories are independent in *behaviour* — each is separately testable by seeding
through the Phase 2 fixture rather than through another story's endpoints. They share
files, which constrains scheduling but not correctness:

- **US1 (P1)**: fully independent. The MVP.
- **US2 (P2)**: independent; its tests seed families and ecosystems via the fixture, not via US3
- **US3 (P3)**: independent; its FR-018 tests seed a bird via the fixture, not via US2
- **US4 (P4)**: independent; seeds birds and classifications via the fixture

Running them in parallel means coordinating on the three controller files and on
`AvesSql.cs`, which US1, US2 and US4 all extend.

### Within Each User Story

Tests are written first and must fail, then DTOs and SQL, then repositories, then services,
then controllers, then the phase's verification task.

### Parallel Opportunities

- Phase 1: T004–T007 in parallel after T003
- Phase 2: T011–T014 in parallel (four separate files, all pure domain); T019, T020 and T022 in parallel
- Phase 3: all five tests T023–T027 in parallel; then T028–T030; then T033/T034; then T036/T037
- Phase 4: all five tests T039–T043 in parallel
- Phase 5: all seven tests T051–T057 in parallel; then T058–T061; then T062/T063
- Phase 6: all three tests T067–T069 in parallel
- Phase 7: T075, T076, T077, T079 and T082 in parallel

---

## Parallel Example: User Story 1

```bash
# All five US1 tests together — five separate files, all expected to fail:
Task: "Contract tests for listarAves/obtenerAve in tests/BirdsCatalog.IntegrationTests/Aves/ReadAvesContractTests.cs"
Task: "Contract tests for the vocabulary reads in tests/BirdsCatalog.IntegrationTests/Vocabularios/ReadVocabulariosContractTests.cs"
Task: "Empty catalog returns 200 with [] in tests/BirdsCatalog.IntegrationTests/Aves/EmptyCatalogTests.cs"
Task: "Malformed 400 vs missing 404 in tests/BirdsCatalog.IntegrationTests/Identifiers/MalformedVsMissingIdTests.cs"
Task: "Raw-byte non-ASCII round trip in tests/BirdsCatalog.IntegrationTests/Encoding/NonAsciiRoundTripTests.cs"

# Then the three read DTO, mapping and SQL tasks together:
Task: "Read DTOs Ave/Familia/Ecosistema in src/BirdsCatalog.Api/Contracts/"
Task: "Explicit mappers in src/BirdsCatalog.Api/Mapping/"
Task: "Joined read SQL in src/BirdsCatalog.Infrastructure/Sql/AvesSql.cs"
```

---

## Implementation Strategy

### MVP First (User Story 1 only)

1. Phase 1 Setup
2. Phase 2 Foundational (blocks everything)
3. Phase 3 US1
4. **STOP and VALIDATE**: quickstart Scenarios 2, 4 and 6 against fixture-seeded data
5. This alone satisfies SC-001 and the read half of SC-006 — a demonstrable catalog

### Incremental Delivery

1. Setup + Foundational → containers up, `/health` green, `/swagger` serving
2. **+ US1** → the catalog is consultable (MVP, 6 operations)
3. **+ US2** → birds are curatable (9 operations)
4. **+ US3** → vocabularies are maintainable (15 operations); `seed-demo-data.ps1` becomes runnable
5. **+ US4** → comparative analysis (all 17); SC-004 becomes provable
6. **+ Polish** → drift check, Schemathesis and CI make the whole thing mechanically enforced

### Parallel Team Strategy

Phases 1 and 2 together, then one developer per story. US1 lands first regardless, since
US2 and US4 extend files it creates.

---

## Notes

- `[P]` means different files with no dependency on an incomplete task
- Every task names an exact path; no task requires reading another task to be actionable
- Three traps carried forward from research, each with a test that catches it: the `:guid`
  route constraint turning a `400` into a `404` (T026, R-004); a JSON encoder that escapes
  accents, so a deserialising test passes while SC-007 fails (T027, R-005); and a foreign key
  declared `CASCADE`, which turns the FR-018 refusal into a `204` and a quietly emptier
  catalog (T055, data-model.md)
- The frozen contract is the source of truth. If code and contract disagree, the code is
  wrong — unless the spec genuinely changed, in which case re-flow spec → plan → contract
- Commit after each task or logical group; one logical change moves spec, contract and code together
