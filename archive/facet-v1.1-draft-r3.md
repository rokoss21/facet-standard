# FACET Language Specification v1.1 (Draft r3)

**Status:** Draft v1.1 (r3)
**Author:** Emil Rokossovskiy <ecsiar@gmail.com> — [@rokoss21](https://github.com/rokoss21/)
**Date:** 2025-09-13
**MIME:** `application/facet`
**File Extension:** `.facet`

---

## Editorial & Normative Updates in v1.1 (r3)

- **E-002 (ABNF additions):** `@import`, `if` attribute, `template_ref` (`{{path}}`), and tokens for conditional expressions.
- **E-003 (ABNF corrections):** Fixed `facet` and `list` productions; removed stray leading spaces; introduced `item_if` to restrict list-item attributes to `if` only.
- **N-004 (Built-in Variables):** Added the `@vars` facet and `{{name}}` interpolation in strings; expanded §7.
- **N-005 (Imports):** Added `@import` directive with allowlist, prohibition of network I/O, and deterministic merge strategy; clarified deep-merge corners.
- **N-006 (Conditional Logic):** Allowed `if="EXPR"` for facets and list items; defined a deterministic expression mini-language.
- **N-007 (Lens Plugins):** Secure lens extension mechanism (registration, sandboxing, limits); expanded §6 and §12.
- **N-008 (Canonization Order):** Pipeline: *imports → variables → conditional filtering → anchors → lenses → JSON* (clarified in §9.1).
- **N-009 (Reserved Names):** Added `@vars` and `@import` to the reserved names set.
- **N-010 (Attributes & Templates):** Attributes configure the facet entity; **variables and lenses MUST NOT apply to attributes**; template interpolation is disabled in attributes; encountering `{{…}}` in attributes **MUST** raise **F304**.
- **N-011 (Typed Variables):** Added compile-time facet `@var_types` for validating `@vars` values (types: `string`, `int`, `float`, `bool`, `array`, `object`; constraints: `enum`, `min`, `max`, `pattern`). New errors: **F451**, **F452**. See §3.12 and §7.5.
- **N-012 (Deterministic Choice Lenses):** Added deterministic lenses `choose(seed)` and `shuffle(seed)` for lists. Missing seed **MUST** raise **F804**. See §6.6.
- **N-013 (Vars Resolution Semantics):** In `@vars`, references are allowed only to **previously declared** variables (top‑down). Forward references **MUST** raise **F404**. Cycles **MUST** raise **F403**. See §3.11 and §7.2.

> **Compatibility:** v1.0 documents remain valid and canonize as before if they do not use new constructs.

---

## Abstract

**FACET (Feature‑Aware Contracted Extension for Text)** is a human‑readable, machine‑deterministic markup language for authoring, managing, and executing instructions for AI systems. FACET unifies plain‑text ergonomics with code‑like rigor through strict indentation, explicit data types, **first‑class output contracts**, and **lenses** — pure, deterministic pipeline transforms applied inline via `|>`. Every FACET document maps to a **single canonical JSON**, enabling lossless round‑trip and reproducible AI pipelines.

This specification defines the syntax, data model, canonization rules, conformance requirements, and security considerations for **FACET v1.1**.

---

## 1. Conformance Keywords

The keywords **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** are as in RFC 2119.

---

## 2. Design Goals

1. **Clarity & Readability** — Minimal syntactic noise; indentation as structure.
2. **Unambiguous Parsing** — No YAML‑style ambiguities; fixed indentation (2 spaces).
3. **Deterministic Serialization** — One canonical JSON; lossless round‑trip.
4. **Prompt‑First Primitives** — Multiline strings, code fences, contracts, examples.
5. **Extensibility** — Block‑based facets `@name` + attributes; lenses; reserved vocabulary that does not prevent domain‑specific ones.
6. **Toolability** — Easy to lint, format, validate; predictable error classes and codes.
7. **Modularity** — Scaling through imports and reuse.
8. **Templating without runtime** — Templates and conditions evaluated at canonization.

---

## 3. Document Model

A FACET document is a UTF‑8 text file consisting of **facets** (blocks), key‑value pairs, lists, inline maps, fenced code blocks, anchors/aliases, **lenses**, comments, variables, and import directives.

### 3.1 Encoding
Documents **MUST** be UTF‑8 encoded. A BOM, if present, **MUST** be ignored.

### 3.2 Whitespace & Newlines
- Line endings **MAY** be `LF` or `CRLF`. Canonization **MUST** normalize to `LF`.
- Trailing spaces **SHOULD** be ignored by parsers and removed by formatters.

### 3.3 Indentation
- Inside facet bodies, indentation is **exactly two spaces** per level. Tabs are invalid and **MUST** cause a parse error.
- Dedentation closes the current nested block.

### 3.4 Comments
- A `#` begins a comment and continues to end of line. Comments are ignored by canonical JSON mapping.
- Inline comments after values are permitted: `key: "value" # note`.

### 3.5 Facets
A **facet** starts with `@` + identifier, optionally followed by **attributes** in parentheses, then a newline and an indented body.

**Attributes** are an inline map of `key=value` pairs. Values in attributes MAY be unquoted identifiers, numbers, booleans, or quoted strings. Attributes are included in the facet's JSON object under the `"_attrs"` key. **Variable substitution and template interpolation MUST NOT occur inside attributes**; encountering `{{…}}` in attributes **MUST** raise **F304**. **Lenses MUST NOT apply to attributes.**

### 3.6 Keys & Values
Key‑value pairs use `key: value`. Keys follow the identifier rule. Values follow the data types in §4. Any value **MAY** be followed by a **lens pipeline** (§6).

### 3.7 Anchors & Aliases
- `&name` defines an anchor for the value on the same line.
- `*name` references the anchored value.
- Anchor names are resolved in a **document‑wide scope after import expansion and conditional filtering**. Redefinition of the same anchor name **MUST** raise **F202**. Aliases to missing anchors **MUST** raise **F201**. Cycles **MUST** be rejected.

### 3.8 Fenced Code Blocks
Triple backticks introduce a **fenced block**. The entire fence body is preserved verbatim and is **not** subject to variable interpolation. Lenses, if present, apply to the **resulting string value** of the fence. Implementations **SHOULD** cap fence body size.

### 3.9 Imports

- **Directive:**
  ```facet
  @import "relative/path/to/file.facet"
  ```
  or
  ```facet
  @import(path="...", strategy="merge")
  ```
- `path` is resolved **relative to the current file** and **MUST** be allowed by a pre‑configured allowlist of root directories.
- Network paths are forbidden. Import cycles **MUST** be detected and cause an error.
- **Merge Strategy** on same‑named facet conflicts:
  - `strategy="merge"` (default):
    - **Map**: Deep merge; the last key encountered wins.
    - **List**: Concatenate in order of appearance.
    - **Type Mismatch**: When the same key has different types across merged maps, the **last value replaces** the previous (type replacement). A last value of `null` sets the key to `null` (does not delete it).
  - `strategy="replace"`: The last encountered version of the facet replaces the previous one entirely.
- Implementations MAY provide a strict merge mode; in strict mode, a type mismatch **MUST** raise **F606** instead of replacing.
- Imports are expanded **before** variable resolution. Errors: **F601** (not found/denied), **F602** (cycle), **F606** (type mismatch in strict mode).

### 3.10 Conditional Attribute `if`

- Any **facet** or **list item** MAY have an `if="EXPR"` attribute.
- The `if` attribute **MUST** be a quoted string; otherwise raise **F704**.
- If the expression is false, the node and all its children are **discarded** from the AST before anchor resolution and lens application.
- The expression language is defined in §3.10.1.

#### 3.10.1 Expression Mini‑Language (`EXPR`)

- **Operands:** Literals (`"str"`, numbers, `true`/`false`/`null`), a variable `name`, or a nested variable path `a.b.c`.
- **Operators:** `==`, `!=`, `<`, `<=`, `>`, `>=`, `in`.
- **Logic:** `and`, `or`, `not`.
- **Grouping:** Parentheses `(` `)` are supported.
- String comparison is lexicographical; number comparison is numerical.
- Errors: **F701** (parse), **F702** (unknown var), **F703** (type error).

### 3.11 Facet `@vars`

- Declares document‑local variables:
  ```facet
  @vars
    username: "Alex"
    topic: "recursion"
    cfg:
      lang: "ru"
  ```
- Values can be any valid scalar, map, or list; accessible via paths (e.g., `cfg.lang`).
- References inside `@vars` are allowed **only to previously declared variables** (top‑down). Forward references **MUST** raise **F404**. **Cycles** **MUST** raise **F403**.
- The `@vars` facet is **compile‑time only** and **MUST NOT** be emitted in the canonical JSON (§9.3).
- Conflict resolution with host‑provided variables is defined in §7.1.

### 3.12 Facet `@var_types` (Compile‑Time Validation)

- Declares types and constraints for variables defined in `@vars`. Keys are **paths relative to `@vars` root**.
  ```facet
  @var_types
    username: { type: "string" }
    retries:  { type: "int", min: 0, max: 10 }
    env:      { type: "string", enum: ["dev","staging","prod"] }
    cfg.lang: { type: "string", pattern: "^[a-z]{2}$" }
  ```
- **Types:** `string`, `int`, `float`, `bool`, `array`, `object`.
- **Constraints:**
  - `enum` (array of literals),
  - numeric `min`/`max` for `int`/`float`,
  - `pattern` (regex) for `string`.
- **Validation:** Performed **after variable resolution** (step 4) and **before conditionals** (step 5). On mismatch raise **F451** (type) or **F452** (constraint).
- `@var_types` is **compile‑time only** and **MUST NOT** be emitted in canonical JSON (§9.3).

---

## 4. Data Types

Unchanged from v1.0. (Extended scalars are serialized as JSON strings; `NaN`/`±Infinity` are invalid.)

---

## 5. Standard Facets (Reserved Names)

The reserved set includes v1.0 names and adds:

- `@vars` — Document‑local variables (compile‑time only; not emitted).
- `@var_types` — Types/constraints for variables (compile‑time only; not emitted).
- `@import` — Import directive (compile‑time only; not emitted).

---

## 6. Lenses (Pipeline Operators)

### 6.1–6.4
(Syntax, semantics, Required/Optional Lenses are unchanged from v1.0.)

### 6.5 Extensible Lenses (Plugins)

- Implementations **MAY** support loading custom user‑defined lenses from a project manifest (see §12.4).
- **Plugin Requirements:**
  - Pure function: `(value, args:list, kwargs:map) -> value'`.
  - **MUST NOT** perform I/O, access network, time, randomness, or global state.
  - Executed in a sandbox with **timeout** and **chain‑depth** limits.
- Name resolution order: 1) Built‑in lenses → 2) Registered plugin lenses.
- Regex‑based lenses **MUST** be subject to timeouts; on timeout raise **F803**.
- Errors: **F801** (load), **F802** (unknown lens), **F803** (purity/timeout).
- Lenses **MUST NOT** apply to attributes; only to body values.

