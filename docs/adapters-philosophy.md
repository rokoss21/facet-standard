# Adapter Philosophy

## Purpose

This document defines the **architectural role and strict limitations** of provider adapters in the FACET ecosystem.

Adapters exist to translate **Canonical JSON** into provider-specific payloads. They are not execution engines, not sources of truth, and not places where logic is allowed to accumulate.

> **Adapters are translators, not decision-makers.**

This principle is foundational to FACET’s long-term correctness, reproducibility, and vendor independence.

---

## The Core Rule

**All semantic decisions MUST be completed before an adapter is invoked.**

Once Canonical JSON exists:

* No logic may be added
* No structure may be inferred
* No defaults may be applied
* No recovery heuristics may run

Adapters perform **mechanical transformation only**.

---

## Why Adapters Must Be Dumb

Modern LLM stacks routinely collapse because adapters grow "helpful" behavior:

* filling in missing fields
* renaming tools dynamically
* reordering messages to satisfy undocumented rules
* retrying failed calls with mutated payloads
* patching provider quirks ad hoc

This creates systems where:

* behavior differs per provider
* bugs cannot be reproduced
* audits become impossible
* fixes introduce new regressions

FACET treats this as an **architectural failure**, not an implementation detail.

---

## Canonical JSON as the Contract Boundary

Adapters consume Canonical JSON and emit provider payloads.

They MUST treat Canonical JSON as:

* immutable
* complete
* authoritative

If a provider rejects a payload derived from valid Canonical JSON, the adapter MUST fail loudly.

> **Adapters are not allowed to “make it work”.**

Failure is information. Mutation is corruption.

---

## What Adapters Are Allowed To Do

Adapters MAY:

* rename fields to match provider APIs
* transform message layouts (e.g. roles → blocks)
* map FACET interfaces to provider tool schemas
* attach provider-required metadata
* split or merge fields when explicitly specified

All transformations MUST be:

* deterministic
* stateless
* reversible in principle

---

## What Adapters Are Forbidden To Do

Adapters MUST NOT:

* infer missing values
* change execution order
* modify tool arguments
* drop or add messages
* reinterpret context priorities
* retry with modified payloads
* apply provider-specific heuristics silently

Any of the above breaks:

* determinism
* reproducibility
* trust

---

## Failure Containment Model

FACET intentionally localizes all provider-specific failures to the adapter layer.

If a provider:

* changes an API
* introduces undocumented constraints
* breaks streaming semantics

Then:

* Canonical JSON remains valid
* stored executions remain replayable
* only the adapter needs updating

This sharply bounds blast radius and prevents systemic corruption.

---

## Adapters vs Frameworks

Most agent frameworks embed logic *inside* provider integrations:

```
Agent Logic
  ↓
Provider Wrapper
  ↓
Model
```

FACET inverts this:

```
FACET Compiler
  ↓
Canonical JSON (IR)
  ↓
Adapter (View)
  ↓
Provider
```

This inversion is what makes determinism possible.

---

## Adapters Are Replaceable

Because adapters are:

* stateless
* mechanical
* non-authoritative

They can be:

* swapped
* versioned independently
* rewritten without touching agent logic

Vendor lock-in becomes structurally impossible.

---

## Design Principle

> **If fixing a bug requires changing adapter logic, the bug was upstream.**

Adapters reveal incompatibilities; they do not hide them.

This is the only sustainable way to build systems that survive provider churn.

---

## Status

This document defines the **normative adapter philosophy** for FACET-based systems.

Any implementation that embeds decision-making logic inside adapters is **non-compliant by design**.
