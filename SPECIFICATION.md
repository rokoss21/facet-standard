# **FACET v2.0 Language Specification**

**Status:** Candidate Recommendation (CR-1)
**Version:** 2.0
**Date:** 2025-09-15
**Author:** Emil Rokossovskiy
**Document Type:** Technical Standard
**Format:** Markdown (UTF-8)
**Intended Audience:** AI systems architects, LLM platform developers, compiler engineers.

---

# **Table of Contents**

1. Introduction
2. Conformance
3. Design Goals
4. Core Concepts
5. Document Structure
6. Abstract Syntax Tree (AST Model)
7. Execution Model
    7.1 Phase 1 — Resolution
    7.2 Phase 2 — Type Checking
    7.3 Phase 3 — Reactive Compute (R-DAG)
    7.4 Phase 4 — Layout (Token Box Model)
    7.5 Phase 5 — Render
    7.6 Execution Modes
8. Token Box Model (Context Algebra)
9. Facet Type System (FTS)
10. Interfaces (`@interface`)
11. Variables (`@vars`) and Inputs (`@input`)
12. Lenses
13. Testing (`@test`)
14. Security Model
15. Profiles
16. Reference CLI (`fct`)
17. Reserved Words
18. Canonical JSON Model (Overview)
19. Change History
20. Normative References
21. Appendix A: Standard Lens Library (Normative)
22. Appendix B: ABNF Grammar (Informative)
23. Appendix C: Error Code Catalog (Normative)
24. Appendix D: JSON Schema Mappings (Reserved)

---

# **1. Introduction**

FACET v2.0 is a **Neural Architecture Description Language (NADL)** designed to define, validate, and execute AI agent behaviors in a deterministic, resource-bounded and type-safe manner.

Unlike v1.x — a template-driven prompt language — **FACET v2.0 is a compiler**:

* It parses FACET files into a strict **AST**
* Performs **type checking** using the Facet Type System (FTS)
* Computes reactive variables via a **Dependency Graph (R-DAG)**
* Manages context limits using a deterministic **Token Box Model**
* Renders vendor-specific payloads (OpenAI, Anthropic, Llama, etc.)

FACET v2.0 introduces:

* Multimodal variables (`image`, `audio`, `embedding`)
* Interfaces with strict typing (`@interface`)
* Deterministic & bounded external lenses
* A first-class **testing system** (`@test`)
* Strict execution phases ensuring reproducibility

**Implementation Note:** This specification defines the **minimum requirements** for FACET v2.0 compliance. Implementations **MAY** provide additional features, lenses, commands, and capabilities beyond these requirements, provided they maintain backward compatibility with the core standard.

FACET aims to become a standard for **AI agent configuration**, replacing brittle text templates with deterministic, declarative, and verifiable specifications.

---

# **2. Conformance**

This document uses the terminology from **RFC 2119**.

The keywords:

**MUST**, **MUST NOT**, **SHALL**, **SHALL NOT**,
**SHOULD**, **SHOULD NOT**, **RECOMMENDED**,
**MAY**, **OPTIONAL**

are to be interpreted as *normative requirements*.

An implementation is **FACET v2.0-compliant** if and only if it satisfies:

* all normative (**MUST**, **MUST NOT**) requirements in this document
* all constraints in the execution model
* all type rules defined by the FTS
* all security constraints
* and preserves canonical behavior for Pure Mode builds

**Conformance Note:** Implementations **MAY** provide additional features beyond the minimum requirements without losing compliance, as long as core functionality remains intact and extensions do not conflict with the standard.

---

# **3. Design Goals**

FACET v2.0 is designed to achieve:

### **3.1 Determinism**

Every FACET document produces the **same canonical state** under Pure Mode, across machines and implementations.

### **3.2 Declarativity**

Users describe *what* an agent should do, not *how* to compute it.

### **3.3 Strict Resource Control**

Token budgets, compute gas, multimodal limits — all **enforced by design**.

### **3.4 Modular Architecture**

Each FACET file composes via imports, interfaces, and test modules.

### **3.5 Multimodal-First**

Images, audio, embeddings are first-class values.

### **3.6 Extensible via Lenses**

New deterministic transformations can be registered at compile time.

### **3.7 Engineering-Grade Tooling**

