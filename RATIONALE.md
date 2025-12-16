# History and Rationale of FACET

## Purpose of This Document

This document records the **historical context, architectural motivations, and rationale** behind the design decisions of FACET.

It exists to answer a recurring future question:

> *Why was FACET designed this way, and not differently?*

This is **not** a changelog and **not** a roadmap.
It is a rationale document intended for:

* future maintainers
* standard reviewers
* enterprise architects
* historians of AI infrastructure

---

## 1. Pre-FACET Era (≈ 2018–2022)

### 1.1 Prompt Engineering as an Anti-Pattern

Early LLM systems treated prompts as:

* opaque strings
* mutable runtime artifacts
* informal contracts

As systems grew, prompt engineering evolved into:

* copy-paste templates
* ad-hoc retries
* regex-based JSON extraction
* post-hoc validation

Failures were handled **after generation**, not prevented.

This era established a false assumption:

> *LLM unreliability is inherent and unavoidable.*

---

### 1.2 Structured Output Did Not Solve the Core Problem

Later approaches introduced:

* JSON schemas in prompts
* function / tool calling APIs
* Pydantic-style validators

However:

* schemas were advisory, not enforced
* providers interpreted constraints differently
* invalid states were still produced
* validation happened **after** the model responded

The system still allowed invalid intermediate states.

---

## 2. FACET v1.x (2022–2024): Lessons Learned

FACET v1.x originated as a **deterministic prompt templating system**.

It introduced:

* structured blocks
* conditional logic
* early lens pipelines
* canonical JSON output

### 2.1 What v1.x Got Right

* determinism mattered
* canonical JSON enabled caching and diffing
* composition beat monolithic prompts

### 2.2 What v1.x Could Not Solve

* no type system
* no execution model
* no formal notion of invalid state
* no prevention of tool-call failures

FACET v1.x reduced chaos, but did not eliminate it.

---

## 3. The Breaking Point (2024–2025)

By 2024, several systemic failures became unavoidable:

* multi-tool agents failing nondeterministically
* provider-specific tool-call rules causing silent breakage
* streaming vs non-streaming divergence
* context truncation corrupting logic
* retries masking correctness bugs

At scale, these failures were:

* expensive
* non-reproducible
* impossible to audit

The industry response remained reactive:

> *Add retries. Add validators. Add guardrails.*

This approach did not converge.

---

## 4. The Core Insight

FACET v2.0 is built on a single foundational realization:

> **You cannot build reliable systems on top of nondeterministic contracts.**

The problem was not LLMs.
The problem was **lack of a contract layer**.

---

## 5. FACET v2.0 (2025): A Structural Reset

FACET v2.0 was intentionally designed as:

* a compiler, not a template engine
* a contract system, not a helper library
* an execution model, not a runtime patch

### 5.1 Determinism as a System Property

FACET does not attempt to make models deterministic.

Instead:

* invalid states are prevented upstream
* contracts are enforced before execution
* outputs are canonicalized

Determinism is achieved **by architecture**, not by probability control.

---

### 5.2 Canonical JSON as Intermediate Representation

FACET introduced **Canonical JSON** as its IR:

* provider-neutral
* hash-stable
* diff-friendly
* replayable

This decouples:

* authoring
* execution
* provider rendering

and prevents vendor lock-in.

---

### 5.3 Execution Phases and R-DAG

FACET formalized execution into five phases:

1. Resolution
2. Type Checking
3. Reactive Compute (R-DAG)
4. Layout (Token Box Model)
5. Render

This eliminated:

* implicit execution order
* hidden side effects
* runtime guesswork

---

### 5.4 Token Box Model

Context handling was redefined as:

* a resource allocation problem
* with explicit priorities
* deterministic compression rules

This replaced:

* truncation heuristics
* "best effort" packing
* silent loss of critical data

---

### 5.5 Adapters as Pure Translators

Adapters were intentionally constrained:

* no logic
* no inference
* no recovery

This preserves:

* auditability
* replayability
* long-term stability

---

## 6. Rejected Alternatives (By Design)

FACET explicitly rejected:

* probabilistic retries
* self-healing prompts
* adaptive prompt rewriting
* runtime schema repair

These techniques obscure failure rather than eliminate it.

---

## 7. Long-Term Positioning

FACET is designed to age like:

* LLVM
* SQL
* JSON Schema

Not like:

* an agent framework
* a vendor SDK
* a prompt toolkit

It is intended to remain:

* boring
* strict
* predictable

for decades.

---

## 8. Historical Attribution

**FACET — Deterministic Contract Layer (since 2025)**

Author: Emil Rokossovskiy (rokoss21)

The central idea predates industry consensus.

When determinism became urgent, the architecture already existed.

---

## Status

This document is **informative**.

It does not define new requirements, but explains why the requirements exist.

---

*End of document.*