### 6.6 Deterministic Choice Lenses (Standard, Recommended)

- **Purpose:** Deterministic, seed‑driven randomness while preserving purity.
- **Lenses:**
  - `choose(seed)` — Input: non‑empty **list**. Output: a single element, chosen deterministically from the list as a pure function of `(list_value, seed)`.
  - `shuffle(seed)` — Input: **list**. Output: permutation of the list, deterministic from `(list_value, seed)`.
- **Seed Requirement:** `seed` **MUST** be provided explicitly (number or string after variable resolution). Missing seed **MUST** raise **F804 (Seed required for deterministic lens)**.
- **Type Rules:** Applying to non‑list values **MUST** raise **F102** (lens type error). Applying `choose` to an empty list **MUST** raise **F102**.
- **Determinism:** Implementations **MUST** use stable, versioned algorithms.

---

## 7. Variables (Templating)

FACET supports two variable sources:

1. **Host variables** (provided externally by the runtime).
2. **Local `@vars`** (defined within the document).

Two forms of reference:

- **Scalar substitution:** `$name` or `${name}` — the variable value is injected **as is** (any JSON type).
- **String interpolation:** `{{path}}` **inside quoted or triple‑quoted strings**; non‑string values are JSON‑serialized.

### 7.1 Scopes & Priority

