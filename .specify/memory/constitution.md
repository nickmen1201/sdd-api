<!-- constitution v1.0.0 · 2026-07-24 · template -->

# Engineering Constitution

Rules of *how* we build. The spec governs *what*. When code violates this file, the
code is wrong; when this file and the spec conflict, stop and ask the user.

**Keywords:** MUST/NEVER = blocks merge. DEFAULT/PREFER = the choice absent a documented
reason (override with one sentence in the spec or an ADR).

This file contains only what the agent would otherwise get wrong or choose differently —
not general good practice it already applies. Keep it that way.

`ACTIVE PROFILES: [X] SWE   [ ] Data Eng   [ ] DS/ML`

---

## Core (always)

- **Intent flows downhill:** spec → plan → contract → tasks → code. A behavior change
  enters at the spec and re-flows; NEVER patch it into code and leave the spec silent.
  Unspecified behavior → STOP and ask, NEVER invent.
- **Reproducible from a clean checkout** via documented commands. Lockfiles committed,
  tool versions pinned, seeds fixed where determinism is feasible.
- **Artifacts live in the repo.** One logical change = one commit moving spec, contract,
  and code together. Conventional Commits (DEFAULT).
- **Secrets** never in the repo — from env/secret manager; CI fails on a leaked credential.
- **Data hygiene defaults** (commonly violated): timestamps UTC + tz-aware + ISO 8601;
  surrogate IDs UUIDv7/ULID, NEVER sequential integers across a boundary; money in integer
  minor units, NEVER float; UTF-8 throughout.
- **Automate the checkable:** any rule a linter/type-checker/test can enforce MUST be
  enforced there. Red CI blocks merge.
- **New dependencies** justified in one line (why, and why not stdlib).

## Profile: SWE

- **Contract-first.** Interfaces get a lintable contract (OpenAPI/proto/SDL/JSON Schema)
  written and linted *before* code, and frozen. No endpoint exists in code that isn't in it.
- Every operation declares all outcomes, errors included. HTTP errors use Problem Details /
  RFC 7807 (DEFAULT). Contract changes classified additive vs breaking.
- Dependencies point inward: domain MUST NOT depend on infrastructure. Map to DTOs at the
  edge — NEVER leak persistence models onto the wire.
- Strict static typing on public interfaces. Contract tests validate the running code
  against the contract file.

## Profile: Data Eng

- Every dataset/interface has an explicit, versioned **data contract**; schema evolution
  additive by DEFAULT, breaking changes versioned.
- Transforms idempotent and replayable — re-running same inputs NEVER double-writes.
- **raw → cleaned → curated**; raw is immutable, NEVER mutated in place.
- Quality checks run *inside* the pipeline (counts, nulls, uniqueness, referential
  integrity, freshness). A failed gate stops promotion downstream.
- Lineage traceable end to end; PII classified and NEVER logged.

## Profile: DS/ML

- Version **data + code + config together**; a result you can't reproduce doesn't exist.
- Every training run tracked (params, data version, code version, metrics). Untracked = invalid.
- Evaluate on a held-out set vs a defined baseline. NEVER report train-set performance as
  generalization; guard train/serve skew.
- Notebooks explore; anything repeatable is promoted to a tested pipeline. NEVER ship a
  notebook to production. Monitor deployed models for drift and decay.

## Agent rules

- Obey this file and the spec; on genuine conflict, STOP and ask.
- Implement in task order, vertical slices; the first slice is human-reviewed before the
  rest follow it as reference.
- Show diffs for contract/schema changes and classify additive vs breaking before coding.

## Versioning

Bump the stamp on meaningful change. Keep **API version** (what changed for consumers,
semver) distinct from **artifact version** (what changed for the team) — same edits drive
both, but a typo fix bumps only the artifact.
