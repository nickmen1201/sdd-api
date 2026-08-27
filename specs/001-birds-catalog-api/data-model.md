# Phase 1 Data Model: Birds of Colombia Catalog

**Feature**: `001-birds-catalog-api` | **Date**: 2026-08-26 | **Plan**: [plan.md](./plan.md)

Three entities, two of which are shared reference vocabularies. Wire shapes are defined in
[contracts/openapi.yaml](./contracts/openapi.yaml) and are not repeated here; this document
covers the domain and the persistent schema.

Naming follows [research.md R-010](./research.md#r-010--language-boundary-spanish-on-the-wire-english-in-the-domain):
English in the C# domain, Spanish on the wire and in the database.

---

## Entities

### Bird — `aves`

| Domain field | Column | Type | Null | Rule |
|---|---|---|---|---|
| `Id` | `id` | `uuid` | no | PK, server-assigned UUIDv4 (FR-013, R-001) |
| `ScientificName` | `nombre_cientifico` | `text` | no | **Unique** across birds (FR-017); mandatory (FR-016) |
| `CommonName` | `nombre_comun` | `text` | no | Mandatory (FR-016) |
| `IndigenousName` | `nombre_indigena` | `text` | **yes** | Optional (FR-016) |
| `IndigenousLanguage` | `lengua_indigena` | `text` | **yes** | Optional (FR-016) |
| `FamilyId` | `familia_id` | `uuid` | no | FK → `familias.id`, `ON DELETE RESTRICT`; mandatory (FR-016); must resolve (FR-014) |
| `EcosystemId` | `ecosistema_id` | `uuid` | no | FK → `ecosistemas.id`, `ON DELETE RESTRICT`; mandatory (FR-016); must resolve (FR-014) |

Exactly one family and one ecosystem, both mandatory. Many birds may share the same family
and the same ecosystem — that is not a duplicate (spec: "Sharing a classification is not a
duplicate").

### Family — `familias`

| Domain field | Column | Type | Null | Rule |
|---|---|---|---|---|
| `Id` | `id` | `uuid` | no | PK, server-assigned UUIDv4 |
| `Name` | `nombre` | `text` | no | **Unique** across families (FR-017); e.g. `Cathartidae` |
| `Order` | `orden` | `text` | no | Taxonomic order, e.g. `Cathartiformes` |

Groups 0..n birds. Deletion refused while any bird references it (FR-018).

### Ecosystem — `ecosistemas`

| Domain field | Column | Type | Null | Rule |
|---|---|---|---|---|
| `Id` | `id` | `uuid` | no | PK, server-assigned UUIDv4 |
| `Name` | `nombre` | `text` | no | **Unique** across ecosystems (FR-017); e.g. `Páramo` |
| `GeographicZone` | `zona_geografica` | `text` | no | e.g. `Andina` |

Hosts 0..n birds. Deletion refused while any bird references it (FR-018).

---

## Relationships

```text
familias 1 ────< aves >──── 1 ecosistemas
         (RESTRICT)   (RESTRICT)
```

- Bird → Family: many-to-one, mandatory, `RESTRICT`.
- Bird → Ecosystem: many-to-one, mandatory, `RESTRICT`.
- There is no relationship between Family and Ecosystem. A family is not confined to an
  ecosystem, and vice versa; they are independent axes of classification.

`RESTRICT` rather than `CASCADE` is load-bearing, not stylistic: FR-018 requires that
removing a referenced vocabulary entry is *refused*, and that "no bird is deleted or left
unclassified". `CASCADE` would delete birds and `SET NULL` would leave them unclassified —
both are explicitly forbidden outcomes.

---

## Validation rules, and where each is enforced

Per [research.md R-007](./research.md#r-007--enforcing-uniqueness-and-referential-refusal-in-the-right-place),
rules are enforced at more than one layer, with different jobs: the schema layer produces
the *message*, the database produces the *guarantee*.

| Rule | Source | Schema / DTO | Service | Database | Response |
|---|---|---|---|---|---|
| Mandatory bird fields present | FR-016 | `required` in the contract; validated on binding | — | `NOT NULL` | 400 |
| Optional fields may be absent or null | FR-016 | nullable in the contract | — | nullable column | 201 / 200 |
| Scientific name unique | FR-017 | — | pre-check for message | `UNIQUE` | 409 |
| Family name unique | FR-017 | — | pre-check for message | `UNIQUE` | 409 |
| Ecosystem name unique | FR-017 | — | pre-check for message | `UNIQUE` | 409 |
| `familia_id` / `ecosistema_id` resolve | FR-014 | `format: uuid` | existence check | FK constraint | 422 |
| Vocabulary entry not deleted while in use | FR-018 | — | dependent-count check | `ON DELETE RESTRICT` | 409 |
| Identifier well-formed | FR-015 | `format: uuid` | — | — | 400 (see R-004) |
| Record exists | FR-015 | — | lookup | — | 404 |

Uniqueness is **exact-match**, per the spec's stated assumption (Open Question 3): `Páramo`
and `paramo` are different names and may coexist. Changing this means replacing the `UNIQUE`
constraint with a unique index on `lower(nombre)` — one line, in one place.

---

## State transitions

None. All three entities are mutable records with no lifecycle, status field, or workflow:
they are created, corrected in place, and removed. The only transition-like rule is that
removal of a family or ecosystem is *conditional* on having no dependents — a precondition,
not a state machine.

---

## Schema DDL

Authoritative copy lives at `db/init/001_schema.sql`; reproduced here so the data model
reads standalone.

```sql
-- UTF-8 database assumed (SC-007, FR-019). pgcrypto supplies gen_random_uuid()
-- as a defence-in-depth default; ids are normally assigned by the service (R-001).
CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE TABLE familias (
    id      uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    nombre  text NOT NULL,
    orden   text NOT NULL,
    CONSTRAINT uq_familias_nombre UNIQUE (nombre)
);

CREATE TABLE ecosistemas (
    id               uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    nombre           text NOT NULL,
    zona_geografica  text NOT NULL,
    CONSTRAINT uq_ecosistemas_nombre UNIQUE (nombre)
);

CREATE TABLE aves (
    id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    nombre_cientifico   text NOT NULL,
    nombre_comun        text NOT NULL,
    nombre_indigena     text NULL,
    lengua_indigena     text NULL,
    familia_id          uuid NOT NULL,
    ecosistema_id       uuid NOT NULL,
    CONSTRAINT uq_aves_nombre_cientifico UNIQUE (nombre_cientifico),
    CONSTRAINT fk_aves_familia    FOREIGN KEY (familia_id)
        REFERENCES familias (id)    ON DELETE RESTRICT,
    CONSTRAINT fk_aves_ecosistema FOREIGN KEY (ecosistema_id)
        REFERENCES ecosistemas (id) ON DELETE RESTRICT
);

-- FR-006 / FR-007 list birds by classification; without these, both are sequential scans.
CREATE INDEX ix_aves_familia_id    ON aves (familia_id);
CREATE INDEX ix_aves_ecosistema_id ON aves (ecosistema_id);
```

Notes on the choices above:

- **`text`, not `varchar(n)`.** PostgreSQL stores both identically and the spec states no
  length limits. An invented limit is a truncation bug waiting on a long Quechua name.
- **The two indexes are not premature optimisation.** They back the foreign keys, which
  PostgreSQL does *not* index automatically; without them every `RESTRICT` check on a
  vocabulary delete (FR-018) scans `aves` in full, as does every FR-006/FR-007 listing.
- **No `created_at` / `updated_at`.** No entity in the spec carries a timestamp; see
  research.md, "Deliberately not researched".

---

## Domain model shape

`Application/Domain` holds plain C# types with no persistence attributes, no Dapper
references and no ASP.NET references — the constitution's inward-dependency rule, enforced
by the project graph.

- `Bird` carries `FamilyId` and `EcosystemId` scalars. It does **not** hold `Family` and
  `Ecosystem` object references, so there is no lazy-loading question and no partially
  populated aggregate.
- Read operations that must return the nested representation (R-002) project into a separate
  `BirdWithClassification` read model, populated by a single joined query with Dapper
  multi-mapping. Reads and writes therefore use different shapes on purpose: a write names a
  reference, a read resolves it.
- The wire DTOs in `Api/Contracts` are distinct types again, in Spanish. Nothing in
  `Application` or `Infrastructure` knows they exist.
