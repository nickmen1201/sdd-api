# Feature Specification: Birds of Colombia Catalog

**Feature Branch**: `main` (no feature branch created — no `before_specify` git hook configured)

**Created**: 2026-08-25

**Status**: Draft

**Input**: Source brief: `formulations_birds_api_20260820_.md` ("Birds of Colombia"). Only that document was used as input; technical/implementation mandates in it were deliberately excluded from this specification.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Consult bird information from one authoritative place (Priority: P1)

An enthusiast, academic, or conservation professional needs information about a Colombian
bird. Instead of piecing it together from scattered sources, they consult the catalog: they
can see the full list of registered birds, and for any single bird they get its scientific
name, common name, indigenous name and the indigenous language it comes from, plus the
taxonomic family and the ecosystem it belongs to.

**Why this priority**: This is the reason the catalog exists — the problem stated in the
brief is that bird information is scattered and inconsistent. Consultation is the value
delivered to every audience named in the brief; everything else exists to make this possible.

**Independent Test**: With a pre-loaded set of bird records, a consumer can list all birds
and retrieve a single bird by its identifier, receiving consistent, complete classification
data in both cases. Delivers value without any curation capability being available.

**Acceptance Scenarios**:

1. **Given** the catalog contains registered birds, **When** a consumer requests the full list, **Then** every registered bird is returned with its descriptive names and its family and ecosystem associations.
2. **Given** a bird is registered, **When** a consumer requests that bird by its identifier, **Then** the complete record for that bird is returned, including which taxonomic family and which ecosystem it is associated with.
3. **Given** an identifier that matches no bird, **When** a consumer requests it, **Then** the consumer is told the bird does not exist, distinguishably from a request that failed for other reasons.
4. **Given** the catalog contains no birds yet, **When** a consumer requests the full list, **Then** an empty result is returned rather than an error.

---

### User Story 2 - Curate the bird records (Priority: P2)

A catalog maintainer keeps the bird information current: they register a newly documented
bird together with its family and ecosystem, correct an existing record when a name or an
association turns out to be wrong, and remove a record that should no longer be published.

**Why this priority**: Without curation the catalog is a fixed dataset that cannot be
corrected or grown; but consultation (P1) already delivers value on a seeded dataset, so
curation comes second.

**Independent Test**: A maintainer can register a bird, see it appear in the catalog,
modify one of its fields, and remove it — each step observable through consultation.

**Acceptance Scenarios**:

1. **Given** a taxonomic family and an ecosystem already exist, **When** a maintainer registers a bird referencing them, **Then** the bird is stored, receives its own identifier, and is retrievable afterwards.
2. **Given** a registered bird, **When** a maintainer changes one or more of its values, **Then** subsequent consultations return the updated values and no other field is altered.
3. **Given** a registered bird, **When** a maintainer removes it, **Then** it no longer appears in the catalog, while its family and ecosystem remain untouched.
4. **Given** a bird submitted with a family or ecosystem that does not exist, **When** the maintainer attempts to register it, **Then** the record is rejected and the reason is stated.

---

### User Story 3 - Maintain the taxonomic families and ecosystems (Priority: P3)

A maintainer manages the two reference vocabularies the birds are classified against: the
taxonomic families (name and taxonomic order) and the ecosystems (name and geographic
zone). They can consult the full list of each, consult one by identifier, and add, correct,
or remove entries.

**Why this priority**: The reference vocabularies are what make the classification
consistent across birds — the standardization the brief asks for. They are needed before
birds can be classified, but they carry no standalone value for the end consumer.

**Independent Test**: A maintainer can create, consult, list, modify, and remove a
taxonomic family and an ecosystem without any bird record existing.

**Acceptance Scenarios**:

1. **Given** the catalog is running, **When** a maintainer registers a taxonomic family with its name and order, **Then** it is stored with its own identifier and becomes available for classifying birds.
2. **Given** the catalog is running, **When** a maintainer registers an ecosystem with its name and geographic zone, **Then** it is stored with its own identifier and becomes available for classifying birds.
3. **Given** a registered family or ecosystem, **When** a maintainer consults it by identifier or lists all of them, **Then** the stored values are returned.
4. **Given** a registered family or ecosystem, **When** a maintainer corrects or removes it, **Then** the change is reflected in subsequent consultations.

---

### User Story 4 - Compare birds across families and ecosystems (Priority: P4)

A researcher wants to see which birds share a taxonomic family, or which birds inhabit a
given ecosystem, in order to do the comparative analysis the brief describes — for example,
listing every bird recorded in the páramo, or every member of the Cathartidae family.