FACET includes testing, static validation, codegen, and profiling.

---

# **4. Core Concepts**

### **4.1 Facets**

Blocks starting with `@name`.
Examples: `@system`, `@user`, `@vars`, `@context`.

### **4.2 Variables**

Declared under `@vars`.
Include static, computed, or injected runtime inputs.

### **4.3 Lenses**

Pure or bounded functions applied to values:

```
value |> trim() |> summarize()
```

### **4.4 Reactive Computation**

Variables may depend on other variables; evaluation occurs as a dependency graph.

### **4.5 Token Box Model**

A formal specification for deterministic context packing.

### **4.6 Interfaces**

Strict tool definitions compiled into OpenAI/Anthropic tool schemas.

### **4.7 Tests**

Executable blocks defining input mocks and assertions.

### **4.8 Profiles**

FACET Core vs FACET Hypervisor.

---

# **5. Document Structure**

A FACET document is a UTF-8 text file consisting of:

* Facet blocks (`@system`, `@vars`, etc.)
* Lists & maps
* Scalars
* Lens pipelines
* Imports
* Test blocks
* Interface definitions

**Normative Rules:**

* Indentation **MUST** use exactly 2 spaces
* Tabs **MUST** cause a syntax error (**F002**)
* Line endings **MUST** be normalized to LF
* Comments begin with `#` and extend to end of line
* Facet attributes **MUST NOT** support string interpolation (`{{...}}`). Implementations **MUST** raise **F402** if interpolation syntax is encountered inside attributes

---

# **6. Abstract Syntax Tree (AST Model)**

Every FACET implementation MUST parse documents into an AST with the following node categories:

### **6.1 FacetNode**

Represents a block such as `@system`, `@user`, `@vars`, `@context`, `@test`, `@interface`.

Contains:

* name
* attributes (if any)
* child nodes (body)

### **6.2 KeyValueNode**

Represents:

```
key: value
```

Where value may be:

* scalar
* map
* list
* lens pipeline

### **6.3 ListNode / ListItemNode**

ListItemNode MAY contain an inline conditional or lens.

### **6.4 LensPipelineNode**

Sequence of transformations applied to a value.
Each lens entry includes:

* lens name
* arguments (positional, named)
* trust level (0/1/2)

### **6.5 DirectiveNode**

Special directives:

* `@import`
* `@input`
* (future extensions)

### **6.6 AST Normalization**

During Phase 1, the AST is normalized by:

* resolving imports
* merging nodes
* flattening lists by key (Smart Merge)
* eliminating unreachable nodes (e.g., false conditionals)

AST MUST be immutable after Phase 2.

---

### **6.7 Smart Merge for Lists (Informative)**

When two imported facets define a list with `key` semantics (e.g., `@examples(key="id")`), implementations **MAY** perform Smart Merge:

* Items are matched by the designated key field (`id` in this example)
* Items with the same key are merged, with the later definition overriding individual fields
* Items with new keys are appended, preserving original relative order

The exact configuration of list merge behavior is implementation-defined and **SHOULD** be documented by the host environment.

**Note:** Full normative definition of Smart Merge is deferred to v2.1.

---

# **7. Execution Model**

FACET execution MUST occur through five ordered phases.

---

## **7.1 Phase 1 — Resolution**

Input:

* Base FACET file
* Allowed import roots

Actions:

1. Parse into AST
2. Resolve all `@import` directives
3. Apply Smart Merge logic
4. Normalize AST
5. Validate reserved facet names

Output:
**Resolved AST**

Errors:

* F601 (import not found)
* F602 (import cycle)

---

## **7.2 Phase 2 — Type Checking**

Input:

* Resolved AST
* FTS type definitions
* Lens registry

Actions:

1. Validate all variables declared under `@vars` against `@var_types`
2. Validate `@interface` definitions
3. Validate lens pipelines:

   * input_type → lens_sig.input
   * lens_sig.output → next lens input
4. Validate facet attributes
5. Validate multimodal asset constraints

Output:
**Typed AST**

Errors:

* F451 (type mismatch)
* F452 (constraint violation)
* F802 (unknown lens)

---

## **7.3 Phase 3 — Reactive Compute (R-DAG)**

Input:

* Typed AST
* Runtime `@input` values

Actions:

1. Construct variable dependency graph
2. Detect cycles
3. Execute nodes in topological order
4. Apply lenses
5. Cache bounded external lenses in Pure Mode

Output:
**Computed variable map (immutable)**

Errors:

* F505 (cyclic dependency)
* F453 (input validation failure)

---

## **7.4 Phase 4 — Layout (Token Box Model)**

Input:

* Computed variable map
* Section mappings
* Token budget

Actions:

* Tokenize all section text
* Apply Critical allocation
* Apply growth/expansion
* Apply compression pipelines
* Apply drop rules
* Produce final ordered list of sections

Output:
**Finalized context**

Errors:

* F901 (critical overflow)

---

## **7.5 Phase 5 — Render**

Input:

* Finalized context
* Target provider (OpenAI/Anthropic/Local)

Actions:

* Build canonical payload
* Convert interfaces into provider tool schemas
* Insert multimodal assets

Output:
**JSON structure ready for API call**

---

## **7.6 Execution Modes**

Implementations MUST support at minimum:

### **Pure Mode**

* Only Pure lenses (Level 0) and Bounded lenses (Level 1 w/ deterministic params)
* Fully deterministic
* No nondeterministic randomness
* Output considered canonical

### **Execution Mode**

* Allows Volatile lenses (Level 2)
* External calls permitted
* Output not canonical


---

# **8. Token Box Model (Context Algebra)**

The Token Box Model defines **deterministic context allocation** for LLM prompts.

Implementations MUST use this model to guarantee reproducible behavior across executions, platforms, and vendors.

---

## **8.1 Overview**

The context is represented as a finite-capacity container (`Budget`) measured in **tokenizer units**.

Each logical prompt block (`@system`, `@user`, `@history`, RAG documents, etc.) is a **Section** with attributes that determine how it is packed into the available space.

---

## **8.2 Section Fields (Normative)**

Each Section MUST have the following properties:

| Field       | Type         | Default | Description                                              |
| ----------- | ------------ | ------- | -------------------------------------------------------- |
| `priority`  | integer      | 500     | Defines removal/survival order (lower = dropped earlier) |
| `base_size` | integer      | auto    | Token length after initial render                        |
| `min`       | integer      | 0       | Minimum guaranteed size                                  |
| `grow`      | number       | 0       | Weight for distributing excess space                     |
| `shrink`    | number       | 0       | Weight for compression/removal                           |
| `strategy`  | LensPipeline | none    | Compression strategy                                     |

A Section is considered **Critical** if and only if:
**`shrink == 0`**

Critical sections MUST NOT be:

* compressed
* truncated
* dropped

---

## **8.3 Algorithm (Normative)**

Let:

* `S = {sections}`
* `B = token budget`
* `size[i] = base_size(S[i])`

### **Step 1 — Compute Fixed Load**

```
Critical = { i | shrink[i] == 0 }
FixedLoad = sum(size[i] for i in Critical)
```

**If FixedLoad > B → raise F901 (Critical Overflow)**

---

### **Step 2 — Compute Free Space**

```
FreeSpace = B - FixedLoad
```

---

### **Step 3 — Expansion (Optional)**

If implementations support expansion:

```
FlexGrow = { i | grow[i] > 0 }
Allocate FreeSpace proportionally by grow weights
target_size[i] = size[i] + growth[i]
```

If expansion is not implemented: skip this step.

---

### **Step 4 — Compression**

If total_size(S) > B:

```
Deficit = total_size(S) - B
Flex = { i | shrink[i] > 0 }

// Sort: lowest priority removed/compressed first
Sort Flex by (priority ASC, shrink DESC)
```

For each flexible section:

1. Apply compression lens pipeline (`strategy`)
2. Recompute size
3. If still > allowed size:

   * Truncate to `min`
4. If still > allowed size:

   * Drop the entire section

Stop when `Deficit <= 0`.

---

### **8.4 Result**

The result of the Layout phase is:

* ordered list of kept sections
* with exact sizes fitting within `Budget`
* deterministic across platforms

---

# **9. Facet Type System (FTS)**

The Facet Type System defines a **strict, language-neutral typing layer** used for:

* variable validation (`@vars` + `@var_types`)
* interface schema generation (`@interface`)
* lens signatures
* multimodal values

