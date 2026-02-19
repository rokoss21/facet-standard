### FACET v2.1.3 — Production Language Specification

**Status:** Recommendation (REC‑PROD)  
**Version:** 2.1.3  
**Date:** 2026‑02‑19  
**Author:** Emil Rokossovskiy  
**Document Type:** Technical Standard  
**Format:** Markdown (UTF‑8)

---

## Table of Contents

1. Scope  
2. Conformance, Profiles, Extensions  
3. File Normalization (UTF‑8, NFC, LF)  
4. Lexical Rules  
5. Concrete Syntax (Facet YAML‑lite)  
6. AST Model (Normative)  
7. Resolution (`@import`) and Merge (Smart Merge)  
8. Facet Type System (FTS)  
9. Lenses (Registry, Trust Levels, Gas, Determinism)  
10. Execution Model (Phases 1–5, Modes)  
11. Token Box Model (Context Algebra)  
12. Standard Facets & Semantics  
13. Interfaces (`@interface`)  
14. Variables (`@vars`), Types (`@var_types`), Inputs (`@input`)  
15. Testing (`@test`)  
16. Policy / Authorization Model (`@policy`, Capability Classes, Runtime Guard)  
17. Security Model  
18. Canonical JSON, Canonicalization, Document Hash  
19. Error Code Catalog (Normative)  
20. Reference CLI (`fct`) (Recommended)  
21. Change History  
Appendix A — Standard Lens Library (Normative)  
Appendix B — ABNF Grammar (Informative)  
Appendix C — Cache Key & Pure Cache‑Only Contract (Normative)  
Appendix D — FTS → JSON Schema Mapping (Normative)  
Appendix E — Conformance Checklist (Normative)  
Appendix F — Execution Artifact, Guard Decisions, Attestation (Normative)

---

## 1. Scope

FACET is a Neural Architecture Description Language (NADL) for defining, validating, and executing AI request construction in a deterministic, type-safe, resource-bounded manner.

FACET standardizes:

- Concrete syntax and normalized source form
- Deterministic import resolution and deterministic merge behavior
- A strict type system (FTS) for variables, lenses, interfaces, and policies
- Reactive computation of variables via a dependency graph (R‑DAG)
- Deterministic context packing via a provider-independent layout model
- Provider-agnostic Canonical JSON output
- Deterministic behavior in Pure Mode
- A security boundary for hermetic compilation and constrained runtime I/O
- A policy / authorization model with fail-closed runtime guard enforcement
- A minimal conformance testing facility
- A minimal provenance / evidence artifact for auditability and attestation

FACET does not standardize model inference behavior, provider request/response protocols, or model internals beyond the requirement that provider payloads preserve Canonical JSON semantics.

---

## 2. Conformance, Profiles, Extensions

This document uses RFC 2119 keywords (**MUST**, **MUST NOT**, **SHOULD**, **MAY**).

An implementation is **FACET v2.1.3 compliant** if and only if it satisfies all normative requirements in this specification for its declared profile(s) and mode(s).

### 2.1 Profiles

#### 2.1.1 Core Profile (Minimal)

Core implementations MUST support:

- Phase 1 (Resolution), Phase 2 (Type Checking), Phase 5 (Render)
- `@meta`, `@import`, `@context`, `@system`, `@user`, `@assistant`, `@vars`, `@var_types`, `@policy`
- Inline list/map literals
- Parsing of the full FACET concrete syntax (§5), followed by profile enforcement

Core restrictions:

- `@vars` values MUST be literals only (no `$` references, no lens pipelines, no `@input`)
- No `@interface`, no `@test`, no R‑DAG evaluation, no Token Box Model, no multimodal canonicalization, no runtime guard, no execution artifact emission
- `@policy` is permitted, but Core MUST treat enforcement points that require execution (`tool_call`, `lens_call`) as **not applicable**; Core MUST still type-check `@policy` and MUST render `metadata.policy_hash` (§18.1)

Core MUST reject any syntactically valid construct disallowed by Core using `F801` (not `F003`).

#### 2.1.2 Hypervisor Profile (Full)

Hypervisor implementations MUST support all features described in this specification, including:

- All phases 1–5
- Token Box Model
- Multimodal canonicalization contract
- `@interface`, `@input`, R‑DAG, `@test`
- Lens registry, gas accounting, and Pure cache-only contract
- `@policy` enforcement, capability/effect classes, runtime guard (fail-closed), and execution artifact provenance (§16, Appendix F)

Unless explicitly stated otherwise, normative requirements target Hypervisor.

### 2.2 Extensions (Namespacing)

Host extensions MUST be namespaced to avoid collisions:

- Facets: `@x.<host>.<name>` or `@x_<host>_<name>`
- Lenses: `x.<host>.<lens_name>`

Error codes:

- The numeric `F000–F999` space is RESERVED for the FACET standard only. Hosts MUST NOT emit new numeric `F`-prefixed codes.
- Host extension diagnostics MUST use a namespaced code string: `X.<host>.<code>` (example: `X.acme.TIMEOUT`).
- A host MAY include a secondary legacy code in auxiliary diagnostics, but the primary error identifier MUST be namespaced.

---

## 3. File Normalization (UTF‑8, NFC, LF)

Implementations MUST apply the following normalization before Phase 1 completes:

1. Input MUST be valid UTF‑8.
2. Text MUST be normalized to Unicode NFC.
3. Line endings MUST be normalized to LF (`\n`).
4. Tabs (`\t`) MUST raise `F002`.

All source spans, hashing, and canonicalization MUST refer to the normalized form.

---

## 4. Lexical Rules

### 4.1 Identifiers (Normative)

Identifiers MUST match:

- `[A-Za-z_][A-Za-z0-9_]*`

Non‑ASCII identifiers MUST raise `F003`.

### 4.2 Strings (Normative)

Strings are double‑quoted and support escapes:

- `\"`, `\\`, `\n`, `\t`, `\r`, `\uXXXX`

Invalid escapes or unclosed strings MUST raise `F003`.

### 4.3 Scalars (Normative)

Supported scalars:

- `true`, `false`, `null`
- integers: optional leading `-`
- floats: decimal form with optional exponent `e/E`

Invalid scalar MUST raise `F003`.

### 4.4 Map keys (Normative)

A map key in the concrete syntax MAY be either:

- an identifier, or
- a quoted string

Semantic restrictions on where string keys are permitted are defined in §12.1.

---

## 5. Concrete Syntax (Facet YAML‑lite)

FACET is indentation-scoped. Indentation MUST be exactly 2 spaces.

### 5.1 Facet blocks

A facet begins with `@name` optionally followed by attributes:

```facet
@system(model="gpt-x", when=true)
  content: "You are a helpful assistant."
```

Facet bodies are maps (block form) and may contain nested maps and lists.

Exception: Certain facets define a specialized body subgrammar in their own sections (notably `@interface` (§13) and `@test` (§15)). Such facets remain syntactically valid FACET documents, but their internal line structure is not constrained to the generic map/list-only body in Appendix B (which is informative).

### 5.1.1 Facet attributes (Normative)

- Attribute syntax: `@facet(k=v, ...)`
- Attribute values MUST be atoms of the attribute-atom set:
  - `string | number | bool | null | varref`
- Lens pipelines inside attributes MUST raise `F003`.
- `@input(...)` MUST NOT appear in facet attributes and MUST raise `F003`.
- Attribute interpolation syntax containing `{{` or `}}` is forbidden and MUST raise `F402`.

### 5.2 Maps and lists

Map entry:

```facet
key: "value"
```

List:

```facet
items:
  - "a"
  - "b"
```

### 5.3 Inline list/map literals (Normative)

Inline list and inline map literals MUST be supported in both Core and Hypervisor:

```facet
tags: ["a", "b", $x]
cfg: { retries: 3, mode: "safe" }
```

Trailing commas are not permitted; violations MUST raise `F003`.

### 5.4 Variable references

- `$name`
- `$name.path.to.field`

Path semantics are defined in §14.8.

### 5.5 Lens pipelines

Pipeline syntax:

```facet
value: $doc |> trim() |> json(indent=2)
```

All profiles MUST parse the full FACET concrete syntax (including `|>` pipelines and `$` references). Profile restrictions are enforced after parsing; therefore any construct that is syntactically valid but disallowed by the active profile/mode MUST raise `F801` (not `F003`).

