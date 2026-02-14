# Barnacle Language Design Review

**Reviewing:** `LANGUAGE_DESIGN.md` v1.0 (2026-02-12, 2,886 lines)
**Reviewer:** Claude
**Date:** 2026-02-14

---

## Executive Summary

The Barnacle language design is ambitious and well-thought-out. The core thesis -- algebraic effects as the universal abstraction mapped to Wasm Component Model interfaces -- is compelling and technically sound. The design successfully unifies error handling, async, state, and I/O under a single mechanism without function coloring.

However, the document contains **8 internal inconsistencies** that would create ambiguity during implementation, **7 design concerns** that need clarification, **5 pattern reuse opportunities** that would strengthen the design's coherence, and **10 missing specifications** that implementers will need. A formal grammar is absent entirely.

### Key findings at a glance

| Category | Count | Severity |
|---|---|---|
| Internal inconsistencies | 8 | High -- blocks parser/type-checker implementation |
| Design concerns | 7 | Medium -- affects architecture decisions |
| Pattern reuse opportunities | 5 | Low -- improves document quality |
| Missing specifications | 10 | Medium -- needed before or during implementation |

### Most critical issues

1. **Handler syntax is defined three different ways** (Section 2.3, 4.3, 11.14)
2. **`Result` vs `Fail<E>` in effect operations** -- the document says `Fail` replaces `Result` but effect operations still return `Result`
3. **Implicit vs explicit return rules are never formally stated** despite governing every function body
4. **Structural typing at Wasm component boundaries** -- how structural types compile to nominal WIT types is unspecified

---

## 1. Internal Inconsistencies

These are places where the document contradicts itself. They must be resolved before implementation begins, as they affect parser grammar, type checker rules, and the effect system semantics.

### 1.1 Handler operation parameter binding: positional vs named

The document uses three incompatible conventions for handler operation clauses:

**Section 2.3 (line 115)** -- Positional binding, positional call:
```barnacle
handler real_console: Console {
    print(msg) -> resume(wasi_print(msg))
    read_line() -> resume(wasi_read_line())
}
```

**Section 4.3 (line 600)** -- Positional binding, named call:
```barnacle
handler io: FileSystem {
    read_file(path) -> resume(wasi_read_file(path: path))
}
```

**Section 11.14 (line 2421)** -- Named binding, named call:
```barnacle
handler real_console: Console {
    print(msg: msg) -> resume(wasi_print(msg: msg))
    read_line() -> resume(wasi_read_line())
}
```

**Question**: In a handler clause like `print(msg) -> ...`, is `msg` a positional pattern binding (matching by position) or a named binding (matching by parameter name)? Since the operation's parameter names are already declared in the effect definition, positional binding is sufficient and less redundant.

**Recommendation**: Adopt positional binding for handler operation clauses (these are pattern-matching contexts, not call sites). Implementation function calls should use named arguments per the general language rule. Canonical form:

```barnacle
handler real_console: Console {
    print(msg) -> resume(wasi_print(msg: msg))
    read_line() -> resume(wasi_read_line())
}
```

### 1.2 `to_result` handler defined inconsistently

The most commonly-referenced handler appears in three forms:

**Line 201 (Section 2.5):**
```barnacle
handler to_result<T, E>: Fail<E> -> Result<T, E> {
    fail(error) -> :err { error }
}
```

**Line 619 (Section 4.3):**
```barnacle
handler to_result: Fail<E> {
    fail(error) -> :err { error: error }
}
```

**Line 2561 (Section 12.3):**
```barnacle
handler to_result<T, E>: Fail<E> -> Result<T, E> { ... }
```

Differences:
- Line 619 is missing the generic parameters `<T, E>` and the return type annotation `-> Result<T, E>`
- Line 202 uses shorthand field syntax `{ error }`, while line 620 uses explicit `{ error: error }`

**Related issue**: The shorthand field syntax `{ error }` meaning `{ error: error }` is used in construction (line 202) but only formally specified for destructuring (line 2263). This syntactic sugar should be formally specified for both contexts.

**Recommendation**: Standardize on the line 201 form with full generic parameters and return type annotation. Formally specify that `{ name }` is shorthand for `{ name: name }` in both construction and destructuring.

### 1.3 `FileSystem` effect signature changes between sections

**Section 2.1 (lines 77-78):**
```barnacle
effect FileSystem {
    read_file(path: String) -> Result<String, IoError>
    write_file(path: String, data: String) -> Result<(), IoError>
}
```

**Section 2.9 (lines 308-309):**
```barnacle
effect FileSystem {
    read_file(path: String) -> String
    write_file(path: String, content: String) -> ()
}
```

Differences:
- Return types: `Result<String, IoError>` vs bare `String`
- Parameter name: `data` vs `content` on `write_file`

This is directly related to the `Result` vs `Fail` question (item 1.4). The Section 2.9 form is more consistent with the `Fail<E>` philosophy.

### 1.4 `Result` in effect operations contradicts the `Fail<E>` philosophy

**The claim (line 172, Section 2.5):**
> This replaces `try`/`catch`, `Result<T,E>`, `throws`, and the `?` operator -- all are sugar or handlers over `Fail`

**The contradiction:**
- Section 2.1 `FileSystem` (line 77): `read_file(path: String) -> Result<String, IoError>`
- Section 2.6 `Database` (line 231): `execute(query: String, params: List<Value>) -> Result<Rows, DbError>`