Implementations MUST fully support FTS to be standard-compliant.

---

## **9.1 Primitive Types**

* `string`
* `int`
* `float`
* `bool`
* `null`
* `any` (discouraged, allowed for generic lenses)

Constraints MAY apply:

* `pattern` (regex, for strings)
* `min`, `max` (numbers)
* `enum` list

Errors:

* F451 — type mismatch
* F452 — constraint violation

---

## **9.2 Multimodal Types**

FTS defines **first-class multimodal types**:

### **Image**

```
type: image
properties:
  max_dim (optional)
  format: "png" | "jpeg" | "webp"
```

### **Audio**

```
type: audio
properties:
  max_duration
  format: "mp3" | "wav" | "ogg"
```

### **Embedding**

```
type: embedding<size=N>
```

Where size MUST be a positive integer.

---

## **9.3 Composite Types**

### **Struct**

```
struct {
  field1: type
  field2: type
}
```

### **List**

```
list<T>
```

### **Map**

```
map<string, T>
```

### **Union**

```
T1 | T2 | ...
```

Used for lenses and interfaces.

---

## **9.4 Type Assignability Rules**

T1 is assignable to T2 if:

1. T1 == T2
2. T1 is subtype of T2
3. For lists: element types are assignable
4. For structs: all required fields match
5. For multimodal types: format & limits fit

---

# **10. Interfaces (`@interface`)**

Interfaces define **typed tool contracts** for AI agent tool calling.

FACET v2.0 implementations MUST support:

* defining interfaces
* type checking interface signatures
* generating canonical JSON schemas during Render phase

---

## **10.1 Syntax**

```facet
@interface WeatherAPI
  fn get_current(city: string) -> struct {
    temp: float
    condition: string
  }
```

Each `fn` definition includes:

* function name
* parameter list (typed)
* return type (FTS type)

---

## **10.2 Usage in @system**

```facet
@system
  tools: [$WeatherAPI]
```

At Render phase, the compiler MUST convert the interface into:

* provider-specific tool schema
* including parameters, type metadata, and error behavior

---

## **10.3 Constraints**

* All parameter names MUST be unique
* Return types MUST be expressible in JSON / LLM tool schema format
* Tool names MUST be unique across document

---

## **10.4 Canonical JSON Mapping (Summary)**

Full mapping is defined in a future Appendix D (JSON Schema Mappings), but simplified:

```
{
  "name": "WeatherAPI",
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "get_current",
        "parameters": {
          "type": "object",
          "properties": {
            "city": { "type": "string" }
          },
          "required": ["city"]
        }
      }
    }
  ]
}
```

---

# **11. Variables (`@vars`) and Inputs (`@input`)**

FACET v2.0 unifies variables into a formal reactive model.

---

## **11.1 @vars Block**

Example:

```facet
@vars
  username: "Alex"
  env: "prod"
  profile: $user_doc |> summarize() |> to_markdown()
```

Rules:

* Variables are evaluated in Phase 3
* Values become immutable once computed
* Variables MAY reference other variables
* Forward references detected during static analysis **MUST** raise **F404**
* Cycles detected during R-DAG construction **MUST** raise **F505**

---

## **11.2 @var_types Block**

Example:

```facet
@var_types
  username: { type: "string" }
  retries:  { type: "int", min: 0, max: 5 }
```

Enforced in Phase 2.

---

## **11.3 Runtime Inputs (`@input`)**

`@input` defines external values that **must be supplied by the Host**.

### Example:

```facet
@vars
  user_query: @input(type="string")
  user_photo: @input(type="image", max_dim=1024)
```

### Rules (Normative)

1. `@input` MUST appear only inside `@vars`.
2. Type MUST be a valid FTS type.
3. Host MUST supply a value at runtime.
4. If input violates constraints → **F453**.
5. Inputs participate in R-DAG evaluation as leaf nodes.

---

# **12. Lenses**

Lenses are pure or controlled transformations.

A lens pipeline looks like:

```
value |> lens1(arg) |> lens2(k=v)
```

---

## **12.1 Lens Registry (Normative)**

Implementations MUST maintain a registry with:

* lens name
* input types
* output type
* trust level (0/1/2)
* deterministic flag

---

## **12.2 Trust Levels**

### **Level 0 — Pure Lenses**

