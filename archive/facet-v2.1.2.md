

### FACET v2.1.2 — Production Language Specification

**Status:** Recommendation (REC‑PROD)  
**Version:** 2.1.2  
**Date:** 2025‑12‑23  
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
16. Security Model  
17. Canonical JSON, Canonicalization, Document Hash  
18. Error Code Catalog (Normative)  
19. Reference CLI (`fct`) (Recommended)  
20. Change History  
Appendix A — Standard Lens Library (Normative)  
Appendix B — ABNF Grammar (Informative)  
Appendix C — Cache Key & Pure Cache‑Only Contract (Normative)  
Appendix D — FTS → JSON Schema Mapping (Normative)  
Appendix E — Conformance Checklist (Normative)

---

## 1. Scope

FACET is a Neural Architecture Description Language (NADL) for defining, validating, and executing AI request construction in a deterministic, type-safe, resource-bounded manner.

FACET standardizes:

- Concrete syntax and normalized source form
- Deterministic import resolution and deterministic merge behavior
- A strict type system (FTS) for variables, lenses, and interfaces
- Reactive computation of variables via a dependency graph (R‑DAG)
- Deterministic context packing via a provider-independent layout model
- Provider-agnostic Canonical JSON output
- Deterministic behavior in Pure Mode
- A security boundary for hermetic compilation and constrained runtime I/O
- A minimal conformance testing facility

FACET does not standardize model inference behavior, provider request/response protocols, or model internals beyond the requirement that provider payloads preserve Canonical JSON semantics.

---

## 2. Conformance, Profiles, Extensions

This document uses RFC 2119 keywords (**MUST**, **MUST NOT**, **SHOULD**, **MAY**).

An implementation is **FACET v2.1.2 compliant** if and only if it satisfies all normative requirements in this specification for its declared profile(s) and mode(s).

### 2.1 Profiles

#### 2.1.1 Core Profile (Minimal)

Core implementations MUST support:

- Phase 1 (Resolution), Phase 2 (Type Checking), Phase 5 (Render)
- `@meta`, `@import`, `@context`, `@system`, `@user`, `@assistant`, `@vars`, `@var_types`
- Inline list/map literals
- Parsing of the full FACET concrete syntax (§5), followed by profile enforcement

Core restrictions:

- `@vars` values MUST be literals only (no `$` references, no lens pipelines, no `@input`)
- No `@interface`, no `@test`, no R‑DAG evaluation, no Token Box Model, no multimodal canonicalization

Core MUST reject any syntactically valid construct disallowed by Core using `F801` (not `F003`).

#### 2.1.2 Hypervisor Profile (Full)

Hypervisor implementations MUST support all features described in this specification, including:

- All phases 1–5
- Token Box Model
- Multimodal canonicalization contract
- `@interface`, `@input`, R‑DAG, `@test`
- Lens registry, gas accounting, and Pure cache-only contract

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
2. Imports MUST be constrained by allowlisted roots (§16.2).
3. Violations of import sandbox rules (absolute paths, `..` traversal, URL imports, or outside allowlisted roots) MUST raise `F601`.
4. Import not found MUST raise `F601`.
5. Import cycles MUST raise `F602`.

### 7.2 Deterministic resolution order (Normative)

Imports MUST be applied in source order: imported content is expanded in-place at the point the `@import` appears, forming a single Resolved Source Form and a single Resolved AST.

### 7.3 Standard facet cardinality (Normative)

Standard facets have fixed cardinality:

Singleton-map facets (deep-merged by key):

- `@meta`, `@context`, `@vars`, `@var_types`

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
- Lists are not deep-merged unless the keyed-list rule applies (§7.4.3).

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

In Execution Mode, Level 1 and Level 2 lenses MAY execute if permitted by the host.

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

Errors: `F451`, `F452`, `F802`

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
   - Tie-break between independent nodes MUST be stable variable key order in the merged ordered map of `@vars` (§7.4.1).
5. Apply lenses (respect mode policy, gas, and caching).
6. Freeze computed variable map (immutable).

Errors: `F401`, `F405`, `F453`, `F505`, `F801`, `F802`, `F803`, `F902`

Forward references are allowed; only unknown references and cycles are errors.

### 10.4 Phase 4 — Layout (Token Box Model)

Input:

- computed messages and section metadata
- `@context` budget and defaults

Output: finalized ordered sections within budget

Errors: `F901`

### 10.5 Phase 5 — Render

Render MUST produce:

- Canonical JSON (§17)
- provider payload (host-defined) that preserves Canonical JSON semantics

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
- If `sum(size[i] for all sections) <= B`, keep all, preserving section order (§17.1.2).

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
  }