- Resolution modes (implementation/CLI‑defined):
  - `resolve=host` (default): Only host variables are used.
  - `resolve=all`: Both host and `@vars` are used.
- In `resolve=all`, **host variables have priority** on name conflicts (e.g., CI secrets override local defaults).

### 7.2 Resolution Order

- Variables are substituted **after imports** and **before** conditional filtering (so `if` sees final values).
- Inside `@vars`, references are allowed only to **previously declared variables** (top‑down). Forward references **MUST** raise **F404**. **Cycles** **MUST** raise **F403**.
- Interpolation and substitution are **disabled** in attributes and fenced blocks; encountering `{{…}}` in attributes **MUST** raise **F304**.

### 7.3 Escaping

- To produce a literal `{{` or `}}` inside an interpolated string, use `\{{` and `\}}` respectively.

### 7.4 Errors

- **F402** — Undefined variable `$name` (scalar form).
- **F402A** — Undefined template variable `{{path}}`.
- **F402B** — Interpolation type error.
- **F403** — Variable cycle detected.
- **F404** — Variable forward reference (in `@vars`).
- **F451** — Variable type mismatch (per `@var_types`).
- **F452** — Variable constraint violation (per `@var_types`).

### 7.5 Typed Variables (Validation via `@var_types`)

- Validation is applied only to variables declared in `@vars`. Unspecified variables are not type‑checked.
- For `enum`, comparison uses literal equality after variable resolution.
- For `pattern`, regex syntax is the same as §4.2 (UTF‑8; `/` escaped as `\/`).