If `Fail<E>` replaces `Result<T, E>`, why do effect operations return `Result`? When a caller gets `Result<String, IoError>` from an effect operation, they must manually unwrap it, which defeats the purpose of the effect system.

**The correct pattern (already used in Section 2.9, lines 303-304):**
```barnacle
effect Database {
    query(sql: String, params: List<Value>) -> List<Row>
    execute(sql: String, params: List<Value>) -> Int
}
```

Here the operations return bare types. If the underlying I/O fails, the handler (not the effect operation) raises `Fail<DbError>`. This keeps the effect interface clean and lets the effect system handle errors uniformly.

**Can you use only `Fail<E>`?** Yes. Effect operations should declare their happy-path return types. The handler implementation decides how errors are surfaced -- either by performing `Fail<E>` (propagating to the caller) or by handling the error internally. The caller's code only sees the happy path; `Fail<E>` in the function's effect set signals that errors can occur.

**Design rule to add**: "Effect operations declare happy-path return types. Errors in effect operations are surfaced through the `Fail<E>` effect, not by returning `Result<T, E>`. `Result<T, E>` is appropriate only at non-effectful data interchange boundaries (e.g., converting from `Fail` to a value via `handle expr with to_result`)."

**Recommendation**: Update all effect definitions in Section 2.1 and 2.6 to use bare return types. Add the design rule above to Section 2.5 or 2.9.

### 1.5 `Console.print` parameter name: `msg` vs `s`

**Effect declaration (line 72):**
```barnacle
effect Console {
    print(msg: String) -> ()
    read_line() -> String
}
```

**Call site (line 1796, Section 9.2):**
```barnacle
Console.print(s: s.length.to_string())
```

The parameter name `s:` does not match the declared `msg:`. Since Barnacle requires named parameters, this is a type error.

**Recommendation**: Fix line 1796 to `Console.print(msg: s.length.to_string())`.

### 1.6 `Database` effect signature varies across sections

Three different `Database` effects appear:

**Section 2.6 (line 231):**
```barnacle
effect Database {
    execute(query: String, params: List<Value>) -> Result<Rows, DbError>
}
```

**Section 2.9 (lines 303-304):**
```barnacle
effect Database {
    query(sql: String, params: List<Value>) -> List<Row>
    execute(sql: String, params: List<Value>) -> Int
}
```

**Section 10.3 (lines 1954-1956):**
```barnacle
export effect Database {
    query(sql: String) -> List<Row>
    execute(sql: String) -> ()
}
```

Differences: parameter names (`query` vs `sql`), type names (`Rows` vs `Row` vs `List<Row>`), return types (`Result<Rows, DbError>` vs `Int` vs `()`), operation names (only `execute` vs both `query` and `execute`), and whether `params` is included.

**Recommendation**: Define one canonical `Database` effect early in the document and reference it consistently. Use the Section 2.9 form (bare return types, both operations, `params` included) as it best demonstrates the `Fail<E>` philosophy.

### 1.7 Implicit vs explicit return rules are ambiguous

**The statement (line 2046, Section 11.1):**
> `return` -- explicit return (required in multi-statement functions)

**The statement (line 2120, Section 11.4):**
> "Explicit return required" (with examples showing `return a` / `return b`)

**Contradicting examples:**

Pattern match arms use implicit values (line 2327):
```barnacle
match value {
    0 -> "zero"
    1 | 2 -> "one or two"
    n if n > 10 -> "big"
    _ -> "other"
}
```

If expressions use implicit values (line 2363):
```barnacle
let max = if a > b { a } else { b }
```

These examples don't use `return`, yet they produce values. The document never explicitly states the rule for when values are implicit vs when `return` is required.

**The apparent intent**: `if`, `match`, and block `{ }` expressions evaluate to the value of their last sub-expression. The `return` keyword is for early return from the *enclosing function*, not for producing values within expressions.

**Recommendation**: Add a clear paragraph in Section 11.4 or 11.13:

> **Expression vs Return**: `if`, `match`, and `{ }` blocks are expressions -- they evaluate to the value of their last sub-expression. The `return` keyword exits the enclosing *function*, not the enclosing block. In multi-statement function bodies, `return` is required to produce the function's value. Single-expression functions (and lambdas) are implicitly their value.
>
> ```barnacle
> // Expression: last sub-expression is the value
> let max = if a > b { a } else { b }
>
> // Function: explicit return required in multi-statement bodies
> fn compute(x: Int) -> Int {
>     let doubled = x * 2
>     return doubled + 1
> }
>
> // Lambda: single-expression is implicitly the value
> let double = x => x * 2
> ```

### 1.8 Minor issues in Section 9.2

**Line 1795:**
```barnacle
export fn print_length(s: &String) {
    Console.print(s: s.length.to_string())
}
```

Issues:
1. Should not be `export` -- this is a trivial example in the Memory Management section, not a public API
2. Missing `with Console` effect annotation -- since `Console.print` performs the `Console` effect, the function signature must declare it
3. Parameter name bug (covered in item 1.5)

**Corrected form:**
```barnacle
fn print_length(s: &String) with Console {
    Console.print(msg: s.length.to_string())
}
```

---

## 2. Design Concerns

These are areas where the design is internally consistent but raises questions about feasibility, completeness, or ergonomics. They should be addressed before or during early implementation.

### 2.1 Structural typing tension with Wasm Component Model

**The tension:**
- Barnacle uses structural typing with width subtyping (lines 1288-1324): `{x: Int, y: Int, z: Int}` can be used where `{x: Int, y: Int}` is expected
- WIT (and the Wasm Component Model) uses nominal typing: types are named, fields must match exactly at component boundaries