```

Rules:

- Interface name MUST be an identifier.
- Function names MUST be unique within the interface.
- Parameter names MUST be unique within the function.
- Parameter and return types MUST be FTS types.

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
- list indexing is not standardized in v2.1.2; any numeric indexing MUST raise `F452` unless a namespaced host extension is enabled

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

In Pure Mode:

- Level‑1 cache misses MUST yield `F803`.
- Disallowed I/O MUST yield `F801`.

---

## 16. Security Model

### 16.1 Hermetic compilation boundary (Normative)

Phases 1–2 MUST be hermetic:

- no network access
- no filesystem access except allowed import roots
- no environment variable leakage
- deterministic execution

### 16.2 Import sandbox (Normative)

`@import` MUST be restricted:

- allowlisted roots
- no absolute paths
- no `..`
- no URLs

Violations MUST raise `F601`.

### 16.3 Runtime I/O restrictions (Normative)

Attempts to perform runtime I/O outside `@input` or registered Level‑1/2 lenses MUST raise `F801`.

---

## 17. Canonical JSON, Canonicalization, Document Hash

### 17.1 Canonical JSON model (Normative)

Phase 5 MUST produce a provider-agnostic Canonical JSON object with:

- `metadata`:
  - `facet_version`: `"2.1.2"`
  - `profile`: `"core"` or `"hypervisor"`
  - `mode`: `"pure"` or `"exec"`
  - `host_profile_id`: string
  - `document_hash`: string
  - `budget_units`: int (effective budget)
  - `target_provider_id`: string
- `tools`: list of canonical tool schemas derived from interfaces
- `messages`: ordered list of:
  - `{ "role": "system"|"user"|"assistant", "content": string | list }`

Canonical JSON represents the request context. It does not include the model’s runtime output.

#### 17.1.1 Tools ordering (Normative)

`tools` list order MUST follow the Resolved Source Form order of `@interface` declarations.

#### 17.1.2 Canonical message ordering (Normative)

Canonical message order MUST be deterministic:

1. All `@system` message blocks in their Resolved Source Form order
2. Then all `@user` message blocks in their Resolved Source Form order
3. Then all `@assistant` message blocks in their Resolved Source Form order

Within each role, relative order MUST be preserved.

Layout (§11) MUST preserve this canonical message ordering when truncating/dropping sections; it MUST NOT reorder messages.

### 17.2 Canonical JSON serialization (Normative)

Canonical JSON MUST be serialized using JSON Canonicalization Scheme (RFC 8785) or equivalent:

- UTF‑8
- stable object key ordering
- stable number formatting

### 17.3 Document hash (Normative)

`metadata.document_hash` MUST be:

- `sha256` of the Resolved Source Form (imports expanded, NFC+LF normalized)

The host MAY additionally publish a hash of Canonical JSON for debugging, but the normative document hash is the Resolved Source Form hash.

---

## 18. Error Code Catalog (Normative)

### 18.1 Syntax & parse (000–099)

- `F001` invalid indentation (must be 2 spaces)
- `F002` tabs forbidden
- `F003` malformed syntax / invalid token / invalid escape / unclosed structure
- `F402` attribute interpolation forbidden

### 18.2 Semantic & type (400–499)

- `F401` variable not found
- `F405` invalid variable path (missing field)
- `F451` type mismatch
- `F452` constraint violation / unsupported construct / invalid placement / invalid signature
- `F453` runtime input validation failed (`@input`)

### 18.3 Graph (500–599)

- `F505` cyclic dependency detected (R‑DAG)

### 18.4 Imports (600–699)

- `F601` import not found / disallowed path
- `F602` import cycle

### 18.5 Runtime/security/layout (800–999)

- `F801` I/O prohibited / lens disallowed in mode/profile
- `F802` unknown lens
- `F803` Pure cache miss (Level‑1 lens attempted without cache hit)
- `F901` critical overflow (Token Box Model)
- `F902` compute gas exhausted

Host extension diagnostics MUST use the `X.<host>.<code>` namespace (§2.2).

---

## 19. Reference CLI (`fct`) (Recommended)

A standard implementation SHOULD provide:

- `fct build file.facet` (Phases 1–2)
- `fct run file.facet --input input.json --pure|--exec`
- `fct test file.facet`
- `fct inspect file.facet --ast ast.json --dag dag.json --layout layout.json`
- `fct codegen file.facet --lang python|ts`

---

## 20. Change History

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
  facet_version: "2.1.2"
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
4. Imports: allowlisted roots, forbid absolute/`..`/URL, detect cycles, sandbox violations as `F601` (§7, §16.2)  
5. Merge: deterministic singleton-map merge with stable key positions; deterministic repeatable-block collection (§7.3–§7.4)  
6. FTS: primitives/composites/unions/multimodal, assignability, constraints (§8)  
7. Lens registry: name/version/types/trust/gas; unknown lens error (§9)  
8. Gas: enforcement and `F902` (§9.4)  
9. Modes: Pure vs Exec enforcement, including `F803` cache-only for Level‑1 (§9.5, App C)  
10. R‑DAG: dependency analysis, topo evaluation, stable tie-break by ordered-map key order, cycle detection (§10.3)  
11. Token Box Model: FACET Units, deterministic ordering, truncation rules, `F901` (§11)  
12. `@context`: budget + defaults schema, stability rule for host-provided budget (§12.2)  
13. `@meta`: atoms-only values; identifier-or-string keys; control-char restriction; string keys forbidden outside `@meta` (§12.1)  
14. `@input`: directive-expression placement and semantics; `F453` validation failures (§14.3)  
15. Canonical JSON: model, ordering rules for tools/messages, RFC 8785 canonicalization, resolved-source document hash (§17)  
16. Interfaces: syntax + JSON Schema mapping conformance (Appendix D) (§13)  
17. Tests: minimal `@test` execution semantics (§15)  
18. Errors: emit standard codes per §18; host diagnostics namespaced (§2.2)  

A Core implementation MUST implement items (1–5, 11 as “not supported” with `F801` on use, 12, 13 as parsing+render-only, 15, 18) and MUST reject Hypervisor-only constructs with `F801`.