---

## 8. Contracts (Output Specification)

Unchanged from v1.0.

---

## 9. Canonical JSON Mapping

### 9.1 Algorithm (Normative)

Given a document `D`:

1. **Normalize newlines** to `LF`.
2. **Tokenize & enforce indentation**: exactly two spaces; tabs cause **F002**.
3. **Imports**: Expand all `@import` directives with allowlist and cycle checks; merge per strategy.
4. **Variables**: Resolve host and local `@vars`; perform `$name` substitution and `{{path}}` interpolation.
4b. **Variable Type Validation**: If `@var_types` is present, validate `@vars` values (see §3.12/§7.5); raise **F451/F452** on failure.
5. **Conditionals**: Evaluate all `if` attributes; remove nodes with false conditions.
6. **Anchors/Aliases**: Resolve aliases; detect cycles and redefinitions (see **F201/F202**).
7. **Lenses**: Apply pipelines (including plugins) left‑to‑right.
8. **Type conversions**: Extended scalars and fenced blocks → JSON strings.
9. **Construct JSON**: Facets → object keys; facet attributes under `"_attrs"`.
10. **Emission**: Stable key order; UTF‑8 encoding.

### 9.2 Merge on Import
- Deterministic; depends only on appearance order and chosen strategy (§3.9).
- **Key order stability:** order is defined by **first appearance** of a key; overwriting a value does **not** change its position.

