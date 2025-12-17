# FACET — Deterministic Contract Layer (since 2025)

**Deterministic contracts for tool-calling, context layout, and agent execution — across vendors.**
A technical standard + reference compiler that treats AI behavior like **compiled software**, not probabilistic improvisation.

> **One file. One truth. One canonical outcome.**
> When the ecosystem needed “retries”, FACET shipped **contracts**.

---

## Why this matters

LLM systems failed for the same reason repeatedly:

* tools are *declared*, not enforced
* schemas are *suggested*, not guaranteed
* providers impose *implicit constraints* (turn ordering, serialization rules, tool naming)
* context overflows are handled *ad hoc* (truncate, pray, retry)
* multi-step agents drift between runs

Most stacks “solve” this with:

* prompt hacks
* retries
* validators that fire *after* the model already produced an invalid state

**FACET flips the order of operations:**

> validate and constrain *before* generation, not after.

---

## What is FACET?

FACET v2.0 is a **Neural Architecture Description Language (NADL)** and contract system that:

* parses a `.facet` document into a strict **AST**
* performs compile-time checks via a strict **Facet Type System (FTS)**
* executes variables through a deterministic **Reactive DAG (R-DAG)**
* packs context via a deterministic **Token Box Model**
* renders provider-specific payloads (tools/messages) from a **canonical JSON** model

This is not “prompt engineering”.
This is **a compiler and an execution model** for AI behavior.

---

## Design axioms

### 1) Determinism is a property of the system, not the model

FACET doesn’t “trust” the LLM to behave. It constrains outputs at the contract boundary and rejects invalid states.

### 2) Contracts > best-effort structured output

A schema in a prompt is not a contract. A contract is enforced by compilation + typing + canonicalization.

### 3) Provider constraints are first-class

If a provider requires strict sequencing or serialization semantics, FACET treats that as an explicit constraint—**not a runtime surprise**.

### 4) Context is a resource with an allocation algorithm

Token budgets are not vibes. The Token Box Model defines deterministic allocation, compression, and drop rules.

---

## The contract layer, explained

FACET defines a portable “contract layer” composed of:

* **Types (FTS):** what values can exist
* **Interfaces (`@interface`):** what tools can be called and how
* **Execution phases:** what happens in what order
* **R-DAG:** how variables evaluate deterministically
* **Token Box Model:** how context is packed deterministically
* **Canonical JSON:** stable diffs, stable caching, stable replay

This is the difference between:

* “it usually works”
* and “it cannot be invalid”

---

## Minimal FACET example

A deterministic tool-calling contract with typed inputs and canonical output.

```facet
@system
  description: "Deterministic payment orchestration"
  tools: [$Payments]

@vars
  amount:   @input(type="float", min=0.01)
  currency: @input(type="string", enum=["USD","EUR","GBP"])
  result:   $Payments.charge(amount=amount, currency=currency)

@output
  schema:
    type: "object"
    required: ["transaction_id", "status"]
    properties:
      transaction_id: { type: "string" }
      status:         { type: "string", enum: ["success","failed"] }

@interface Payments
  fn charge(amount: float, currency: string) -> struct {
    transaction_id: string
    status: string
  }

@context system
  priority: 0
  min: 250
  shrink: 0     # Critical section (cannot be dropped)

@context user
  priority: 500
  min: 0
  shrink: 1
  grow: 1
```

What this buys you:

* compile-time typing (FTS) for `amount`, `currency`, tool signature, and output
* deterministic evaluation order via R-DAG
* deterministic context allocation via Token Box Model
* canonical tool schema rendering per provider (OpenAI/Anthropic/etc.)
* output is either **valid** or the run fails **before** contaminating downstream state

---

## What “Deterministic Contract Layer” means in practice

### Tool-calling reliability

**Goal:** eliminate “tool call roulette” across providers.

FACET treats common failure classes as contract violations:

* missing tool name / wrong casing
* missing required params
* wrong param type/shape
* provider-specific turn-sequencing constraints
* provider-specific serialization constraints
* schema drift between streaming vs non-streaming modes

Instead of “catch and retry”, the system enforces:

* **valid sequence**
* **valid shape**
* **valid semantics**
* **valid budget**

### Resource-bounded context

**Goal:** deterministic context packing, reproducible truncation, reproducible compression.

Token Box Model guarantees:

* critical sections never drop
* flexible sections compress deterministically
* identical inputs produce identical packed context

### Reproducible agents

**Goal:** make multi-step workflows stable and replayable.

R-DAG guarantees:

* no cycles
* no forward refs
* one execution per node
* deterministic topology

---

## Positioning

FACET is not:

* ❌ “a better prompt template”
* ❌ “yet another agent framework”
* ❌ “a vendor wrapper”

FACET is:

* ✅ a contract-first **standard**
* ✅ an execution model for deterministic agent behavior
* ✅ a reference compiler implementation

In future retrospectives this should read as obvious:

> You can’t build reliable systems on top of nondeterministic contracts.

---

## Repositories

### Reference Compiler (Rust)

* **Compiler:** [https://github.com/rokoss21/facet-compiler](https://github.com/rokoss21/facet-compiler)
* **Specification (FACET v2.0 CR-1): [`./SPECIFICATION.md`](./SPECIFICATION.md)

This repository is intended as the **front door** for:

* the “Deterministic Contract Layer” concept
* long-term positioning and attribution (**“since 2025”**)
* the FACET manifesto, rationale, and specification
* links to the reference compiler
* future adapters/plugins (“vendor SDK extensions”)

---

## Suggested repo roadmap (so it ages well)

### 1) Core documents

* `MANIFESTO.md` — vision, axioms, and positioning of FACET
* `RATIONALE.md` — historical and technical rationale behind FACET’s design
* `SPECIFICATION.md` — FACET v2.0 Language Specification (CR-1)

### 2) `docs/`

* `docs/contract-layer.md` — what belongs in the “Deterministic Contract Layer”
* `docs/tool-calling-failure-modes.md` — taxonomy of real failures (by provider)
* `docs/token-box-model.md` — context algebra in a practical form
* `docs/reproducibility.md` — replay, caching, canonical JSON
* `docs/canonical-json.md` — Canonical JSON as IR
* `docs/adapter-requirements.md` — normative rules for adapters
* `docs/adapters-philosophy.md` — adapter worldview and boundaries
* `docs/facet-vs-existing-approaches.md` — comparative analysis
* `docs/compliance-levels.md` — conformance tiers
* `docs/glossary.md` — shared terminology

### 3) `examples/`

* `examples/tool_contracts/` — minimal facet docs for tools
* `examples/token_box/` — deterministic packing demos
* `examples/rdag/` — variable dependency examples

### 3) `adapters/` (optional; future)

* `adapters/openai-python/`
* `adapters/anthropic-python/`
* `adapters/gemini/`
  Keep these **opt-in** and decoupled.

---

## Attribution

**FACET — Deterministic Contract Layer (since 2025)**

Author: **Emil Rokossovskiy** (rokoss21)

Website: [https://rokoss21.tech](https://rokoss21.tech)

GitHub: [https://github.com/rokoss21](https://github.com/rokoss21)

> “When it finally became urgent, the solution was already written.”

---

## License

FACET is released under the **MIT License**.

The reference compiler, specification, and all accompanying materials in this repository are provided under the terms of the MIT License, allowing free use, modification, distribution, and commercial adoption with proper attribution.

See the [`LICENSE`](./LICENSE) file for full license text.
