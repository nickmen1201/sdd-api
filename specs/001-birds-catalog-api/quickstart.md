# Quickstart & Validation Guide: Birds of Colombia Catalog

**Feature**: `001-birds-catalog-api` | **Date**: 2026-08-26 | **Plan**: [plan.md](./plan.md)

How to run the catalog and prove it satisfies the spec. Every scenario below maps to a
success criterion (SC-00x) or a requirement (FR-0xx) and states its expected outcome.

Wire shapes are in [contracts/openapi.yaml](./contracts/openapi.yaml); the schema is in
[data-model.md](./data-model.md). Neither is duplicated here.

---

## Prerequisites

| Tool | Version | Needed for |
|---|---|---|
| Docker Desktop | with Compose v2 | Running the API and PostgreSQL |
| .NET SDK | 10.x (pinned in `global.json`) | Building and running tests locally |
| PowerShell | 7+ | The seed script |

Everything except the SDK is containerised. No local PostgreSQL install is required.

---

## Run it

```powershell
docker compose up --build -d
docker compose ps          # api and postgres should both be healthy
```

The `api` service waits for PostgreSQL's `pg_isready` healthcheck before starting, so
first-run startup is deterministic rather than a race (research.md R-012).

| Endpoint | URL |
|---|---|
| API base | <http://localhost:8080> |
| Swagger UI | <http://localhost:8080/swagger> |
| Generated OpenAPI document | <http://localhost:8080/swagger/v1/swagger.json> |
| Health | <http://localhost:8080/health> |

**Readiness check** — poll this rather than sleeping a fixed interval:

```powershell
curl.exe -fsS http://localhost:8080/health
```

**Schema changes** require recreating the volume, because PostgreSQL only runs
`docker-entrypoint-initdb.d` on an empty data directory (research.md R-006):

```powershell
docker compose down -v; docker compose up --build -d
```

**Tear down**: `docker compose down -v`

---

## Scenario 1 — Seed the demonstration dataset (SC-003)

SC-003 requires the dataset to be registered **through the catalog itself**, not inserted
by SQL. The seed script therefore drives the public HTTP API, which is also what makes it a
test: if any endpoint is broken, seeding fails.

```powershell
./scripts/seed-demo-data.ps1 -BaseUrl http://localhost:8080
```

The script creates 3 families, 3 ecosystems and 5 birds, in that order (birds last, since
FR-014 requires their references to already exist). It captures each server-assigned id
from the `201` response and reuses it for the birds that follow.

The dataset is chosen so that the P4 cross-entity listings are meaningfully exercised —
some families and ecosystems are deliberately shared:

| Bird (`nombre_cientifico`) | Common name | Family | Ecosystem |
|---|---|---|---|
| `Vultur gryphus` | Cóndor de los Andes | Cathartidae | Páramo |
| `Coragyps atratus` | Gallinazo negro | **Cathartidae** | Bosque andino |
| `Ensifera ensifera` | Colibrí picoespada | Trochilidae | **Páramo** |
| `Oxypogon guerinii` | Barbudito paramuno | **Trochilidae** | **Páramo** |
| `Ramphastos ambiguus` | Tucán de pico negro | Ramphastidae | Bosque húmedo tropical |

Cathartidae is shared by 2 birds, Trochilidae by 2, and Páramo by 3. Only
`Vultur gryphus` carries the optional indigenous fields (`Kuntur` / `Quechua`); the rest
leave them null, which exercises both branches of FR-016.

**Expected**: 11 × `201 Created`, each with a `Location` header. The script then re-reads
every bird individually and in the list, and exits non-zero if any is missing.

**Proves**: SC-003, and FR-008/FR-011/FR-012 end to end.

---

## Scenario 2 — A single request returns the complete classification (SC-001)

```powershell
curl.exe -s http://localhost:8080/api/aves | ConvertFrom-Json | Select-Object -First 1
```

**Expected**: each bird carries `familia` **with its `orden`** and `ecosistema` **with its
`zona_geografica`** embedded as full objects, not bare ids. One request, no follow-ups.

**Proves**: SC-001, FR-001, FR-002, FR-003.

---

## Scenario 3 — Cross-entity listings return exactly the right subset (SC-004)

Using the ids the seed script printed:

```powershell
curl.exe -s http://localhost:8080/api/familias/$cathartidaeId/aves     # expect 2
curl.exe -s http://localhost:8080/api/ecosistemas/$paramoId/aves       # expect 3
```

**Expected**: exactly `Vultur gryphus` + `Coragyps atratus` for Cathartidae, and exactly
`Vultur gryphus` + `Ensifera ensifera` + `Oxypogon guerinii` for Páramo. No others — 100%
precision and recall against the seeded data.

An entry with no birds returns `200` with `[]`, never an error.

**Proves**: SC-004, FR-006, FR-007.

---

## Scenario 4 — Malformed identifier is not confused with a missing one (FR-015)

This is the sharpest correctness check in the feature, and the one most likely to regress
if the `:guid` route constraint is ever reintroduced (research.md R-004).