### 5.6 Directive-expressions (Normative)

FACET defines a directive-expression:

- `@input(...)`

Directive-expressions are expressions and may appear only where explicitly permitted by this specification (§14.3). Any directive-expression used in an invalid position MUST raise `F452`.

---

## 6. AST Model (Normative)

Implementations MUST produce an AST with at least:

- `FacetNode(name, attrs, body, span)`
- `MapNode(entries[], span)` (ordered)
- `ListNode(items[], span)` (ordered)
- `KeyValueNode(key, valueExpr, span)` where `key` is `IdentifierKey` or `StringKey`
- `StringNode(value, span)`
- `ScalarNode(kind, value, span)` where `kind ∈ {int,float,bool,null}`
- `VarRefNode(name, pathSegments[], span)`
- `LensPipelineNode(baseExpr, steps[], span)`
- `LensCallNode(name, args[], namedArgs{}, span)`
- `InputExprNode(attrs, span)` representing `@input(...)`
- `ImportDirectiveNode(pathString, span)` for top-level `@import`

For `@policy`, implementations MUST represent policy rule structures using the standard `MapNode`/`ListNode`/scalar nodes and MUST preserve ordering as required by §16.2 and §7.4.

AST spans MUST reference normalized NFC+LF source coordinates.

Ordered maps MUST preserve key insertion order as defined by the merge rules in §7.4.

---

## 7. Resolution (`@import`) and Merge (Smart Merge)

### 7.1 `@import` (Normative)

Top-level directive:

```facet
@import "relative/path/module.facet"
```

Rules:

1. Imports MUST resolve relative to the importing file.
2. Imports MUST be constrained by allowlisted roots (§17.2).
3. Violations of import sandbox rules (absolute paths, `..` traversal, URL imports, or outside allowlisted roots) MUST raise `F601`.
4. Import not found MUST raise `F601`.
5. Import cycles MUST raise `F602`.

### 7.2 Deterministic resolution order (Normative)

Imports MUST be applied in source order: imported content is expanded in-place at the point the `@import` appears, forming a single Resolved Source Form and a single Resolved AST.

### 7.3 Standard facet cardinality (Normative)

Standard facets have fixed cardinality:

Singleton-map facets (deep-merged by key):

- `@meta`, `@context`, `@vars`, `@var_types`, `@policy`

Repeatable-block facets (collected as ordered lists of blocks):

- `@interface`, `@system`, `@user`, `@assistant`, `@test`

For repeatable-block facets, each occurrence represents one block instance. Merging/applying imports MUST preserve the deterministic block occurrence order from the Resolved Source Form.

If a host introduces an extension facet, the host MUST define its cardinality. If cardinality is unknown, implementations MUST raise `F452`.

### 7.4 Merge rules (Normative)

#### 7.4.1 Singleton-map merge

Singleton-map facets MUST be merged as ordered maps:

- When a key appears for the first time, it is inserted at that position.
- When a key appears again, its value is overridden by the later value, but the key’s position MUST remain the position of its first insertion.
- If both old and new values are maps, they MUST be deep-merged recursively using the same ordered-map rules.
- Lists are not deep-merged unless the keyed-list rule applies (§7.4.3) or a facet-specific list merge rule is defined (see §16.2.3 for `@policy`).

#### 7.4.2 Repeatable-block merge

Repeatable-block facets MUST be merged by concatenating block instances in encounter order as they appear in the Resolved Source Form.

#### 7.4.3 Keyed list merge (Normative when enabled)

If a list-bearing map field declares `key="field"` as a facet attribute on its containing facet, list merge MUST be keyed:

- Each list item MUST be a map containing the key field; missing key → `F452`
- Items match by string equality of `item[field]`
- Matched items deep merge as maps; later overrides earlier
- Order preserves first appearance; new keys append in encounter order

### 7.5 Phase 1 output

Phase 1 outputs:

- Resolved Source Form (imports expanded, normalized)
- Resolved AST with deterministic block ordering and deterministic ordered-map key ordering

---

## 8. Facet Type System (FTS)

FTS is used for:

- `@vars` validation (`@var_types`)
- Lens signatures and pipeline type checking
- Interface schemas (`@interface`)
- Policy condition and rule type checking (§16.3)

### 8.1 Primitive types

- `string`, `int`, `float`, `bool`, `null`, `any`

Constraints:

- numbers: `min`, `max`
- strings: `pattern` (safe subset; see §9.9)
- `enum`: list of literal values

Violations:

- type mismatch → `F451`
- constraint violation → `F452`

### 8.2 Composite types

- `struct { field: T, ... }` (fields required by default)
- `list<T>`
- `map<string, T>`
- `T1 | T2 | ...` (union)

Optional fields MUST be expressed as `T | null`.

### 8.3 Multimodal types

- `image` with constraints: `format ∈ {png,jpeg,webp}`, `max_dim` (int)
- `audio` with constraints: `format ∈ {mp3,wav,ogg}`, `max_duration` (seconds, float)
- `embedding<size=N>` where `N` is a positive integer

### 8.4 Assignability (Normative)

`T1` is assignable to `T2` iff:

1. `T2 == any`, OR
2. `T1` and `T2` are the same primitive and satisfy constraints, OR
3. union: `T1` assignable to at least one member of `T2`, OR
4. list/map: element types assignable, OR
5. struct: all required fields of `T2` exist in `T1` and are assignable.

---

## 9. Lenses (Registry, Trust Levels, Gas, Determinism)

### 9.1 Lens registry (Normative)

Hypervisor implementations MUST maintain a lens registry. Each lens entry MUST include:

- `name` (string)
- `version` (string; MUST change if behavior changes)
- `input_type` (FTS)
- `output_type` (FTS)
- `trust_level ∈ {0,1,2}`
- deterministic gas function: `gas_cost(input, args) -> int`
- determinism class: `pure | bounded | volatile`

Additionally, for policy/guard support (§16.4–§16.6):

- `effect_class` (string) for `trust_level ∈ {1,2}` (see §16.5)
  - missing/invalid `effect_class` for Level‑1/2 MUST raise `F456` at registry construction time or first use

Unknown lens MUST raise `F802`.

### 9.2 Trust levels

- Level 0 — Pure: deterministic, no I/O
- Level 1 — Bounded: potentially external but governed by cache contract
- Level 2 — Volatile: nondeterministic and/or unbounded external effects

### 9.3 Pipeline type checking (Normative)

In Phase 2, the compiler MUST validate that each lens step accepts the previous output type (assignability). Violations MUST raise `F451`.

### 9.4 Gas model (Normative)

The host MUST define `GasLimit`. Every lens invocation consumes gas. If gas exceeds `GasLimit`, execution MUST raise `F902`.

### 9.5 Pure Mode policy (Normative)

In Pure Mode:

- Level 2 lenses MUST be rejected: `F801`
- Level 1 lenses MUST run in cache-only mode:
  - cache hit: allowed
  - cache miss: `F803`
- Level 0 lenses: allowed

### 9.6 Execution Mode policy

In Execution Mode, Level 1 and Level 2 lenses MAY execute if permitted by the host and permitted by `@policy` / guard enforcement (§16.6).

### 9.7 Cache key requirements (Normative)

Level 1 lenses MUST be cache-addressable by the contract in Appendix C.

### 9.8 Determinism of layout strategies (Normative)

Any lens pipeline used as a Layout `strategy` MUST be:

- Level‑0 only in Pure Mode (else `F801`)
- deterministic
- idempotent on identical input
- total over all valid NFC+LF `string` inputs (must not throw for any valid string)
- independent of locale, time, environment variables, filesystem, or network

### 9.9 Regex safety (Normative)

Any lens performing regex evaluation MUST use a linear-time safe engine (RE2-class) or a proven safe subset. Otherwise, the lens MUST NOT be registered as Level‑0.

---

## 10. Execution Model (Phases 1–5, Modes)

Hypervisor execution MUST follow these phases in order.

### 10.1 Phase 1 — Resolution

- Normalize input (§3)
- Parse to AST (§5, §6)
- Resolve imports (§7.1–§7.2)
- Apply merge rules (§7.3–§7.4)

Errors: `F001–F003`, `F601`, `F602`, `F402`

### 10.2 Phase 2 — Type Checking

