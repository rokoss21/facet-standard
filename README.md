# FACET — Deterministic Contract Layer (since 2025)

**Deterministic contracts for tool-calling, context layout, and agent execution — across vendors.**
A technical standard + reference compiler that treats AI behavior like **compiled software**, not probabilistic improvisation.

> **One file. One truth. One canonical outcome.**
> When the ecosystem needed “retries”, FACET shipped **contracts**.

---

## Specification versions

| Version | Status | Document |
|---------|--------|----------|
| **v2.1.3** | ✅ Recommendation (REC-PROD) — **current** | [`SPECIFICATION.md`](./SPECIFICATION.md) |
| v2.1.2 | Recommendation (REC-PROD) | [`archive/facet-v2.1.2.md`](./archive/facet-v2.1.2.md) |
| v2.0 CR-1 | Candidate Recommendation | [`archive/facet-v2.0-cr1.md`](./archive/facet-v2.0-cr1.md) |
| v1.1 draft r3 | Archive | [`archive/facet-v1.1-draft-r3.md`](./archive/facet-v1.1-draft-r3.md) |

**v2.1.3** is the current production standard. It introduces a complete Policy / Authorization Model (`@policy`), capability/effect classes, a fail-closed Runtime Guard, provenance artifact with hash-chain, and policy semantics versioning (`policy_version`).

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

FACET is a **Neural Architecture Description Language (NADL)** and contract system that:

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

A deterministic tool-calling contract: typed runtime inputs, a policy-guarded tool, and
explicit context layout. This example is verified against the reference compiler
(`facet-fct build` / `run`, v2.1.3).

```facet
@context
  budget: 8000
  defaults: { priority: 500, min: 0, grow: 0, shrink: 1 }

@var_types
  request: "string"
  currency: "string"
  customer_verified: "bool"

@vars
  request:           @input(type="string")
  currency:          @input(type="string", default="USD")
  customer_verified: @input(type="bool", default=false)

@interface Payments
  fn charge(amount: float, currency: string) -> struct {
    transaction_id: string
    status: string
  } (effect="payment")

@policy
  allow: [
    { id: "expose-charge", op: "tool_expose", name: "Payments.charge", effect: "payment" },
    { id: "call-charge",   op: "tool_call",   name: "Payments.charge", effect: "payment" }
  ]
  deny: [
    { id: "no-charge-unverified", op: "tool_call", name: "Payments.charge", unless: $customer_verified }
  ]

@system
  id: "sys.rules"
  priority: 0
  min: 250
  # Critical section: never compressed or dropped
  shrink: 0
  tools: [$Payments]
  content: "Deterministic payment orchestration. Call Payments.charge exactly once."

@user
  id: "user.request"
  priority: 500
  shrink: 1
  grow: 1
  content: $request
```

Run it:

```bash
facet-fct build --input payments.facet
facet-fct run   --input payments.facet --runtime-input payments.input.json --exec
```

What this buys you:

* compile-time typing (FTS) for runtime inputs and for the full tool signature (§8, §14)
* deterministic evaluation order via R-DAG (§10.3)
* deterministic context allocation via the Token Box Model — the system block is Critical
  (`shrink: 0`) and cannot be dropped, whatever else has to give (§11)
* canonical tool schema rendering per provider, derived from FTS via the normative
  JSON Schema mapping (Appendix D)
* fail-closed authorization: `Payments.charge` is `effect="payment"` and is denied unless
  `customer_verified` is true — the guard decides **before** the call is initiated (§16)
* a stable `document_hash` and `policy_hash` in canonical metadata, so a contract change
  is a visible, diffable event (§18)

### Syntax notes that trip people up

FACET is YAML-lite, not a C-family language. The most common mistakes:

* indentation is **exactly 2 spaces**; a tab raises `F002`
* comments are whole-line `#`; there are no trailing comments, no `//`, no `;`
* braces are only inline literals (`{ retries: 3 }`, `["a", "b"]`) — there are no code blocks
* `@import "./file.facet"` imports a whole file; there are no named imports
* there is no control flow: no loops, no `if`, no assignment, no string concatenation, no
  arithmetic. `@vars` values are literals, `$` references, `@input(...)` and lens pipelines,
  nothing else (§14.1). The only branch in the language is the boolean gate `when=` (§12.6)
* `@interface` declares **tools**, not data structures; every `fn` MUST declare `effect=`
  (missing effect → `F456`). Data shapes live in `@var_types` as FTS types
* the model is never called by FACET. `run` emits Canonical JSON; your host calls the
  provider and owns retries, storage, and side effects
* loops and state machines belong to the host: one `run` = one deterministic request.
  Mutable state enters through `@input`

The full facet set in v2.1.3 is `@meta @import @context @system @user @assistant @vars
@var_types @interface @policy @test`. Anything else is not FACET.

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
* **Specification (current — FACET v2.1.3 REC-PROD):** [`./SPECIFICATION.md`](./SPECIFICATION.md)
* **Previous versions:** [`./archive/`](./archive/)

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
* `SPECIFICATION.md` — FACET v2.1.3 Language Specification (REC-PROD, **current**)
* `archive/facet-v2.1.2.md` — FACET v2.1.2 Language Specification (REC-PROD)
* `archive/facet-v2.0-cr1.md` — FACET v2.0 Language Specification (CR-1)

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

### 4) `adapters/` (optional; future)

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
