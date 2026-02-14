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
export fn greet(name: String) -> () with Console {
    Console.print(msg: "Hello, {name}!")
}

// Private — effects inferred by the compiler
fn helper() {
    Console.print(msg: "internal")
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
fn map<T, U, E>(self list: List<T>, transform: (T) -> U with E) -> List<U> with E {
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
fn parse_int(s: String) -> Int with Fail<ParseError> {
    match try_parse(s) {
        :some { value } -> return value
        :none -> Fail.fail(error: ParseError { input: s })
    }
}

// Handling — wrap in handle at any point
let result = handle parse_int("abc") with to_result
// result: Result<Int, ParseError>

let value = handle parse_int("abc") with or_default(0)
// value: Int (always succeeds)
```

Effect inference means you DON'T need to annotate every function in the call chain with `Fail<E>` — only public boundaries and the origin. This avoids the Java checked exceptions problem. Internal functions infer their full effect set.

`handle` is an expression — it can go anywhere: in a let binding, loop body, function argument, match arm, etc. Wherever you place it, the `Fail` effect is consumed at that point.

#### Standard Handlers for Fail

The standard library provides common handlers:

```barnacle
// Convert to Result type
handler to_result<T, E>: Fail<E> -> Result<T, E> {
    fail(error) -> :err { error }
}

// Provide a default value
fn or_default<T, E>(default: T): Handler<Fail<E>, T> {
    handler {
        fail(_) -> default
    }
}

// Panic on error
handler to_panic<E>: Fail<E> {
    fail(error) -> panic(message: "Unhandled error: {error}")
}

// Transform error type
fn map_error<E1, E2>(f: (E1) -> E2): Handler<Fail<E1>, Fail<E2>> {
    handler {
        fail(error) -> Fail.fail(error: f(error))
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
export fn join_path(a: String, b: String) -> String { ... }  // pure function
export fn extension(path: String) -> Option<String> { ... }  // pure function

// User code
import effect FileSystem from std/fs
import fn join_path from std/fs
fn process() with FileSystem {
    let path = join_path(a: dir, b: "config.toml")  // pure — direct call
    FileSystem.read_file(path: path)                 // effect — needs handler
}
```

Pure functions compose freely. Effectful operations require handlers. This distinction is enforced at compile time.

### 2.8 Effect Subtyping and Polymorphism

Effects support subtyping through effect sets:

```barnacle
// Empty effect set = pure function
fn pure() -> Int { return 42 }

// Single effect
fn one() -> () with Console { Console.print(msg: "one") }

// Multiple effects
fn two() -> () with Console, FileSystem { 
    Console.print(msg: "reading")
    FileSystem.read_file(path: "config")
}

// Effect polymorphism — generic over any effect set
fn generic<E>(f: () -> () with E) -> () with E {
    f()
}

// Can call with any effect set
generic(f: pure)       // E = {}
generic(f: one)        // E = {Console}
generic(f: two)        // E = {Console, FileSystem}
```

### 2.9 Effects as Interfaces, Not DI Containers

**Critical Design Guidance**: Effects should model **capabilities** (I/O, database, network, filesystem), not business logic abstractions.

#### The Right Way: Effects for Capabilities

```barnacle
// ✅ Good: Effects represent I/O capabilities
effect Database {
    query(sql: String, params: List<Value>) -> List<Row>
    execute(sql: String, params: List<Value>) -> Int
}

effect FileSystem {
    read_file(path: String) -> String
    write_file(path: String, content: String) -> ()
}

effect Network {
    http_request(url: String, method: :get | :post) -> Response
}

// ✅ Good: Business logic is regular functions that USE effects
export fn get_active_users() -> List<User> with Database {
    let rows = Database.query(sql: "SELECT * FROM users WHERE active = true", params: [])
    return rows |> map(transform: row => parse_user(row))
}

export fn save_user(user: User) -> () with Database, Fail<DbError> {
    let result = Database.execute(
        sql: "INSERT INTO users (name, email) VALUES (?, ?)", 
        params: [user.name, user.email]
    )
    if result == 0 {
        Fail.fail(error: DbError { message: "Insert failed" })
    }
}
```

#### The Wrong Way: Effects as Business Logic Wrappers

```barnacle
// ❌ Bad: Effect wraps business logic, not I/O
effect UserService {
    get_active_users() -> List<User>
    create_user(name: String, email: String) -> User
    delete_user(id: Int) -> ()
    update_email(id: Int, email: String) -> ()
    // ... 50 more operations
}

// This is just dependency injection with extra steps!
// Write functions instead:
```

#### Why This Matters

1. **Effects should be thin** — 3-10 operations, not 50+
2. **Effects should be reusable** — `Database` effect works for many domains
3. **Effects should map to I/O** — they represent actual side effects, not pure logic
4. **Business logic should be functions** — composable, testable, no ceremony

#### Named Effect Instances for Multiple Backends

Use `as` to distinguish multiple instances of the same effect:

```barnacle
import effect Database from app/db as primary
import effect Database from app/replica as replica

export fn replicate_users() -> () with primary.Database, replica.Database {
    let users = primary.Database.query(sql: "SELECT * FROM users", params: [])
    for user in users {
        replica.Database.execute(
            sql: "INSERT INTO users (id, name, email) VALUES (?, ?, ?)",
            params: [user.id, user.name, user.email]
        )
    }
}
```

#### Lints and Guardrails

The Barnacle linter warns on:
- **Too many operations per effect** (> 20 operations): suggests splitting or using functions
- **Too many custom effects** (> 5 in a module): suggests consolidating or using functions
- **Effects without I/O operations**: suggests using regular functions instead
- **Business domain names in effects**: suggests renaming to capability-based names

The **path of least resistance** (writing functions) is the **correct path**. Effects are for when you need to abstract over implementation (testing, swapping backends, sandboxing).

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
    read_file(path) -> resume(wasi_read_file(path: path))
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
    fail(error) -> :err { error: error }
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
        scheduler.enqueue(continuation: cont)
        return resume()
    }
}
```

**Compilation**: Initially via **CPS (Continuation-Passing Style) transform**. The function is transformed to pass continuations explicitly:

```barnacle
// Original
export fn work() with Async {
    Async.suspend()
    return compute()
}

// After CPS transform (conceptual)
export fn work$cps(k: Continuation) {
    return Async.suspend$cps(handler: k => {
        return compute()
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

```wit
export check-function: func(id: func-id) -> list<diagnostic>
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
import compiler syntax from compiler:syntax
import compiler types from compiler:types
import compiler symbols from compiler:symbols

export fn name() -> String { 
    return "no-mut-pub-field" 
}

export fn check_type(id: TypeId) -> List<Diagnostic> {
    let def = types.type_definition(id: id)
    return match def {
        TypeDef.Struct(s: s) -> {
            return s.fields
                |> filter(predicate: f => f.visibility == :pub && f.mutable)
                |> map(transform: f => {
                    return Diagnostic {
                        level: :warning,
                        span: syntax.get_span(id: f.id),
                        message: "Public mutable fields break encapsulation",
                        code: "no-mut-pub-field",
                        fixes: [
                            TextEdit {
                                span: f.vis_span,
                                replacement: "",
                            }
                        ],
                    }
                })
        }
        _ -> {
            return []
        }
    }
}

export fn check_function(_: FuncId) -> List<Diagnostic> { 
    return [] 
}
export fn check_module(_: ModuleId) -> List<Diagnostic> { 
    return [] 
}
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
    replacement: :some { value: "fetch_v2" }
}
export fn old_fetch() -> () with Http { ... }

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
export fn fetch(url: String) -> Response with Http { ... }
```

Lint rules can query and pattern-match on annotations with full type safety:

```barnacle
let deprecated_annot = annotations.get<Deprecated>(id: func_id)
return match deprecated_annot {
    :some { value: d } -> {
        return generate_deprecation_warning(d: d)
    }
    :none -> {
        return []
    }
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
export fn title() -> String { 
    return "Extract to effect" 
}

export fn applicable(ctx: ActionContext) -> Bool {
    let node = ctx.node_at_cursor
    return match syntax.parent(id: node) {
        :some { value: parent } -> {
            if syntax.kind(id: parent) == FunctionDef {
                let func = types.function(id: parent)
                return !func.effects.is_empty()
            } else {
                return false
            }
        }
        :none -> {
            return false
        }
    }
}

export fn apply(ctx: ActionContext) -> List<TextEdit> {
    let func = types.function(id: ctx.node_at_cursor)
    let effect_name = func.name ++ "Effect"
    
    let operations = func.effects
        |> flat_map(fn: f => {
            return types.effect_operations(id: f)
        })
    
    let effect_decl = generate_effect_decl(name: effect_name, operations: operations)
    
    return [
        TextEdit {
            span: before_function(func: func),
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
    let routes = analyze_module(app: app:routes)
    generate_client(routes: routes, output_dir: "./generated/client.ts")
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
        ${fields.map(fn: f => { 
            return ts`${f.name}: ${ts_type(type: f.type)};` 
        }).join(separator: "\n")}
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
export fn analyze_query<T>(q: Quote<T>) -> QueryPlan {
    return match q {
        Quote.BinaryOp(left: left, op: op, right: right) -> { ... }
        Quote.Call(func: func, args: args) -> { ... }
        _ -> { ... }
    }
}

let plan = analyze_query(q: 'user.filter(predicate: u => u.age > 18))
```

### 7.2 Stdlib Generators (in standard library)

Specific generators are written using core primitives. They run as Wasm components at compile time, configured in the `world`.

#### TypeScript Client Generator

```barnacle
// std/codegen/typescript.barnacle
export fn generate_typescript_client(module: Module, output: String) with Reflect, FileSystem {
    let funcs = Reflect.functions_in(module: module)
        |> filter(predicate: f => f.visibility == :export)
    
    let interfaces = funcs
        |> map(transform: f => {
            return ts`
            export async function ${f.name}(${params(func: f)}): Promise<${return_type(func: f)}> {
                return fetch(path: '/api/${f.name}', options: {
                    method: 'POST',
                    body: JSON.stringify(${arg_object(func: f)}),
                }).then(handler: r => r.json());
            }
            `
        })
    
    let output_code = ts`
        // Auto-generated TypeScript client
        ${interfaces.join(separator: "\n\n")}
    `
    
    return FileSystem.write_file(path: output, content: render(code: output_code))
}
```

#### OpenAPI Spec Generator

```barnacle
export fn generate_openapi(module: Module) with Reflect -> OpenApiSpec {
    let routes = Reflect.functions_in(module: module)
        |> filter(predicate: f => has_annotation<Route>(item: f))
    
    return OpenApiSpec {
        openapi: "3.0.0",
        info: { title: "API", version: "1.0.0" },
        paths: routes.map(transform: r => {
            let route_annot = get_annotation<Route>(item: r)
            return {
                path: route_annot.path,
                endpoint: {
                    method: route_annot.method,
                    content: {
                        parameters: r.params.map(transform: p => openapi_param(param: p)),
                        responses: {
                            "200": openapi_response(type: r.return_type),
                        },
                    }
                }
            }
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
| Row types `{ name: String, age: Int }` | SQL backends, Arrow engines |
| Expression capture `'expr` | Lineage tracking, DAG visualization |
| `Never` type, `Fail` effect | `Result` type, error handling utilities |
| Effect system, handlers | Specific effects (Http, Database, Logger) |
| Core types (Int, String, List, Option) | Collections, algorithms, utilities |

The core is minimal and orthogonal. Everything else is built on top in userland.

---

## 8. Type System

### 8.1 Core Types

#### Primitives

```barnacle
Bool            // true, false
Int             // 64-bit signed integer (default)
Float           // 64-bit floating point (default)
String          // UTF-8 string
Byte            // Single byte (8-bit unsigned)
Never           // uninhabited type (for functions that don't return)
()              // unit type
```

#### Sized Variants (for interop)

When you need explicit sizes for FFI, Wasm interop, or performance:

```barnacle
// Signed integers
Int8, Int16, Int32, Int64

// Unsigned integers
UInt8, UInt16, UInt32, UInt64

// Floating point
Float32, Float64
```

Use the readable `Int` and `Float` types by default. Only use sized variants when you need explicit control.

#### Compound Types

```barnacle
// Row type (structural, named fields)
type Point { x: Float, y: Float }

// Anonymous row type (inline structural)
{ name: String, age: Int }

// Enum (algebraic data type with atom variants)
enum Option<T> {
    :some { value: T },
    :none,
}

enum Result<T, E> {
    :ok { value: T },
    :err { error: E },
}

// List
List<T>

// Map (requires non-identifier keys or atoms)
Map<K, V>

// Set
Set<T>

// Function type
(A, B) -> C with E
```

### 8.2 Structural Typing by Default

**Barnacle uses structural typing by default** — types are shapes, and any value with matching fields fits:

```barnacle
// Named type alias for a structural shape
type Point { x: Float, y: Float }

// Anonymous row type (inline)
export fn get_name(obj: { name: String }) -> String {
    return obj.name
}

// Works with any value that has a `name: String` field
get_name(obj: { name: "Alice", age: 30 })
get_name(obj: { name: "Bob", city: "NYC", active: true })
```

Named types via `type Name { fields }` are **named aliases for shapes**, not distinct nominal types. This makes APIs flexible while maintaining readability.

#### Width Subtyping

Barnacle supports **width subtyping** — a type with more fields can be used where fewer are expected:

```barnacle
type Person { name: String }
type Employee { name: String, id: Int, department: String }

export fn greet(person: Person) -> String {
    return "Hello, {person.name}!"
}

let emp: Employee = { name: "Alice", id: 42, department: "Engineering" }
greet(person: emp)
```

The extra fields are ignored at the call site. This enables evolution of data structures without breaking existing code.

### 8.3 Generics

Full support for parametric polymorphism:

```barnacle
export fn identity<T>(value: T) -> T { 
    return value 
}

export fn map<T, U, E>(self list: List<T>, transform: (T) -> U with E) -> List<U> with E {
    // Generic over value types T, U and effect set E
}

type Box<T> { value: T }

enum Tree<T> {
    :leaf { value: T },
    :node { left: Tree<T>, right: Tree<T> },
}
```

### 8.4 Effect Polymorphism

Functions can be generic over effect sets:

```barnacle
export fn retry<T, E>(f: () -> T with E, times: Int) -> T with E {
    return f()
}
```

### 8.5 Tag System — Compile-Time Evidence

**Tags replace distinct/nominal typing entirely.** They provide zero-runtime-cost evidence that values have passed through specific validations, state transitions, or transformations.

#### Basic Tag Syntax

Tags are composable type evidence using the `& :tag` syntax:

```barnacle
// Add evidence to a type
String & :email                    // A string that's been validated as an email
Int & :positive                    // An integer that's been validated as positive
Connection & :connected & :authenticated   // A connection with multiple evidences
```

Tags are:
- **Compile-time only** — zero runtime cost, erased after type checking
- **Additive** — you can accumulate multiple tags
- **Not fabricable** — can only come from functions that produce them
- **Widening-safe** — tags can always be stripped (widened to untagged)

#### Creating Tagged Values

Use `tag()` to add evidence (accumulates with existing tags):

```barnacle
export fn validate_email(input: String) -> String & :email with Fail<ValidationError> {
    if input.contains(needle: "@") {
        return tag(value: input, as: :email)
    } else {
        return Fail.fail(error: ValidationError { message: "Invalid email" })
    }
}

export fn ensure_positive(value: Int) -> Int & :positive with Fail<ValueError> {
    if value > 0 {
        return tag(value: value, as: :positive)
    } else {
        return Fail.fail(error: ValueError { message: "Value must be positive" })
    }
}
```

Use `retag()` to replace all tags (for state transitions):

```barnacle
export fn disconnect(conn: Connection & :connected) -> Connection & :disconnected {
    close_socket(socket: conn)
    return retag(value: conn, as: :disconnected)
}
```

#### Tag Patterns and Use Cases

**1. Validation Evidence**

```barnacle
type User {
    email: String & :email,
    username: String & :non_empty,
    age: Int & :positive,
}

export fn create_user(email: String, username: String, age: Int) -> User with Fail<ValidationError> {
    let valid_email = validate_email(input: email)
    let valid_username = validate_non_empty(input: username)
    let valid_age = ensure_positive(value: age)
    
    return { email: valid_email, username: valid_username, age: valid_age }
}
```

**2. State Machines / Typestate**

```barnacle
type Connection { socket: Socket }

export fn connect(host: String) -> Connection & :connected with Network { ... }
export fn authenticate(conn: Connection & :connected) -> Connection & :connected & :authenticated with Network { ... }
export fn send_message(conn: Connection & :connected & :authenticated, msg: String) -> () with Network { ... }

// Type system prevents you from sending messages without authentication:
let conn = connect(host: "example.com")
// send_message(conn: conn, msg: "hello")
let authed = authenticate(conn: conn)
send_message(conn: authed, msg: "hello")
```

**3. Capabilities / Permissions**

```barnacle
type Request { path: String, headers: Map<String, String> }

export fn require_auth(req: Request) -> Request & :authenticated with Fail<AuthError> { ... }
export fn require_admin(req: Request & :authenticated) -> Request & :authenticated & :admin with Fail<AuthError> { ... }

export fn delete_user(req: Request & :authenticated & :admin, user_id: Int) -> () with Database {
    return Database.execute(query: "DELETE FROM users WHERE id = ?", params: [user_id])
}
```

**4. Resource Lifecycle**

```barnacle
type File { handle: FileHandle }

export fn open_file(path: String) -> File & :open with FileSystem { ... }
export fn make_writable(file: File & :open) -> File & :open & :writable with FileSystem { ... }
export fn write(file: File & :open & :writable, data: String) -> () with FileSystem { ... }
export fn close(file: File & :open) -> File & :closed with FileSystem { ... }
```

**5. Data Provenance / Pipeline Stages**

```barnacle
type Dataset<T> { rows: List<T> }

export fn clean(data: Dataset<T>) -> Dataset<T> & :cleaned { ... }
export fn dedupe(data: Dataset<T> & :cleaned) -> Dataset<T> & :cleaned & :deduped { ... }
export fn normalize(data: Dataset<T> & :cleaned & :deduped) -> Dataset<T> & :cleaned & :deduped & :normalized { ... }

export fn analyze(data: Dataset<User> & :cleaned & :deduped & :normalized) -> Report {
    return process_data(data: data)
}
```

**6. Builder Replacement**

Instead of traditional builder patterns, use tags to track required fields:

```barnacle
type HttpRequest { url: Option<String>, method: Option<String>, headers: Map<String, String> }

export fn new_request() -> HttpRequest {
    return { url: :none, method: :none, headers: {} }
}

export fn with_url(req: HttpRequest, url: String) -> HttpRequest & :has_url {
    return tag(value: { ...req, url: :some { value: url } }, as: :has_url)
}

export fn with_method(req: HttpRequest, method: String) -> HttpRequest & :has_method {
    return tag(value: { ...req, method: :some { value: method } }, as: :has_method)
}

export fn send(req: HttpRequest & :has_url & :has_method) -> Response with Network {
    return make_request(req: req)
}
```

**7. Protocol State**

```barnacle
// SMTP protocol states
export fn smtp_connect() -> SmtpSession & :connected { ... }
export fn smtp_helo(session: SmtpSession & :connected) -> SmtpSession & :greeted { ... }
export fn smtp_mail_from(session: SmtpSession & :greeted) -> SmtpSession & :has_sender { ... }
export fn smtp_rcpt_to(session: SmtpSession & :has_sender) -> SmtpSession & :has_recipient { ... }
export fn smtp_data(session: SmtpSession & :has_recipient) -> SmtpSession & :sending_data { ... }
```

**8. Taint Tracking**

```barnacle
// User input is tainted by default
export fn get_user_input() -> String & :tainted with Console { ... }

export fn sanitize_html(input: String & :tainted) -> String & :sanitized {
    let clean = strip_tags(html: input)
    return tag(value: clean, as: :sanitized)
}

export fn render_html(content: String & :sanitized) -> String {
    return "<div>{content}</div>"
}
```

#### Tags vs Effects: Complementary Tools

- **Tags** provide compile-time evidence (this value has been validated)
- **Effects** (like `Fail<E>`) provide runtime branching (this operation can fail)
- **Together**: If code is reachable after a validation function with `Fail`, the tag is guaranteed

```barnacle
export fn process_email(input: String) -> () with Fail<ValidationError> {
    let email = validate_email(input: input)
    return send_email(to: email)
}
```

#### Tag Stripping (Widening)

Tags can always be stripped through widening:

```barnacle
let tagged: Int & :positive = ensure_positive(value: 42)
let plain: Int = tagged
```

This allows tagged values to be used in contexts that don't care about the evidence.

### 8.6 Atoms and Enums Unified

Atoms are lightweight values prefixed with `:`, similar to symbols in Lisp or atoms in Erlang. They unify with the enum system to provide both ad-hoc and named unions.

#### Atoms as Values

```barnacle
:ok                    // Simple atom
:error                 // Another atom
:get                   // HTTP method as atom
:post                  // Another HTTP method

// Atoms with associated data
:some { value: 42 }
:err { error: "file not found" }
:point { x: 1.0, y: 2.0 }
```

#### Inline Atom Unions

Use `|` to create anonymous unions of atoms:

```barnacle
// Function that returns one of several atoms
export fn status() -> :ok | :pending | :error {
    return :ok
}

// Pattern match on atom unions
return match status() {
    :ok -> {
        return "Success"
    }
    :pending -> {
        return "In progress"
    }
    :error -> {
        return "Failed"
    }
}
```

#### Named Enums (Closed Sets)

Use `enum` to define a closed, named set of atom variants:

```barnacle
enum Option<T> {
    :some { value: T },
    :none,
}

enum Result<T, E> {
    :ok { value: T },
    :err { error: E },
}

enum HttpMethod {
    :get,
    :post,
    :put,
    :delete,
    :patch,
}
```

#### Structural Compatibility

Atoms in named enums are structurally compatible with atom unions:

```barnacle
enum Color {
    :red,
    :green,
    :blue,
}

export fn is_primary(color: :red | :blue) -> Bool {
    return true
}

let c: Color = :red
is_primary(color: c)
```

#### Syntax Semantics

- `|` means "one of these" (union)
- `&` means "all of these" (intersection/evidence)

```barnacle
// Union: value is ONE of these atoms
:get | :post | :delete

// Intersection: value has ALL these tags
String & :validated & :sanitized
```

### 8.7 Subtyping

Barnacle has limited subtyping:

1. **Width subtyping**: `{ x: Int, y: Int, z: Int }` is a subtype of `{ x: Int, y: Int }`
2. **Effect subtyping**: `A with E1` is a subtype of `A with E2` if `E1 ⊆ E2`
3. **Tag widening**: `T & :tag` is a subtype of `T` (tags can be stripped)
4. **Atom union subtyping**: Single atoms are subtypes of unions containing them

No class-based inheritance. Use composition and structural typing instead.

### 8.8 Traits (Future Work)

> **Note**: Trait system design is still in progress. The syntax shown here is provisional.

Traits will define shared behavior and generic constraints:

```barnacle
trait Show {
    fn show(self) -> String
}

trait Eq {
    fn eq(self, other: Self) -> Bool
}

// Implementation syntax TBD
// Constraint syntax TBD
```

Traits are separate from effects. Traits are about polymorphism (compile-time dispatch), effects are about side-effect tracking.

### 8.9 Null Safety

No null. Use `Option<T>`:

```barnacle
enum Option<T> {
    :some { value: T },
    :none,
}

export fn divide(a: Int, b: Int) -> Option<Int> {
    if b == 0 {
        return :none
    } else {
        return :some { value: a / b }
    }
}
```

### 8.10 No Methods on Types

Barnacle has **no methods attached to types**. Instead:

- Functions with a `self` parameter
- The pipeline operator `|>` for chaining

```barnacle
// Define functions with self parameter (positional for pipeline)
export fn filter<T>(self data: List<T>, predicate: (T) -> Bool) -> List<T> { ... }
export fn map<T, U>(self data: List<T>, transform: (T) -> U) -> List<U> { ... }

// Use pipeline operator for "method call" syntax
users 
    |> filter(predicate: u => u.active) 
    |> map(transform: u => u.name)
    |> sort()

// Equivalent to:
sort(data: map(transform: u => u.name, self: filter(predicate: u => u.active, self: users)))
```

The `self` parameter is special:
- It's the **only positional parameter** (all others are named)
- Pipeline `|>` feeds into the `self` parameter
- This creates the ergonomics of method chaining without tying functions to types

Types are pure data. Modules organize related functions.

### 8.11 Row Types and Structural Records

Records are structural (duck-typed):

```barnacle
export fn get_name(obj: { name: String }) -> String {
    return obj.name
}

// Works with any record that has a `name: String` field
get_name(obj: { name: "Alice", age: 30 })
get_name(obj: { name: "Bob", city: "NYC" })
```

This enables row polymorphism for flexible APIs.

### 8.12 No Tuples

Barnacle does not have tuples. Use row types with named fields instead:

```barnacle
// ❌ No tuples
// export fn divide(a: Int, b: Int) -> (Int, Int)

// ✅ Use row types
export fn divide(a: Int, b: Int) -> { quotient: Int, remainder: Int } {
    // Note: A production version would check for b == 0 and use Fail<E>
    return { quotient: a / b, remainder: a % b }
}

let result = divide(a: 17, b: 5)
let { quotient, remainder } = result
```

Named fields are self-documenting and eliminate the need to remember positional order.

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
let x = Box.new(value: 42)
let y = x

// Explicit borrow with &
export fn print_length(s: &String) {
    Console.print(s: s.length.to_string())
}

let s = "hello"
print_length(s: &s)
```

No lifetimes or borrow checker. The compiler inserts reference count operations automatically, with optimizations to elide unnecessary ones.

### 9.3 Optimizations

#### Last Use Analysis

```barnacle
let x = expensive_computation()
let y = transform(x: x)
return y
```

The compiler tracks last uses and skips reference count operations when possible.

#### Copy-on-Write

```barnacle
let s = "hello"
let s2 = s.append(suffix: " world")
```

#### Reuse Analysis

```barnacle
let list = [1, 2, 3]
let list2 = list.map(transform: x => x * 2)
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

A module is a single `.barnacle` file. The file path determines the module path.

```barnacle
// user.barnacle
import type User from app/models
import effect Database from app/db
import fn capitalize from std/string

export type User {
    name: String,
    email: String,
}

export fn create_user(name: String, email: String) -> User with Database {
    let user = { name: capitalize(s: name), email: email }
    Database.insert(table: "users", data: user)
    return user
}

export fn validate_email(email: String) -> Bool {
    return email.contains(needle: "@")
}
```

File structure:
```
app/
  barnacle.toml       # package manifest
  src/
    main.barnacle
    user.barnacle
    db.barnacle
    models.barnacle
```

### 10.2 Import Syntax

Barnacle uses **explicit import categories** to make it clear what kind of entity is being imported:

```barnacle
// Import types
import type User, Post from app/models
import type Point from geometry/shapes

// Import effects
import effect Console from std/io
import effect Database from app/db

// Import functions
import fn map, filter from std/list
import fn capitalize from std/string

// Import handlers
import handler postgres from app/db/handlers

// Import annotations (metadata)
import annotation deprecated, test from std/meta

// Import module (qualified access)
import module std/string
// Use as: string.capitalize(...)

// Rename with `as`
import effect Database from app/db as primary
import effect Database from app/replica as replica
import type Map from std/collections as HashMap

// Multiple from same module
import type User, Post, Comment from app/models
import effect Http, WebSocket from std/net
```

#### Named Effect Instances

The `as` keyword enables multiple instances of the same effect:

```barnacle
import effect Database from app/db as primary
import effect Database from app/replica as secondary

export fn replicate_data() -> () with primary.Database, secondary.Database {
    let data = primary.Database.query(sql: "SELECT * FROM users")
    return secondary.Database.insert_batch(data: data)
}
```

### 10.3 Export Syntax

Use `export` (not `pub`) for public visibility:

```barnacle
// Export type
export type User { name: String, email: String }

// Export function
export fn create_user(name: String, email: String) -> User { ... }

// Export effect
export effect Database {
    query(sql: String) -> List<Row>
    execute(sql: String) -> ()
}

// Re-export from another module
export type User from app/models
export fn validate from app/validation
export effect Database from app/db
```

Private by default — anything without `export` is module-private.

### 10.4 Handlers and World

Handlers are typically wired in `world` declarations, not imported into application code:

```barnacle
// app/main.barnacle
import effect Database from app/db
import effect Console from std/io

export fn main() -> () with Database, Console {
    Console.print(msg: "Starting app...")
    let users = Database.query(sql: "SELECT * FROM users")
    return Console.print(msg: "Found {users.length} users")
}

// world configuration (separate file or section)
world app {
    entry main
    handle Database with postgres(conn: "postgres://localhost/mydb")
    handle Console with stdio
}
```

Local `handle` blocks can still use handlers directly:

```barnacle
import handler mock_db from app/test_helpers

export fn test_create_user() {
    let result = handle create_user(name: "Alice", email: "alice@example.com") with mock_db
    return assert(condition: result.name == "Alice")
}
```

### 10.5 Package Management

Packages follow the Wasm component model. A package is a collection of modules compiled to a Wasm component.

`barnacle.toml`:
```toml
[package]
name = "app"
version = "0.1.0"

[dependencies]
"std" = "1.0"
"fermyon/http" = "0.5"
```

Package namespaces:
- `std/io` — standard library
- `app/models` — local package modules
- `fermyon/http` — third-party packages

Dependencies are Wasm components. No source code dependencies — everything is distributed as compiled components with WIT interfaces.

### 10.6 No Circular Imports

Circular imports are not allowed. The module graph must be a DAG (directed acyclic graph).

If you need shared types between modules, extract them to a common module that both depend on.

---

## 11. Syntax Overview

### 11.1 Keywords and Visibility

```barnacle
fn          // function declarations
let         // immutable bindings
let mut     // mutable bindings
const       // compile-time constants
export      // public visibility (NOT pub)
effect      // effect declarations
handler     // effect handlers
handle      // handle expression with handler
with        // effect annotation, handler binding
resume      // resume continuation in handler
match       // pattern matching
return      // explicit return (required in multi-statement functions)
if          // conditional expression
else        // else branch
for         // loops
while       // while loops
break       // break from loop
continue    // continue loop
comptime    // compile-time execution
world       // top-level configuration
annotation  // metadata declarations
use         // disposable/resource cleanup
import      // import declarations
type        // type alias declarations
enum        // enum declarations
```

### 11.2 Comments

```barnacle
// Single line comment

/* 
   Multi-line comment
   /* Nestable */
*/

/// Documentation comment
/// These appear in generated docs and LSP hovers
```

### 11.3 Variables and Constants

```barnacle
// Immutable binding (default)
let x = 42
let y: Int = 42

// Mutable binding
let mut counter = 0
counter = counter + 1

// Compile-time constant
const MAX_USERS: Int = 1000
const PI: Float = 3.14159
```

### 11.4 Functions

**All function parameters are named** (except `self` which is positional for pipeline):

```barnacle
// Function declaration
export fn add(a: Int, b: Int) -> Int {
    return a + b
}

// Function with default values
export fn connect(host: String, port: Int = 5432) -> Connection {
    return create_connection(host: host, port: port)
}

// Call with named arguments
connect(host: "localhost", port: 5432)
connect(host: "localhost")

// Function with self parameter (for pipeline)
export fn filter<T>(self data: List<T>, predicate: (T) -> Bool) -> List<T> {
    return data.filter(predicate: predicate)
}

// Pipeline usage
users |> filter(predicate: u => u.active)
// Equivalent to: filter(self: users, predicate: u => u.active)

// Explicit return required
export fn max(a: Int, b: Int) -> Int {
    if a > b {
        return a
    } else {
        return b
    }
}
```

### 11.5 Lambdas (Arrow Syntax)

```barnacle
// Single parameter
x => x * 2

// Multiple parameters
(a, b) => a + b

// No parameters
() => 42

// Single-expression lambdas are implicitly their value
let double = x => x * 2

// Multi-statement lambdas require explicit return
let complex = (a, b) => {
    let sum = a + b
    let product = a * b
    return { sum, product }
}

// Use in higher-order functions
users |> map(transform: u => u.name)
users |> filter(predicate: u => u.age > 18)
```

Note: Lambdas use `=>` (arrow), NOT `|x|` syntax.

### 11.6 String Literals

```barnacle
// Simple string
let name = "Alice"

// Interpolation with {}
let greeting = "Hello, {name}!"
let calculation = "2 + 2 = {2 + 2}"

// Escape literal braces
let literal = "\{not interpolated\}"

// Multi-line strings (triple quotes)
let poem = """
    Roses are red,
    Violets are blue,
    Barnacle is awesome,
    And so are you.
"""

// Raw strings (no escaping, no interpolation)
let regex = r"^\d{3}-\d{2}-\d{4}$"
let path = r"C:\Users\Alice\Documents"
```

### 11.7 Numeric Types

```barnacle
// Default integers and floats (readable names)
let count: Int = 42              // 64-bit signed integer
let price: Float = 19.99         // 64-bit float

// Sized variants for interop/performance
let byte: Int8 = 127
let short: Int16 = 32000
let int: Int32 = 2_147_483_647
let long: Int64 = 9_223_372_036_854_775_807

let unsigned: UInt32 = 4_294_967_295
let small: UInt8 = 255

let single: Float32 = 3.14
let double: Float64 = 2.718281828

// Bool and Byte
let flag: Bool = true
let byte_val: Byte = 0xFF
```

### 11.8 Collection Literals

```barnacle
// List
let numbers = [1, 2, 3, 4, 5]
let names = ["Alice", "Bob", "Charlie"]

// Row type value (identifier keys)
let person = { name: "Alice", age: 30 }
let point = { x: 1.0, y: 2.0 }

// Map (string literal keys)
let config = { "host": "localhost", "port": 5432 }

// Map (atom keys)
let options = { :timeout: 30000, :retry: true }

// Map (computed keys)
let dynamic = { [compute_key()]: "value" }

// Set (function call, not literal)
let unique = Set("alice", "bob", "charlie")

// Parser distinguishes row vs map by key type:
// - Identifier keys -> row type
// - Expression keys (string literals, atoms, computed) -> map
```

### 11.9 Spread Operator

```barnacle
// Flat spread (rows and maps)
let user = { name: "Alice", age: 30 }
let updated = { ...user, age: 31 }

// List spread
let a = [1, 2, 3]
let b = [4, 5, 6]
let combined = [...a, ...b]  // [1, 2, 3, 4, 5, 6]

// Deep spread (dotted paths)
let person = {
    name: "Alice",
    address: { city: "NYC", state: "NY" }
}
let relocated = { ...person, address.city: "Portland", address.state: "OR" }

// Deep spread desugars to nested spreads at compile time
```

### 11.10 Destructuring

```barnacle
// Flat destructuring
let { name, age } = user
let { x, y } = point

// Deep destructuring (dotted paths)
let { address.city, address.state } = person

// Rename with `as` (NOT `:`)
let { name as user_name } = user
let { address.city as hometown } = person

// Rest operator (flat only, no mixing with deep destructuring)
let { name, email, ...rest } = user

// In let bindings
let { quotient, remainder } = divide(a: 17, b: 5)

// In for loops
for { name, age } in users {
    Console.print(msg: "{name} is {age} years old")
}

// In match expressions
match result {
    :ok { value } -> value
    :err { error } -> handle_error(error)
}

// NOT in lambda parameters (use explicit let inside)
let process = user => {
    let { name, age } = user
    return "{name} ({age})"
}
```

**Note**: Maps use `.get()` method, not destructuring.

### 11.11 Pipeline Operator

The pipeline operator `|>` feeds into the `self` parameter:

```barnacle
// Instead of nested calls
let result = process(transform(filter(data, predicate: pred), mapper: map_fn))

// Use pipeline
let result = data
    |> filter(predicate: pred)
    |> transform(mapper: map_fn)
    |> process

// Real-world example
let active_user_names = users
    |> filter(predicate: u => u.active)
    |> map(transform: u => u.name)
    |> sort()
```

The `self` parameter is the **only positional parameter** in Barnacle. All other parameters are named.

### 11.12 Pattern Matching

```barnacle
// Match with arrow syntax (->)
match value {
    0 -> "zero"
    1 | 2 -> "one or two"
    n if n > 10 -> "big"
    _ -> "other"
}

// Destructuring atoms
match result {
    :ok { value } -> value
    :err { error } -> "Error: {error}"
}

match status {
    :pending -> "Waiting..."
    :complete { result } -> "Done: {result}"
    :failed { reason } -> "Failed: {reason}"
}

// Destructuring rows
match point {
    { x: 0, y: 0 } -> "origin"
    { x, y } -> "at {x}, {y}"
}

// List patterns
match list {
    [] -> "empty"
    [x] -> "singleton: {x}"
    [first, second, ...rest] -> "at least two"
}
```

### 11.13 Control Flow

```barnacle
// If is an expression
let max = if a > b { a } else { b }

// If with multiple branches
let category = if score >= 90 {
    "A"
} else if score >= 80 {
    "B"
} else if score >= 70 {
    "C"
} else {
    "F"
}

// While loop
let mut i = 0
while i < 10 {
    Console.print(msg: "i = {i}")
    i = i + 1
}

// For loop
for user in users {
    Console.print(msg: user.name)
}

// For loop with index
for { item, index } in users.enumerate() {
    Console.print(msg: "{index}: {item.name}")
}

// Break and continue
for x in numbers {
    if x < 0 {
        continue
    }
    if x > 100 {
        break
    }
    process(x)
}
```

### 11.14 Effects and Handlers

```barnacle
// Effect declaration
effect Console {
    print(msg: String) -> ()
    read_line() -> String
}

// Function with effects
export fn greet(name: String) -> () with Console {
    return Console.print(msg: "Hello, {name}!")
}

// Handler
handler real_console: Console {
    print(msg: msg) -> resume(wasi_print(msg: msg))
    read_line() -> resume(wasi_read_line())
}

// Handle expression
handle greet(name: "Alice") with real_console
```

### 11.15 Disposable Pattern (use)

The `use` keyword provides sugar over effect handlers for resource cleanup:

```barnacle
// use binding runs cleanup when block exits
use file = open_file(path: "config.toml")
let contents = read_file(file: file)
// file automatically closed here

// Multiple use bindings dispose in reverse order
use conn = connect(host: "localhost")
use tx = begin_transaction(conn: conn)
let result = execute_query(tx: tx)
commit(transaction: tx)
// tx disposed, then conn disposed

// Cleanup runs even on Fail
export fn process_file(path: String) -> String with FileSystem, Fail<IoError> {
    use file = open_file(path: path)
    if !is_valid(file: file) {
        return Fail.fail(error: IoError { message: "Invalid file" })
    }
    return read_file(file: file)
}
```

### 11.16 Operators

```barnacle
// Arithmetic
+ - * / %

// Comparison
== != < > <= >=

// Logical
&& || !

// Bitwise
& | ^ << >> ~

// String concatenation (for now, may change to + in future)
++

// Pipeline
|>

// Tag composition (type-level)
&   // in types: T & :tag

// Union (type-level)
|   // in types: :a | :b
```

### 11.17 No Semicolons

Semicolons are **not required** and are **not used** in Barnacle. They can optionally be used as a statement separator if you want multiple statements on one line, but this is discouraged.

```barnacle
// ✅ Idiomatic
let x = 1
let y = 2
let z = x + y

// ❌ Avoid (but technically allowed)
let x = 1; let y = 2; let z = x + y
```

Use line breaks and proper indentation instead.

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
export fn test_addition() {
    return assert_eq(a: 1 + 1, b: 2)
}

@Test
export fn test_with_effects() with Console {
    return handle {
        Console.print(msg: "test")
        return assert_eq(a: 1 + 1, b: 2)
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
    examples: ["add(a: 1, b: 2)"],
    see_also: ["subtract", "multiply"],
}
export fn add(a: Int, b: Int) -> Int { 
    return a + b 
}
```

Documentation is extracted at compile time and rendered as HTML, Markdown, or JSON.

---

## 14. Open Questions & Future Work

### 14.1 Memory Management Strategy

**Question**: Should Barnacle use reference counting, tracing GC, or Wasm GC?

**Considerations**:
- Reference counting: Predictable performance, but can leak cycles
- Tracing GC: Handles cycles, but requires runtime support
- Wasm GC: Native support in Wasm, but still evolving

**Current plan**: Perceus-style reference counting with optimizations. Consider Wasm GC as a future backend target.

### 14.2 Trait/Protocol System for Generic Constraints

**Question**: How should trait constraints work with generics?

**Current design**:
```barnacle
fn sort<T>(self list: List<T>) -> List<T> where T: Ord
```

**Open questions**:
- Syntax for trait bounds (`where T: Trait` vs `T: Trait` in signature)
- Associated types
- Higher-kinded types
- Trait coherence and orphan rules

**Plan**: Design in progress. Inspired by Rust traits + Haskell type classes.

### 14.3 Operator Overloading

**Question**: Should custom types support operator overloading?

**Use cases**:
- Mathematical types (complex numbers, vectors, matrices)
- DSLs (query builders, numeric code)

**Considerations**:
- Trait-based approach (e.g., `Add`, `Mul` traits)
- Risk of abuse (unclear code)
- Integration with numeric types

**Plan**: Likely yes, via traits. Details TBD.

### 14.4 Fail<E> — Stdlib vs Compiler-Known

**Question**: Should `Fail<E>` be a regular stdlib effect or compiler-known special case?

**Considerations**:
- Compiler-known: Can provide better error messages and optimizations
- Stdlib: Keeps compiler simpler, more principled

**Current plan**: Start as stdlib effect, add compiler hints if needed.

### 14.5 Self-Hosting Roadmap

**Question**: When should Barnacle be self-hosted (compiler written in Barnacle)?

**Milestones**:
1. Bootstrap compiler in Rust (✅ in progress)
2. Core language features stabilize (target: Q2 2026)
3. Begin self-hosted compiler development (target: Q3 2026)
4. Switch to self-hosted by default (target: Q4 2026)

**Benefits**: Dogfooding, better compiler errors, metaprogramming in Barnacle itself.

### 14.6 Package Registry

**Question**: How should the package ecosystem work?

**Options**:
1. Central registry (like crates.io, npm)
2. Decentralized (git URLs, IPFS)
3. Hybrid approach

**Current thinking**:
- Wasm component registries (warg, etc.)
- Integration with existing Wasm ecosystem
- Support for private registries

**Plan**: Follow WebAssembly Package Registry (warg) standards as they mature.

### 14.7 Field Visibility for Exported Types

**Question**: Should exported types have fine-grained field visibility?

**Current design**: Leaning towards **all fields visible** when type is exported.

```barnacle
// Option A: All fields visible (simpler)
export type User {
    name: String,
    email: String,
    password_hash: String,  // visible if User is exported
}

// Option B: Per-field visibility (more complex, may add later)
export type User {
    export name: String,
    export email: String,
    password_hash: String,  // private even if User is exported
}
```

**Tradeoffs**:
- All visible: Simple, encourages small focused types, structural typing friendly
- Per-field: More control, but adds complexity

**Plan**: Start with all-visible. Add per-field visibility if strong use cases emerge.

### 14.8 Generic Constraint Syntax

**Question**: What syntax for trait/protocol constraints on generics?

**Options**:
```barnacle
// Option 1: Where clause (Rust-style)
fn sort<T>(self list: List<T>) -> List<T> where T: Ord

// Option 2: Inline (Swift/Kotlin-style)
fn sort<T: Ord>(self list: List<T>) -> List<T>

// Option 3: Effect-like (experimental)
fn sort<T>(self list: List<T>) -> List<T> with T: Ord
```

**Plan**: Explore ergonomics in practice. Likely where clause for consistency with effect syntax.

### 14.9 Wasm Stack Switching

**Question**: When should we migrate from CPS transform to stack switching for multi-shot effects?

**Considerations**:
- Stack switching proposal is not yet standardized
- Performance improvements are significant
- Backward compatibility with CPS-compiled code

**Plan**: Support both strategies, with a compiler flag to choose. Migrate ecosystem gradually.

### 14.10 Concurrency Model

**Question**: What concurrency primitives should be built-in vs library?

**Current design**:
- `Async` effect is built-in
- Async handlers (executors) are in stdlib
- Structured concurrency via handlers

**Open**: Parallel computation (not just concurrent), work-stealing, data parallelism.

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