* No I/O
* Deterministic
* Allowed in all modes

Examples:
`trim()`, `to_json()`, `sort_by()`

---

### **Level 1 — Bounded External Lenses**

* External calls allowed (LLM, RAG)
* MUST be deterministic given the same inputs
* MUST pin model version
* MUST be cacheable

Allowed only in Pure Mode if `temperature=0`.

---

### **Level 2 — Volatile Lenses**

* Non-deterministic
* Allowed only in Execution Mode

Examples:
`llm_sample(temperature=1.0)`

---

## **12.3 Lens Composition Rules**

* Lens A output_type MUST match lens B input_type
* Pipeline with mixed trust levels inherits the **highest level**
* Lens errors MUST abort the R-DAG execution

---

# **13. Testing (`@test`)**

Testing is a required component of FACET v2.0 Hypervisor Profile.

**Implementation Note:** Implementations **MAY** provide additional testing features such as extended assertion types, output formats, and integration with external testing frameworks.

---

## **13.1 Syntax**

```facet
@test "basic greeting"
  vars:
    username: "TestUser"

  mock:
    WeatherAPI.get_current: { temp: 10, cond: "Rain" }

  assert:
    - output contains "umbrella"
    - output sentiment "helpful"
    - cost < 0.01
```

---

## **13.2 Execution Rules**

1. Each test defines an isolated execution environment.
2. Inputs override `@input` values.
3. Mocked tools bypass external calls.
4. The full 5-phase pipeline is executed.
5. Assertions are evaluated against:

   * output
   * telemetry (cost/tokens/latency)

---

## **13.3 Failure Conditions**

* Unmet assertion → test failure
* Missing mock for Level 1 lens → error
* Lens failure → test abort

---


# **14. Security Model**

Security is a **foundational requirement** of FACET v2.0.
All implementations MUST enforce these constraints to qualify as compliant.

---

## **14.1 Hermetic Build Requirements**

During Phases 1–3, a FACET compiler MUST operate in a **hermetic environment**:

* No unrestricted filesystem access
* No external network access
* No environment variable leakage
* Only explicitly allowed imports may be resolved

### **Imports**

`@import` is restricted by:

1. **Allowlist Root Directories**
2. No absolute paths
3. No parent traversal (`..`)
4. No network URLs

This ensures deterministic reproducibility.

---

## **14.2 Runtime I/O Restrictions**

Only the following are allowed at runtime:

* Lenses explicitly registered as Level 1 or 2
* Input values provided through `@input`

Any other form of I/O MUST raise **F801 (I/O Prohibited)**.

---

## **14.3 Multimodal Asset Sanitization**

All `image`, `audio`, or `embedding` inputs MUST be sanitized:

* Stripped metadata
* Re-encoded using safe codecs
* Normalized shape/dimensions according to FTS constraints

This eliminates:

* EXIF-based payload injections
* malformed binary inputs
* fingerprinting vectors

---

## **14.4 Gas Model (Compute Quotas)**

To prevent infinite lens chains or excessive computation:

### **Each lens consumes “gas”.**

The host MUST define:

* max gas (`GasLimit`)
* gas cost functions per lens
* gas accounting per DAG step

Exceeding gas MUST raise **F902 (Compute Exhaustion)**.

This prevents accidental or malicious compute blowups.

---

## **14.5 Pure Mode Safety**

Pure Mode MUST guarantee:

* No external I/O
* No randomness
* All lens outputs reproducible
* Caches for Level 1 lenses enabled
* Stable, hash-derived randomness only (if required for deterministic sorting/shuffling)

This mode is required for:

* testing
* compilation
* caching
* canonicalization

---

# **15. Profiles**

FACET v2.0 defines **profiles** to allow lightweight and enterprise implementations.

---

## **15.1 Profile: Core**

Minimal, text-oriented FACET suitable for prompt templates.

**Includes:**

* Phases 1–2–5
* @vars (static only)
* basic lenses
* no multimodal types
* no interfaces
* no R-DAG

**Target users:** prompt engineers, simple systems.

---

## **15.2 Profile: Hypervisor (Full)**

The complete FACET v2.0 environment.

**Includes:**

* All five execution phases
* Token Box Model
* Multimodal asset support
* Interfaces (`@interface`)
* Testing (`@test`)
* Reactive DAG
* Complete security sandboxing

