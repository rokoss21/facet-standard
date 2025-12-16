# Compliance Levels

## Purpose

This document defines **compliance levels** for FACET-related implementations.

While the FACET v2.0 specification defines what is *correct*, compliance levels define **how completely** a given component (compiler, adapter, runtime, SDK integration) adheres to the FACET contract model.

This allows the ecosystem to:

* distinguish partial integrations from full implementations
* avoid false claims of determinism
* set clear expectations for enterprise use
* evolve the standard without breaking attribution or trust

Compliance levels are **declarative** and **auditable**.

---

## Core Principle

**Not all FACET integrations are equal — and that must be explicit.**

A component MUST declare its compliance level.

Silently claiming "FACET-compatible" without meeting the requirements of a level is considered **non-compliant**.

---

## Compliance Levels Overview

FACET defines four compliance levels:

| Level | Name          | Scope                              |
| ----: | ------------- | ---------------------------------- |
|    L0 | Conceptual    | Documentation / ideas only         |
|    L1 | Structural    | Canonical JSON & schema adherence  |
|    L2 | Deterministic | Full determinism & reproducibility |
|    L3 | Reference     | Spec-complete, reference-grade     |

---

## Level 0 — Conceptual Compliance (L0)

**Audience:** blog posts, design docs, experimental prototypes

### Definition

The implementation:

* references FACET concepts (contracts, determinism, Canonical JSON)
* does NOT implement formal compilation or guarantees

### Allowed Claims

* "FACET-inspired"
* "FACET concepts applied"
* "Contract-based approach"

### Forbidden Claims

* deterministic execution
* reproducibility guarantees
* FACET-compatible

### Notes

L0 is **not an implementation level**.
It exists to allow discussion without misleading users.

---

## Level 1 — Structural Compliance (L1)

**Audience:** SDK extensions, tooling, lightweight integrations

### Definition

The implementation:

* produces or consumes **Canonical JSON**
* follows canonical ordering and explicit null rules
* enforces schema shape stability

### Required Properties

* stable key ordering
* explicit `null` for missing optional fields
* deterministic serialization

### Non-Requirements

* full R-DAG execution
* Token Box Model
* strict determinism across runs

### Allowed Claims

* "FACET-compatible (structural)"
* "Canonical JSON compliant"

### Common Examples

* logging / auditing tools
* snapshot testing harnesses
* visualization layers

---

## Level 2 — Deterministic Compliance (L2)

**Audience:** production agent systems, enterprise deployments

### Definition

The implementation:

* fully enforces **deterministic execution**
* produces identical Canonical JSON for identical inputs
* rejects invalid states *before* provider execution

### Required Properties

* strict Facet Type System (FTS)
* deterministic R-DAG execution
* deterministic Token Box Model layout
* canonical JSON as the single source of truth
* no retries as a correctness mechanism

### Guarantees

* reproducible outputs
* stable hashing
* replayable executions
* deterministic failure modes

### Allowed Claims

* "Deterministic"
* "FACET-compliant"
* "Reproducible agent execution"

---

## Level 3 — Reference Compliance (L3)

**Audience:** standards bodies, auditors, long-term infrastructure

### Definition

The implementation:

* satisfies **all** FACET v2.0 normative requirements
* passes the official FACET golden test suite
* is suitable as a **reference implementation**

### Required Properties

* full spec coverage (all execution phases)
* golden tests with published fixtures
* strict adapter requirements
* hermetic execution guarantees
* documented versioning and change history

### Privileges

Only L3 implementations may claim:

* "FACET Reference Implementation"
* "Spec-complete"
* "FACET Standard"

---

## Adapters and Compliance

Provider adapters have **their own compliance axis**.

An adapter may be:

* L1 compliant (structural mapping only)
* L2 compliant (deterministic mapping + golden tests)

Adapters can **never** be L3 on their own.
They inherit system-level compliance.

---

## Misrepresentation Clause

Claiming a higher compliance level than implemented is a **spec violation**.

Non-compliant claims:

* "Deterministic" without reproducibility
* "FACET-compatible" without Canonical JSON
* "Standard" without spec coverage

Such claims invalidate trust and interoperability.

---

## Rationale

Compliance levels exist to prevent:

* marketing-driven overclaims
* partial integrations masquerading as standards
* ecosystem fragmentation

A deterministic contract layer only works if trust is explicit.

---

## Summary

FACET compliance is not binary.

It is **tiered, explicit, and enforceable**.

If a system does not declare its compliance level, it has none.

---

## Status

This document defines **normative compliance levels** for the FACET ecosystem.
