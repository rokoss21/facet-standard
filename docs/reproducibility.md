# Reproducibility

## Purpose

This document defines **reproducibility guarantees** in FACET-compliant systems.

Reproducibility means that an AI execution can be:

* replayed exactly
* audited after the fact
* diffed across versions
* cached safely
* reasoned about as a deterministic system

FACET treats reproducibility as a **first-class property**, not an emergent side-effect.

---

## The Core Problem

Most LLM-based systems are *not reproducible*.

Even when developers fix:

* model version
* temperature
* prompt text

results still drift due to:

* implicit context truncation
* provider-specific tool sequencing rules
* non-deterministic serialization
* runtime retries and fallbacks
* hidden defaults in SDKs

This makes:

* debugging unreliable
* audits unverifiable
* regression testing meaningless

FACET exists specifically to eliminate these failure modes.

---

## FACET’s Reproducibility Contract

A FACET execution is reproducible **if and only if** all of the following are identical:

* FACET document
* runtime inputs (`@input` values)
* execution mode (Pure)
* lens registry and versions
* compiler version
* `policy_version` (version of `@policy` DSL and guard evaluation semantics)

Under these conditions:

* execution order is fixed
* context layout is fixed
* tool schemas are fixed
* Canonical JSON output is fixed

This is a **hard guarantee**, not a best-effort promise.

---

## Deterministic Execution Stack

Reproducibility emerges from the interaction of six enforced layers:

1. **Typed AST** — no ambiguous values
2. **Reactive DAG (R-DAG)** — fixed evaluation order by merged ordered-map insertion position
3. **Token Box Model** — deterministic context packing in FACET Units
4. **Runtime Guard** — deterministic policy enforcement; every guarded operation is logged as a `GuardDecision` event
5. **Canonical JSON** — stable intermediate representation including `policy_hash` and `policy_version`
6. **Provider Adapters** — isolated, stateless renderers

If any layer fails to produce a valid deterministic output, execution **aborts**.

FACET never emits a “mostly correct” state.

---

## Canonical JSON as the Replay Artifact

Canonical JSON is the **replay key**.

Given a stored Canonical JSON document:

* provider payloads can be regenerated
* execution history can be reconstructed
* outputs can be revalidated
* hashes can be recomputed

This enables:

* exact replay in incident analysis
* post-hoc audits
* cross-provider migration

Canonical JSON decouples **execution truth** from **provider behavior**.

## Execution Artifact as the Audit Record

For Hypervisor runs, the **Execution Artifact** extends reproducibility into the governance layer.

It captures:

* every guarded operation and its policy decision (`GuardDecision`)
* a cryptographic hash-chain over all decisions (`provenance.hash_chain`)
* an optional digital signature (`attestation`)

The hash-chain seed includes `facet_version`, `host_profile_id`, `document_hash`, `policy_hash`, `policy_version`, `profile`, and `mode`. This makes the chain unique to the specific execution configuration — it cannot be confused with a chain from a different policy version or mode even if the document is identical.

The Execution Artifact enables **tamper-evident audit trails** for regulated environments.

---

## Snapshot Testing and CI

FACET enables **snapshot-based testing** of AI systems.

In `@test` blocks:

* Canonical JSON is emitted
* JSON is hashed and stored as a golden snapshot
* future runs compare against the snapshot

Any difference indicates:

* a semantic change. or
* a regression. or
* an intentional evolution.

This allows AI behavior to be tested like compiler output.

---

## What Reproducibility Enables

Reproducibility unlocks capabilities that are otherwise impossible:

* deterministic caching and memoization
* cost prediction and budgeting
* stable rollbacks
* compliance and audit trails
* long-term system evolution

Without reproducibility, AI systems remain operationally fragile.

---

## What FACET Does *Not* Claim

FACET does **not** claim:

* models are deterministic
* outputs are always identical across providers
* creative sampling is eliminated

FACET claims:

* nondeterminism is **explicit, bounded, and isolated**
* deterministic layers remain deterministic
* probabilistic behavior cannot corrupt system state

---

## Design Principle

> Reproducibility is not about freezing intelligence.
> It is about freezing **contracts**.

FACET enforces reproducibility by turning AI execution into a **compiled, replayable process**, not an ephemeral interaction.

---

## Status

This document defines the **normative reproducibility guarantees** for FACET v2.1.3 Hypervisor Profile.

All compliant implementations MUST uphold these guarantees in Pure Mode execution.