- Validate `@var_types` and `@vars` expressions and pipelines
- Validate `@interface` and schema mappability
- Validate placement constraints (e.g. `@input` placement)
- Validate facet attributes (`when` type, etc.)
- Validate multimodal constraints
- Validate `@policy` schema and policy condition typing (§16.3)

Errors: `F451`, `F452`, `F456`, `F802`

AST MUST be treated as immutable after Phase 2 success.

### 10.3 Phase 3 — Reactive Compute (R‑DAG)

Inputs:

- typed Resolved AST
- runtime values for `@input` variables (Hypervisor)

Rules:

1. Build a dependency graph from `$var` references in `@vars`.
2. Unknown variable reference MUST raise `F401`.
3. Cycles MUST raise `F505`.
4. Evaluate in topological order.
   - Tie-break between independent nodes MUST follow the **merged ordered-map insertion order** of `@vars` as produced by §7.4.1 (first-insertion position preserved under overrides).
   - Overriding a variable's value via a later merged source MUST NOT change that variable's position in the ordered map. The position of first insertion is the canonical position used for tie-break traversal. Implementations that reorder keys upon override violate determinism.
5. Apply lenses (respect mode policy, gas, caching, and runtime guard/policy enforcement where applicable; §16.6).
6. Freeze computed variable map (immutable).
7. Materialize an **Effective Policy** object and compute `policy_hash` (§16.2.4, §18.3).

Errors: `F401`, `F405`, `F453`, `F505`, `F801`, `F802`, `F803`, `F902`, `F454`, `F455`

Forward references are allowed; only unknown references and cycles are errors.

### 10.4 Phase 4 — Layout (Token Box Model)

Input:

- computed messages and section metadata
- `@context` budget and defaults

Output: finalized ordered sections within budget

Errors: `F901`

### 10.5 Phase 5 — Render

Render MUST produce:

- Canonical JSON (§18)
- provider payload (host-defined) that preserves Canonical JSON semantics

If the host emits an Execution Artifact (Hypervisor run/test), it MUST conform to Appendix F.

---

## 11. Token Box Model (Context Algebra)

### 11.1 Normative budget unit: FACET Units

Layout MUST measure content in FACET Units:

$$
\text{facet\_units}(s) = \text{byte\_length}(\text{UTF‑8}(\text{NFC+LF normalized } s))
$$

Provider token counts MAY be reported as telemetry but MUST NOT affect normative layout.

### 11.2 Sections (Normative)

Each message block (`@system`, `@user`, `@assistant`) produces exactly one Layout section.

Each section has:

| Field | Type | Default |
|---|---|---|
| `id` | string | derived if omitted (§11.2.1) |
| `priority` | int | 500 |
| `min` | int | 0 |
| `grow` | float | 0 |
| `shrink` | float | 0 |
| `strategy` | lens pipeline | none |
| `content` | string | required |

A section is Critical iff `shrink == 0`. Critical sections MUST NOT be compressed, truncated, or dropped.

#### 11.2.1 Deterministic section id derivation (Normative)

If a message block does not specify `id`, the implementation MUST derive:

- Determine the message’s canonical role rank (`system=0`, `user=1`, `assistant=2`).
- Within each role, count occurrences in canonical message order starting at 1.
- Set `id = "<role>#<n>"` (example: `user#2`).

### 11.3 Deterministic packing algorithm (Normative)

Let `B` be budget in FACET Units and `size[i] = facet_units(content[i])`.

1) Critical load  
- `FixedLoad = sum(size[i] for critical sections)`  
- If `FixedLoad > B` → `F901`

2) If total fits  
- If `sum(size[i] for all sections) <= B`, keep all, preserving section order (§18.1.2).

3) Compress/drop flexible  
- Let `Flex = { i | shrink[i] > 0 }`
- Sort `Flex` by stable key:
  1. `priority` ascending
  2. `shrink` descending
  3. original section order ascending

Iterate `Flex` in that order while total size > B:

- If `strategy` is set: apply strategy to `content[i]` (Pure Mode: Level‑0 only; else `F801`)
- Recompute `size[i]` and total
- If still over budget: truncate deterministically from the end down to satisfy budget but not below `min`
  - truncation MUST NOT split UTF‑8 sequences
- If still over budget and `size[i] == min`: drop the entire section (unless Critical)

Result MUST be deterministic across implementations.

---

## 12. Standard Facets & Semantics

### 12.1 `@meta` (Normative minimal schema)

`@meta` is optional. If present, it MUST be a map whose values are atoms only:

- `string | number | bool | null`

`@meta` keys MUST be either:

- identifiers, or
- strings

If a `@meta` key is a string, it MUST NOT contain control characters (Unicode code points U+0000–U+001F and U+007F).

String keys are permitted only in `@meta`. Any string-keyed map entry outside `@meta` MUST raise `F452`.

`@meta` values MUST NOT contain `$` references, `@input`, or lens pipelines.

Host extensions SHOULD use string keys with a namespaced dotted form, for example:

```facet
@meta
  "x.acme.build_id": "..."
```

### 12.2 `@context` (Normative minimal schema)

`@context` defines layout configuration.

Minimum schema:

```facet
@context
  budget: 32000
  defaults:
    priority: 500
    min: 0
    grow: 0
    shrink: 0
```

Rules:

- `budget` MUST be an integer ≥ 0 measured in FACET Units.
- `defaults` MAY include any of: `priority|min|grow|shrink`.
- Missing values default to §11.2 defaults.

If `@context` is absent, the host MUST supply a budget and MUST surface it in canonical metadata.

If `@context` is absent and the host supplies a budget, that host-provided budget MUST be a deterministic function of the execution configuration. At minimum it MUST be stable for a fixed tuple:

- `(host_profile_id, facet_version, profile, mode, target_provider_id)`

### 12.3 Message facets: `@system`, `@user`, `@assistant` (Normative)

Each message block MUST be a map and MAY include:

- `content` (required)
- layout fields: `id|priority|min|grow|shrink|strategy`
- `when` (boolean gate)

`@system` MAY include:

- `tools`: list of interface references (`$InterfaceName`)

`when` gating is applied before policy-based message gating (`message_emit`) if implemented (§16.6.4).

### 12.4 Content forms (Normative)

A message `content` MUST be either:

- a `string`, or
- a list of content items, each of which is one of:
  - `{ type: "text", text: string }`
  - `{ type: "image", asset: <canonical asset> }`
  - `{ type: "audio", asset: <canonical asset> }`

### 12.5 Canonical asset model (Normative)

Canonical assets MUST be represented by semantic digest:

```json
{
  "kind": "image",
  "format": "jpeg",
  "digest": { "algo": "sha256", "value": "…" },
  "shape": { "width": 1024, "height": 768 }
}
```

or for audio:

```json
{
  "kind": "audio",
  "format": "wav",
  "digest": { "algo": "sha256", "value": "…" },
  "shape": { "duration": 3.2 }
}
```

Asset canonicalization is host-profile-defined. Therefore:

- Canonical JSON MUST include `metadata.host_profile_id`.
- `host_profile_id` MUST be stable and versioned.
- Any change that can alter semantic digests (codec pipeline, resampling, colorspace, normalization rules) MUST change `host_profile_id`.

### 12.6 Boolean gating (`when`) (Normative)

Facet attributes MAY include `when=<atom>` where atom is:

- `true|false`, or
- `$var` that evaluates to `bool`

If `when` evaluates to `false`, that message block MUST be omitted from layout and render.

Non-boolean `when` MUST raise `F451`.

---

## 13. Interfaces (`@interface`)

### 13.1 Syntax (Normative)

```facet
@interface WeatherAPI
  fn get_current(city: string) -> struct {
    temp: float
    condition: string
  } (effect="read")
```

Rules:

- Interface name MUST be an identifier.
- Function names MUST be unique within the interface.
- Parameter names MUST be unique within the function.
- Parameter and return types MUST be FTS types.
- Each function MUST declare an `effect` attribute (see §16.5) using the attribute atom set:
  - `effect` MUST be a string
  - Lens pipelines inside function attributes MUST raise `F003`
  - `@input(...)` MUST NOT appear in function attributes and MUST raise `F003`
- Missing `effect` MUST raise `F456`.

Duplicate interface names in the Resolved AST MUST raise `F452`.

### 13.2 Schema mappability (Normative)