**Why this priority**: This is the comparative-analysis payoff of having the data
centralized, but it is only meaningful once birds and both reference vocabularies are in
place.

**Independent Test**: With several birds sharing a family and several sharing an ecosystem,
requesting the birds of a given family returns exactly that subset, and the same for a given
ecosystem.

**Acceptance Scenarios**:

1. **Given** several birds are classified under the same taxonomic family, **When** a consumer requests the birds of that family, **Then** exactly those birds are returned and no others.
2. **Given** several birds are recorded in the same ecosystem, **When** a consumer requests the birds of that ecosystem, **Then** exactly those birds are returned and no others.
3. **Given** a family or ecosystem exists but has no birds associated with it, **When** a consumer requests its birds, **Then** an empty result is returned rather than an error.
4. **Given** an identifier that matches no family or ecosystem, **When** a consumer requests its birds, **Then** the consumer is told it does not exist.

---

### Edge Cases

- A consumer asks for a bird, family, or ecosystem by an identifier that does not exist, or by a value that is not a valid identifier at all — the two situations must remain distinguishable to the caller.
- A bird is submitted referencing a taxonomic family or an ecosystem that has not been registered.
- A maintainer removes a taxonomic family or an ecosystem that birds are still classified under — see FR-018.
- A bird has no documented indigenous name or indigenous language (many species do not), or its ecosystem is not known at the time of registration — see FR-016.
- The same bird is submitted twice, or two birds are submitted with the same scientific name — see FR-017.
- A correction is submitted for a bird, family, or ecosystem that does not exist (or no longer does).
- The catalog is queried before any data has been loaded.
- Names carry Spanish and indigenous-language characters (e.g. "Cóndor de los Andes", "Páramo", "Kuntur"); they must be stored and returned unaltered.

## Requirements *(mandatory)*

### Functional Requirements

**Catalog consultation**

- **FR-001**: The system MUST let a consumer retrieve the complete list of registered birds.
- **FR-002**: The system MUST let a consumer retrieve a single bird by its identifier.
- **FR-003**: A bird returned by the system MUST carry its scientific name, common name, indigenous name, indigenous language, its taxonomic family association, and its ecosystem association.
- **FR-004**: The system MUST let a consumer retrieve the complete list of taxonomic families, and a single taxonomic family by its identifier, with its name and taxonomic order.
- **FR-005**: The system MUST let a consumer retrieve the complete list of ecosystems, and a single ecosystem by its identifier, with its name and geographic zone.
- **FR-006**: The system MUST let a consumer retrieve all birds classified under a given taxonomic family.
- **FR-007**: The system MUST let a consumer retrieve all birds recorded in a given ecosystem.

**Catalog curation**

- **FR-008**: The system MUST let a maintainer register a new bird with its descriptive names, its taxonomic family, and its ecosystem.
- **FR-009**: The system MUST let a maintainer correct an existing bird record.
- **FR-010**: The system MUST let a maintainer remove a bird from the catalog.
- **FR-011**: The system MUST let a maintainer register, correct, and remove taxonomic families.
- **FR-012**: The system MUST let a maintainer register, correct, and remove ecosystems.

**Integrity and consistency**

