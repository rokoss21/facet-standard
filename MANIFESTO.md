# FACET Manifesto

## Deterministic Contract Layer for Intelligence

**FACET is not a product. Not a framework. Not a wrapper.**
FACET is a *line in the sand*.

It exists because modern AI systems crossed a threshold where probabilistic behavior stopped being acceptable.

---

## The Broken Assumption

For years, the industry operated on a convenient lie:

> *"If we prompt carefully enough, models will behave."*

This assumption failed at scale.

Same input → different output.
Schemas treated as suggestions.
Tools called incorrectly, out of order, or not at all.
Context windows overflow silently.
Retries replace guarantees.

What we called *engineering* was often just **ritualized hope**.

---

## FACET’s Core Claim

**Determinism is not a property of models.**
**Determinism is a property of systems.**

FACET does not try to make LLMs smarter.
It makes *AI behavior accountable*.

---

## Contracts, Not Prompts

A schema in a prompt is not a contract.

A contract:

* is validated *before* execution
* is enforced *during* execution
* is rejected if it cannot be satisfied

FACET treats every interaction as a compiled agreement:

* typed inputs
* typed tools
* typed outputs
* bounded resources

If the contract cannot be fulfilled — the system **fails fast**, not silently.

---

## Context Is a Resource, Not a Guess

Token limits are not vibes.

FACET introduces the **Token Box Model** because:

* truncation without rules is corruption
* compression without determinism is drift
* retries without guarantees are debt

Context must be:

* allocated
* prioritized
* compressed
* dropped

**Deterministically.**

---

## Execution Must Be Explainable

FACET systems:

* execute in ordered phases
* evaluate variables via an explicit dependency graph
* forbid cycles and forward references
* produce canonical JSON

If a system cannot be replayed, diffed, cached, or reasoned about — it is not production-grade.

---

## Providers Are Not Abstracted Away

Every LLM provider has constraints:

* turn ordering rules
* serialization quirks
* tool invocation semantics
* streaming differences

FACET makes these constraints *explicit*.

No hidden behavior.
No runtime surprises.
No “it works on OpenAI but not on Gemini”.

---

## Failures Belong at Compile Time

FACET moves failure **left**:

* before runtime
* before deployment
* before production incidents

Invalid states are unrepresentable.

This is the same philosophy that made:

* type systems matter
* compilers necessary
* infrastructure reproducible

---

## What FACET Is Not

FACET is not:

* prompt engineering
* an agent framework
* a retry strategy
* a wrapper around APIs

FACET is the *missing layer* underneath all of them.

---

## The Long View

FACET was written early — deliberately.

Before the failures became headline incidents.
Before agents were trusted with money, autonomy, and decisions.
Before reproducibility became a regulatory requirement.

When the problem becomes undeniable, the solution should already exist.

FACET is that solution.

---

## Attribution

**FACET — Deterministic Contract Layer (since 2025)**

Author: Emil Rokossovskiy (rokoss21)

Website: [https://rokoss21.tech](https://rokoss21.tech)

GitHub: [https://github.com/rokoss21](https://github.com/rokoss21)

> *When reliability became mandatory, contracts replaced hope.*