**Target users:** AI agents, autonomous systems, enterprise LLM orchestration.

---

# **16. Reference CLI (`fct`)**

A standard implementation SHOULD provide a CLI tool called **`fct`**.

This ensures feature parity across implementations.

**Implementation Note:** Implementations **MAY** provide additional CLI commands and options beyond the minimum set defined in this section for enhanced usability and tooling.

---

## **16.1 `fct build`**

```
fct build file.facet
```

Runs:

* Phase 1: Resolution
* Phase 2: Type Checking

Outputs:

* validation report
* typed AST

---

## **16.2 `fct run`**

```
fct run file.facet --input input.json
```

Runs:

* full 5-phase pipeline
* produces canonical JSON payload

Options:

* `--pure`
* `--exec`
* `--profile core/hypervisor`

---

## **16.3 `fct test`**

```
fct test file.facet
```

Discovers and executes all `@test` blocks.

Outputs:

* test results
* failures
* telemetry (tokens, cost, latency)

---

## **16.4 `fct codegen`**

```
fct codegen file.facet --lang python
```

Generates:

* Pydantic/TS types
* SDKs for tools defined via `@interface`
* type-safe API stubs

---

## **16.5 `fct inspect`**

```
fct inspect file.facet --graph dag.json
```

Outputs:

* AST
* R-DAG graph
* Token allocation flamegraph

Useful for debugging or introspection.

---

# **17. Reserved Words**

The following facet names and keywords are RESERVED and MUST NOT be redefined:

### **Facet Names**

```
@meta
@system
@user
@assistant
@vars
@var_types
@context
@plan
@import
@tools
@input
@interface
@test
```

### **Keywords**

```
fn
struct
list
map
enum
type
```

---

# **18. Canonical JSON Model (Overview)**

FACET MUST render its final output into a **canonical JSON structure** before provider-specific transformations.

This JSON MUST include:

1. system messages
2. user messages
3. assistant instructions (if any)
4. tool definitions
5. multimodal items

The canonical order MUST be:

1. metadata
2. system
3. tools
4. examples
5. history
6. user
7. assistant

This order ensures deterministic packing and stable diffs.

Full JSON mapping tables will be defined in a future Appendix D (JSON Schema Mappings).

---

# **19. Change History**

### **v2.0 (CR-1)**

* Added Token Box Model
* Added R-DAG computation
* Added `@input`
* Added `@test`
* Added multimodal type system
* Added interfaces
* Added CLI spec
* Fully formalized execution phases
* Security model defined

### **v1.1**

* Conditional logic
* Imports
* Lenses v1
* Variables and templating

### **v1.0**

* Basic block structure
* Deterministic JSON output
* First stable release

---

# **20. Normative References**

1. RFC 2119 — Key words for use in RFCs
2. RFC 5234 — Augmented BNF for Syntax Specifications (ABNF)
3. JSON Schema Draft 2020-12
4. OpenAI Assistants & Tools Specifications
5. Anthropic Tool Use Protocol
6. Unicode Standard v15
7. Markdown CommonMark Spec
8. FACET v1.1 Specification

---

# **Appendix A (Normative): Standard Lens Library**

To ensure portability, all FACET v2.0 implementations **MUST** provide the following built-in lenses (Trust Level 0).

**Implementation Note:** Implementations **MAY** provide additional lenses beyond this minimum set for enhanced functionality. Such extensions should be clearly documented and not conflict with the standard library.

---

## **A.1 Text Lenses**

* `trim()`: Removes whitespace from both ends.
* `lowercase()`, `uppercase()`: Case conversion.
* `split(separator: string) -> list<string>`: Splits string by delimiter.
* `replace(pattern: string, replacement: string)`: Regex replacement.
* `indent(level: int)`: Adds 2 spaces × level indentation to each line.

---

## **A.2 Data Structure Lenses**

* `json(indent: int = 0)`: Serializes any value to JSON string.
* `keys() -> list<string>`: Returns map keys.
* `values() -> list<any>`: Returns map values.
* `filter(query: string)`: Filters list based on expression (e.g., `item.active == true`).
* `map(key: string)`: Extracts a specific field from a list of objects.
* `sort_by(key: string, desc: bool = false)`: Stable sort.