```powershell
curl.exe -s -o nul -w "%{http_code}`n" http://localhost:8080/api/ave/not-a-uuid
curl.exe -s -o nul -w "%{http_code}`n" http://localhost:8080/api/ave/3f1c9a2e-5b7d-4e88-9c31-6a0f4d2b8e71
```

**Expected**: `400` then `404` — never `404` twice. Both bodies are
`application/problem+json` and say which of the two happened.

**Proves**: FR-015, SC-006.

---

## Scenario 5 — Integrity rules are refusals, not surprises

| Attempt | Expected | Rule |
|---|---|---|
| `POST /api/aves` with an existing `nombre_cientifico` | `409`, duplicate named in `detail` | FR-017 |
| `POST /api/aves` with a `familia_id` that resolves to nothing | `422`, dangling reference named | FR-014 |
| `POST /api/aves` omitting `nombre_comun` | `400`, field named in `errors` | FR-016 |
| `POST /api/aves` omitting `nombre_indigena` | `201`, field returned as `null` | FR-016 |
| `DELETE /api/familias/{cathartidaeId}` while 2 birds reference it | `409`; family still present; **both birds still present and still classified** | FR-018 |
| Delete those 2 birds, then `DELETE` the family | `204` | FR-018 |
| `DELETE /api/aves/{id}` for a bird | `204`; its family and ecosystem untouched | FR-010 |
| Register several birds against the same family and ecosystem | all `201` | spec: sharing is not a duplicate |

The FR-018 case is worth performing manually at least once: after the refused delete,
re-list the family's birds and confirm the count is unchanged. The failure mode this guards
against — `ON DELETE CASCADE` silently removing birds — produces a `204` and a quietly
emptier catalog.

**Proves**: FR-014, FR-016, FR-017, FR-018, FR-010, SC-005, SC-006.

---

## Scenario 6 — Non-ASCII round-trips byte-for-byte (SC-007)

```powershell
curl.exe -s http://localhost:8080/api/aves --output response.json
Select-String -Path response.json -Pattern 'Cóndor de los Andes','Páramo','Kuntur' -Encoding utf8
```

**Expected**: the literal accented characters appear in the raw bytes. If the response
contains `Páramo` instead, the JSON encoder is misconfigured — see research.md R-005.
That escaped form still decodes correctly, so a test that deserialises before asserting
will pass while SC-007 fails; assert on the **raw bytes**.

**Proves**: SC-007, FR-019.

---

## Automated suites

### Unit and integration tests

```powershell
dotnet test
```

Integration tests start their own PostgreSQL via Testcontainers and apply
`db/init/001_schema.sql`, so they need Docker running but not `docker compose up`. They
are independent of the seeded demo data.

### Contract drift check — the code must not diverge from the frozen contract

```powershell
./scripts/check-contract-drift.ps1
```

Fetches `/swagger/v1/swagger.json` from the running container and diffs it against
`specs/001-birds-catalog-api/contracts/openapi.yaml`.

**Expected**: no difference. A difference is a **failure, not a prompt to regenerate** —
the frozen contract is the source of truth, and the fix is to change the code, or to
re-flow the change from the spec if the behaviour genuinely should change
(research.md R-008).

The script also asserts the emitted document declares `openapi: 3.1.x`. If that assertion
fails, apply the documented fallback in R-008: switch emission to
`Microsoft.AspNetCore.OpenApi` and keep Swashbuckle for Swagger UI only.

### Contract conformance — Schemathesis

```powershell
docker compose --profile contract-tests run --rm schemathesis
```

Runs Schemathesis against the **frozen contract**, not the generated document — testing
generated output against the code that generated it would prove nothing. Checks enabled:
`status_code_conformance`, `response_schema_conformance`, `content_type_conformance`,
`not_a_server_error`, with a fixed seed for reproducibility.

**Expected**: all checks pass. Two failure modes worth recognising:

- *Undocumented status code* — the code returns something the contract does not declare.
  Add it to the contract only if the spec justifies it; otherwise the code is wrong.
- *`404` where `400` was expected on a generated non-UUID path value* — the `:guid` route
  constraint has crept back in (research.md R-004).

---

## Full validation from a clean checkout

The sequence a reviewer runs to verify the whole feature:

```powershell
docker compose down -v
docker compose up --build -d
curl.exe -fsS http://localhost:8080/health
./scripts/seed-demo-data.ps1 -BaseUrl http://localhost:8080
dotnet test
./scripts/check-contract-drift.ps1
docker compose --profile contract-tests run --rm schemathesis
```

All six steps exit zero on a healthy build. Together they cover SC-001 through SC-007 and
every FR in the spec.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| API exits immediately on first `up` | PostgreSQL not yet accepting connections | Confirm `depends_on: condition: service_healthy` is present (R-012) |
| Schema changes have no effect | `initdb` only runs on an empty volume | `docker compose down -v` then up again (R-006) |
| Accents appear as `\u00xx` | JSON encoder left at the default | Configure `JavaScriptEncoder.Create(UnicodeRanges.All)` (R-005) |
| Malformed id returns `404` | `{id:guid}` route constraint in use | Remove the constraint; let model binding produce the `400` (R-004) |
| Deleting a family removes birds | Foreign key declared `CASCADE` | Must be `ON DELETE RESTRICT` (data-model.md) |
| Duplicate names slip through under load | Uniqueness checked only in the service layer | The `UNIQUE` constraint is the guarantee; the service check is only for the message (R-007) |
