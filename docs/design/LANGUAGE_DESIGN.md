# Barnacle Language Design Document

**Version:** 1.0  
**Last Updated:** 2026-02-12  
**Status:** Living Document

---

## Table of Contents

1. [Project Vision & Goals](#1-project-vision--goals)
2. [Algebraic Effects — The Core Abstraction](#2-algebraic-effects--the-core-abstraction)
3. [The `world` Keyword — Configuration as Code](#3-the-world-keyword--configuration-as-code)
4. [Compilation Pipeline](#4-compilation-pipeline)
5. [Compiler Reflection & Metaprogramming](#5-compiler-reflection--metaprogramming)
6. [Dead Code / Liveness Analysis](#6-dead-code--liveness-analysis)
7. [Code Generation / Cross-Language Projection](#7-code-generation--cross-language-projection)
8. [Type System](#8-type-system)
9. [Memory Management](#9-memory-management)
10. [Module System & Imports](#10-module-system--imports)
11. [Syntax Overview](#11-syntax-overview)
12. [Standard Library Architecture](#12-standard-library-architecture)
13. [Tooling & Developer Experience](#13-tooling--developer-experience)
14. [Open Questions & Future Work](#14-open-questions--future-work)
15. [References & Influences](#15-references--influences)

---

## 1. Project Vision & Goals

Barnacle is a new programming language that compiles to WebAssembly Components. Its core thesis:

- **Algebraic effects as the universal abstraction** — side effects (I/O, async, errors, state) are tracked in the type system via effects. No function coloring. No special async/await syntax. No split between checked/unchecked exceptions.
- **The Wasm Component Model is the runtime model** — effects map to WIT imports, handlers map to WIT exports, composition maps to component linking. The effect system and the component model are two views of the same thing.
- **Compile-time metaprogramming via the same effect system** — lint rules, refactors, code generators, and analysis tools are written in Barnacle itself, compiled to Wasm components, and run inside the compiler/LSP as sandboxed plugins.
- **First-class LSP** — the compiler is built on Salsa (incremental computation) from day one, meaning the LSP and the compiler share the same query database. The LSP is not an afterthought.
- **Complete dead code analysis** — no side effects at module scope + explicit entry points in `world` declarations = provably sound liveness analysis.

### Key Design Principles

1. **Explicitness at boundaries, inference internally** — Public APIs must be explicit about their effects. Internal implementation details are inferred.
2. **Zero-cost abstractions** — The common case (tail-resumptive effects) compiles to direct function calls with no overhead.
3. **Composability over batteries-included** — Core primitives are orthogonal and compose well. Specific solutions live in the standard library or ecosystem.
4. **Wasm-first, not Wasm-compatible** — The language is designed around Wasm's capabilities and constraints, not retrofitted to target it.
5. **Make invalid states unrepresentable** — Use the type system to prevent bugs at compile time.

### Key References/Influences

- **Koka** — algebraic effects, evidence-passing compilation, Perceus reference counting
- **Eff** — academic reference for algebraic effects
- **OCaml 5** — production effect handlers via stack switching
- **Effekt** — multi-backend effect compilation
- **Unison** — "abilities" (their name for effects)
- **Grain** — purpose-built Wasm language, ANF IR, Binaryen backend
- **AssemblyScript** — TypeScript-like Wasm language, tight Wasm type mapping
- **Spin (Fermyon)** — WIT-first design, component composition, WASI integration
- **Rust (rust-analyzer)** — Salsa-based incremental compilation, `rowan` lossless CST
- **LINQ (C#)** — expression capture for data pipelines

---

## 2. Algebraic Effects — The Core Abstraction

Algebraic effects are the fundamental building block of Barnacle. They provide a unified way to handle side effects, errors, async operations, state, and more — all without special syntax or language features.

### 2.1 Declaring Effects

Effects are declared as named sets of operations with typed inputs and outputs:

```barnacle
effect Console {
    print(msg: String) -> ()
    read_line() -> String
}

effect FileSystem {
    read_file(path: String) -> Result<String, IoError>
    write_file(path: String, data: String) -> Result<(), IoError>
}

effect Async {
    suspend() -> ()
    spawn<T>(task: () -> T) -> Handle<T>
    await<T>(handle: Handle<T>) -> T
}
```

An effect declaration defines a set of operations that can be **performed** (called) but whose implementation is provided separately via **handlers**.

### 2.2 Performing Effects

Functions that perform effects gain those effects in their signature. Effects are **inferred** on private/internal functions and **required to be explicit** on `pub` functions:

```barnacle
// Public — effects must be declared (this is your API contract)
pub fn greet(name: String) -> () with Console {
    Console.print("Hello, " ++ name ++ "!")
}

// Private — effects inferred by the compiler
fn helper() {
    Console.print("internal")
}
// compiler infers: with Console
```

This design eliminates the verbosity of checked exceptions (Java) while maintaining the safety of effect tracking at API boundaries.

### 2.3 Handling Effects

Handlers give meaning to effects. The `handle expr with handler` syntax wraps an expression and intercepts its effects:

```barnacle
handler real_console: Console {
    print(msg) -> resume(wasi_print(msg))
    read_line() -> resume(wasi_read_line())
}

handler mock_console: Console {
    let log = mut []
    print(msg) -> { log.push(msg); resume(()) }
    read_line() -> resume("mocked input")
}

// Same function, different handlers:
handle main() with real_console    // production
handle main() with mock_console    // testing
```

`resume` is the key primitive — the handler decides whether to resume, how many times, and with what value. This enables:

- **Exceptions**: don't call `resume` → early exit
- **Async**: suspend, store continuation, resume later
- **Generators**: resume multiple times
- **State**: thread state through resume calls
- **Logging/monitoring**: intercept, record, then resume
- **Transactions**: buffer operations, commit/rollback

### 2.4 No Function Coloring

There is no async/sync split. `Async` is just another effect, same as `Console` or `Database`. A function can call any other function as long as its effect set is a superset of the callee's:

```barnacle
fn a() with Console { ... }
fn b() with Async { ... }
fn c() with Console, Async {
    a()  // ✅ Console ⊆ {Console, Async}
    b()  // ✅ Async ⊆ {Console, Async}
}
```

Higher-order functions are **effect-polymorphic** — they're generic over the effect set:

```barnacle
fn map<T, U, E>(list: List<T>, f: (T) -> U with E) -> List<U> with E {
    // works with pure functions, effectful functions, async functions — all the same
}
```

This eliminates the need for `map` + `async_map` + `try_map` + `async_try_map` etc. One `map` function works for all cases.

### 2.5 Errors as Effects (Fail)

Errors are just the `Fail<E>` effect:

```barnacle
effect Fail<E> {
    fail(error: E) -> Never
}
```

This replaces `try`/`catch`, `Result<T,E>`, `throws`, and the `?` operator — all are sugar or handlers over `Fail`:

```barnacle
// Creating an error — just perform the effect
fn parse_int(s: String) -> I32 {
    match try_parse(s) {
        Some(n) -> n
        None -> Fail.fail(ParseError { input: s })
    }
}
// compiler infers: with Fail<ParseError>

// Handling — wrap in handle at any point
let result = handle parse_int("abc") with to_result
// result: Result<I32, ParseError>

let value = handle parse_int("abc") with or_default(0)
// value: I32 (always succeeds)
```

Effect inference means you DON'T need to annotate every function in the call chain with `Fail<E>` — only public boundaries and the origin. This avoids the Java checked exceptions problem. Internal functions infer their full effect set.

`handle` is an expression — it can go anywhere: in a let binding, loop body, function argument, match arm, etc. Wherever you place it, the `Fail` effect is consumed at that point.

#### Standard Handlers for Fail

The standard library provides common handlers:

```barnacle
// Convert to Result type
handler to_result<T, E>: Fail<E> -> Result<T, E> {
    fail(error) -> Result.Err(error)
}

// Provide a default value
fn or_default<T, E>(default: T): Handler<Fail<E>, T> {
    handler {
        fail(_) -> default
    }
}

// Panic on error
handler to_panic<E>: Fail<E> {
    fail(error) -> panic(format("Unhandled error: {}", error))
}

// Transform error type
fn map_error<E1, E2>(f: (E1) -> E2): Handler<Fail<E1>, Fail<E2>> {
    handler {
        fail(error) -> Fail.fail(f(error))
    }
}
```

### 2.6 Interchangeable Effects (Abstraction Over Implementation)

Effects enable programming against abstract interfaces. Multiple handlers can satisfy the same effect:

```barnacle
effect Database {
    execute(query: String, params: List<Value>) -> Result<Rows, DbError>
}

handler postgres(conn: String): Database { ... }
handler mysql(conn: String): Database { ... }
handler in_memory_db(): Database { ... }  // for tests — pure, no I/O

// Same code, different backends:
handle app() with postgres("postgres://...")
handle app() with in_memory_db()
```

This is similar to dependency injection or interface-based programming, but with full type safety and zero runtime overhead.

### 2.7 Pure Imports vs Effects

Modules can export both pure functions and effect declarations. Pure functions are just functions — no ceremony, no effects, direct calls:

```barnacle
// std/fs.barnacle
effect FileSystem { ... }              // effect
pub fn join_path(a: String, b: String) -> String { ... }  // pure function
pub fn extension(path: String) -> Option<String> { ... }  // pure function

// User code
import std/fs.{FileSystem, join_path}
fn process() with FileSystem {
    let path = join_path(dir, "config.toml")  // pure — direct call
    FileSystem.read_file(path)                 // effect — needs handler
}
```

Pure functions compose freely. Effectful operations require handlers. This distinction is enforced at compile time.

### 2.8 Effect Subtyping and Polymorphism

Effects support subtyping through effect sets:

```barnacle
// Empty effect set = pure function
fn pure() -> I32 { 42 }

// Single effect
fn one() -> () with Console { Console.print("one") }

// Multiple effects
fn two() -> () with Console, FileSystem { 
    Console.print("reading")
    FileSystem.read_file("config")
}

// Effect polymorphism — generic over any effect set
fn generic<E>(f: () -> () with E) -> () with E {
    f()
}

// Can call with any effect set
generic(pure)       // E = {}
generic(one)        // E = {Console}
generic(two)        // E = {Console, FileSystem}
```

---

## 3. The `world` Keyword — Configuration as Code

A `world` declaration is the single source of truth for how code is built, checked, and run. It replaces build files, configuration files, and CLI flags with a first-class language construct.

### 3.1 Basic World Declaration

```barnacle
world server {
    // Entry point
    entry main

    // Effect handler bindings
    use barnacle:std/console with real_console
    use app:db/postgres with postgres { connection: env("DATABASE_URL") }

    // Lint rules (Wasm components run at compile time)
    lint barnacle:std/lints/deprecated
    lint app:lints/api_standards {
        min_api_version: "2.0",
        require_rate_limit: true,
    }

    // Code generation
    generate app:codegen/ts_client {
        module: app:routes,
        output_dir: "./generated/ts",
    }

    // Code actions (available in LSP)
    action app:refactors/extract_effect
}
```

### 3.2 Multiple Worlds in One Project

Different worlds for different purposes:

```barnacle
world test {
    entry test_runner
    use barnacle:std/console with mock_console
    use app:db/mock with in_memory_db
    lint barnacle:std/lints/deprecated { level: Error }
}

world benchmark {
    entry bench_main
    use barnacle:std/console with quiet_console
    use app:db/postgres with postgres { connection: env("BENCH_DB_URL") }
    optimize: true
}

world production {
    entry main
    use barnacle:std/console with real_console
    use app:db/postgres with postgres { connection: env("DATABASE_URL") }
    lint barnacle:std/lints/deprecated { level: Warn }
    optimize: true
    debug_info: false
}
```

### 3.3 World Composition

Worlds can extend or compose with others:

```barnacle
world base {
    use barnacle:std/console with real_console
    lint barnacle:std/lints/deprecated
}

world dev extends base {
    entry dev_main
    use app:db/postgres with postgres { connection: "localhost:5432" }
    optimize: false
    debug_info: true
}

world prod extends base {
    entry main
    use app:db/postgres with postgres { connection: env("DATABASE_URL") }
    optimize: true
    debug_info: false
}
```

### 3.4 CLI Integration

Worlds are selected via the CLI:

```bash
barnacle build --world server        # builds the server world
barnacle test --world test           # runs tests
barnacle run --world dev             # runs dev world
```

The compiler reads the world declaration and:
1. Determines which code is live (from the entry point)
2. Applies the specified lint rules
3. Wires effect handlers to implementations
4. Runs code generators
5. Compiles to the target Wasm component

---

## 4. Compilation Pipeline

### 4.1 Stages

The compilation pipeline consists of the following stages:

```
Source (.barnacle files)
  ↓
Lexing (logos or hand-rolled)
  ↓
Parsing → CST (lossless, rowan-based, for LSP)
  ↓
HIR (desugared, name-resolved, effect-inferred)
  ↓
Type checking + effect checking
  ↓
MIR / ANF (A-Normal Form, control flow, monomorphization, effect lowering)
  ↓
Wasm core module (wasm-encoder)
  ↓
[Optional: wasm-opt / Binaryen optimization]
  ↓
Component wrapping (wit-component, Canonical ABI)
  ↓
Composition / linking (wasm-tools compose / wac)
  ↓
Final .wasm Component
```

#### Stage Details

1. **Lexing**: Tokenization of source text. May use `logos` crate or hand-rolled lexer for maximum control.

2. **Parsing**: Builds a Concrete Syntax Tree (CST) using `rowan` for lossless representation. Preserves all whitespace, comments, and syntax for IDE/LSP features.

3. **HIR (High-level IR)**: Desugared, name-resolved representation. Effect inference happens here. Still relatively high-level and close to source.

4. **Type Checking**: Full type checking including effect checking. Ensures type safety and effect soundness.

5. **MIR (Mid-level IR)**: A-Normal Form representation. All control flow is explicit. Monomorphization of generics. Effect lowering (effects → function calls/CPS/exceptions).

6. **Wasm Core Module**: Direct emission to Wasm binary using `wasm-encoder`. No text format intermediate step.

7. **Optimization**: Optional pass through `wasm-opt` or Binaryen for optimizations beyond what the compiler does.

8. **Component Wrapping**: Wrap the core module in a Wasm component using `wit-component` and the Canonical ABI.

9. **Composition**: Link multiple components together using `wasm-tools compose` or `wac` based on the world declaration.

### 4.2 Salsa Query Architecture

Every stage is a Salsa tracked query, enabling incremental recomputation:

```rust
#[salsa::input]
struct SourceFile { 
    text: String, 
    path: PathBuf 
}

#[salsa::tracked]
fn parse(db: &dyn Db, file: SourceFile) -> ConcreteSyntaxTree { 
    // Parse the source text into a CST
}

#[salsa::tracked]
fn lower_to_hir(db: &dyn Db, file: SourceFile) -> hir::Module { 
    // Desugar and resolve names
}

#[salsa::tracked]
fn infer_effects(db: &dyn Db, module: hir::Module) -> EffectMap {
    // Infer effects for all functions in the module
}

#[salsa::tracked]
fn type_check(db: &dyn Db, module: hir::Module) -> TypedModule { 
    // Type check the module
}

#[salsa::tracked]
fn lower_to_mir(db: &dyn Db, module: TypedModule) -> mir::Module {
    // Convert to A-Normal Form
}

#[salsa::tracked]
fn codegen(db: &dyn Db, module: mir::Module) -> WasmModule {
    // Generate Wasm bytecode
}
```

The compiler CLI and LSP share the exact same database. When a file changes:
1. Salsa invalidates only the affected queries
2. Downstream queries are recomputed on demand
3. IDE features (completions, diagnostics, etc.) get updated incrementally

### 4.3 Effect Compilation — Tiered Strategy

Effects compile differently based on their usage patterns:

#### Tier 1: Tail-Resumptive Handlers (Zero Overhead)

The common case — I/O, logging, configuration, most real-world effects. The handler calls `resume` exactly once at the end:

```barnacle
handler io: FileSystem {
    read_file(path) -> resume(wasi_read_file(path))
}
```

**Compilation**: Direct function call. No overhead. The effect operation becomes a regular function call to the handler implementation. This maps 1:1 to WIT imports.

```wasm
;; Effect operation
FileSystem.read_file("config.toml")

;; Compiles to
call $wasi_read_file  ;; direct call, zero overhead
```

#### Tier 2: Non-Resumptive Handlers (Exceptions)

Handlers that don't resume — used for exceptions and early exit:

```barnacle
handler to_result: Fail<E> {
    fail(error) -> Result.Err(error)  // no resume
}
```

**Compilation**: Wasm exception handling (`try`/`catch`/`throw` instructions) or return-code checking, depending on target support:

```wasm
;; With Wasm exceptions
(try $result
  (do ... (throw $error_tag))
  (catch $error_tag
    (return (struct.new $Result $err ...))))

;; Or with return codes (fallback)
(call $operation)
(if (result.is_error)
  (return (error_value)))
```

#### Tier 3: Multi-Shot / Suspending Handlers (CPS Transform)

Handlers that resume multiple times or store continuations — async, generators, backtracking:

```barnacle
handler async_executor: Async {
    suspend() -> {
        let cont = capture_continuation()
        scheduler.enqueue(cont)
        // resume later when scheduled
    }
}
```

**Compilation**: Initially via **CPS (Continuation-Passing Style) transform**. The function is transformed to pass continuations explicitly:

```barnacle
// Original
fn work() with Async {
    Async.suspend()
    compute()
}

// After CPS transform (conceptual)
fn work$cps(k: Continuation) {
    Async.suspend$cps(fn() {
        compute()
        k()
    })
}
```

**Future**: Migrate to **Wasm stack switching** once the proposal stabilizes. This will provide native support for suspending and resuming computations with much better performance.

### 4.4 WIT Generation from Effects

The compiler generates WIT interfaces from effect declarations:

```barnacle
// Barnacle effect
effect Http {
    fetch(url: String, method: Method) -> Result<Response, HttpError>
}

// Generated WIT
interface http {
    enum method { get, post, put, delete }
    
    record response {
        status: u32,
        headers: list<tuple<string, string>>,
        body: list<u8>,
    }
    
    variant http-error {
        network-error(string),
        timeout,
        invalid-url(string),
    }
    
    fetch: func(url: string, method: method) -> result<response, http-error>;
}
```

**Effect declarations → WIT interfaces**:
- Effect name → interface name
- Operations → functions
- Types → WIT types (with automatic translation)

**`with Effect` on a function → `import` in the world**:
- A function that performs an effect imports that interface

**Handlers → components that `export` the interface**:
- A handler provides an implementation (export)

**Component composition → handler wiring**:
- The `world` declaration specifies which handler satisfies which effect

This creates a perfect mapping: **effects ARE component interfaces**.

---

## 5. Compiler Reflection & Metaprogramming

### 5.1 Lint Rules as Wasm Components

The compiler exposes its internals (CST, HIR, types, symbols) as WIT interfaces. Lint rules are ordinary Barnacle functions compiled to Wasm components and run inside the compiler/LSP.

#### Per-Item Signature (Salsa-Cacheable)

Lint functions take a **single item** (not the whole world) and return diagnostics as data. This makes them Salsa-cacheable:

```barnacle
// Per-item signature — Salsa caches per (lint, item) pair
export check_function: func(id: func-id) -> list<diagnostic>
```

When you edit one function, only that function's lint results are recomputed. This is crucial for IDE performance.

#### Compiler WIT Interfaces

The compiler provides WIT interfaces for lint/analysis/codegen components:

```wit
world lint-rule {
    import syntax;       // query the CST
    import types;        // query the type checker
    import symbols;      // query name resolution
    
    export name: func() -> string;
    export check-function: func(id: func-id) -> list<diagnostic>;
    export check-type: func(id: type-id) -> list<diagnostic>;
    export check-module: func(id: module-id) -> list<diagnostic>;
}

interface syntax {
    // Query CST nodes
    get-node: func(id: node-id) -> node;
    get-span: func(id: node-id) -> span;
    get-text: func(id: node-id) -> string;
    children: func(id: node-id) -> list<node-id>;
}

interface types {
    // Query type information
    type-of: func(expr-id: expr-id) -> type-id;
    type-definition: func(type-id: type-id) -> type-def;
    effects-of: func(func-id: func-id) -> list<effect-id>;
}

interface symbols {
    // Query name resolution
    definition: func(name-id: name-id) -> option<def-id>;
    references: func(def-id: def-id) -> list<name-id>;
    scope: func(node-id: node-id) -> scope-id;
}
```

#### Example Lint Rule

```barnacle
// lints/no_mut_pub_field.barnacle
import compiler:syntax
import compiler:types
import compiler:symbols

export fn name() -> String { "no-mut-pub-field" }

export fn check_type(id: TypeId) -> List<Diagnostic> {
    let def = types.type_definition(id)
    match def {
        TypeDef.Struct(s) -> {
            s.fields
                .filter(fn(f) { f.visibility == Pub && f.mutable })
                .map(fn(f) {
                    Diagnostic {
                        level: Warning,
                        span: syntax.get_span(f.id),
                        message: "Public mutable fields break encapsulation",
                        code: "no-mut-pub-field",
                        fixes: [
                            TextEdit {
                                span: f.vis_span,
                                replacement: "",  // make private
                            }
                        ],
                    }
                })
        }
        _ -> []
    }
}

export fn check_function(_: FuncId) -> List<Diagnostic> { [] }
export fn check_module(_: ModuleId) -> List<Diagnostic> { [] }
```

The compiler runs this component for each type definition, caches the results, and shows diagnostics in the IDE.

### 5.2 Strongly Typed Annotations

Annotations are declared as types, not arbitrary strings:

```barnacle
annotation Deprecated {
    reason: String,
    since: String,
    replacement: Option<String>,
}

@Deprecated { 
    reason: "Use v2 API", 
    since: "0.3.0", 
    replacement: Some("fetch_v2") 
}
pub fn old_fetch() -> () with Http { ... }

annotation Doc {
    summary: String,
    examples: List<String>,
    see_also: List<String>,
}

@Doc {
    summary: "Fetches data from a URL",
    examples: ["fetch(\"https://example.com\")"],
    see_also: ["fetch_v2", "Http.post"],
}
pub fn fetch(url: String) -> Response with Http { ... }
```

Lint rules can query and pattern-match on annotations with full type safety:

```barnacle
let deprecated_annot = annotations.get<Deprecated>(func_id)
match deprecated_annot {
    Some(d) -> {
        // Generate deprecation warning with replacement suggestion
    }
    None -> []
}
```

### 5.3 Code Actions / Refactors

Code actions (quick fixes, refactors) use the same component model:

```wit
world code-action {
    import syntax;
    import types;
    import symbols;
    
    export title: func() -> string;
    export applicable: func(ctx: action-context) -> bool;
    export apply: func(ctx: action-context) -> list<text-edit>;
}

record action-context {
    cursor-position: position,
    selection: option<range>,
    node-at-cursor: node-id,
}

record text-edit {
    span: span,
    replacement: string,
}
```

#### Example Code Action

```barnacle
// actions/extract_effect.barnacle
export fn title() -> String { "Extract to effect" }

export fn applicable(ctx: ActionContext) -> Bool {
    // Check if cursor is on a function with effects
    let node = ctx.node_at_cursor
    match syntax.parent(node) {
        Some(parent) if syntax.kind(parent) == FunctionDef -> {
            let func = types.function(parent)
            !func.effects.is_empty()
        }
        _ -> false
    }
}

export fn apply(ctx: ActionContext) -> List<TextEdit> {
    // Generate an effect declaration from function effects
    let func = types.function(ctx.node_at_cursor)
    let effect_name = func.name ++ "Effect"
    
    let operations = func.effects.flat_map(fn(eff) {
        types.effect_operations(eff)
    })
    
    let effect_decl = generate_effect_decl(effect_name, operations)
    
    [
        TextEdit {
            span: before_function(func),
            replacement: effect_decl,
        }
    ]
}
```

### 5.4 Sandboxing

All lint/action/generator components run in Wasmtime inside the compiler. Security properties:

1. **Isolated execution**: Each component runs in its own Wasm instance with separate memory.
2. **Capability-based**: Components can only access what the compiler provides through WIT imports. No filesystem, no network, no system calls.
3. **Deterministic**: Same inputs → same outputs. No non-determinism.
4. **Crash-safe**: A buggy lint can't crash the LSP — it traps cleanly and reports an error.
5. **Resource-limited**: Memory and execution time limits prevent runaway components.

This enables **safe third-party plugins** — you can install community lint rules without risk.

---

## 6. Dead Code / Liveness Analysis

### 6.1 Why It's Sound

Three properties make dead code analysis provably complete:

1. **`world` declares entry points** — we know exactly where execution starts. No implicit main functions, no module initializers.

2. **No side effects at module scope** — importing a module has zero runtime cost. Module scope only allows pure declarations:
   - Function definitions
   - Type definitions
   - Constant definitions
   - Import statements
   - Effect declarations
   
   No code runs at import time. No static constructors. No hidden initialization.

3. **All effects tracked in the type system** — no hidden I/O, no reflection, no dynamic dispatch, no eval. Every side effect is explicit in the type signature.

Given these properties, dead code analysis is a straightforward graph reachability problem from the entry points declared in the `world`.

### 6.2 Liveness in the Compiler

```rust
#[salsa::tracked]
fn direct_references(db: &dyn Db, def: DefId) -> Vec<DefId> {
    // Returns all definitions directly referenced by this definition
    // - Functions called
    // - Types used
    // - Constants referenced
}

#[salsa::tracked]
fn live_set(db: &dyn Db, world: WorldId) -> HashSet<DefId> {
    let entry_points = world_entry_points(db, world);
    let mut live = HashSet::new();
    let mut queue = VecDeque::from(entry_points);
    
    while let Some(def) = queue.pop_front() {
        if live.insert(def) {
            // First time seeing this def — add its references
            queue.extend(direct_references(db, def));
        }
    }
    
    live
}

#[salsa::tracked]
fn is_live(db: &dyn Db, world: WorldId, def: DefId) -> bool {
    live_set(db, world).contains(&def)
}
```

Exposed to lint/analysis components as a `Liveness` query interface:

```wit
interface liveness {
    is-live: func(world: world-id, def: def-id) -> bool;
    live-functions: func(world: world-id) -> list<func-id>;
    live-types: func(world: world-id) -> list<type-id>;
}
```

### 6.3 Tree Shaking

Dead code is never lowered to MIR or Wasm. The compilation pipeline:

1. **Parse and HIR**: All code is parsed and desugared (needed for IDE features)
2. **Type check**: Only live code is type-checked (optimization — but type errors in dead code are still warnings)
3. **MIR lowering**: Only live functions are lowered
4. **Codegen**: Only live code is emitted

Benefits:
- **Smaller output**: No dead code in the final Wasm binary
- **Faster compilation**: Don't generate code for unused functions
- **Better error messages**: "This code is dead" is better than a cryptic type error in unused code

### 6.4 Dead Code Warnings

The compiler can optionally warn about dead code:

```barnacle
world server {
    entry main
    lint barnacle:std/lints/dead_code { level: Warn }
}
```

This helps catch:
- Unused helper functions
- Incomplete refactoring
- Forgotten TODOs
- API functions that are no longer used

But it doesn't prevent compilation — dead code is just a warning, not an error.

---

## 7. Code Generation / Cross-Language Projection

Barnacle supports compile-time code generation for creating clients, documentation, and specifications in other languages.

### 7.1 Core Primitives (in the compiler)

#### `comptime` Keyword

Marks compile-time execution:

```barnacle
comptime {
    // This runs at compile time
    let routes = analyze_module(app:routes)
    generate_client(routes, "./generated/client.ts")
}
```

#### `Reflect` Effect

Compile-time type reflection, backed by Salsa queries:

```barnacle
effect Reflect {
    type_of(expr: Expr) -> Type
    fields_of(type: Type) -> List<Field>
    functions_in(module: Module) -> List<Function>
    signature_of(func: Function) -> Signature
}
```

Available only at compile time, not at runtime.

#### Quasiquotation Syntax

Write code templates in target languages with typed holes:

```barnacle
let ts_code = ts`
    export interface ${type_name} {
        ${fields.map(fn(f) { ts`${f.name}: ${ts_type(f.type)};` }).join("\n")}
    }
`
```

The `ts\`...\`` quasiquote:
- Returns a `Quote<TypeScript>` value (structured, not just a string)
- Interpolations `${}` are type-checked
- Helper functions like `ts_type` convert Barnacle types to TypeScript syntax

Similar quasiquotes for Python, JSON, OpenAPI, etc.

#### `Quote` Type

Captures an expression as a typed AST node instead of evaluating it:

```barnacle
fn analyze_query<T>(q: Quote<T>) -> QueryPlan {
    // Can inspect the AST of q
    match q {
        Quote.BinaryOp(left, op, right) -> { ... }
        Quote.Call(func, args) -> { ... }
        _ -> { ... }
    }
}

let plan = analyze_query('user.filter(fn(u) { u.age > 18 }))
// The expression is not evaluated, it's captured as an AST
```

### 7.2 Stdlib Generators (in standard library)

Specific generators are written using core primitives. They run as Wasm components at compile time, configured in the `world`.

#### TypeScript Client Generator

```barnacle
// std/codegen/typescript.barnacle
pub fn generate_typescript_client(module: Module, output: String) with Reflect, FileSystem {
    let funcs = Reflect.functions_in(module).filter(fn(f) { f.visibility == Pub })
    
    let interfaces = funcs.map(fn(f) {
        ts`
        export async function ${f.name}(${params(f)}): Promise<${return_type(f)}> {
            return fetch('/api/${f.name}', {
                method: 'POST',
                body: JSON.stringify(${arg_object(f)}),
            }).then(r => r.json());
        }
        `
    })
    
    let output_code = ts`
        // Auto-generated TypeScript client
        ${interfaces.join("\n\n")}
    `
    
    FileSystem.write_file(output, render(output_code))
}
```

#### OpenAPI Spec Generator

```barnacle
pub fn generate_openapi(module: Module) with Reflect -> OpenApiSpec {
    let routes = Reflect.functions_in(module)
        .filter(fn(f) { has_annotation<Route>(f) })
    
    OpenApiSpec {
        openapi: "3.0.0",
        info: { title: "API", version: "1.0.0" },
        paths: routes.map(fn(r) {
            let route_annot = get_annotation<Route>(r)
            (route_annot.path, {
                route_annot.method: {
                    parameters: r.params.map(openapi_param),
                    responses: {
                        "200": openapi_response(r.return_type),
                    },
                },
            })
        }).into_map(),
    }
}
```

#### Usage in World

```barnacle
world api {
    entry main
    
    generate barnacle:std/codegen/typescript {
        module: app:routes,
        output: "./client/src/generated/api.ts",
    }
    
    generate barnacle:std/codegen/openapi {
        module: app:routes,
        output: "./docs/openapi.json",
    }
}
```

When you run `barnacle build --world api`, the generators run and produce files.

### 7.3 The Core vs Stdlib Boundary

| Core (in compiler) | Stdlib/Ecosystem (in .barnacle) |
|---|---|
| `comptime`, `Reflect`, quasiquote syntax, `Quote` type | TypeScript/Python/OpenAPI generators |
| Pipeline operator `\|>` | Data libraries (DataSet, filter, group_by) |
| Row types `{ name: String, age: I32 }` | SQL backends, Arrow engines |
| Expression capture `'expr` | Lineage tracking, DAG visualization |
| `Never` type, `Fail` effect | `Result` type, error handling utilities |
| Effect system, handlers | Specific effects (Http, Database, Logger) |
| Core types (I32, String, List, Option) | Collections, algorithms, utilities |

The core is minimal and orthogonal. Everything else is built on top in userland.

---

## 8. Type System

### 8.1 Core Types

#### Primitives

```barnacle
Bool            // true, false
I8, I16, I32, I64
U8, U16, U32, U64
F32, F64
Char            // Unicode scalar value
String          // UTF-8 string
Never           // uninhabited type (for functions that don't return)
()              // unit type
```

#### Compound Types

```barnacle
// Tuple
(I32, String, Bool)

// Struct (nominal)
struct Point {
    x: F64,
    y: F64,
}

// Record (structural, anonymous)
{ name: String, age: I32 }

// Enum (algebraic data type)
enum Option<T> {
    Some(T),
    None,
}

enum Result<T, E> {
    Ok(T),
    Err(E),
}

// List
List<T>

// Map
Map<K, V>

// Function type
(A, B) -> C with E
```

### 8.2 Generics

Full support for parametric polymorphism:

```barnacle
fn identity<T>(x: T) -> T { x }

fn map<T, U, E>(list: List<T>, f: (T) -> U with E) -> List<U> with E {
    // Generic over value types T, U and effect set E
}

struct Box<T> {
    value: T,
}

enum Tree<T> {
    Leaf(T),
    Node(Tree<T>, Tree<T>),
}
```

### 8.3 Effect Polymorphism

Functions can be generic over effect sets:

```barnacle
fn retry<T, E>(f: () -> T with E, times: I32) -> T with E {
    // Generic over effect set E
    // Works with any effect, including Fail
}
```

### 8.4 Subtyping

Barnacle has limited subtyping:

1. **Effect subtyping**: `A with E1` is a subtype of `A with E2` if `E1 ⊆ E2`
2. **Row polymorphism**: `{ x: I32, y: I32, z: I32 }` is a subtype of `{ x: I32, y: I32 }`

No class-based inheritance. Use composition and traits instead.

### 8.5 Traits

Traits define shared behavior:

```barnacle
trait Show {
    fn show(self) -> String
}

trait Eq {
    fn eq(self, other: Self) -> Bool
}

impl Show for Point {
    fn show(self) -> String {
        "Point { x: ${self.x}, y: ${self.y} }"
    }
}

impl<T> Show for List<T> where T: Show {
    fn show(self) -> String {
        "[" ++ self.map(fn(x) { x.show() }).join(", ") ++ "]"
    }
}
```

Traits are separate from effects. Traits are about polymorphism (compile-time dispatch), effects are about side-effect tracking.

### 8.6 Null Safety

No null. Use `Option<T>`:

```barnacle
enum Option<T> {
    Some(T),
    None,
}

fn divide(a: I32, b: I32) -> Option<I32> {
    if b == 0 {
        None
    } else {
        Some(a / b)
    }
}
```

### 8.7 Row Types and Structural Records

Records are structural (duck-typed):

```barnacle
fn get_name(obj: { name: String }) -> String {
    obj.name
}

// Works with any record that has a `name: String` field
get_name({ name: "Alice", age: 30 })
get_name({ name: "Bob", city: "NYC" })
```

This enables row polymorphism for flexible APIs.

---

## 9. Memory Management

### 9.1 Strategy: Perceus-style Reference Counting

Barnacle uses **automatic reference counting** with optimizations inspired by Koka's Perceus algorithm.

Key insights:
1. Most objects are short-lived and never shared
2. Reference count operations can be elided when ownership is clear
3. Linear types can avoid reference counting entirely

### 9.2 Ownership and Borrowing

Barnacle has a simplified ownership model:

```barnacle
// Move semantics by default
let x = Box.new(42)
let y = x    // x is moved to y, x is no longer usable

// Explicit borrow with &
fn print_length(s: &String) {
    Console.print(s.length.to_string())
}

let s = "hello"
print_length(&s)  // s is borrowed, still usable after
```

No lifetimes or borrow checker. The compiler inserts reference count operations automatically, with optimizations to elide unnecessary ones.

### 9.3 Optimizations

#### Last Use Analysis

```barnacle
let x = expensive_computation()
let y = transform(x)  // x's last use — no ref count increment needed
return y
```

The compiler tracks last uses and skips reference count operations when possible.

#### Copy-on-Write

```barnacle
let s = "hello"
let s2 = s.append(" world")  // If s is unique, mutate in place; otherwise, copy
```

#### Reuse Analysis

```barnacle
let list = [1, 2, 3]
let list2 = list.map(fn(x) { x * 2 })  // Reuse list's allocation for list2
```

When a collection is consumed and a new one is produced, reuse the allocation if the ref count is 1.

### 9.4 Cycles

Reference counting can leak cycles. Strategies:

1. **Weak references**: Break cycles explicitly with `Weak<T>`
2. **Cycle detection**: Optional cycle collector (GC) can be enabled
3. **Design patterns**: Most code doesn't create cycles. Prefer tree structures.

### 9.5 Integration with Wasm

Wasm GC proposal provides native GC support. Barnacle can target either:
- **Wasm core + linear memory**: Manual memory management with RC
- **Wasm GC**: Native GC with structs and arrays

The compiler backend is pluggable to support both targets.

---

## 10. Module System & Imports

### 10.1 Module Structure

A module is a single `.barnacle` file:

```barnacle
// user.barnacle
import std/string.{capitalize}
import app/db.{Database}

pub struct User {
    pub name: String,
    pub email: String,
}

pub fn create_user(name: String, email: String) -> User with Database {
    let user = User { name: capitalize(name), email }
    Database.insert("users", user)
    user
}

fn validate_email(email: String) -> Bool {
    // private helper
    email.contains("@")
}
```

### 10.2 Import Syntax

```barnacle
// Import everything
import std/string

// Import specific items
import std/string.{capitalize, lowercase}

// Import with alias
import std/collections.{Map as HashMap}

// Import module with alias
import std/collections as coll
```

### 10.3 Export Rules

- `pub` items are exported
- Non-`pub` items are private to the module
- Effects are always exported (they're interfaces, not implementations)

### 10.4 Package Management

Packages follow the Wasm component model. A package is a collection of modules compiled to a Wasm component.

```
app/
  barnacle.toml       # package manifest
  src/
    main.barnacle
    user.barnacle
    db.barnacle
```

`barnacle.toml`:
```toml
[package]
name = "app"
version = "0.1.0"

[dependencies]
"barnacle:std" = "1.0"
"myorg:db" = "0.5"
```

Dependencies are Wasm components. No source code dependencies — everything is distributed as compiled components with WIT interfaces.

---

## 11. Syntax Overview

### 11.1 Basic Syntax

```barnacle
// Comments
// Single line comment

/* 
   Multi-line comment
*/

// Variables
let x = 42
let y: I32 = 42
let mut z = 0

// Functions
fn add(a: I32, b: I32) -> I32 {
    a + b
}

// Blocks are expressions
let x = {
    let a = 1
    let b = 2
    a + b  // last expression is the result
}

// If is an expression
let max = if a > b { a } else { b }

// Pattern matching
match option {
    Some(x) -> x * 2
    None -> 0
}
```

### 11.2 Operators

```barnacle
// Arithmetic
+ - * / %

// Comparison
== != < > <= >=

// Logical
&& || !

// Bitwise
& | ^ << >> ~

// String concatenation
++

// Pipeline
|>

// Function composition
>>
```

### 11.3 Pipeline Operator

```barnacle
// Instead of nested calls
let result = process(transform(filter(data, pred), mapper))

// Use pipeline
let result = data
    |> filter(pred)
    |> transform(mapper)
    |> process
```

### 11.4 Pattern Matching

```barnacle
match value {
    0 -> "zero"
    1 | 2 -> "one or two"
    n if n > 10 -> "big"
    _ -> "other"
}

// Destructuring
match point {
    Point { x: 0, y: 0 } -> "origin"
    Point { x, y } -> "at ${x}, ${y}"
}

// List patterns
match list {
    [] -> "empty"
    [x] -> "singleton"
    [x, y, ..rest] -> "at least two"
}
```

### 11.5 String Interpolation

```barnacle
let name = "Alice"
let age = 30
Console.print("${name} is ${age} years old")
```

---

## 12. Standard Library Architecture

### 12.1 Core Module Structure

```
barnacle:std/
  core/           # Core types, traits, primitives
    option.barnacle
    result.barnacle
    list.barnacle
    string.barnacle
    
  effects/        # Standard effects
    console.barnacle
    fail.barnacle
    async.barnacle
    state.barnacle
    
  io/             # I/O effects and handlers
    filesystem.barnacle
    network.barnacle
    
  collections/    # Data structures
    map.barnacle
    set.barnacle
    vector.barnacle
    
  codegen/        # Code generators
    typescript.barnacle
    python.barnacle
    openapi.barnacle
    
  lints/          # Standard lint rules
    deprecated.barnacle
    unused.barnacle
    style.barnacle
```

### 12.2 Minimalist Philosophy

The standard library is intentionally small. It provides:
1. Core types and traits
2. Standard effects (Console, Fail, Async, State)
3. Common data structures
4. WIT interfaces for compiler extension

Everything else (web frameworks, databases, graphics, etc.) lives in the ecosystem.

### 12.3 Effect Handlers in Stdlib

```barnacle
// Standard handlers for common effects

// Console handlers
handler real_console: Console { ... }
handler mock_console: Console { ... }
handler quiet_console: Console { ... }

// Fail handlers
handler to_result<T, E>: Fail<E> -> Result<T, E> { ... }
handler to_panic<E>: Fail<E> { ... }
handler or_default<T, E>(default: T): Fail<E> -> T { ... }

// Async handlers
handler thread_pool_executor: Async { ... }
handler single_threaded_executor: Async { ... }

// State handlers
handler cell_state<S>(initial: S): State<S> { ... }
handler transaction_state<S>: State<S> { ... }
```

---

## 13. Tooling & Developer Experience

### 13.1 Command-Line Interface

```bash
# Build
barnacle build [--world WORLD] [--optimize]

# Run
barnacle run [--world WORLD] [ARGS...]

# Test
barnacle test [--world WORLD]

# Check (type check without codegen)
barnacle check

# Format
barnacle fmt

# LSP
barnacle lsp

# REPL
barnacle repl

# Component tools
barnacle component new <name>
barnacle component add-dependency <package>
barnacle component publish
```

### 13.2 LSP Features

Built on Salsa from day one:
- **Completions**: Context-aware, effect-aware
- **Go to definition**: Instant, even across components
- **Find references**: All references to a symbol
- **Hover**: Type info, effect info, documentation
- **Diagnostics**: Errors, warnings, lint results
- **Code actions**: Refactors, quick fixes, from Wasm components
- **Formatting**: Built-in formatter
- **Inlay hints**: Inferred types, effects
- **Semantic highlighting**: Precise, incremental

### 13.3 Testing

Tests are just functions marked with `@Test`:

```barnacle
@Test
fn test_addition() {
    assert_eq(1 + 1, 2)
}

@Test
fn test_with_effects() with Console {
    handle {
        Console.print("test")
        assert_eq(1 + 1, 2)
    } with mock_console
}
```

The test runner (specified in `world test { entry test_runner }`) discovers and runs all `@Test` functions.

### 13.4 Debugging

- **Source maps**: Wasm source maps for debugging in browser dev tools
- **Printf debugging**: Effects make it easy to add logging without changing function signatures
- **Interactive debugger**: DWARF support for native debuggers (lldb, gdb)

### 13.5 Documentation

Documentation is written in annotations:

```barnacle
@Doc {
    summary: "Adds two numbers",
    examples: ["add(1, 2) // returns 3"],
    see_also: ["subtract", "multiply"],
}
pub fn add(a: I32, b: I32) -> I32 { a + b }
```

Documentation is extracted at compile time and rendered as HTML, Markdown, or JSON.

---

## 14. Open Questions & Future Work

### 14.1 Wasm Stack Switching

**Question**: When should we migrate from CPS transform to stack switching for Tier 3 effects?

**Considerations**:
- Stack switching proposal is not yet standardized
- Performance improvements are significant
- Backward compatibility with CPS-compiled code

**Plan**: Support both strategies, with a compiler flag to choose. Migrate ecosystem gradually.

### 14.2 Dependent Types

**Question**: Should Barnacle support dependent types?

**Use cases**:
- Length-indexed lists
- Refined types (positive integers, non-empty strings)
- Type-safe DSLs

**Concerns**:
- Complexity for compiler and users
- Type inference becomes undecidable
- May not fit with Wasm's runtime model

**Plan**: Explore in a research branch. Not for 1.0.

### 14.3 Linear Types

**Question**: Should we add linear types for resource management?

**Use cases**:
- File handles that must be closed
- Network connections
- Tokens representing unique capabilities

**Plan**: Effect system already provides resource safety for many cases (handlers ensure cleanup). Linear types are a possible future addition if the need arises.

### 14.4 Concurrency Model

**Question**: What concurrency primitives should be built-in vs library?

**Current design**:
- `Async` effect is built-in
- Async handlers (executors) are in stdlib
- Structured concurrency via handlers

**Open**: Parallel computation (not just concurrent), work-stealing, data parallelism.

### 14.5 Module Versioning

**Question**: How do we handle breaking changes in Wasm component interfaces?

**Options**:
1. Semantic versioning + major version in package name
2. Interface versioning in WIT
3. Adapter components for compatibility

**Plan**: Follow Wasm component model conventions as they evolve.

### 14.6 Interactive Development

**Question**: Should we support hot code reloading?

**Use cases**:
- Web development
- Game development
- Interactive applications

**Plan**: Effects make this feasible (no global state). Explore in tooling.

### 14.7 Metaprogramming Boundaries

**Question**: What's the right balance between compile-time and runtime metaprogramming?

**Current design**:
- Compile-time: `comptime`, `Reflect`, code generation
- Runtime: Traits, effect handlers

**Open**: Procedural macros, type-level computation, plugin systems.

---

## 15. References & Influences

### Academic Papers

- **Plotkin & Pretnar**: "Handling Algebraic Effects" (2013) — foundational paper on effect handlers
- **Daan Leijen**: "Type Directed Compilation of Row-Typed Algebraic Effects" (2017) — Koka's effect compilation
- **Bauer & Pretnar**: "Programming with Algebraic Effects and Handlers" (2015) — Eff language
- **Biernacki et al.**: "Handle with Care: Relational Interpretation of Algebraic Effects and Handlers" (2018)

### Languages

- **Koka**: https://koka-lang.github.io/koka/doc/index.html
- **Eff**: https://www.eff-lang.org/
- **Effekt**: https://effekt-lang.org/
- **Unison**: https://www.unison-lang.org/
- **Grain**: https://grain-lang.org/
- **OCaml 5**: https://v2.ocaml.org/manual/effects.html

### Wasm Ecosystem

- **Wasm Component Model**: https://component-model.bytecodealliance.org/
- **WIT (Wasm Interface Types)**: https://component-model.bytecodealliance.org/design/wit.html
- **WASI**: https://wasi.dev/
- **Wasmtime**: https://wasmtime.dev/
- **wit-bindgen**: https://github.com/bytecodealliance/wit-bindgen

### Tooling

- **Salsa**: https://github.com/salsa-rs/salsa
- **rowan**: https://github.com/rust-analyzer/rowan
- **rust-analyzer**: https://rust-analyzer.github.io/
- **Tree-sitter**: https://tree-sitter.github.io/tree-sitter/

### Relevant Projects

- **Fermyon Spin**: https://www.fermyon.com/spin
- **wasmCloud**: https://wasmcloud.com/
- **Extism**: https://extism.org/

---

## Appendix A: Effect System Formal Semantics

(To be expanded with formal typing rules, operational semantics, and soundness proofs)

---

## Appendix B: Wasm Compilation Strategies

(To be expanded with detailed compilation strategies for each effect tier)

---

## Appendix C: Salsa Query Graph

(To be expanded with the full query dependency graph for the compiler)

---

## Appendix D: WIT Interface Catalog

(To be expanded with all standard WIT interfaces exported by the compiler)

---

**End of Document**
