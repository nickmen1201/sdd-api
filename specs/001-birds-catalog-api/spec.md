# Feature Specification: Birds of Colombia Catalog

**Branch**: `main` · **Created**: 2026-08-25 · **Status**: Draft
**Input**: `formulations_birds_api_20260820_.md`. Implementation mandates in that brief (ID type, layering, repository pattern, operation signatures) are excluded here — they bind the plan, not the spec.

Behaviour is specified as Gherkin. Each `@FR-xxx` tag is a requirement ID and appears
exactly once — the tagged scenario *is* that requirement. Where a rule differs per entity,
the tag sits on the `Examples:` block it applies to. `@P1..@P4` are delivery priorities —
each Feature is independently testable and shippable in tag order.

## Clarifications

### Session 2026-08-26

- Q: Which bird fields are mandatory? (FR-016) → A: Indigenous name and indigenous language are optional; scientific name, common name, family and ecosystem are all mandatory.
- Q: What uniqueness rules apply? (FR-017) → A: A scientific name must be unique across birds; a family name unique across families; an ecosystem name unique across ecosystems. Sharing a *classification* is not a duplicate — many birds may reference the same family and the same ecosystem.
- Q: Removing a family or ecosystem that birds still use? (FR-018) → A: Refuse the removal while dependents exist.

## Key Entities

| Entity | Fields | Relations |
|---|---|---|
| **Bird** (Ave) | scientific name **(unique)**, common name, indigenous name *(optional)*, indigenous language *(optional)* | belongs to exactly 1 Family, exactly 1 Ecosystem — both mandatory |
| **Family** (Familia Taxonómica) | name **(unique)** (*Cathartidae*), taxonomic order (*Cathartiformes*) | groups 0..n Birds |
| **Ecosystem** (Ecosistema) | name **(unique)** (*Páramo*), geographic zone (*Andina*) | hosts 0..n Birds |

All three carry a system-assigned identifier: unique, stable for the record's lifetime,
not inferable from creation order. Fields not marked *(optional)* are mandatory.

---

## Feature: Consult the catalog

```gherkin
@P1
Feature: Consult the catalog
  As an enthusiast, academic, or conservation professional
  I want a bird's full classification from one authoritative place
  So that I don't reassemble it from scattered, inconsistent sources

  Background:
    Given the catalog is running

  @FR-001
  Scenario: List every registered bird
    Given the catalog contains registered birds
    When a consumer requests the list of birds
    Then every registered bird is returned, each with the full payload of FR-003

  @FR-002
  Scenario: Retrieve one bird
    Given a bird is registered
    When a consumer requests that bird by its identifier
    Then the complete record is returned, with the full payload of FR-003

  Scenario Outline: Retrieve reference data
    Given a <entity> is registered
    When a consumer requests it by its identifier, and requests the full list
    Then the stored <fields> are returned in both cases

    @FR-004
    Examples: Taxonomic families
      | entity | fields                   |
      | family | name and taxonomic order |

    @FR-005
    Examples: Ecosystems
      | entity    | fields                   |
      | ecosystem | name and geographic zone |

  @FR-015
  Scenario: Unknown identifier
    Given an identifier matching no record
    When a consumer requests it
    Then the consumer is told the record does not exist
    And that outcome is distinguishable from a malformed or invalid request

  @FR-015
  Scenario: Malformed identifier
    Given a value that is not a valid identifier at all
    When a consumer requests it
    Then the consumer is told the request is invalid, not that the record is missing

  Scenario: Empty catalog
    Given no birds have been registered
    When a consumer requests the list of birds
    Then an empty result is returned, not an error
```

---

## Feature: Curate bird records

```gherkin
@P2
Feature: Curate bird records
  As a catalog maintainer
  I want to register, correct and remove birds
  So that the catalog stays current instead of being a frozen dataset

  @FR-008
  Scenario: Register a bird
    Given a family and an ecosystem already exist
    When a maintainer registers a bird referencing them
    Then the bird is stored with its own identifier
    And it is retrievable afterwards

  @FR-009
  Scenario: Correct a bird
    Given a registered bird
    When a maintainer changes one or more of its values
    Then subsequent consultations return the updated values
    And no other field is altered

  @FR-010
  Scenario: Remove a bird
    Given a registered bird
    When a maintainer removes it
    Then it no longer appears in the catalog
    And its family and its ecosystem remain untouched

  @FR-014
  Scenario: Reject a dangling classification
    Given a family or ecosystem identifier that matches no registered entry
    When a maintainer registers or corrects a bird referencing it
    Then the operation is rejected and the reason is stated

  @FR-015
  Scenario: Curate a record that does not exist
    Given an identifier matching no bird, family or ecosystem
    When a maintainer submits a correction or a removal for it
    Then the maintainer is told the record does not exist
```

---

## Feature: Maintain the reference vocabularies

```gherkin
@P3
Feature: Maintain the reference vocabularies
  As a catalog maintainer
  I want to manage families and ecosystems
  So that birds are classified consistently against a standard vocabulary

  Scenario Outline: Lifecycle of a vocabulary entry
    When a maintainer registers a <entity> with its <fields>
    Then it is stored with its own identifier
    And it becomes available for classifying birds
    When the maintainer corrects it
    Then subsequent consultations return the corrected values
    When the maintainer removes it
    Then it no longer appears in the list of that vocabulary

    @FR-011
    Examples: Taxonomic families
      | entity | fields                   |
      | family | name and taxonomic order |

    @FR-012
    Examples: Ecosystems
      | entity    | fields                   |
      | ecosystem | name and geographic zone |

  @FR-018
  Scenario Outline: Removing a vocabulary entry in use is refused
    Given at least one bird is classified under a <entity>
    When a maintainer removes that <entity>
    Then the removal is refused and the reason is stated
    And the <entity> is still registered
    And no bird is deleted or left unclassified

    Examples:
      | entity    |
      | family    |
      | ecosystem |

  Scenario Outline: Removal succeeds once nothing depends on it
    Given no bird is classified under a <entity>
    When a maintainer removes that <entity>
    Then it is removed

    Examples:
      | entity    |
      | family    |
      | ecosystem |
```

