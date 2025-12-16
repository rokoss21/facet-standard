# Token Box Model

## Purpose

The Token Box Model defines a **deterministic context allocation algorithm** for LLM execution.

Its purpose is to replace ad-hoc truncation, heuristic compression, and retry-based prompt handling with a **formal, reproducible layout model**.

In FACET, context is not a side-effect of string concatenation — it is a **compiled artifact**.

---

## Problem Statement

Modern LLM systems fail under context pressure because:

* token limits are enforced late (after prompt assembly)
* truncation is implicit and non-deterministic
* critical instructions may be silently dropped
* different runs drop different parts of context
* provider tokenizers behave differently

This leads to:

* non-reproducible agent behavior
* debugging instability
* production-only failures

The Token Box Model addresses this by making **context layout explicit, typed, and deterministic**.

---

## Core Concept

The context is treated as a **finite-capacity container** with a fixed token budget.

Each logical block of prompt data is represented as a **Section** with explicit layout constraints.

The compiler is responsible for fitting all sections into the available budget **without violating invariants**.

---

## Section Definition

Each Section has the following properties:

| Field       | Type         | Description                             |
| ----------- | ------------ | --------------------------------------- |
| `priority`  | int          | Removal order (lower = dropped earlier) |
| `base_size` | int          | Token count after render                |
| `min`       | int          | Minimum guaranteed size                 |
| `grow`      | float        | Weight for expansion                    |
| `shrink`    | float        | Weight for compression                  |
| `strategy`  | LensPipeline | Compression strategy                    |

### Critical Sections

A Section is **Critical** if:

```
shrink == 0
```

Critical sections:

* MUST NOT be compressed
* MUST NOT be truncated
* MUST NOT be dropped

If all critical sections do not fit, execution **MUST fail**.

---

## Deterministic Algorithm

Let:

* `S` = all sections
* `B` = token budget
* `size[i]` = base_size of section `i`

### Step 1 — Fixed Load

```
Critical = { i | shrink[i] == 0 }
FixedLoad = sum(size[i] for i in Critical)
```

If:

```
FixedLoad > B
```

→ **FAIL** with `ContextCriticalOverflow`

---

### Step 2 — Free Space

```
FreeSpace = B - FixedLoad
```

---

### Step 3 — Expansion (Optional)

```
Expandable = { i | grow[i] > 0 }
```

FreeSpace MAY be distributed proportionally:

```
extra[i] = FreeSpace * (grow[i] / sum(grow))
```

---

### Step 4 — Compression

If total size exceeds budget:

```
Deficit = total_size - B
Flexible = { i | shrink[i] > 0 }
Sort Flexible by (priority ASC, shrink DESC)
```

For each section:

1. Apply compression strategy
2. Recompute size
3. Truncate to `min` if needed
4. Drop section if still oversized

Stop when `Deficit <= 0`.

---

## Determinism Guarantees

Given:

* identical sections
* identical priorities
* identical token budget

The resulting context layout is:

* byte-for-byte identical
* order-stable
* provider-independent

This makes context **cacheable, diffable, and replayable**.

---

## Why This Matters

Without a formal layout model:

* retries hide bugs
* prompt behavior drifts
* context loss is invisible

With the Token Box Model:

* failures are explicit
* critical instructions are protected
* behavior is reproducible

This turns context handling from a heuristic into an **engineering discipline**.

---

## Relationship to FACET Execution

The Token Box Model is executed in:

**Phase 4 — Layout**

Inputs:

* computed variable values
* rendered sections
* token budget

Output:

* finalized ordered context

Any violation aborts execution before provider interaction.

---

## Design Principle

> Context is not text.
> Context is a resource.

The Token Box Model makes that resource explicit, bounded, and deterministic.

---

## Status

This document defines the **normative Token Box Model** for FACET v2.0 and later.

All compliant implementations MUST follow this algorithm when performing context layout.