**The gap:** The document never explains how structural types are compiled to nominal WIT types at component boundaries. When a Barnacle function `export fn process(data: { name: String, age: Int }) -> ...` is compiled to a Wasm component, what WIT type does `{ name: String, age: Int }` become?

**Options:**
1. **Auto-generate WIT records** for each structural shape used at export/import boundaries. E.g., `record process-data { name: string, age: u64 }`. Risk: name collisions, proliferation of types.
2. **Require explicit `type` aliases at `export` boundaries**. E.g., `export type ProcessData { name: String, age: Int }` and require export functions to use named types. This is more restrictive but maps cleanly to WIT.
3. **Compiler-inserted field selection** at boundaries: when a wide type is passed to a narrow interface, the compiler generates code to project the relevant fields. This preserves width subtyping but adds overhead.

**Question**: Which approach does Barnacle intend? This affects both the type checker (boundary checking rules) and the code generator (WIT emission).

**Recommendation**: Add a subsection to Section 4.4 (WIT Generation from Effects) explaining the boundary compilation strategy. Option 2 (require named types at `export` boundaries) is most natural -- it aligns with the "explicit at boundaries, inferred internally" principle.

### 2.2 Tag system lacks tag algebra specification

The tag system (Section 8.5, lines 1357-1556) is one of Barnacle's most distinctive features, but the algebra governing tag behavior is underspecified.

**What is specified:**
- `tag(value, as: :tag)` -- adds a tag (accumulates with existing tags) (line 1380)
- `retag(value, as: :tag)` -- replaces ALL existing tags (line 1400)
- Tags can always be stripped (widened) (line 1549)

**What is missing:**

1. **Selective tag removal**: `retag()` replaces all tags, but there's no way to remove a single tag while preserving others. For example, if a value has `String & :validated & :sanitized` and a transformation invalidates the sanitization but not the validation, there's no `remove_tag(value, tag: :sanitized)` to produce `String & :validated`.

2. **Tag propagation through operations**: When you spread a tagged record (`{ ...tagged_record, name: "new" }`), do the spread fields retain their tags? The builder example (line 1495) suggests `tag()` is called explicitly after spread, but this is ambiguous:
   ```barnacle
   export fn with_url(req: HttpRequest, url: String) -> HttpRequest & :has_url {
       return tag(value: { ...req, url: :some { value: url } }, as: :has_url)
   }
   ```
   Does the `...req` spread preserve any tags that `req` might have (e.g., `:has_method`)?

3. **Tag flow through function calls**: If a function takes `String & :validated` and returns `String`, does the output retain the `:validated` tag? Or is it stripped because the return type says `String`? The document implies tags must be explicitly added, but this creates verbosity for functions that don't affect the tagged property.

4. **Tag intersection**: Two tagged types `T & :a` and `T & :b` -- can they be intersected to require `T & :a & :b`? This is implied by the multi-tag examples but never formally stated.

**Recommendation**: Add a "Tag Algebra" subsection defining:
- Tags are explicitly added via `tag()` and never implicitly propagated
- `retag()` replaces all tags; add `untag(value, tag:)` for selective removal
- Spread (`...`) does NOT propagate tags (the result is untagged unless `tag()` is called)
- Tag intersection in type positions (`T & :a & :b`) requires the value to carry all listed tags

### 2.3 Borrowing model is underspecified

**Section 9.2 (lines 1786-1803):**
```barnacle
// Explicit borrow with &
export fn print_length(s: &String) {
    Console.print(s: s.length.to_string())
}
```

**Line 1803:**
> No lifetimes or borrow checker. The compiler inserts reference count operations automatically.

**The confusion**: If reference counting handles everything automatically, what does `&` mean? Without a borrow checker, there's no static guarantee that a `&String` reference won't outlive the owned value. If `&` is just an optimization hint (skip the RC increment because the callee promises not to store it), the document should say so.

**Proposed clarification**: `&T` means "this parameter is a *borrowed* reference -- the callee will access the value but will not store it beyond the function's execution." The compiler uses this as an RC elision hint: no reference count increment on call, no decrement on return. This is safe without a borrow checker because:
1. The callee cannot assign `&T` to a field or global (type system prevents it)
2. The callee cannot return `&T` (return type must be `T`, not `&T`)
3. The callee can only pass `&T` down to other `&T` parameters (transitive non-escaping)

**Question**: Can closures capture `&T` values? If so, the closure could escape the borrow's scope. This needs specification.

### 2.4 `world extends` lacks override and conflict resolution semantics

**Section 3.3 (lines 449-471):**
```barnacle
world base {
    use barnacle:std/console with real_console
    lint barnacle:std/lints/deprecated
}

world dev extends base {
    entry dev_main
    use app:db/postgres with postgres { connection: "localhost:5432" }
}
```

**Unspecified behaviors:**
1. Can `dev extends base` override a handler from `base`? E.g., can `dev` rebind `Console` to a different handler?
2. Are lint rules additive? If `base` has `lint deprecated` and `dev` does not mention it, is the lint still active?
3. Can a lint be removed from a parent? E.g., `lint -barnacle:std/lints/deprecated` to explicitly disable it?
4. What if two extended worlds conflict? E.g., `world both extends a, b` where `a` and `b` provide different handlers for the same effect?