---

## **A.3 Logic Lenses**

* `default(value: any)`: Returns input if not null, else returns default.
* `ensure_list()`: Wraps scalar in list `[x]`, returns list as is.

---

## **A.4 Multimodal Lenses (Stub)**

Implementations **SHOULD** provide these, but specific codec support is implementation-defined.

* `image_resize(width: int, height: int)`
* `audio_trim(start: float, end: float)`

---

# **Appendix B (Informative): ABNF Grammar**

The syntax is defined using ABNF (RFC 5234).

```abnf
; Document Structure
FACET-DOC    = *BLOCK
BLOCK        = AT-BLOCK / COMMENT / NL
AT-BLOCK     = "@" IDENT [ATTRS] NL IND BODY DED

; Attributes
ATTRS        = "(" [ATTR *("," WS ATTR)] ")"
ATTR         = IDENT "=" VALUE

; Values & Types
VALUE        = SCALAR / STRING / LIST / MAP / VAR / PIPELINE
VAR          = "$" IDENT *( "." IDENT )
PIPELINE     = VALUE *( WS "|>" WS LENS-CALL )

; Lenses
LENS-CALL    = IDENT "(" [ARGS] ")"
ARGS         = ARG *("," WS ARG)
ARG          = [IDENT "="] VALUE        ; positional or named

; Variables Block
VARS-DECL    = IDENT ":" WS (VALUE / INPUT-DIR) NL
INPUT-DIR    = "@input" "(" [ATTRS] ")"

; Comments
COMMENT      = "#" *(%x20-7E) NL

; Basic Tokens
IDENT        = ALPHA *(ALPHA / DIGIT / "_")
STRING       = DQUOTE *(%x20-21 / %x23-7E) DQUOTE
SCALAR       = 1*DIGIT / "true" / "false" / "null"
WS           = 1*SP                      ; TAB is forbidden (F002)
NL           = LF / CRLF
IND          = 2SP
DED          = ; dedent (implementation-defined)
BODY         = *(VARS-DECL / KEY-VALUE / LIST-ITEM)
KEY-VALUE    = IDENT ":" WS VALUE NL
LIST-ITEM    = "-" WS VALUE NL
```

---

# **Appendix C (Normative): Error Code Catalog**

Implementations **MUST** use these codes for diagnostics.

---

## **C.1 Syntax & Parse Errors (000–099)**

| Code   | Description                              |
| ------ | ---------------------------------------- |
| F001   | Invalid indentation (must be 2 spaces)   |
| F002   | Tabs are forbidden                       |
| F003   | Unclosed block/string/parenthesis        |

---

## **C.2 Semantic & Type Errors (400–499)**

| Code   | Description                                      |
| ------ | ------------------------------------------------ |
| F401   | Variable not found                               |
| F402   | Attribute interpolation forbidden                |
| F404   | Variable forward reference (invalid evaluation order) |
| F451   | Type Mismatch (FTS validation failed)            |
| F452   | Constraint Violation (min/max/pattern)           |
| F453   | Runtime Input Validation failed (`@input`)       |

---

## **C.3 Graph & Logic Errors (500–599)**

| Code   | Description                                      |
| ------ | ------------------------------------------------ |
| F505   | Reactive Variable Cycle detected (R-DAG)         |

---

## **C.4 Import & Resolution Errors (600–699)**

| Code   | Description                                      |
| ------ | ------------------------------------------------ |
| F601   | Import not found                                 |
| F602   | Import cycle                                     |

---

## **C.5 Runtime & Security Errors (800–999)**

| Code   | Description                                      |
| ------ | ------------------------------------------------ |
| F801   | I/O Prohibited (Hermetic Build violation)        |
| F802   | Unknown Lens                                     |
| F901   | Context Critical Overflow (Box Model failed)     |
| F902   | Compute Gas Exhausted                            |

---

# **Appendix D (Reserved): JSON Schema Mappings**

This appendix is reserved for future versions of the FACET specification and will normatively define the canonical JSON mapping for FTS types and `@interface` schemas.

**Status:** Reserved for v2.1

---

**End of Specification**

---

**Status:** Candidate Recommendation (CR-1)
**Document Hash:** To be computed upon final approval.
**Last Updated:** 2025-09-15