All interface types MUST be mappable to JSON Schema per Appendix D. If not, MUST raise `F452`.

### 13.3 Tool reference from `@system`

```facet
@system
  tools: [$WeatherAPI]
  content: "..."
```

Unknown interface reference MUST raise `F452`.

---

## 14. Variables (`@vars`), Types (`@var_types`), Inputs (`@input`)

### 14.1 `@vars` (Normative)

`@vars` is a singleton ordered map of variable definitions.

Names MUST be unique after merge. If a variable name is overridden by a later merged value, it is not an error; the later value replaces the earlier value and the variable retains the order position of its first insertion (§7.4.1).

Hypervisor allows variable value expressions consisting of:

- literals, maps, lists
- `$` references
- lens pipelines
- `@input(...)` directive-expression, subject to §14.3

Core allows literals only and MUST reject any disallowed variable construct with `F801`.

### 14.2 `@var_types` (Normative)

`@var_types` is a singleton ordered map from variable name to FTS type expression.

If a variable has a declared type, its computed value MUST satisfy it; violations MUST raise `F451` or `F452`.

### 14.3 `@input(...)` directive-expression (Normative)

`@input(...)` is a directive-expression that denotes a runtime-supplied input value.

Placement:

- `@input(...)` MUST appear only as the base expression of a `@vars` entry value, optionally followed by a lens pipeline.
- `@input(...)` MUST NOT appear inside composite literals (not nested inside a list/map/struct literal).
- Any invalid placement MUST raise `F452`.

Attributes:

- `type` (REQUIRED): string containing an FTS type expression
- `default` (OPTIONAL): an atom (`string|number|bool|null`)

Examples:

```facet
@vars
  query: @input(type="string")
  n: @input(type="int", default=3)
  q: @input(type="string") |> trim()
```

Semantics:

- If `default` is present and the host does not supply an input value, the default MUST be used.
- If `default` is absent, the host MUST supply an input value.
- Supplied or defaulted values MUST validate against the `type`; violations MUST raise `F453`.
- Invalid FTS type strings MUST raise `F452`.

### 14.4 Evaluation semantics

Variables are computed in Phase 3 using R‑DAG rules (§10.3).

### 14.5 Unknown variables

Any reference to a missing variable MUST raise `F401`.

### 14.6 Cycles

Any dependency cycle MUST raise `F505`.

### 14.7 Input materialization

`@input` values are materialized during Phase 3 before dependent computations.

### 14.8 Variable path traversal (Normative)

For `$x.a.b`:

- `$x` must exist or `F401`
- each path segment on a map/struct must exist or `F405`
- list indexing is not standardized in v2.1.3; any numeric indexing MUST raise `F452` unless a namespaced host extension is enabled

---

## 15. Testing (`@test`) (Hypervisor Required)

### 15.1 Minimum syntax (Normative)

```facet
@test "basic"
  vars:
    username: "TestUser"
  input:
    query: "hello"
  mock:
    WeatherAPI.get_current: { temp: 10, condition: "Rain" }
  assert:
    - canonical.messages[0].role == "system"
    - canonical contains "hello"
    - telemetry.gas_used < 5000
```

### 15.2 Semantics (Normative)

- Each test runs in an isolated environment.
- `vars:` overrides variable values in the test environment (must still type-check).
- `input:` supplies `@input` values by variable name.
- `mock:` intercepts interface calls by fully-qualified tool function name.
- Assertions are evaluated against:
  - `canonical`: the Canonical JSON object produced by Phase 5
  - `telemetry`: host-defined telemetry fields
  - `execution` (RECOMMENDED): the Execution Artifact object (Appendix F) produced by the run/test engine, when applicable

In Pure Mode:

- Level‑1 cache misses MUST yield `F803`.
- Disallowed I/O MUST yield `F801`.
- Policy/guard failures MUST yield `F454` or `F455` as applicable (§16.6).

---

## 16. Policy / Authorization Model (`@policy`, Capability Classes, Runtime Guard)

### 16.1 Goals and determinism (Normative)

FACET policy is designed to be:

- **declarative** and **deterministic**
- **type-checkable** in Phase 2
- enforceable with a **runtime guard** in Hypervisor
- **fail-closed**: if a dangerous operation cannot be proven allowed, it MUST be rejected

Policy evaluation MUST be independent of locale, time, environment variables, filesystem, or network.

`policy_version` identifies the revision of the `@policy` DSL and guard evaluation semantics standardized in §16 and Appendix F. It covers: condition evaluation rules, rule match algorithm, effect class semantics, guard timing, and enforcement points. Any change to these semantics that is incompatible with existing Effective Policy Objects MUST result in a new `policy_version` value, even if `facet_version` does not change (e.g. in an errata or clarification release). `policy_version` is emitted in `metadata.policy_version` (§18.1) and included in `policy_hash` (§16.2.4) and hash-chain seed (§F.4).

### 16.2 `@policy` facet

#### 16.2.1 Cardinality and placement (Normative)

- `@policy` is a standard **singleton-map facet** (§7.3) and is deep-merged (§7.4.1) with additional list-merging rules (§16.2.3).
- `@policy` MAY be absent.
- `@policy` MUST NOT use string keys outside rules where allowed by general map key rules; string keys outside `@meta` are still forbidden (§12.1).

#### 16.2.2 Minimal schema (Normative)

`@policy` MUST be a map whose known top-level keys are:

- `defaults` (OPTIONAL): map
- `deny` (OPTIONAL): list of `PolicyRule`
- `allow` (OPTIONAL): list of `PolicyRule`

Unknown top-level keys MUST raise `F452`.

A `PolicyRule` MUST be a map with keys:

- `id` (OPTIONAL): string identifier (see §16.2.3)
- `op` (REQUIRED): string enum (§16.4.1)
- `name` (REQUIRED for `tool_*` and `lens_call`): string matcher (§16.2.5)
- `effect` (OPTIONAL): string effect matcher (§16.5.3)
- `when` (OPTIONAL): `PolicyCond`
- `unless` (OPTIONAL): `PolicyCond`

Unknown keys inside a `PolicyRule` MUST raise `F452`.

#### 16.2.3 Merge rules for `allow` / `deny` lists (Normative)

Within `@policy` only, if both earlier and later merged values contain `allow` and/or `deny` lists, implementations MUST merge them as follows:

- If a rule item has an `id` field:
  - it participates in a keyed merge with key `id` (string)
  - matched rules deep-merge as maps; later overrides earlier
  - ordering preserves first appearance; new ids append in encounter order
- If a rule item does not have `id`:
  - it MUST be appended in encounter order (no matching)

If `id` exists but is not a string, MUST raise `F452`.

#### 16.2.4 Policy hash (Normative)

The implementation MUST compute an **Effective Policy Object** after Phase 1 merging and after Phase 2 validation. The object is the fully merged `@policy` map with all lists in their effective merged order.

`metadata.policy_hash` MUST be:

- `null` if `@policy` is absent, else
- `"sha256:" + hex( sha256( JCS({ policy_version: <policy_version>, policy: EffectivePolicyObject }) ) )`

Where `policy_version` is the value emitted in `metadata.policy_version` (§18.1). Including `policy_version` in the hash ensures that semantically incompatible policy interpretations produce distinct hashes even when the source `@policy` text is identical.

#### 16.2.5 Name matchers (Normative)

For `PolicyRule.name`:

- If `name` contains no `*`, it MUST be an exact match string.
- If `name` ends with `.*`, it MUST match any string with that prefix before the `*`:
  - `PaymentAPI.*` matches `PaymentAPI.charge`, `PaymentAPI.refund`
- Any other use of `*` MUST raise `F452`.

#### 16.2.6 Canonical identifier form for name matching (Normative)

The `name` field in a `PolicyRule` and in `OpDesc.name` MUST use a **canonical identifier form**:

- The canonical form is **case-sensitive**.
- The canonical form MUST contain **no whitespace characters**.
- For `tool_*` operations: canonical form is `"<InterfaceName>.<fn_name>"` exactly as the identifiers are defined in the `@interface` AST node (§6, §13.1).
- For `lens_call`: canonical form is the lens registry `name` (§9.1), exactly as registered.
- For `message_emit`: canonical form is the derived or explicit section `id` (§11.2.1).

