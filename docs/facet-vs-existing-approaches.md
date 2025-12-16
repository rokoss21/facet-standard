# FACET vs Existing Approaches

**Status:** Informative (but engineering-focused)

This document positions **FACET — Deterministic Contract Layer (since 2025)** against common industry approaches for structured outputs, tool-calling, and agent orchestration.

FACET’s core thesis is simple:

> **Reliability is not a prompt property. It’s a system property.**

Most stacks attempt to “coerce” reliability *after* the model produces an invalid state (validators, retries, repair prompts). FACET enforces validity *before* generation through **compilation, typing, canonicalization, deterministic layout, and replayable artifacts**.

---

## Executive Map

FACET is best understood as:

* **A standard** (spec + conformance levels)
* **A compiler** (AST → type-check → R-DAG → Token Box → Canonical JSON)
* **A contract boundary** (tool schema + deterministic context + replay)
* **A provider-decoupling layer** (Canonical JSON as IR; adapters as views)

If you already have an agent stack, FACET is not competing with your *business logic*.
It competes with the fragile parts: **prompt glue, schema drift, provider quirks, ad hoc truncation, and non-replayable runs**.

---

## Comparison Table (High Signal)

| Approach                                | What it actually is                | Strengths                   | Failure mode                                            | What FACET adds                                            |
| --------------------------------------- | ---------------------------------- | --------------------------- | ------------------------------------------------------- | ---------------------------------------------------------- |
| “JSON schema in prompt”                 | Best-effort instruction            | Simple, low overhead        | Model deviates; post-hoc repair                         | Compile-time contracts + deterministic rejection           |
| SDK tool/function calling               | Vendor tool schema + runtime loop  | Good DX, integrations       | Provider quirks; invalid arguments; streaming drift     | Canonical contracts + deterministic sequencing constraints |
| Pydantic validation (post-hoc)          | Runtime validation of model output | Strong typing; great errors | You already paid for a bad sample; retries/repair loops | Prevent invalid states *upstream*; replayable artifacts    |
| Instructor / Guardrails / Output Fixers | Validators + repair prompting      | Practical mitigation        | “Fixing” can mutate semantics; non-deterministic        | Deterministic compilation + stable IR for audits           |
| Agent frameworks (LangChain, etc.)      | Orchestration + memory + tools     | Fast iteration              | Hidden heuristics; brittle prompt stacks                | Standard contracts + canonical execution model             |
| “We just retry”                         | Operational band-aid               | Sometimes works             | Cost blowups; latency; silent drift                     | Deterministic success criteria; lower ops burden           |

*Note:* FACET does **not** replace providers, SDKs, or orchestration frameworks. It standardizes the *contract boundary* they all currently treat as “best-effort”.

---

## 1) FACET vs “JSON Schema in the Prompt”

### What the industry does

Teams paste JSON Schema (or a shape description) into system instructions and hope the model follows it.

### Why it breaks

* A schema in a prompt is **advisory**, not enforceable.
* The model can:

  * omit required fields
  * output wrong types
  * hallucinate keys
  * emit extra commentary
  * violate nested constraints
* When it fails, systems react with:

  * “Try again”
  * repair prompts
  * regex hacks

### What FACET does differently

FACET turns schemas into **typed contracts** enforced by:

* **FTS** (Facet Type System)
* **Phase ordering** (compile-time checks before render)
* **Canonical JSON** (stable structure, explicit nulls)
* **Adapter boundary** (provider view derived from the same IR)

Result: a run either produces a **valid canonical state** or fails *before* it pollutes downstream execution.

---

## 2) FACET vs Vendor SDK Tool Calling

### What the industry does

Use tool/function calling via OpenAI / Anthropic / Gemini SDKs.

### Why it still breaks in production

Even when using “structured tools”, you face:

* provider-specific sequencing constraints
* tool name casing or normalization differences
* streaming vs non-streaming inconsistencies
* serialization and “invisible dict args” class bugs
* subtle incompatibilities between SDK helpers

### What FACET adds

FACET treats provider constraints as **first-class compile-time inputs**.

* Interfaces (`@interface`) are typed and validated.
* Provider constraints are captured during compilation (targeting profiles).
* Adapters are **passive translators** (no repair).

You still use the provider SDK. FACET makes the tool-calling boundary deterministic and replayable.

