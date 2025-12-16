# Adapter Requirements

## Purpose

This document defines **normative requirements** for FACET-compatible provider adapters.

Adapters are the only layer allowed to translate **Canonical JSON** into provider-specific payloads (OpenAI, Anthropic, Gemini, local runtimes, etc.).

They exist to **map**, not to interpret, fix, enrich, or re‑decide behavior.

> Adapters are translators, not collaborators.

---

## Core Principle

**Adapters MUST be behaviorally passive.**

They MUST NOT:

* introduce new logic
* infer missing data
* reorder execution semantics
* apply provider-specific heuristics
* silently recover from invalid states

All intelligence, validation, and determinism belong **above** the adapter boundary.

---

## Architectural Position

FACET enforces a strict layered architecture:

```
.facet document
      ↓
Typed AST
      ↓
R-DAG execution
      ↓
Token Box Model
      ↓
Canonical JSON   ← SINGLE SOURCE OF TRUTH
      ↓
Provider Adapter
      ↓
Provider Payload
```

Adapters operate **only** on Canonical JSON.
They MUST NOT accept partially-compiled or provider-shaped inputs.

---

## Mandatory Adapter Properties

A compliant adapter MUST satisfy **all** of the following.

### 1. Deterministic Mapping

Given identical Canonical JSON input:

* adapter output MUST be byte-for-byte identical
* mapping MUST be pure and stateless
* no randomness, clocks, environment state, or I/O allowed

Adapters MUST be referentially transparent functions:

```
output = adapter(canonical_json)
```

---

### 2. No Semantic Repair

Adapters MUST NOT attempt to "fix" provider constraints by modifying semantics.

Forbidden behaviors include:

* renaming tools to match provider casing quirks
* injecting missing fields
* reordering messages to satisfy undocumented rules
* splitting or merging tool calls

If Canonical JSON violates a provider constraint, the adapter MUST fail loudly.

> Silent recovery is corruption.

---

### 3. Provider Constraints Are Declarative Inputs

All provider-specific constraints MUST be declared **upstream**, during compilation.

Examples:

* required tool-call turn ordering
* serialization restrictions
* streaming limitations
* tool name casing rules

Adapters may only **apply** constraints that were already resolved into Canonical JSON.

They MUST NOT discover or infer constraints dynamically.

This requirement implies that provider targeting is an explicit compilation choice
(e.g. `target = "gemini"`, `profile = "strict_chat"`), not a runtime adaptation.

Adapters MUST NOT compensate for missing or incorrect target selection.

---

### 4. One-to-One Structural Mapping

Adapters MUST preserve structure:

* one canonical tool → one provider tool definition
* one canonical message → one provider message
* explicit `null` fields MUST remain explicit

Adapters MUST NOT:

* collapse multiple messages
* expand single messages
* drop empty or null fields

---

### 5. Failure Containment

Adapters MUST be a **failure boundary**.

If a provider:

* rejects a payload
* changes undocumented behavior
* introduces breaking changes

The failure MUST:

* surface as an adapter error
* NOT mutate Canonical JSON
* NOT poison caches or history

Canonical JSON remains valid and replayable.

---

## Explicit Prohibitions

Adapters MUST be safe to execute in zero-trust environments.

Adapters MUST NOT:

* perform validation (already done by compiler)
* run type checks
* execute lenses
* call LLMs
* fetch external resources
* access filesystem or network

Adapters are not execution engines.

---

## Versioning Requirements

Adapters MUST:

* declare supported Canonical JSON version(s)
* fail on incompatible versions
* be forward-incompatible by default

This prevents silent misinterpretation of newer contracts.

---

## Testing Requirements

Every adapter implementation MUST include:

1. **Golden tests**

   * Canonical JSON input → exact provider payload snapshot

2. **Negative tests**

   * invalid Canonical JSON → deterministic failure

3. **Round-trip safety**

   * adapter output MUST NOT affect canonical replay hashes

Snapshot tests MUST be stable across environments.

---

## Relationship to Reproducibility

Adapters MUST NOT compromise reproducibility guarantees.

Reproducibility is defined entirely by:

* FACET document
* inputs
* execution mode
* Canonical JSON

Adapters are excluded from the reproducibility contract.

They are replaceable.

---

## Design Rationale

Why adapters are intentionally constrained:

* to prevent vendor lock-in
* to localize API churn
* to enable long-term replay and auditing
* to keep the compiler authoritative

Once adapters are allowed to "help", determinism collapses.

---

## Summary

Adapters exist to answer one question only:

> "How does this Canonical JSON look in *this* provider’s dialect?"

Anything beyond that violates the contract.

---

## Status

This document defines **normative requirements** for FACET-compatible adapters.

Any adapter violating these rules is **non-compliant by design**.
