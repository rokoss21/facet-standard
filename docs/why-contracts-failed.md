# Why AI Contracts Failed (Before FACET)

> *Why schemas, prompts, and validators were never enough — and why a deterministic contract layer became inevitable.*

---

## The Original Promise

From the earliest days of production LLM systems, engineers tried to impose structure:

* JSON schemas in prompts
* "Please respond strictly in JSON"
* Tool definitions as docstrings
* Output validators (Pydantic, JSON Schema, regex)

The idea was simple: **treat the model as if it were a well-behaved component**.

That assumption failed.

Not because engineers were careless — but because the architecture was wrong.

---

## Failure Mode #1: Contracts Were Text, Not Enforcement

A schema inside a prompt is not a contract.

It is *advice*.

The model can:

* reorder fields
* omit required keys
* hallucinate extra properties
* emit partial JSON
* switch formats mid-stream

Validation happens **after** generation — when the system is already corrupted.

At that point, all you can do is:

* retry
* truncate
* patch
* pray

This is not enforcement. This is damage control.

---

## Failure Mode #2: Validation Was Post-Hoc

Most stacks adopted the same pattern:

1. Generate text
2. Try to parse it
3. Fail
4. Retry with a stronger prompt

This creates systemic problems:

* retries amplify cost and latency
* failures are nondeterministic
* retry logic becomes untestable
* agents drift between runs

A contract that fires **after** violation is not a contract.

---

## Failure Mode #3: Providers Enforced Hidden Rules

LLM providers impose constraints that are:

* undocumented
* model-specific
* enforced at runtime

Examples observed in production:

* tool calls must follow a user turn (Gemini)
* function arguments silently dropped (OpenAI)
* tool names mutated by casing (Claude)
* complex schemas rejected with 400/500 errors

Developers discovered these rules empirically — via failures.

No schema could express them.
No validator could prevent them.

---

## Failure Mode #4: Context Was Treated as a String

Context windows were handled as text buffers:

* concatenate sections
* count tokens
* truncate when overflowing

This led to:

* critical instructions being dropped
* examples disappearing unpredictably
* system prompts overwritten by history

Truncation was nondeterministic.
Compression was ad hoc.

There was **no allocation algorithm**.

---

## Failure Mode #5: Agents Drifted Over Time

Multi-step agents relied on:

* mutable state
* re-generated context
* probabilistic outputs

The same workflow:

* behaved differently across runs
* failed differently across environments
* could not be replayed or audited

There was no canonical state.

---

## The Root Cause

All of these failures shared one assumption:

> *LLMs can be treated like functions if we ask nicely.*

They cannot.

Probabilistic systems require **deterministic boundaries**.

Contracts must be:

* enforced before generation
* checked at compile time
* expressed as types, graphs, and budgets

Not as strings.

---

## Why a New Layer Was Required

FACET did not appear because prompts were bad.

It appeared because **the entire contract model was inverted**.

Correct order:

1. Define what is allowed
2. Constrain generation
3. Reject invalid states early
4. Produce canonical output

Everything else is patchwork.

---

## The Shift

FACET introduced a different stance:

* determinism is a system property
* contracts live outside the model
* validation precedes execution
* context is a resource, not a string

This was not an optimization.

It was a correction.

---

## Looking Back (From the Future)

In retrospect, this failure was inevitable.

You cannot build reliable systems on top of:

* advisory schemas
* post-hoc validation
* probabilistic contracts

The moment AI moved from demos to infrastructure,
**a deterministic contract layer became unavoidable**.

FACET was simply early.

---

*FACET — Deterministic Contract Layer (since 2025)*