- **FR-013**: Every bird, taxonomic family, and ecosystem MUST be identified by a unique identifier that is assigned by the system, stable for the lifetime of the record, and not guessable or inferable from the order in which records were created.
- **FR-014**: The system MUST reject a bird whose taxonomic family or ecosystem does not correspond to a registered entry, and state why.
- **FR-015**: The system MUST distinguish, in its response, between a request for a record that does not exist and a request that is malformed or invalid.
- **FR-016**: The system MUST enforce the following presence rules when registering or correcting a bird: [NEEDS CLARIFICATION: the brief lists the fields but not which are mandatory. Are indigenous name and indigenous language optional (many species have none, and the brief's own condor example carries no ecosystem)? Is the ecosystem association required at registration time, or may a bird be recorded before its ecosystem is known?]
- **FR-017**: The system MUST apply the following uniqueness rules: [NEEDS CLARIFICATION: the brief states no uniqueness constraints. Must a scientific name be unique across birds — and must a family name and an ecosystem name be unique — or are duplicates permitted?]
- **FR-018**: When a maintainer removes a taxonomic family or an ecosystem that birds are still classified under, the system MUST [NEEDS CLARIFICATION: the brief does not say. Refuse the removal while dependent birds exist, remove the dependent birds along with it, or keep the birds and leave that classification unassigned?]
- **FR-019**: The system MUST preserve Spanish and indigenous-language text exactly as submitted, in storage and in every response.

**Demonstration dataset**

- **FR-020**: The catalog MUST be demonstrable with at least 5 birds registered together with their taxonomic families and ecosystems, chosen so that at least one family and at least one ecosystem are shared by more than one bird, allowing FR-006 and FR-007 to be exercised meaningfully.

### Key Entities

- **Bird (Ave)**: A bird species recorded in the catalog. Carries its scientific name, its common name, its indigenous name and the indigenous language that name belongs to. Each bird is classified under exactly one taxonomic family and associated with exactly one ecosystem.
- **Taxonomic Family (Familia Taxonómica)**: A taxonomic grouping used to classify birds consistently. Carries a name (e.g. *Cathartidae*) and the taxonomic order it belongs to (e.g. *Cathartiformes*). One family may group many birds.
- **Ecosystem (Ecosistema)**: A habitat in which birds are recorded. Carries a name (e.g. *Páramo*) and the geographic zone it belongs to (e.g. *Andina*). One ecosystem may host many birds.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A consumer can obtain a bird's complete classification — descriptive names, taxonomic family with its order, and ecosystem with its geographic zone — from this catalog alone, with no other source consulted.
- **SC-002**: All 16 catalog operations described in the brief (5 for birds, 6 for families, 5 for ecosystems, including the two cross-entity listings) are available and exercised successfully in an end-to-end demonstration.
- **SC-003**: A demonstration dataset of at least 5 birds, sharing at least one taxonomic family and at least one ecosystem between them, is registered end to end through the catalog itself, and every registered bird is afterwards retrievable both individually and in the full list.
- **SC-004**: Listing the birds of a given family, and of a given ecosystem, returns exactly the expected subset of the demonstration dataset — 100% precision and 100% recall, verified against the seeded data.
- **SC-005**: 100% of birds in the catalog resolve to an existing taxonomic family and an existing ecosystem — no record points at a classification that is not registered.
- **SC-006**: Every failed request (unknown record, invalid input, integrity violation) returns a response from which the caller can determine what went wrong without inspecting logs or source code.
- **SC-007**: Text in Spanish and in indigenous languages is returned byte-for-byte identical to what was submitted, verified on names containing accented and non-ASCII characters.

## Assumptions

- Scope is bounded to exactly the operations enumerated in the source brief. Free-text search, filtering by arbitrary field, sorting, pagination, bulk import, images, audio, geographic coordinates, and conservation status are **not** part of this feature — the brief mentions none of them.
- The catalog is read by external consumers programmatically (the brief's stated purpose is programmatic access for enthusiasts, academics, and conservation professionals). No user interface is in scope.
- A bird belongs to exactly one taxonomic family and one ecosystem, as the brief's data model expresses each as a single association. Multi-ecosystem or multi-family birds are out of scope.
- Taxonomic families and ecosystems are shared reference data: several birds may point at the same family and the same ecosystem, and the brief explicitly asks for the demonstration data to be chosen so this is the case.
- The brief specifies no users, roles, authentication, or authorization, and no distinction between who may read the catalog and who may modify it. This specification therefore describes the *maintainer* and *consumer* as roles in the workflow, not as enforced permissions; **access control is treated as out of scope for this feature** and must be added as a separate feature if the catalog is exposed publicly with write operations. Flagged here rather than assumed away — see Open Questions.
- No volume, traffic, latency, or availability targets are stated in the brief, so none are asserted here; the success criteria measure correctness and completeness instead. If this catalog is to serve production traffic, performance targets need to be established separately.
- The source brief also mandates specific implementation choices (identifier data type, architectural layering, the repository pattern, and the exact operation signatures). Those are deliberately excluded from this specification, which covers *what* and *why*; they belong to the planning phase and remain binding there.

## Open Questions

Recorded here rather than resolved by guesswork. The first three are also marked inline in
the requirements above.

1. **Mandatory vs optional bird fields** (FR-016) — which of the descriptive fields must be present, and is the ecosystem association required at registration?
2. **Uniqueness rules** (FR-017) — may two birds share a scientific name; may two families or two ecosystems share a name?
3. **Removing reference data still in use** (FR-018) — refuse, cascade, or leave the birds unclassified?
4. **Access control** — the brief names no roles or permissions; is the catalog read-only to the public with restricted curation, or fully open? (See Assumptions; currently out of scope.)
5. **Correcting a record** — must a correction supply the complete record (replacing every field) or may it supply only the fields that change? The brief's operation inventory implies whole-record updates but does not state it.