### 9.3 Compile‑time Facets (Not Emitted)
- `@import`, `@vars`, and `@var_types` are **compile‑time** and **MUST NOT** appear in the canonical JSON output.

---

## 10. Grammar (ABNF) — Delta from v1.0

> Only productions that change or are added are shown. The v1.0 rule **E‑001** (single `{}` for `inline_map`) remains in force.

```abnf
; Facets
facet           = "@" ident [attrs] NL IND block
                / import_directive             ; @import has no body

import_directive= "@import" SP (quoted / import_paren)

import_paren    = "(" SP? "path" "=" quoted
                  [ "," SP? "strategy" "=" (DQUOTE "merge" DQUOTE / DQUOTE "replace" DQUOTE) ]
                  SP? ")"

; Attributes (unchanged carrier; semantics of `if` in §3.10)
attrs           = "(" attr *( "," SP? attr ) ")"
attr            = ident "=" attrval
attrval         = number / boolean / quoted / ident

; Lists — restrict inline attributes on items to `if` only
list            = "-" SP value [ SP item_if ] lens* NL
item_if         = "(" SP? "if" "=" quoted SP? ")"

; Template interpolation within strings
path            = ident *( "." ident )

template_ref    = "{{" SP? path SP? "}}"

quoted          = DQUOTE *( ESC / template_ref / CHAR ) DQUOTE
triple          = 3DQUOTE *( template_ref / CHAR / NL ) 3DQUOTE

; Notes:
; - EXPR for `if` is parsed by a dedicated expression parser; here it is carried as a quoted string in attrs.
; - Other v1.0 productions (document, block, kv, scalar, inline_map, regex, etc.) remain unchanged.
```

> `@var_types` uses the regular facet grammar; its content is a map of type-spec objects (no ABNF changes required beyond the standard map rules).

---

## 11. Errors and Diagnostics

New or expanded error codes:

- **F201** — Anchor error (undefined alias, cycle detected).
- **F202** — Anchor redefinition (duplicate `&name`).
- **F304** — Attribute interpolation prohibited.
- **F305** — Unsupported list-item attribute.
- **F402** — Undefined scalar variable `$name`.
- **F402A** — Undefined template variable `{{path}}`.
- **F402B** — Interpolation type error.
- **F403** — Variable cycle detected.
- **F404** — Variable forward reference (in `@vars`).
- **F451** — Variable type mismatch (per `@var_types`).
- **F452** — Variable constraint violation (per `@var_types`).
- **F601** — Import not found/denied.
- **F602** — Import cycle detected.
- **F606** — Import merge type mismatch (strict mode).
- **F701** — If‑expression parse error.
- **F702** — If‑expression unknown variable.
- **F703** — If‑expression type error.
- **F704** — If expression must be quoted.
- **F801** — Lens plugin load failure.
- **F802** — Unknown custom lens.
- **F803** — Lens purity violation / timeout.
- **F804** — Seed required for deterministic lens.

> All v1.0 diagnostics (e.g., **F001**, **F002**, **F003**, **F101**, **F102**, **F301**, **F401**, **F999**) remain.

---

## 12. Security Considerations

### 12.1 Imports
- **MUST** be restricted to an allowlist of root directories and file masks.
- **MUST** forbid absolute paths and any relative path escaping roots.
- **MUST** forbid network URLs.
- **SHOULD** limit import depth and total count.

### 12.2 Variables
- Host variables **MUST NOT** be implicitly read from the environment; they **MUST** be passed explicitly by the host.
- `@vars`/`@var_types` **MUST NOT** perform file loading; any `.env/.json` ingestion is an out‑of‑band preprocessing step.

### 12.3 Conditionals
- Expression parser **MUST** be implemented without `eval` and conform to the limited grammar.

