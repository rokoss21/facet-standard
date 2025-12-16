# Glossary

This glossary defines **normative terminology** used across the FACET standard, specification, and ecosystem documents.

All terms listed here are intended to be interpreted consistently across implementations, adapters, documentation, and discussions.

---

## FACET

**FACET** — A deterministic contract layer and language (NADL) for defining, validating, and executing AI system behavior.

FACET treats AI behavior as **compiled software**, not probabilistic improvisation.

---

## Determinism

**Determinism** — The property that identical inputs produce identical outputs.

In FACET, determinism is defined at the **system level**, not at the model level.

Determinism applies to:

* execution order
* context layout
* tool-calling semantics
* canonical JSON output

---

## Contract

**Contract** — A formally defined, enforced agreement describing:

* valid inputs
* valid outputs
* execution constraints
* resource bounds

A contract differs from a schema in that it is **enforced before execution**, not merely validated after generation.

---

## Contract Layer

**Contract Layer** — The architectural boundary where AI behavior is constrained, validated, and canonicalized.

The contract layer prevents invalid states from entering execution.

FACET implements a contract layer via:

* types (FTS)
* interfaces
* execution phases
* Canonical JSON

---

## NADL (Neural Architecture Description Language)

**NADL** — A declarative language used to describe AI system architecture, behavior, and constraints.

FACET v2.0 is a NADL.

---

## Canonical JSON

**Canonical JSON** — A deterministic, normalized intermediate representation (IR) produced by FACET.

Canonical JSON is:

* provider-agnostic
* structurally stable
* hashable
* replayable

It is the **single source of truth** for execution, caching, testing, and auditing.

---

## IR (Intermediate Representation)

**Intermediate Representation (IR)** — A normalized internal form used between compilation stages.

Canonical JSON serves as FACET’s IR, analogous to LLVM IR.

---

## AST (Abstract Syntax Tree)

**AST** — A structured representation of a parsed FACET document.

The AST is:

* immutable after type checking
* the input to R-DAG construction

---

## FTS (Facet Type System)

**Facet Type System (FTS)** — A strict, language-neutral type system used by FACET.

FTS governs:

* variable types
* tool interfaces
* lens signatures
* multimodal values

---

## Interface (`@interface`)

**Interface** — A typed contract defining a callable tool.

Interfaces specify:

* function name
* parameters
* return type

Interfaces compile into provider-specific tool schemas.

---

## Lens

**Lens** — A transformation function applied to values within FACET.

Lenses are categorized by trust level:

* Level 0 — Pure (fully deterministic)
* Level 1 — Bounded external (deterministic under constraints)
* Level 2 — Volatile (non-deterministic)

---

## R-DAG (Reactive Dependency Graph)

**R-DAG** — A directed acyclic graph representing variable dependencies.

R-DAG guarantees:

* no cycles
* deterministic evaluation order
* single execution per node

---

## Token Box Model

**Token Box Model** — A deterministic algorithm for context allocation under token budgets.

It defines:

* critical vs flexible sections
* compression rules
* drop order

---

## Adapter

**Adapter** — A pure translation layer that maps Canonical JSON to provider-specific payloads.

Adapters:

* MUST be deterministic
* MUST NOT add logic
* MUST NOT mutate semantics

Adapters are translators, not collaborators.

---

## Provider Payload

**Provider Payload** — The final request format required by a specific AI provider API.

Provider payloads are derived views of Canonical JSON.

---

## Pure Mode

**Pure Mode** — An execution mode in which all behavior is fully deterministic.

Pure Mode forbids:

* randomness
* unrestricted I/O
* volatile lenses

Pure Mode outputs are canonical.

---

## Execution Mode

**Execution Mode** — A permissive mode allowing volatile lenses and external side effects.

Execution Mode outputs are not canonical.

---

## Snapshot Testing (Golden Tests)

**Snapshot Testing** — A testing method where output is compared against a known-good snapshot.

FACET uses snapshot testing for:

* Canonical JSON
* adapter outputs
* regression detection

---

## Vendor Lock-in

**Vendor Lock-in** — Dependency on a specific provider’s undocumented or unstable behavior.

FACET mitigates vendor lock-in by:

* enforcing provider-agnostic Canonical JSON
* isolating provider logic in adapters

---

## Reproducibility

**Reproducibility** — The ability to replay executions and obtain identical results.

Reproducibility in FACET is defined by:

* FACET document
* inputs
* execution mode
* Canonical JSON

---

## Invalid State

**Invalid State** — Any state that violates a contract, type, constraint, or execution rule.

FACET prevents invalid states **before execution**.

---

## Summary

This glossary defines the shared language of the FACET ecosystem.

Correct use of these terms is required for:

* specification compliance
* adapter implementation
* meaningful technical discussion

---

**Status:** Normative reference document