Implementations MUST normalize both the rule `name` and the runtime `OpDesc.name` to canonical form before comparison. A `PolicyRule.name` that contains whitespace or deviates from canonical form (e.g. wrong case) MUST raise `F452` at policy parse / Phase 2 validation time.

#### 16.2.7 PolicyRule field semantics — conjunctive filter (Normative)

All fields within a `PolicyRule` act as a **conjunctive (AND) filter** over the `OpDesc`. A rule matches an operation if and only if **all** present conditions are satisfied simultaneously:

- `op` matches `OpDesc.op` (always required)
- `name` matches `OpDesc.name` per §16.2.5–§16.2.6 (required for `tool_*` and `lens_call`)
- `effect` matches `OpDesc.effect_class` per §16.5.4 (only if `effect` field is present)
- `when` evaluates to `true` (only if `when` field is present)
- `unless` evaluates to `false` (only if `unless` field is present)

No field is an independent match. Implementations MUST NOT treat any single field as sufficient for a rule match unless all present fields are satisfied concurrently. A rule that matches on `name` but not on `effect` does **not** match.

### 16.3 Policy conditions (`PolicyCond`) (Normative)

`PolicyCond` is a deterministic boolean expression over computed variables (`@vars`) and MAY reference `$var` paths (§14.8). It MUST NOT perform I/O.

`PolicyCond` MUST be one of:

- boolean literal: `true` / `false`
- variable reference: `$name` or `$name.path` (result MUST be `bool`)
- map form:
  - `{ not: <PolicyCond> }`
  - `{ all: [<PolicyCond>, ...] }`
  - `{ any: [<PolicyCond>, ...] }`

Rules:

- `all` / `any` lists MUST be non-empty; empty list MUST raise `F452`.
- Any `VarRefNode` used in `PolicyCond` MUST resolve to an existing variable, else `F401`.
- Any invalid path segment MUST raise `F405`.
- The resulting type of every `PolicyCond` MUST be `bool`; violations MUST raise `F451`.
- `PolicyCond` MUST NOT contain lens pipelines or `@input(...)`; occurrences MUST raise `F452`.
  - Note: policy MAY still depend on runtime inputs indirectly via variables that use `@input` in `@vars`.

#### Short-circuit evaluation semantics (Normative)

`all` and `any` MUST use short-circuit (lazy) evaluation:

- `all`: evaluates arguments left-to-right; stops and returns `false` at the first argument that evaluates to `false`. Arguments after the stopping point MUST NOT be evaluated and MUST NOT generate errors (including `F401`, `F405`, or `F455`).
- `any`: evaluates arguments left-to-right; stops and returns `true` at the first argument that evaluates to `true`. Arguments after the stopping point MUST NOT be evaluated and MUST NOT generate errors.

This guarantees deterministic short-circuit behavior across all implementations. An implementation that evaluates arguments unconditionally and raises an error on an unevaluated branch violates this requirement.

### 16.4 Policy enforcement points and operation descriptors

#### 16.4.1 Standard operations (Normative)

The policy/guard model standardizes the following operation kinds:

- `tool_expose` — exposing a tool schema in `canonical.tools`
- `tool_call` — invoking an interface function at runtime
- `lens_call` — invoking a lens step at runtime (or during Phase 3 compute)
- `message_emit` — emitting an included message block into `canonical.messages`

A `PolicyRule.op` MUST be one of the strings above; otherwise `F452`.

#### 16.4.2 Operation descriptor (`OpDesc`) (Normative)

For every policy decision, the implementation MUST construct an `OpDesc` containing at least:

- `op` (string): operation kind
- `name` (string): fully qualified name
  - `tool_*`: `"<InterfaceName>.<fn_name>"`
  - `lens_call`: `"<lens_name>"`
  - `message_emit`: `"<role>#<n>"` (derived id; §11.2.1) or explicit id if present
- `effect_class` (string | null): see §16.5
- `mode` (`"pure"` or `"exec"`)
- `profile` (`"core"` or `"hypervisor"`)

`OpDesc` is an internal conceptual entity; its externally testable representation is `GuardDecision` (Appendix F).

#### 16.4.3 Definition: side-effecting operation (Normative)

A **side-effecting operation** is any operation whose `OpDesc.effect_class` is not `"read"`.

- An operation with `effect_class == null` MUST be treated as **side-effecting and unsafe** for fail-closed purposes. The guard MUST NOT assume a non-mutating default for null-class operations; absence of an effect class is not equivalent to `read`.
- An operation with `effect_class == "read"` is the only class that MAY be treated as non-mutating for guard classification purposes. All other non-null classes MUST be treated as potentially state-mutating.

This definition is a classification aid for host logic and informative descriptions. **Normative guard timing requirements in §16.6.1a are defined by operation kind, `trust_level`, and `mode` — not by this classification alone.** The classification is useful for reasoning about which operations require extra scrutiny, but guard timing is controlled by the explicit enumeration in §16.6.1a, not by this definition.

### 16.5 Capability / effect classes

#### 16.5.1 Standard effect classes (Normative)

FACET standardizes the following effect classes with normative semantics:

- `read` — observational only; MUST NOT mutate any persistent state observable outside the current execution. An operation declared `read` that actually mutates state is a host configuration error and does not reduce the host's security obligations.
- `write` — mutates persistent state; MUST be treated as at least as sensitive as `payment` for default-deny purposes.
- `external` — crosses the hermetic boundary to interact with an external system; may be read or write in nature; subsumes `network` unless a more specific class is declared.
- `payment` — initiates or authorizes a financial transaction; MUST be treated as the highest-sensitivity class by default policy. Implementations MUST NOT infer `payment` is equivalent to `write`; they are distinct and non-substitutable.
- `filesystem` — reads or writes filesystem paths outside the import sandbox; does not imply `network`.
- `network` — performs network I/O; does not imply `payment` or `filesystem`.

Implementations MUST NOT treat `read` as interchangeable with any mutating class (`write`, `payment`, `filesystem`, `network`). Effect classes carry disjoint semantic obligations and MUST be matched precisely.

Hosts MAY introduce additional effect classes but MUST namespace them:

- `x.<host>.<effect>`

#### 16.5.2 Effects for tools (Normative)

Every `@interface fn` MUST declare `effect` as a string in the standard or namespaced effect classes (§13.1, §16.5.1). Missing or invalid effect MUST raise `F456`.

The tool’s `effect_class` in `OpDesc` MUST be the declared `effect`.

#### 16.5.3 Effects for lenses (Normative)

Every lens registry entry with `trust_level ∈ {1,2}` MUST declare `effect_class` (§9.1). Missing or invalid effect MUST raise `F456`.

The lens’s `effect_class` in `OpDesc` MUST be the registry `effect_class`.

#### 16.5.4 Policy matching on effects (Normative)

If a `PolicyRule.effect` is present:

- it MUST be a string effect matcher:
  - exact match with no `*`, OR
  - suffix `.*` prefix match as in §16.2.5
- it MUST be matched against `OpDesc.effect_class`
- If `OpDesc.effect_class` is `null` and a rule requires an effect match, the rule MUST NOT match.

### 16.6 Runtime Guard and fail-closed execution (Hypervisor Normative)

#### 16.6.1 Guard requirement (Normative)

Hypervisor implementations MUST enforce a **Runtime Guard** over dangerous operations.

The guard MUST be invoked for:

- every `tool_call`
- every `lens_call` where `trust_level == 2`
- every `lens_call` where `trust_level == 1` in `exec` mode

Additionally:

- the host MAY guard `tool_expose` and/or `message_emit`; if implemented, it MUST be consistent with this section and MUST emit guard decisions (Appendix F).

#### 16.6.1a Guard evaluation timing (Normative)

The guard decision MUST be evaluated **before** any external initiation associated with the guarded operation.

**Permitted before ALLOW** — deterministic, Level-0, hermetic local computation needed to construct the `OpDesc` and `InputObject` (including `input_hash` per §F.3). This preparation MUST NOT perform any I/O, invoke any lens with `trust_level ≥ 1`, or initiate any external call.

**Prohibited before ALLOW:**

- external invocation of any `tool_call`, regardless of declared `effect_class`;
- lens execution for any `lens_call` with `trust_level ∈ {1, 2}` in `exec` mode;
- lens execution for any `lens_call` with `trust_level == 2` in `pure` mode;
- inclusion of a tool schema in `canonical.tools` for any `tool_expose`;
- inclusion of message content in the canonical message list for any `message_emit`, if the host guards that operation.