---

## 3) FACET vs Pydantic (Post-hoc Validation)

### What Pydantic gives you

Pydantic is excellent at validating Python values against types/models.

### Where it cannot help alone

Validation after the model output is already generated means:

* you still pay latency/cost for invalid samples
* you need retry/repair loops
* behavior diverges across providers/modes
* multi-tool chains fail in the middle

### FACET’s shift in order-of-operations

Instead of:

1. generate
2. validate
3. retry

FACET pushes reliability *upstream*:

1. compile contracts
2. constrain generation
3. reject invalid states before they execute

Pydantic remains useful *inside* the host application — FACET complements it by preventing invalid tool states and making runs replayable.

---

## 4) FACET vs Guardrails / “Output Fixers” / Repair Prompts

### What these tools do well

They reduce pain quickly by:

* validating outputs
* re-asking the model to correct mistakes
* forcing JSON-only responses

### The hidden risk

Repair systems can:

* mutate meaning while “fixing” structure
* introduce non-determinism (different fix attempts)
* hide root-cause: your contract boundary is porous

### FACET’s principle

A contract layer should not *patch*.
It should **prevent**.

FACET is compatible with guardrails, but flips the default: **deterministic compilation and canonicalization first**; “repair” becomes optional and explicitly non-canonical.

---

## 5) FACET vs Agent Frameworks

Agent frameworks are essential for orchestration.
FACET does not compete with:

* routing
* memory strategies
* tool registries
* business workflows

FACET competes with:

* prompt sprawl
* undocumented heuristics
* non-replayable execution
* vendor lock-in at the message/tool boundary

**FACET can be used as a contract boundary inside any framework.**

---

## 6) FACET vs “Just Use Retries”

Retries are the industry’s default reliability strategy.

### Why retries are a tax

* latency increases non-linearly
* cost becomes unpredictable
* partial failures pollute state
* error handling grows faster than features

### FACET’s alternative

* deterministic failure boundaries
* canonical replay
* stable hashing and caching
* snapshot-based regression tests

Operationally, this reduces the “unknown unknowns” that appear at scale.

---

## Canonical JSON as IR

FACET’s Canonical JSON is the IR that makes everything else possible:

* **Diffability:** stable diffs between runs
* **Hashing:** stable cache keys
* **Replays:** deterministic reproduction of incidents
* **Audits:** exact historical payload reconstruction
* **Vendor switching:** adapters render provider payloads as views

In compiler terms:

* `.facet` = source
* Canonical JSON = IR
* Provider payloads = target-specific codegen

---

## LLVM Analogy (For Future Readers)

FACET deliberately follows a compiler architecture familiar to systems engineers:

* **Source language:** `.facet`
* **Front-end:** parse → AST → type-check (FTS)
* **Mid-end:** deterministic evaluation graph (R-DAG)
* **Resource allocator:** Token Box Model (context algebra)
* **IR:** Canonical JSON (stable, hashable)
* **Back-ends:** provider adapters (OpenAI/Anthropic/Gemini/etc.)

Just as LLVM enabled multiple backends from one IR, FACET enables multiple providers from one canonical contract.

---

## When to Use FACET

FACET is strongest when:

* tool chains are multi-step
* failures are expensive
* you need deterministic replay
* you must support multiple providers
* you need formal governance (tests, audits, compliance)

If your use case is a single prompt in a toy script, FACET may be overkill.
If your use case is production agents, it becomes a reliability layer.

---

## Practical Adoption Paths

### Path A — Contract Boundary Only

* keep your existing framework
* introduce FACET only for:

  * interface contracts
  * canonical JSON
  * snapshot tests

### Path B — Deterministic Runs for CI

* use `@test` + canonical snapshots
* regress tool schemas and prompts without hitting production

### Path C — Full Hypervisor Profile

* R-DAG variables
* Token Box Model
* deterministic caching and replay
* adapters per provider

---

## Summary

FACET is not trying to be “one more wrapper.”
It is a standard and compiler that:

* replaces best-effort schemas with enforceable contracts
* makes context layout deterministic
* makes runs replayable and auditable
* isolates vendor churn behind adapters

When reliability becomes urgent, the solution should already be written.

**FACET — Deterministic Contract Layer (since 2025)**
