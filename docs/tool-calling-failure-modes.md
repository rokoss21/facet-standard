# Tool-Calling Failure Modes

## Purpose

This document catalogs **real, recurring failure modes** observed in LLM tool-calling systems across major providers and agent frameworks.

Its goal is to:

* make failures *explicit and enumerable*
* demonstrate that these failures are **systemic**, not user error
* show why post-hoc validation and retries are structurally insufficient
* define the problem space a deterministic contract layer must solve

This is not a critique of any single provider.
It is a taxonomy of failure modes that emerge when **probabilistic generation is asked to satisfy implicit contracts**.

---

## Core Observation

Tool calling today fails not because models are weak, but because:

> **Tool contracts are implicit, informal, and enforced only after generation.**

LLMs are expected to infer:

* schema shape
* parameter types
* tool names
* sequencing rules
* provider-specific constraints

…without those constraints being part of the execution model.

The result is a predictable set of failure classes.

---

## Failure Class 1: Schema Shape Violations

### Description

The model produces a tool call whose JSON structure does not match the declared schema.

### Examples

* missing required fields
* extra unexpected fields
* wrong nesting depth
* arrays where objects are expected

### Real-World Symptoms

* Pydantic validation errors
* silent field dropping
* runtime exceptions after generation

### Why Retries Fail

Retries re-sample from the same unconstrained distribution.
They reduce probability but **do not eliminate invalid states**.

---

## Failure Class 2: Type Mismatches

### Description

The model emits values of the wrong type for otherwise valid fields.

### Examples

* numbers as strings (`"42"` instead of `42`)
* booleans as text (`"true"`)
* objects serialized as strings

### Real-World Symptoms

* deserialization failures
* silent coercion bugs
* inconsistent behavior across SDKs

### Root Cause

Schemas exist only as *instructions*, not as **constraints on generation**.

---

## Failure Class 3: Tool Name Drift

### Description

The model references a tool name that does not exactly match the declared identifier.

### Examples

* casing drift (`process_payment` → `Process_Payment`)
* partial names (`search` → `search_docs`)
* hallucinated tool names

### Impact

* downstream dispatch failure
* silent no-op behavior
* hard-to-debug agent stalls

---

## Failure Class 4: Parameter Visibility Loss

### Description

Certain parameter shapes are ignored or dropped by provider APIs or SDK layers.

### Examples

* `dict` arguments not visible to OpenAI-powered agents
* binary payloads failing serialization

### Impact

* tools invoked with incomplete inputs
* agents behaving inconsistently between sync and stream modes

### Root Cause

Mismatch between:

* declared tool schema
* provider transport format
* SDK serialization logic

---

## Failure Class 5: Sequencing Violations

### Description

The model produces a *valid-looking* tool call at an **invalid point in the conversation**.

### Examples

* Gemini requiring tool calls immediately after user or tool response turns
* tool calls emitted after assistant messages

### Symptoms

* provider-side `INVALID_ARGUMENT` errors
* conversation reset or termination

### Why This Is Fundamental

Sequencing rules are **provider-specific** and **not visible to the model**.

---

## Failure Class 6: Streaming vs Non-Streaming Drift

### Description

The same agent behaves differently in streaming and non-streaming modes.

### Examples

* tool calls appearing only in one mode
* different output shapes
* missing final tool invocation

### Impact

* non-reproducible behavior
* broken production parity

---

## Failure Class 7: Multi-Tool Chain Collapse

### Description

Agents fail when chaining multiple tools in a single reasoning flow.

### Symptoms

* early termination
* partial execution
* invalid intermediate state

### Root Cause

Each tool call compounds uncertainty.
Without contracts, error probability grows multiplicatively.

---

## Failure Class 8: Context-Induced Tool Corruption

### Description

Tool calls degrade as context grows or is truncated heuristically.

### Examples

* truncated tool schema
* partial parameter emission
* hallucinated defaults

### Root Cause

Context overflow handled by truncation, not allocation.

---

## Why Validation and Retries Cannot Fix This

Post-generation validation:

* detects invalid states **after they exist**
* cannot prevent invalid intermediate steps
* cannot guarantee convergence

Retries:

* reduce probability
* increase cost
* do not change the state space

This is equivalent to catching compiler errors at runtime.

---

## The Missing Layer

All listed failures share one property:

> **They occur because tool contracts are not part of the execution model.**

A deterministic system must:

* encode schema, types, and sequencing *before generation*
* reject invalid states *before emission*
* treat provider constraints as first-class

This is the problem space FACET addresses at the contract layer.

---

## Status

This document defines an **informative but implementation-grounded taxonomy** of tool-calling failures.

It is intended to support:

* adapter design
* contract systems
* future standardization efforts

The failures described here are not hypothetical.
They are observed, reproducible, and systemic.