---

## Feature: Compare birds across families and ecosystems

```gherkin
@P4
Feature: Compare birds across families and ecosystems
  As a researcher
  I want the birds of a given family or ecosystem
  So that I can do comparative analysis on centralized data

  Scenario Outline: List the birds of a classification
    Given several birds are classified under the same <entity>
    And other birds are classified elsewhere
    When a consumer requests the birds of that <entity>
    Then exactly those birds are returned and no others

    @FR-006
    Examples: Taxonomic families
      | entity |
      | family |

    @FR-007
    Examples: Ecosystems
      | entity    |
      | ecosystem |

  Scenario Outline: Classification with no birds
    Given a <entity> exists with no birds associated
    When a consumer requests its birds
    Then an empty result is returned, not an error

    Examples:
      | entity    |
      | family    |
      | ecosystem |

  @FR-015
  Scenario Outline: Birds of a classification that does not exist
    Given an identifier matching no <entity>
    When a consumer requests its birds
    Then the consumer is told the <entity> does not exist

    Examples:
      | entity    |
      | family    |
      | ecosystem |
```

---

## Feature: Data integrity

```gherkin
Feature: Data integrity
  Rules that hold across every operation above.

  @FR-003
  Scenario: Bird payload
    When a bird is returned by any operation
    Then it carries its scientific name, common name, indigenous name,
        indigenous language, its family association and its ecosystem association

  @FR-013
  Scenario: Identifiers
    When any bird, family or ecosystem is created
    Then the system assigns a unique identifier
    And that identifier is stable for the lifetime of the record
    And it is not guessable or inferable from creation order

  @FR-019
  Scenario: Spanish and indigenous-language text
    When a record containing "Cóndor de los Andes", "Páramo" or "Kuntur" is submitted
    Then those values are stored and returned unaltered

  @FR-016
  Scenario Outline: Presence rules for bird fields
    Given a bird submitted without its <field>
    When a maintainer registers or corrects it
    Then the submission is <outcome>

    Examples: Mandatory
      | field           | outcome                                |
      | scientific name | rejected, and the reason is stated     |
      | common name     | rejected, and the reason is stated     |
      | family          | rejected, and the reason is stated     |
      | ecosystem       | rejected, and the reason is stated     |

    Examples: Optional
      | field               | outcome                              |
      | indigenous name     | accepted, and the field left empty   |
      | indigenous language | accepted, and the field left empty   |

  @FR-017
  Scenario Outline: Names are unique within their own kind
    Given a <entity> is registered with a given <field>
    When a maintainer registers or corrects another <entity> carrying the same <field>
    Then the operation is rejected and the reason is stated

    Examples:
      | entity    | field           |
      | bird      | scientific name |
      | family    | name            |
      | ecosystem | name            |

  Scenario: Sharing a classification is not a duplicate
    Given a family and an ecosystem are registered
    When several birds are registered referencing that same family and that same ecosystem
    Then every one of them is accepted
```

---

## Success Criteria

- **SC-001**: A consumer obtains a bird's complete classification — names, family with its order, ecosystem with its zone — from this catalog alone.
- **SC-002**: All 16 operations from the brief (5 birds, 6 families, 5 ecosystems, incl. the two cross-entity listings) are exercised successfully end to end.
- **SC-003** (`@FR-020`): A demonstration dataset of ≥5 birds — with ≥1 family and ≥1 ecosystem shared by more than one bird, so `@P4` is meaningfully exercised — is registered through the catalog itself, and every bird is afterwards retrievable individually and in the list.
- **SC-004**: The `@P4` listings return exactly the expected subset of that dataset — 100% precision and recall against the seeded data.
- **SC-005**: 100% of birds resolve to an existing family and an existing ecosystem.
- **SC-006**: Every failed request states what went wrong without the caller reading logs or source.
- **SC-007**: Accented and non-ASCII text round-trips byte-for-byte.

## Assumptions

- Scope is exactly the operations in the brief. **Out of scope**: search, filtering, sorting, pagination, bulk import, images, audio, coordinates, conservation status, and any UI — the brief mentions none.
- Consumers are programmatic; no user interface.
- One bird → one family, one ecosystem (the brief's model). Families and ecosystems are shared reference data.
- No volume, latency or availability targets are stated, so none are asserted; the criteria measure correctness instead. Production targets need to be set separately.
- **Access control is out of scope.** The brief names no users, roles or permissions; "maintainer" and "consumer" above are workflow roles, not enforced ones. Required as a separate feature before exposing writes publicly.

## Open Questions

1. Access control — restricted curation or fully open? (Out of scope above.)
2. Corrections — whole-record replacement or partial? The brief's operation inventory implies whole-record but does not say.
3. Uniqueness comparison — is `@FR-017` exact-match, or should "Páramo" collide with "paramo"/"PÁRAMO"? Assumed **exact match** above; changing it is a one-line edit to that scenario.