An operation with `effect_class == null` MUST require a guard decision before any prohibited action above.

An implementation that initiates any prohibited action before obtaining an explicit guard ALLOW violates fail-closed semantics and MUST raise `F455`.

#### 16.6.2 Decision algorithm (Normative)

For a given `OpDesc`, define `active(rule)`:

- `when` is absent OR `when` evaluates to `true`
- AND `unless` is absent OR `unless` evaluates to `false`

Match order:

1. Evaluate `deny` rules in encounter order; first matching active deny → DENY (policy violation).
2. Else evaluate `allow` rules in encounter order; first matching active allow → ALLOW.
3. Else apply defaults (§16.6.3).

If evaluation of a condition fails (e.g. type mismatch, missing variable) at runtime, the guard MUST treat this as **undecidable** and MUST deny with `F455` (fail-closed), unless the implementation can prove a deterministic, safe deny reason `F454` applies.

#### 16.6.3 Default policy (Normative)

Defaults apply when no allow/deny rule matches.

- For `tool_call`:
  - If `effect_class ∈ {write, payment, filesystem}`, default MUST be DENY.
  - Otherwise default MUST be DENY unless the host explicitly configures a deterministic, documented, and testable default allow for a subset of non-state-changing effect classes.
- For `lens_call` of Level‑1/2:
  - default MUST be DENY.
- For `tool_expose`:
  - default SHOULD be DENY.
- For `message_emit`:
  - default MAY be ALLOW (subject to `when`), unless configured otherwise.

Any host-defined default allow MUST be a deterministic function of execution configuration and MUST be testable via `@test`.

#### 16.6.4 Interaction with `when` gating (Normative)

- `when=false` MUST omit the block/tool reference before any policy `message_emit` or `tool_expose` decision is considered.
- `when=true` does not imply policy allow.

#### 16.6.5 Guard decision recording (Normative)

For every guarded operation attempt (including denied), Hypervisor MUST record a `GuardDecision` event as specified in Appendix F.

#### 16.6.6 Fail-closed errors (Normative)

`F454` and `F455` are strictly distinct and MUST NOT be substituted for each other:

- **`F454` — Policy deny**: the guard evaluated the Effective Policy successfully and reached a **deterministic DENY** decision via: a matching active deny rule, or a deterministic default deny (§16.6.3) with no matching allow rule. `F454` represents a reproducible, well-defined policy rejection. Implementations MUST emit `F454` whenever the deny outcome is deterministic.

- **`F455` — Guard undecidable**: the guard encountered an **evaluation failure** that prevented a deterministic outcome. Causes include:
  - a variable required by a `PolicyCond` is missing at runtime (`F401` condition)
  - a type error occurs during condition evaluation
  - `effect_class` is missing or invalid where required by the matched rule
  - the guard or rule-evaluation logic encounters an internal failure

`F455` MUST NOT be used if a deterministic deny decision (`F454`) can be made. Implementations MUST distinguish between the two cases: if the policy can be fully evaluated and the result is DENY, it is `F454`; if evaluation cannot complete, it is `F455`.

---

## 17. Security Model

### 17.1 Hermetic compilation boundary (Normative)

Phases 1–2 MUST be hermetic:

- no network access
- no filesystem access except allowed import roots
- no environment variable leakage
- deterministic execution

### 17.2 Import sandbox (Normative)

`@import` MUST be restricted:

- allowlisted roots
- no absolute paths
- no `..`
- no URLs

Violations MUST raise `F601`.

### 17.3 Runtime I/O restrictions (Normative)

Attempts to perform runtime I/O outside `@input` or registered Level‑1/2 lenses MUST raise `F801`.

Additionally, Hypervisor MUST apply runtime guard enforcement per §16.6. Any attempt to execute a guarded operation without a guard decision MUST be treated as fail-closed and MUST raise `F455`.

---

## 18. Canonical JSON, Canonicalization, Document Hash

### 18.1 Canonical JSON model (Normative)

Phase 5 MUST produce a provider-agnostic Canonical JSON object with:

- `metadata`:
  - `facet_version`: `"2.1.3"`
  - `profile`: `"core"` or `"hypervisor"`
  - `mode`: `"pure"` or `"exec"`
  - `host_profile_id`: string
  - `document_hash`: string
  - `policy_hash`: string | null
  - `policy_version`: string — version identifier for the `@policy` DSL and guard evaluation semantics in use, independent of `facet_version`. MUST be `"1"` for v2.1.3. MUST be incremented by the FACET standard if policy semantics change in a future version.
  - `budget_units`: int (effective budget)
  - `target_provider_id`: string
- `tools`: list of canonical tool schemas derived from interfaces
- `messages`: ordered list of:
  - `{ "role": "system"|"user"|"assistant", "content": string | list }`

Canonical JSON represents the request context. It does not include the model’s runtime output.

#### 18.1.1 Tools ordering (Normative)

`tools` list order MUST follow the Resolved Source Form order of `@interface` declarations.

#### 18.1.2 Canonical message ordering (Normative)

Canonical message order MUST be deterministic:

1. All `@system` message blocks in their Resolved Source Form order
2. Then all `@user` message blocks in their Resolved Source Form order
3. Then all `@assistant` message blocks in their Resolved Source Form order

Within each role, relative order MUST be preserved.

Layout (§11) MUST preserve this canonical message ordering when truncating/dropping sections; it MUST NOT reorder messages.

#### 18.1.3 Tool exposure and policy (Normative)

If `@system.tools` is present in any included `@system` blocks, `canonical.tools` MUST be the union of referenced interfaces, ordered by §18.1.1.

Hypervisor MUST apply policy `tool_expose` filtering before emitting `canonical.tools`. Denied tools MUST be omitted rather than emitted with a marker, and a `GuardDecision` with `decision: "denied"` MUST be recorded (Appendix F, §16.6.5).

Core MAY apply `tool_expose` policy filtering; if implemented, it MUST follow the same omission semantics.

### 18.2 Canonical JSON serialization (Normative)

Canonical JSON MUST be serialized using JSON Canonicalization Scheme (RFC 8785) or equivalent:

- UTF‑8
- stable object key ordering
- stable number formatting

### 18.3 Document hash (Normative)

`metadata.document_hash` MUST be:

- `sha256` of the Resolved Source Form (imports expanded, NFC+LF normalized)

The host MAY additionally publish a hash of Canonical JSON for debugging, but the normative document hash is the Resolved Source Form hash.

---

## 19. Error Code Catalog (Normative)

### 19.1 Syntax & parse (000–099)

- `F001` invalid indentation (must be 2 spaces)
- `F002` tabs forbidden
- `F003` malformed syntax / invalid token / invalid escape / unclosed structure
- `F402` attribute interpolation forbidden

### 19.2 Semantic & type (400–499)

- `F401` variable not found
- `F405` invalid variable path (missing field)
- `F451` type mismatch
- `F452` constraint violation / unsupported construct / invalid placement / invalid signature
- `F453` runtime input validation failed (`@input`)
- `F454` policy violation (denied by policy or default policy deny where decision is well-defined)
- `F455` guard undecidable / missing evidence (fail-closed)
- `F456` effect not declared / invalid effect class

### 19.3 Graph (500–599)

- `F505` cyclic dependency detected (R‑DAG)

### 19.4 Imports (600–699)

- `F601` import not found / disallowed path
- `F602` import cycle

### 19.5 Runtime/security/layout (800–999)

- `F801` I/O prohibited / lens disallowed in mode/profile
- `F802` unknown lens
- `F803` Pure cache miss (Level‑1 lens attempted without cache hit)
- `F901` critical overflow (Token Box Model)
- `F902` compute gas exhausted

Host extension diagnostics MUST use the `X.<host>.<code>` namespace (§2.2).

---

## 20. Reference CLI (`fct`) (Recommended)

A standard implementation SHOULD provide:

- `fct build file.facet` (Phases 1–2)
- `fct run file.facet --input input.json --pure|--exec`
- `fct test file.facet`
- `fct inspect file.facet --ast ast.json --dag dag.json --layout layout.json --policy policy.json`
- `fct codegen file.facet --lang python|ts`