**Recommendation**: Add to Section 3.3:
- Child declarations override parent declarations for the same binding (last-writer-wins)
- Lint rules are inherited and additive; use `remove lint <name>` to exclude an inherited lint
- Diamond conflicts (multiple parents binding the same effect) are compilation errors requiring explicit resolution in the child world

### 2.5 No error recovery discussion for the parser

The document mentions rowan-based lossless CST (line 503) and Salsa incremental computation (line 544), but never discusses error recovery -- which is the most challenging aspect of building an LSP-quality parser.

**Why this matters**: During editing, source code is nearly always in an invalid state (incomplete expressions, unbalanced braces, missing semicolons). The parser must produce a useful CST even from malformed input, or the LSP will provide no feedback while the user types.

**Recommendation**: Add a paragraph in Section 4.1 (Stage Details, Parsing) noting:
- The parser will implement error recovery using synchronization tokens (closing braces, newlines at statement boundaries, keywords like `fn`, `type`, `effect`)
- Error nodes are first-class CST nodes, preserving malformed syntax for IDE display
- This follows rust-analyzer's approach where every byte of source text is represented in the CST

### 2.6 Closures and effects interaction is unspecified

**Section 2.4 (line 155):**
```barnacle
fn map<T, U, E>(self list: List<T>, transform: (T) -> U with E) -> List<U> with E {
    // works with pure functions, effectful functions, async functions -- all the same
}
```

Closures can be effect-polymorphic, but the document doesn't address two important interactions:

1. **Mutable captures under multi-shot handlers**: If a closure captures a `mut` binding and is used in a context with a multi-shot handler (Tier 3 -- generators, backtracking), the handler may resume the continuation multiple times. Each resumption would share the same mutable binding, leading to unexpected aliasing.

   ```barnacle
   let mut counter = 0
   let inc = () => { counter = counter + 1; return counter }
   // Under a multi-shot handler, multiple resumes share `counter`
   ```

2. **Effect inference on closures**: Is the effect set `E` in `(T) -> U with E` inferred from the closure body? If `transform: u => Database.query(sql: "...")` is passed to `map`, does the compiler infer `E = {Database}`?