### 12.4 Lens Plugins & Deterministic Lenses
- Plugins run in a sandbox with disabled FS/network/time/random.
- **MUST** enforce timeout and max lens‑chain length.
- Deterministic lenses (`choose`, `shuffle`) **MUST** fail without an explicit seed and **MUST** use versioned algorithms.
- Plugin search paths **SHOULD** be declared in a project manifest (e.g., `facet.config.json`).
- Implementations **SHOULD** support reproducible builds via version‑pinned plugin manifests.

---

## 13. IANA Considerations

Unchanged from v1.0.

---

## 14. Interoperability & Migration

- **From Mustache/Jinja:** `{{...}}` in strings maps to FACET interpolation; logic `{% if %}` becomes `if` attributes.
- **Libraries:** Extract common sections into `.facet` files and include via `@import`.

---

## 15. Style Guide (Non‑Normative)

- **Variables:** Prefer host vars for secrets; `@vars` for local defaults. Use `@var_types` to stabilize teams’ expectations.
- **Imports:** Group at file top; prefer `merge` for configurations, `replace` for templates.
- **Conditionals:** Favor simple flags in `@vars` over complex expressions.
- **Lenses:** Keep chains short; heavy transforms belong to pre‑processing; prefer `choose(seed)`/`shuffle(seed)` over ad‑hoc randomness.

---

## 16. Reference: Standard Facet Keys (Informative)

- `@meta`: `id`, `version`, `author`, `tags`
- `@system`: `role`, `style`, `constraints`
- `@user`: `request`, `data`
- `@assistant`: `prompt`, `style`, `format`
- `@plan`: steps (list of strings)
- `@examples`: arbitrary map/list
- `@tools`: each tool has `name`, `call`, `policy`
- `@output`: `format`, `require`, `schema`
- `@safety`: `policies`, `blocked`, `notes`
- `@vars`: compile‑time variables (not emitted)
- `@var_types`: compile‑time variable typing (not emitted)
- `@import`: compile‑time include (not emitted)

---

## 17. Worked Examples (v1.1)

### 17.1 Vars + Interpolation + Typed Validation
```facet
@vars
  username: "Alex"
  retries: 3
  env: "prod"
  greeting_choices:
    - "hi"
    - "hello"
    - "hey"

@var_types
  username: { type: "string" }
  retries:  { type: "int", min: 0, max: 5 }
  env:      { type: "string", enum: ["dev","staging","prod"] }

@user
  prompt: "Hello, {{username}}! Env={{env}}. Retries={{retries}}."
```

### 17.2 Deterministic Choice Lens
```facet
@vars
  seed: 42
  greeting_choices: ["hi","hello","hey"]
  greeting: $greeting_choices |> choose(seed=42)

@user
  prompt: "{{greeting}}, world!"
```

### 17.3 Imports + Merge
```facet
@import "common/system_prompts.facet"
@import(path="common/output_contracts.facet", strategy="merge")

@user
  request: "Explain recursion in Python"
```

### 17.4 Conditional Facets & List Items
```facet
@vars
  mode: "expert"
  features: ["recursion", "tail-calls"]

@system(role="Deep Technical Expert", if="mode == 'expert'")
  constraints:
    - "Use precise terminology"

@plan
  - "Intro"
  - "Tail recursion" (if="'tail-calls' in features")
```

---

## 18. CLI & Config (Informative)

A reference implementation **SHOULD** provide CLI commands and a configuration manifest.

- **CLI Examples**:
  ```bash
  # Canonize with variables and import roots; enforce var typing
  facet canon --resolve=all --import-root ./common --var "mode=expert" file.facet

  # Pre‑process a template into a pure FACET doc
  facet template --vars vars.json in.facet > expanded.facet
  ```

- **Manifest Example** (`facet.config.json`):
  ```json
  {
    "imports": {
      "roots": ["./facets", "./common"],
      "allow_glob": ["**/*.facet"]
    },
    "lenses": {
      "plugins": ["./lenses/text_filters.py"],
      "timeout_ms": 50,
      "max_chain": 16
    }
  }
  ```