If an Execution Artifact is produced during `run` or `test`, it SHOULD be emitted as `execution.json` and MUST conform to Appendix F if present.

---

## 21. Change History

### v2.1.3 (rev. 2026-02-19 — targeted normative clarifications)

Normative additions within v2.1.3 to close formal gaps identified post-publication:

- **§16.2.6** — Added canonical identifier form requirement for `PolicyRule.name` and `OpDesc.name`: case-sensitive, no whitespace, `<InterfaceName>.<fn>` exactly as in AST; non-canonical form MUST raise `F452` at Phase 2.
- **§16.2.7** — Formalized that all `PolicyRule` fields are a conjunctive (AND) filter; a rule matches only when all present fields are satisfied simultaneously.
- **§16.3** — Added short-circuit evaluation semantics for `all`/`any`: left-to-right, stops at first false/true; arguments after stopping point MUST NOT be evaluated and MUST NOT raise errors.
- **§16.5.1** — Added normative semantic constraints to standard effect classes: `read` MUST NOT mutate state; `payment` is distinct from `write`; classes are non-interchangeable and non-substitutable.
- **§16.6.1a** — Added guard evaluation timing: guard decision MUST be made before any associated side-effecting or non-Level-0 computation begins.
- **§16.6.6** — Sharpened `F454` vs `F455` distinction: `F454` = deterministic deny (policy evaluated successfully); `F455` = evaluation failure preventing a deterministic outcome. `F455` MUST NOT be used when `F454` is applicable.
- **§18.1.3** — Promoted Hypervisor `tool_expose` policy filtering from SHOULD to MUST; denied tools MUST be omitted and a denied `GuardDecision` MUST be recorded.
- **Appendix F.4** — Added `profile` and `mode` to the H0 hash-chain seed to prevent collisions across execution configurations.
- **Appendix E** — Updated conformance checklist items 15, 17, 18, 19 to reflect above additions.
- **§10.3** — Clarified R-DAG tie-break: independent nodes MUST follow merged ordered-map insertion order; override MUST NOT change a variable's position.
- **§16.4.3** (new) — Formal definition of "side-effecting operation": any operation with `effect_class ≠ "read"`; `effect_class == null` treated as unsafe/side-effecting.
- **§16.6.1a** — Replaced ambiguous "side-effecting or non-Level-0" with a precise enumeration of guarded operations and explicit reference to §16.4.3.
- **§16.2.4** — `policy_hash` computation now wraps `{ policy_version, policy: EffectivePolicyObject }` to bind hash to DSL semantics version.
- **§18.1** — Added `policy_version` field to Canonical JSON metadata (MUST be `"1"` for v2.1.3).
- **§F.2** — Added `policy_version` to Execution Artifact metadata.
- **§F.4** — Added `policy_version` to H0 hash-chain seed.

### v2.1.3 (original)

- Added Policy / Authorization Model with standard `@policy` facet, deterministic condition DSL, and enforcement points (`tool_expose`, `tool_call`, `lens_call`, `message_emit`)
- Added capability/effect classes for interface functions (`effect`) and for Level‑1/2 lenses (`effect_class`)
- Added mandatory Hypervisor Runtime Guard with fail-closed semantics and new error codes `F454–F456`
- Added `metadata.policy_hash` to Canonical JSON and standardized policy hashing
- Added Execution Artifact specification (provenance events, guard decisions, hash-chain, optional attestation) in Appendix F
- Updated conformance checklist accordingly

### v2.1.2

- Defined `@input(...)` as a directive-expression, permitted only as a `@vars` base expression (optionally piped), and added ABNF coverage
- Allowed `@meta` keys to be identifiers or strings; string keys forbidden outside `@meta`
- Fixed Canonical JSON ordering requirements for `tools` and `messages`

### v2.1.1

- Reserved `F000–F999` for FACET only; host diagnostics must use `X.<host>.<code>`
- Required full-syntax parsing across profiles; disallowed constructs MUST raise `F801`
- Tightened `@meta` values to atoms only and forbade compute constructs
- Strengthened layout strategy requirements (total, locale/time/env independent)
- Required stability of host-provided budget by execution configuration tuple
- Fixed import error preference: sandbox violations MUST raise `F601`

---

## Appendix A — Standard Lens Library (Normative)

Hypervisor implementations MUST provide these Level‑0 lenses:

### A.1 Text

- `trim() -> string`
- `lowercase() -> string` (locale-independent)
- `uppercase() -> string` (locale-independent)
- `split(separator: string) -> list<string>`
- `replace(pattern: string, replacement: string) -> string` (safe regex subset)
- `indent(level: int) -> string` (2 spaces × level)

### A.2 Data

- `json(indent: int = 0) -> string`
- `keys() -> list<string>`
- `values() -> list<any>`
- `map(field: string) -> list<any>`
- `sort_by(field: string, desc: bool = false) -> list<any>`
- `default(value: any) -> any`
- `ensure_list() -> list<any>`

---

## Appendix B — ABNF Grammar (Informative)

This ABNF describes the normalized NFC+LF source form. Newlines are LF (`%x0A`).

Note: This ABNF covers the generic YAML-lite expression syntax used in most facets. Certain facets define specialized internal line grammars (e.g. `@interface` and `@test`) that are specified in their respective sections and are not fully captured here.

```abnf
FACET-DOC   = *(WSLINE / TOP)
WSLINE      = *(SP / COMMENT) NL
COMMENT     = "#" *(%x20-10FFFF) NL

TOP         = IMPORT / FACET
IMPORT      = "@import" SP STRING NL

FACET       = "@" IDENT [ATTRS] NL BODY
ATTRS       = "(" [ATTR *( "," *SP ATTR)] ")"
ATTR        = IDENT "=" ATTR-ATOM
ATTR-ATOM   = STRING / NUMBER / BOOL / NULL / VARREF

BODY        = 1*(IND LINE)
LINE        = (KV / LISTITEM) NL

KV          = KEY ":" *SP VALUE
KEY         = IDENT / STRING

LISTITEM    = "-" SP VALUE

VALUE       = ATOM *(SP "|>" SP LENS-CALL)

ATOM        = STRING / NUMBER / BOOL / NULL / VARREF / INLINE-LIST / INLINE-MAP / INPUT-DIR
INPUT-DIR   = "@input" [ATTRS]

LENS-CALL   = IDENT "(" [ARGS] ")"
ARGS        = ARG *( "," *SP ARG )
ARG         = [IDENT "="] VALUE

VARREF      = "$" IDENT *("." IDENT)

INLINE-LIST = "[" *SP [VALUE *(*SP "," *SP VALUE)] *SP "]"
INLINE-MAP  = "{" *SP [KEY ":" *SP VALUE *(*SP "," *SP KEY ":" *SP VALUE)] *SP "}"

STRING      = DQUOTE *CHAR DQUOTE
CHAR        = ESC / %x20-21 / %x23-5B / %x5D-10FFFF
ESC         = "\" ( DQUOTE / "\" / "n" / "t" / "r" / "u" 4HEXDIG )

NUMBER      = ["-"] 1*DIGIT ["." 1*DIGIT] [("e"/"E") ["-"/"+"] 1*DIGIT]
BOOL        = "true" / "false"
NULL        = "null"

IDENT       = ALPHA *(ALPHA / DIGIT / "_")

SP          = %x20
NL          = %x0A
IND         = SP SP
DQUOTE      = %x22
```

---

## Appendix C — Cache Key & Pure Cache‑Only Contract (Normative)

For Level‑1 lenses:

`CacheKey = sha256( JCS({
  lens: { name, version },
  input: canonical_input_value,
  args: canonical_args,
  named_args: canonical_named_args,
  host_profile_id,
  facet_version: "2.1.3"
}))`

Where:

- `canonical_*` values are RFC 8785 canonical JSON encodings
- multimodal values MUST include semantic digest and declared constraints

Pure Mode rule:

- Level‑1 MUST NOT perform network calls; cache hit only, else `F803`.

---

## Appendix D — FTS → JSON Schema Mapping (Normative)

### D.1 Primitives

- `string` → `{ "type": "string" }`
- `int` → `{ "type": "integer" }`
- `float` → `{ "type": "number" }`
- `bool` → `{ "type": "boolean" }`
- `null` → `{ "type": "null" }`
- `any` → `{}`

Constraints:

- `min/max` → `minimum/maximum`
- `pattern` → `pattern`
- `enum` → `enum`

### D.2 Struct

`struct { a: T1, b: T2 }` →

```json
{
  "type": "object",
  "properties": { "a": <T1>, "b": <T2> },
  "required": ["a", "b"],
  "additionalProperties": false
}
```

Optional fields MUST be expressed as `T | null` and map to `oneOf` including `{ "type": "null" }`.

### D.3 List / Map

- `list<T>` → `{ "type": "array", "items": <T> }`
- `map<string,T>` → `{ "type": "object", "additionalProperties": <T> }`

### D.4 Union

`T1 | T2` → `{ "oneOf": [<T1>, <T2>] }`

### D.5 Embedding

`embedding<size=N>` →

```json
{
  "type": "array",
  "items": { "type": "number" },
  "minItems": N,
  "maxItems": N
}
```

---

## Appendix E — Conformance Checklist (Normative)

A Hypervisor implementation MUST implement:

1. Normalization: UTF‑8 validation, NFC normalization, LF normalization (§3)  
2. Parsing: 2-space indentation, inline list/map, pipelines, attributes restrictions, directive-expression parsing (§5, App B)  
3. AST: required node classes and normalized spans; ordered-map preservation (§6)  
4. Imports: allowlisted roots, forbid absolute/`..`/URL, detect cycles, sandbox violations as `F601` (§7, §17.2)  
5. Merge: deterministic singleton-map merge with stable key positions; deterministic repeatable-block collection (§7.3–§7.4)  
6. FTS: primitives/composites/unions/multimodal, assignability, constraints (§8)  
7. Lens registry: name/version/types/trust/gas/determinism; effect_class for Level‑1/2; unknown lens error (§9, §16.5)  
8. Gas: enforcement and `F902` (§9.4)  
9. Modes: Pure vs Exec enforcement, including `F803` cache-only for Level‑1 (§9.5, App C)  
10. R‑DAG: dependency analysis, topo evaluation, tie-break by merged ordered-map insertion order per §7.4.1 (override MUST NOT change key position), cycle detection (§10.3)
11. Token Box Model: FACET Units, deterministic ordering, truncation rules, `F901` (§11)  
12. `@context`: budget + defaults schema, stability rule for host-provided budget (§12.2)  
13. `@meta`: atoms-only values; identifier-or-string keys; control-char restriction; string keys forbidden outside `@meta` (§12.1)  
14. `@input`: directive-expression placement and semantics; `F453` validation failures (§14.3)  
15. Canonical JSON: model, ordering rules for tools/messages, RFC 8785 canonicalization, resolved-source document hash, policy_hash; mandatory emission of `policy_version` in metadata; Hypervisor MUST apply `tool_expose` guard filtering before emitting `canonical.tools` (§18)

16. Interfaces: syntax + JSON Schema mapping conformance; mandatory fn effect attribute (§13, §16.5)  
17. Policy: `@policy` parsing, merge rules for allow/deny, condition typing, deterministic evaluation; conjunctive filter semantics per rule; canonical identifier form for name matching; short-circuit evaluation of `all`/`any`; `policy_version` included in `policy_hash` computation; formal definition of side-effecting operations (§16.2–§16.4.3)
18. Runtime Guard: enforcement points, guard timing (before any associated computation), precise enumeration of guarded operations, fail-closed semantics, policy decision algorithm, `F454` (deterministic deny) vs `F455` (undecidable) distinction; `effect_class == null` treated as side-effecting (§16.6)
19. Provenance: Execution Artifact emission with guard decisions and hash-chain; `profile`, `mode`, and `policy_version` included in H0 seed; `policy_version` in Execution Artifact metadata (Appendix F)
20. Tests: minimal `@test` execution semantics; policy/guard interactions in tests (§15)  
21. Errors: emit standard codes per §19; host diagnostics namespaced (§2.2)  

A Core implementation MUST implement items (1–5, 12, 13 as parsing+render-only, 15, 16 as “not supported” with `F801` on use of `@interface`, 17 as type-check+policy_hash emission only, 21) and MUST reject Hypervisor-only constructs with `F801`.

---

## Appendix F — Execution Artifact, Guard Decisions, Attestation (Normative)

### F.1 Overview

Hypervisor `run` and `test` environments SHOULD emit an **Execution Artifact**. If emitted, it MUST be deterministic (given identical inputs, caches, and execution configuration) and MUST be serialized using RFC 8785 (JCS).

The Execution Artifact is distinct from Canonical JSON:

- Canonical JSON is the provider-agnostic request context (§18).
- Execution Artifact is an audit/provenance record of guarded operations and their decisions.

### F.2 Execution Artifact schema (Normative)

An Execution Artifact MUST be a JSON object with:

- `metadata` (object):
  - `facet_version`: `"2.1.3"`
  - `host_profile_id`: string
  - `document_hash`: string (same as `canonical.metadata.document_hash`)
  - `policy_hash`: string | null (same as `canonical.metadata.policy_hash`)
  - `policy_version`: string (same as `canonical.metadata.policy_version`)
- `provenance` (object):
  - `events`: list of `GuardDecision` objects in increasing `seq`
  - `hash_chain`: object describing a hash chain over events
- `attestation` (object | null): optional signature envelope (§F.5)

### F.3 GuardDecision event (Normative)

A `GuardDecision` MUST be a JSON object with:

- `seq` (int): starts at 1, increments by 1 with no gaps
- `op` (string): one of `tool_call | lens_call | tool_expose | message_emit`
- `name` (string): operation name (see §16.4.2)
- `effect_class` (string | null)
- `mode` (string): `"pure"` or `"exec"`
- `decision` (string): `"allowed"` or `"denied"`
- `policy_rule_id` (string | null): `id` of the first matching rule that determined the decision, else `null`
- `input_hash` (string): `"sha256:" + hex(sha256(JCS(InputObject)))`

Where `InputObject` MUST be:

- For `tool_call`:
  - `{ interface: <InterfaceName>, fn: <fn_name>, args: <canonical_args_object>, host_profile_id, facet_version: "2.1.3" }`
- For `lens_call`:
  - `{ lens: { name, version }, input: <canonical_input_value>, args: <canonical_args>, named_args: <canonical_named_args>, host_profile_id, facet_version: "2.1.3" }`
  - Note: this aligns with Appendix C’s cache key contract but is distinct (event input hash does not need to equal cache key).
- For `tool_expose`:
  - `{ interface: <InterfaceName>, host_profile_id, facet_version: "2.1.3" }`
- For `message_emit`:
  - `{ message_id: <id>, role: <role>, host_profile_id, facet_version: "2.1.3" }`

Canonical values MUST follow RFC 8785 (JCS).

### F.4 Hash chain (Normative)

`provenance.hash_chain` MUST be:

```json
{
  "algo": "sha256",
  "head": "sha256:..."
}
```

The chain MUST be computed as:

- `H0 = sha256( JCS({ facet_version: "2.1.3", host_profile_id, document_hash, policy_hash, policy_version, profile, mode }) )`

  Where `profile` is `"core"` or `"hypervisor"` and `mode` is `"pure"` or `"exec"`, matching `canonical.metadata.profile` and `canonical.metadata.mode`; `policy_version` matches `canonical.metadata.policy_version`. Including all execution configuration fields in the seed prevents hash-chain collisions across configurations and policy semantic versions.
- For each event `Ei` with `seq=i`:
  - `Hi = sha256( JCS({ prev: "sha256:" + hex(H{i-1}), event: Ei }) )`
- `head = "sha256:" + hex(Hn)` where `n = len(events)`

### F.5 Attestation (Optional, Normative if present)

If `attestation` is non-null, it MUST be an object with:

- `algo` (string): `"ed25519"` or a namespaced algorithm `"x.<host>.<algo>"`
- `key_id` (string): host-defined key identifier
- `sig` (string): base64url signature over the bytes of `provenance.hash_chain.head` string (UTF‑8)

The host MUST document the public key discovery mechanism for `key_id` out of band. FACET does not standardize key distribution.

### F.6 Test visibility (Normative)

If `execution` is exposed to `@test` assertions (§15.2), it MUST be the exact Execution Artifact object emitted by the engine (or a strict subset that preserves `metadata`, `provenance.events`, and `provenance.hash_chain`).