**Recommendation**: Add a note:
- Effect inference on closures: yes, `E` is inferred by unifying with the effects performed in the closure body
- `mut` captures in multi-shot handler contexts: either deep-copy mutable captures on each resume (Koka's approach), or prohibit `mut` captures in closures that cross multi-shot handler boundaries (simpler but more restrictive). This should be an Open Question if not yet decided.

### 2.7 `use` (Disposable pattern) lacks formal effect mapping

**Section 11.15 (line 2431):**
> The `use` keyword provides sugar over effect handlers for resource cleanup

But the document never shows what effect or desugaring `use` maps to. There is no `Disposable` effect or protocol defined anywhere.

**Proposed desugaring:**
```barnacle
// Source
use file = open_file(path: "config.toml")
let contents = read_file(file: file)
// ... file automatically closed here

// Desugars to (conceptual):
let file = open_file(path: "config.toml")
let result = handle {
    let contents = read_file(file: file)
    // ... rest of block
} with {
    // On scope exit (normal or Fail), call close
    finally -> close(file: file)
}
```

**Question**: Does `use` introduce a new effect, a new handler, or is it purely syntactic sugar that calls a `close`/`dispose` method? If the latter, how does the compiler know *which* cleanup function to call?

**Recommendation**: Define either:
- A `Disposable` protocol/trait: `trait Disposable { fn dispose(self) -> () }` and `use` calls `dispose` on scope exit
- Or a naming convention: `use file = open_file(...)` calls `close_file(file)` (matching `open_file` -> `close_file`)

The trait approach is more general and should be preferred once the trait system is designed.

---

## 3. Pattern Reuse Opportunities

These are places where the document repeats similar patterns that could be unified or documented as reusable abstractions.

### 3.1 Unify handler definition syntax into one canonical grammar

The document uses three distinct handler forms:

**Named handler (line 114):**
```barnacle
handler real_console: Console {
    print(msg) -> resume(wasi_print(msg: msg))
}
```

**Parameterized handler function (line 206):**
```barnacle
fn or_default<T, E>(default: T): Handler<Fail<E>, T> {
    handler {
        fail(_) -> default
    }
}
```

**Typed handler with return annotation (line 201):**
```barnacle
handler to_result<T, E>: Fail<E> -> Result<T, E> {
    fail(error) -> :err { error }
}
```

These can be unified under one grammar:

```
HandlerDecl = "handler" [name] [GenericParams] [Params] ":" EffectType ["->" ReturnType] "{" OperationClauses "}"
```

Where:
- Omitting `name` creates an anonymous handler (for inline use)
- Omitting `GenericParams` uses inference
- Omitting `Params` creates a non-parameterized handler
- Omitting `ReturnType` means the handler doesn't change the expression's type
- The parameterized handler function form is syntactic sugar for a function that returns a handler value

**Recommendation**: Add this unified grammar to Section 2.3 or 11.14 and show how each existing variant is a special case.

### 3.2 Consolidate the "explicit at boundaries, inferred internally" meta-principle

This pattern appears in at least three domains but is never stated as a unified principle:

| Domain | At boundaries (`export`) | Internally (private) |
|---|---|---|
| **Effects** (line 92) | Effects must be declared | Effects are inferred |
| **Types** (implied) | Types should be explicit | Types can be inferred |
| **Tags** (line 1375) | Tags are required by consumers | Tags are accumulated by producers |

**Recommendation**: State this as an explicit meta-principle in Section 1 (Key Design Principles):

> **Principle: Explicit at boundaries, inferred internally.** Public APIs (`export`) require explicit type, effect, and tag annotations. Internal implementations use inference. This principle applies uniformly to types (explicit return types on `export fn`), effects (explicit `with Effect` on `export fn`), and tags (explicit tag requirements on `export fn` parameters). This ensures API contracts are clear while keeping implementation code concise.

Then reference this principle from Section 2.2 (effect inference), Section 8 (type inference), and Section 8.5 (tags).

### 3.3 Document the "Evidence Pipeline" pattern

Three examples in Section 8.5 are instances of the same abstract pattern:

**Validation evidence (lines 1411-1427):**
```
raw input -> validate_email() -> String & :email -> validate_non_empty() -> String & :non_empty -> create_user()
```

**Data provenance (lines 1469-1481):**
```
raw data -> clean() -> & :cleaned -> dedupe() -> & :deduped -> normalize() -> & :normalized -> analyze()
```

**Taint tracking (lines 1518-1532):**
```
user input & :tainted -> sanitize_html() -> & :sanitized -> render_html()
```

**The abstract pattern (Evidence Pipeline):**
1. **Acquire**: Obtain a value, optionally with an initial tag (e.g., `:tainted` for user input)
2. **Transform**: Pass through a pipeline of functions that validate/process the value, each adding evidence tags and potentially performing `Fail<E>` for invalid inputs
3. **Consume**: Pass to a function that requires specific tags, guaranteeing the value has passed all necessary stages

**Recommendation**: Add an "Evidence Pipeline" subsection to Section 8.5 that names this pattern, shows the abstract structure, and notes that the validation, provenance, and taint examples are all instances.

### 3.4 Define a base `compiler-plugin` WIT world

Three plugin worlds share the same imports:

**`lint-rule` (lines 741-749):**
```wit
world lint-rule {
    import syntax;
    import types;
    import symbols;
    // ...
}
```

**`code-action` (lines 875-883):**
```wit
world code-action {
    import syntax;
    import types;
    import symbols;
    // ...
}
```

**Code generators** (implied but not formalized as a world)

**Recommendation**: Factor out the shared interface:
```wit
world compiler-plugin {
    import syntax;
    import types;
    import symbols;
}

world lint-rule includes compiler-plugin {
    export name: func() -> string;
    export check-function: func(id: func-id) -> list<diagnostic>;
    export check-type: func(id: type-id) -> list<diagnostic>;
    export check-module: func(id: module-id) -> list<diagnostic>;
}

world code-action includes compiler-plugin {
    export title: func() -> string;
    export applicable: func(ctx: action-context) -> bool;
    export apply: func(ctx: action-context) -> list<text-edit>;
}

world code-generator includes compiler-plugin {
    import filesystem;
    export generate: func(module: module-id) -> list<generated-file>;
}
```

This reduces duplication and makes the plugin API surface discoverable.

### 3.5 Consider `try { expr }` sugar for `handle expr with to_result`

The `Fail<E>` to `Result<T, E>` conversion is the most common handler usage:

```barnacle
let result = handle parse_int("abc") with to_result
// result: Result<Int, ParseError>
```

Since this pattern is ubiquitous (any time effectful code interfaces with a `Result`-expecting API), consider dedicated syntax:

```barnacle
let result = try parse_int("abc")
// Desugars to: handle parse_int("abc") with to_result
// result: Result<Int, ParseError>
```

**Benefits**: Reduces boilerplate, provides a familiar keyword for error handling, makes the `Fail<E>` -> `Result` bridge ergonomic.

**Risk**: `try` is a loaded keyword in many languages. It might confuse users who expect `try/catch` semantics.

**Recommendation**: Add this as an Open Question in Section 14 with the pros/cons above. If adopted, it would also be consistent with the reverse direction: `try!` or `unwrap` for `Result` -> `Fail<E>`.

---

## 4. Missing Specifications

These items are referenced or implied by the document but never formally specified.

### 4.1 No formal grammar

The syntax is described entirely by example in Section 11. There is no BNF, PEG, or other formal grammar. This makes it impossible to:
- Verify that syntax examples are consistent with each other
- Determine parser ambiguities
- Build a parser from the specification

**See Appendix A** for a proposed formal grammar sketch.

### 4.2 No operator precedence table

Section 11.16 (lines 2456-2482) lists operators but provides no precedence or associativity rules.

**Critical ambiguity**: `&` is used for both bitwise AND (expression-level) and tag composition (type-level). `|` is used for both bitwise OR (expression-level) and union types (type-level). The document states this at lines 2478-2481 but provides no disambiguation rule beyond "in types" vs "in expressions." What about contexts where both are valid?

**Recommendation**: Add a precedence table:

| Precedence | Operators | Associativity | Notes |
|---|---|---|---|
| 1 (lowest) | `\|>` | Left | Pipeline |
| 2 | `\|\|` | Left | Logical OR |
| 3 | `&&` | Left | Logical AND |
| 4 | `==` `!=` | None | Comparison (non-chaining) |
| 5 | `<` `>` `<=` `>=` | None | Ordering |
| 6 | `\|` | Left | Bitwise OR |
| 7 | `^` | Left | Bitwise XOR |
| 8 | `&` | Left | Bitwise AND |
| 9 | `<<` `>>` | Left | Bitwise shift |
| 10 | `+` `-` `++` | Left | Additive, string concat |
| 11 | `*` `/` `%` | Left | Multiplicative |
| 12 | `!` `~` `-` (prefix) | Right | Unary |

In type contexts: `&` means tag intersection, `|` means union. The parser distinguishes by syntactic position (type annotations vs expressions).

### 4.3 `for` loop iteration protocol is undefined

**Lines 2384-2401**: `for` loops are used but no iteration protocol exists:
```barnacle
for user in users { ... }
for { item, index } in users.enumerate() { ... }
```

**Questions:**
- What types can appear after `in`? Only `List<T>`? Any `Iterable`?
- What is `enumerate()`? A method? A function with `self` parameter?
- How does the for loop desugar? Into a while loop with `next()`? Into `map`?

**Recommendation**: Define an `Iterable` trait (when the trait system is designed) or specify that `for x in collection` desugars to specific function calls. In the interim, specify that `for` works with `List<T>` and any type that provides `fn next(self) -> Option<T>`.

### 4.4 Equality semantics are undefined

**Line 327**: `if result == 0` -- integer equality
**Line 1679**: `trait Eq { fn eq(self, other: Self) -> Bool }` -- listed as future work

**Questions:**
- Is `==` structural equality for record types? (i.e., `{x: 1, y: 2} == {x: 1, y: 2}` is `true`?)
- Is `==` defined for all types, or only primitives until the trait system exists?
- Does width subtyping affect equality? Is `{x: 1, y: 2} == {x: 1, y: 2, z: 3}` a type error or `true`?

**Note**: The `Eq` trait at line 1679 uses positional parameters: `fn eq(self, other: Self)`. The `other` parameter is positional, which contradicts the "all parameters are named" rule (line 2094). This should be `fn eq(self, other: Self)` where `other` is named at call sites.

**Recommendation**: Specify structural equality for primitives and records. For custom equality, defer to the trait system. Width-subtyped comparisons should be type errors (both sides must have the same type).

### 4.5 `Byte` vs `UInt8` relationship

**Line 1233**: `Byte` -- "Single byte (8-bit unsigned)"
**Line 1247**: `UInt8` -- listed as a sized integer variant
**Line 2199**: `let small: UInt8 = 255`
**Line 2206**: `let byte_val: Byte = 0xFF`

Both are 8-bit unsigned integers. Are they the same type? Aliases? Distinct types with explicit conversion?

**Recommendation**: Specify that `Byte` is a type alias for `UInt8`. This is the simplest option and avoids unnecessary conversion overhead.

### 4.6 String interpolation protocol

**Lines 2166-2167:**
```barnacle
let greeting = "Hello, {name}!"
let calculation = "2 + 2 = {2 + 2}"
```

**Question**: What types can appear inside `{}`? If `name` is a `User` (struct), what string representation is used? Is there a `Show`/`Display` trait?

**Recommendation**: Until the trait system is designed, restrict string interpolation to types with built-in string representations: `String`, `Int`, `Float`, `Bool`, and atoms. For other types, require explicit conversion: `"User: {user.name}"` rather than `"User: {user}"`.

### 4.7 Numeric conversion semantics

**Questions:**
- Are conversions between `Int` and `Int32` implicit or explicit?
- What is the overflow behavior for `Int32`, `UInt8`, etc.? Trap? Wrap?
- Is `Int` + `Float` allowed? Does it produce `Float`?

**Recommendation**: All numeric conversions should be explicit via named functions: `to_int32(value: x)`, `to_float(value: x)`, etc. Overflow should trap (panic) by default, with an opt-in wrapping version: `wrapping_add(a: x, b: y)`. No implicit int-to-float conversion.

### 4.8 `mut` interaction with structural typing

**Question**: Is `mut` a property of bindings only, or can record fields be mutable?

```barnacle
let mut x = { name: "Alice" }
x.name = "Bob"  // Is this allowed?

type User { name: mut String }  // Is this valid syntax?
```

If fields can be mutable, how does this interact with width subtyping? Can a `{ name: mut String }` be passed where `{ name: String }` is expected?

**Recommendation**: Specify that `mut` applies to bindings only. Record fields are always immutable; to "update" a field, create a new record with spread: `{ ...user, name: "Bob" }`. This is consistent with the Perceus copy-on-write optimization (Section 9.3) and eliminates mutable aliasing concerns.

### 4.9 Lambda parameter naming exception

**Line 2094**: "All function parameters are named (except `self` which is positional for pipeline)"

But lambda parameters are also positional:
```barnacle
x => x * 2
(a, b) => a + b
```

This is an implicit exception that's never stated.

**Recommendation**: Amend to: "All declared function parameters are named (except `self` which is positional for pipeline). Lambda parameters are positional, since lambdas are anonymous and typically short-lived."

### 4.10 Field shorthand in record construction

**Line 202**: `fail(error) -> :err { error }` -- uses `{ error }` shorthand
**Line 2263**: `let { name, age } = user` -- shorthand in destructuring

Shorthand `{ x }` meaning `{ x: x }` is used in construction (line 202) but only formally described for destructuring (line 2263).

**Recommendation**: Formally specify: "In record construction, `{ name }` is shorthand for `{ name: name }`, binding the field to the variable of the same name. This is the dual of destructuring shorthand."

---

## Appendix A: Formal Grammar Sketch

This is a semi-formal grammar using PEG-like notation. It covers the major syntactic constructs defined in the design document. Terminals are in quotes, non-terminals are in PascalCase, `?` means optional, `*` means zero-or-more, `+` means one-or-more, `/` means ordered choice.

### A.1 Top-Level Declarations

```peg
Program         = Declaration*

Declaration     = ImportDecl
                / ExportDecl
                / FunctionDecl
                / TypeDecl
                / EnumDecl
                / EffectDecl
                / HandlerDecl
                / WorldDecl
                / AnnotationDecl
                / ConstDecl

ImportDecl      = "import" ImportCategory ImportNames "from" ModulePath ("as" Identifier)?
ImportCategory  = "type" / "effect" / "fn" / "handler" / "annotation" / "module"
ImportNames     = Identifier ("," Identifier)*
ModulePath      = Identifier ("/" Identifier)*

ExportDecl      = "export" (FunctionDecl / TypeDecl / EnumDecl / EffectDecl / ReExport)
ReExport        = ImportCategory ImportNames "from" ModulePath
```

### A.2 Function Declarations

```peg
FunctionDecl    = Annotation* "fn" Identifier GenericParams? "(" ParamList? ")" ReturnType? EffectAnnot? Block

ParamList       = Param ("," Param)*
Param           = SelfParam / NamedParam
SelfParam       = "self" Identifier ":" Type
NamedParam      = Identifier ":" Type ("=" Expression)?
ReturnType      = "->" Type
EffectAnnot     = "with" EffectList
EffectList      = EffectRef ("," EffectRef)*
EffectRef       = QualifiedName GenericArgs?

GenericParams   = "<" GenericParam ("," GenericParam)* ">"
GenericParam    = Identifier
GenericArgs     = "<" Type ("," Type)* ">"
```

### A.3 Type Declarations

```peg
TypeDecl        = "type" Identifier GenericParams? "{" FieldList "}"
FieldList       = Field ("," Field)* ","?
Field           = Identifier ":" Type

EnumDecl        = "enum" Identifier GenericParams? "{" VariantList "}"
VariantList     = Variant ("," Variant)* ","?
Variant         = Atom ("{" FieldList "}")?

AnnotationDecl  = "annotation" Identifier "{" FieldList "}"
```

### A.4 Effect and Handler Declarations

```peg
EffectDecl      = "effect" Identifier GenericParams? "{" OperationList "}"
OperationList   = OperationDef+
OperationDef    = Identifier "(" ParamList? ")" "->" Type

HandlerDecl     = "handler" Identifier? GenericParams? HandlerParams? ":" EffectRef HandlerReturn? "{" HandlerClauses "}"
HandlerParams   = "(" ParamList ")"
HandlerReturn   = "->" Type
HandlerClauses  = HandlerClause+
HandlerClause   = Identifier "(" PatternList? ")" "->" Expression
                / LetBinding  // for local state in handlers

PatternList     = Pattern ("," Pattern)*
```

### A.5 World Declarations

```peg
WorldDecl       = "world" Identifier ("extends" Identifier ("," Identifier)*)? "{" WorldBody "}"
WorldBody       = WorldItem*
WorldItem       = EntryDecl
                / UseDecl
                / HandleDecl
                / LintDecl
                / GenerateDecl
                / ActionDecl
                / WorldOption

EntryDecl       = "entry" Identifier
UseDecl         = "use" ModulePath "with" Identifier (RecordLiteral)?
HandleDecl      = "handle" EffectRef "with" Identifier (RecordLiteral)?
LintDecl        = "lint" ModulePath (RecordLiteral)?
GenerateDecl    = "generate" ModulePath (RecordLiteral)?
ActionDecl      = "action" ModulePath
WorldOption     = Identifier ":" Expression
```

### A.6 Expressions

```peg
Expression      = PipelineExpr

PipelineExpr    = LogicalOrExpr ("|>" FunctionCall)*

LogicalOrExpr   = LogicalAndExpr ("||" LogicalAndExpr)*
LogicalAndExpr  = CompareExpr ("&&" CompareExpr)*
CompareExpr     = BitwiseOrExpr (CompareOp BitwiseOrExpr)?
CompareOp       = "==" / "!=" / "<" / ">" / "<=" / ">="

BitwiseOrExpr   = BitwiseXorExpr ("|" BitwiseXorExpr)*
BitwiseXorExpr  = BitwiseAndExpr ("^" BitwiseAndExpr)*
BitwiseAndExpr  = ShiftExpr ("&" ShiftExpr)*
ShiftExpr       = AdditiveExpr (("<<" / ">>") AdditiveExpr)*
AdditiveExpr    = MultExpr (("+" / "-" / "++") MultExpr)*
MultExpr        = UnaryExpr (("*" / "/" / "%") UnaryExpr)*

UnaryExpr       = ("!" / "~" / "-") UnaryExpr / PostfixExpr
PostfixExpr     = PrimaryExpr (FieldAccess / FunctionCall)*
FieldAccess     = "." Identifier
FunctionCall    = "(" ArgList? ")"

PrimaryExpr     = Identifier
                / Literal
                / AtomExpr
                / RecordLiteral
                / ListLiteral
                / LambdaExpr
                / IfExpr
                / MatchExpr
                / HandleExpr
                / BlockExpr
                / ComptimeExpr
                / "(" Expression ")"

ArgList         = NamedArg ("," NamedArg)*
NamedArg        = Identifier ":" Expression
```

### A.7 Literals

```peg
Literal         = IntLiteral / FloatLiteral / StringLiteral / BoolLiteral / UnitLiteral

IntLiteral      = [0-9] ([0-9] / "_")*
                / "0x" [0-9a-fA-F] ([0-9a-fA-F] / "_")*
                / "0b" [01] ([01] / "_")*
FloatLiteral    = [0-9]+ "." [0-9]+ (("e" / "E") ("+" / "-")? [0-9]+)?
StringLiteral   = SimpleString / MultiLineString / RawString
SimpleString    = '"' (StringChar / Interpolation)* '"'
MultiLineString = '"""' (StringChar / Interpolation | Newline)* '"""'
RawString       = 'r"' [^"]* '"'
Interpolation   = "{" Expression "}"
BoolLiteral     = "true" / "false"
UnitLiteral     = "()"

AtomExpr        = Atom (RecordLiteral)?
Atom            = ":" Identifier
```

### A.8 Compound Expressions

```peg
RecordLiteral   = "{" RecordEntries? "}"
RecordEntries   = RecordEntry ("," RecordEntry)* ","?
RecordEntry     = SpreadEntry / FieldEntry / ShorthandEntry / DeepUpdateEntry / MapEntry
SpreadEntry     = "..." Expression
FieldEntry      = Identifier ":" Expression
ShorthandEntry  = Identifier                          // { name } == { name: name }
DeepUpdateEntry = Identifier ("." Identifier)+ ":" Expression
MapEntry        = (StringLiteral / AtomExpr / ComputedKey) ":" Expression
ComputedKey     = "[" Expression "]"

ListLiteral     = "[" (ListEntry ("," ListEntry)* ","?)? "]"
ListEntry       = SpreadEntry / Expression

LambdaExpr      = LambdaParams "=>" LambdaBody
LambdaParams    = Identifier                          // single param
                / "(" (Identifier ("," Identifier)*)? ")"  // multiple params
LambdaBody      = Expression / Block

IfExpr          = "if" Expression Block ("else" "if" Expression Block)* ("else" Block)?
MatchExpr       = "match" Expression "{" MatchArm+ "}"
MatchArm        = Pattern Guard? "->" Expression

HandleExpr      = "handle" Expression "with" Expression
ComptimeExpr    = "comptime" Block
```

### A.9 Statements and Blocks

```peg
Block           = "{" Statement* Expression? "}"

Statement       = LetBinding
                / Assignment
                / ForLoop
                / WhileLoop
                / UseBinding
                / ReturnStmt
                / BreakStmt
                / ContinueStmt
                / Expression

LetBinding      = "let" "mut"? Pattern (":" Type)? "=" Expression
Assignment      = LValue "=" Expression
LValue          = Identifier ("." Identifier)*

UseBinding      = "use" Identifier "=" Expression
ReturnStmt      = "return" Expression
BreakStmt       = "break"
ContinueStmt    = "continue"

ForLoop         = "for" Pattern "in" Expression Block
WhileLoop       = "while" Expression Block
```

### A.10 Patterns

```peg
Pattern         = LiteralPattern
                / AtomPattern
                / RecordPattern
                / ListPattern
                / IdentifierPattern
                / WildcardPattern
                / OrPattern

LiteralPattern  = IntLiteral / FloatLiteral / StringLiteral / BoolLiteral
AtomPattern     = Atom (RecordPattern)?
RecordPattern   = "{" RecordPatEntry ("," RecordPatEntry)* ","? ("," "..." Identifier)? "}"
RecordPatEntry  = Identifier (":" Pattern)?           // { name } or { name: pat }
                / Identifier ("." Identifier)+ ("as" Identifier)?  // deep destructure
ListPattern     = "[" (Pattern ("," Pattern)* ("," "..." Identifier)?)? "]"
IdentifierPattern = Identifier
WildcardPattern = "_"
OrPattern       = Pattern "|" Pattern
Guard           = "if" Expression
```

### A.11 Types

```peg
Type            = TaggedType

TaggedType      = UnionType ("&" TagRef)*
TagRef          = Atom

UnionType       = BaseType ("|" BaseType)*

BaseType        = FunctionType
                / RecordType
                / NamedType
                / BorrowType
                / UnitType
                / NeverType
                / "(" Type ")"

FunctionType    = "(" TypeList? ")" "->" Type EffectAnnot?
TypeList        = Type ("," Type)*
RecordType      = "{" FieldList "}"
NamedType       = QualifiedName GenericArgs?
BorrowType      = "&" Type
UnitType        = "()"
NeverType       = "Never"

QualifiedName   = Identifier ("." Identifier)*
```

### A.12 Constants and Annotations

```peg
ConstDecl       = "const" Identifier ":" Type "=" Expression

Annotation      = "@" Identifier RecordLiteral?
```

### A.13 Notes on Grammar

1. **Newline sensitivity**: Statements are separated by newlines. Semicolons are optional separators. Expressions that span multiple lines use standard continuation rules (open bracket, operator at end of line, etc.).

2. **Keyword disambiguation**: `&` and `|` are operators in expression context and type constructors in type context. The parser uses syntactic position to disambiguate: after `:` in a parameter/field/return type annotation, the parser enters type-parsing mode.

3. **Record vs Map literal**: The parser distinguishes by key type. Identifier keys (`{ name: "Alice" }`) produce record types. Expression keys -- string literals (`{ "name": "Alice" }`), atoms (`{ :key: value }`), or computed keys (`{ [expr]: value }`) -- produce map types.

4. **Handler clause binding**: Handler operation clauses use positional binding: `print(msg) -> ...` binds the first parameter of the `print` operation to the name `msg`. This is a pattern-matching context, not a call site.

5. **Expression-bodied constructs**: `if`, `match`, `handle`, and `{ }` blocks evaluate to their last sub-expression. `return` exits the enclosing function.

---

**End of Review**
