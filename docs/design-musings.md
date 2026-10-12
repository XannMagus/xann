# Design musings and future possibilities

Design notebook, 2026-10-12. I am keeping these ideas and alternatives here to
return to as Xann develops. These notes distinguish design intent from alternatives
that remain open. They are not the definitive language specification or part of
the current lexer work. The example syntax is provisional.

The [syntax reference](syntax.md) records specified rules. Moving an idea from
these notes into that reference requires a separate design decision.

## Current scope

The current session concerns a lexer FSM whose state types enforce legal
transitions, fixes for lexer shortcomings, and Unicode identifiers with NFC
normalization. The later language ideas below are recorded so they can be revisited
without expanding that implementation scope.

## Mutability and constants

Immutability by default is a language rule, recorded in the
[syntax reference](syntax.md#bindings-and-mutability).

The existing examples use `let` for mutable bindings and `const` for immutable
bindings. A possible replacement distinguishes all three concepts directly:

```text
let width = readWidth();  // immutable runtime binding
mut count = 0;           // mutable runtime binding
const scale = 2;         // compile-time constant
```

This would use `mut` in place of Rust's `let mut`, while retaining Rust's meanings
for `let` and `const`. The declaration spelling is not final. Reference/ownership
rules and the permitted compile-time expressions still need specification.

The language draws on Rust, PHP, and Haskell.

## Constant evaluation

I want compile-time constants to have a demonstrable value before the frontend
finishes. Scalar and array expressions are the starting candidates. Pure functions
with no calls to other functions are a possible restriction, but I am open to the
most permissive rule that still establishes the value during compilation.

Purity, eligibility for constant evaluation, and successful constant evaluation
are distinct. Purity alone does not establish termination or make every operation
available to the compile-time evaluator. A function can loop without calling
another function; nested calls between eligible pure functions need not introduce
effects.

One possible approach is to evaluate supported expressions and eligible pure calls
within finite resource limits, rejecting an initializer if no value can be
established. Exceeding an evaluation budget would be a constant-evaluation failure,
not proof of impurity or nontermination. Proving termination in advance is a
different, stricter alternative. Eligibility rules and budgets remain open.

The constant evaluator could eventually share execution machinery with the
interpreter while exposing only operations permitted during constant evaluation.
The dependency arrangement must preserve the frontend/backend boundaries.

## Interpreter validation and responsiveness

I want the whole program checked before any runtime execution, including in the
interpreter. A valid prefix must not run if a later part is statically invalid.
Runtime failures remain possible after successful validation.

This is still interpretation if the checked representation is executed directly.
Generating native code and immediately running it would instead be a compiler
operating in compile-and-run mode. Keeping distinct execution paths could help
debugging and comparisons of interpreter and compiler behavior.

Checking large programs on every run may make the interpreter cumbersome. Possible
mitigations include retaining checked programs for repeated execution, incremental
analysis of changed definitions and affected dependents, cached dependency checks,
and bounded constant evaluation. Reused checks must remain valid under the current
source and analysis assumptions; caching must not weaken whole-program validation.

The initial checker can be straightforward. More expensive inference, ownership
analysis, or generic instantiation should be deliberate decisions, and performance
work should follow measurements. A future REPL could check each submitted unit
before execution without abandoning the validation boundary.

## Examples and source extension

The examples express intended capabilities; their spelling and structure remain
open to redesign. In particular, `examples/00.lang` is not a fixed grammar contract
or parser acceptance test. It explores classes, constructors, methods, loops,
arrays, filesystem operations, and time operations. Executing it also requires
library facilities, not just parsing its syntax.

I am considering `.xn` as the source extension to match the name Xann. The files
still use `.lang`. Examples can become acceptance tests once their syntax and
semantics are settled.

## Classes, constructors, and fields

I am not satisfied with the current class example: it lacks explicit constructor
syntax and visibility. If I keep classes, I want data and behavior together in a
form such as:

```text
class DirectoryReader {
    private recentFiles: []string;

    public constructor(
        private directoryPath: string,
        maximumAge: number = 24,
    ) {
        this.recentFiles = [];
    }
}
```

If classes are retained, these are the rules I want:

- An explicit constructor is mandatory, even when it has no parameters.
- A constructor parameter carrying visibility declares and initializes a field,
  inspired by PHP constructor property promotion.
- Parameters without visibility are ordinary parameters.
- Fields not promoted from parameters are declared before the constructor, with
  a visibility keyword replacing `let`. They cannot have default values at their
  declaration and must be initialized in the constructor. Either violation must
  be rejected by the frontend before execution or compilation.
- Constructor parameters otherwise use regular function parameter syntax,
  including default values.
- `private`, `protected`, and `public` are the familiar initial visibility options;
  modules, namespaces, and packages have not been designed yet.

Checking initialization on every successful constructor path requires semantic
analysis; rejecting defaults in field declarations is a syntax check. Both must
finish before producing a checked program. Constructor parameter defaults remain
allowed; the prohibition concerns fields declared before the constructor.

If classes are adopted, both restrictions must be enforced before the frontend
produces a checked program: parsing rejects field declaration defaults, and
semantic analysis rejects missing required constructor initialization. Invalid
code must receive diagnostics explaining why it is invalid and must never be
accepted as executable input by either backend. The raw AST may still be inspected.
See the [frontend validation boundary](development.md#frontend-validation-boundary).

Questions still open include visibility defaults, static fields, and whether
classes or inheritance belong in the language at all.
The exact private/protected/public access rules are still to be specified.

## Named arguments and callable contracts

I want named arguments for constructors and ordinary function calls:

```text
new DirectoryReader(directoryPath: "/path/to/directory", maximumAge: 48);
```

Parameter names become part of a callable's interface. I accept that constraint:
an override must preserve the overridden parameter names. Trait implementations
could follow the same rule.

Parameter names can be omitted when arguments are supplied in declaration order.
Named arguments should allow a subset of defaulted parameters to be supplied
without repeating defaults for earlier omitted parameters:

```text
fn connect(host: string, port: number = 443, timeout: number = 30)

connect("example.org");
connect("example.org", 8080);
connect(host: "example.org", timeout: 60);
```

I am considering requiring all arguments at a callsite to be named or all to be
positional. This simplifies binding rules but requires naming required arguments
when overriding a later optional argument. Allowing a positional prefix followed
by named arguments is the alternative; neither rule is final yet.

Possible binding rules are: positional arguments follow declaration order and may
omit trailing defaulted parameters; named arguments may appear in any order and
omit any defaulted parameter; required parameters cannot be omitted. Unknown or
duplicate argument names would be errors, including a parameter supplied both
positionally and by name if mixed calls are allowed.

Argument expressions must not perform external side effects. A function invoked
to produce an argument value or reference must be verified pure. This controls
effects inside the argument expression; it does not prohibit effectful work in
preceding statements. Default-value evaluation still needs specification.

Evaluating argument expressions in callsite order, independently of binding order,
is a possible rule, not a settled requirement. Purity removes ordinary external
side effects but does not itself settle termination or error-order behavior.

## Purity

A pure function must produce equivalent output values for exactly the same input
values and must not change external state. Reads of files, clocks, random sources,
or other changing state beyond its inputs and local computation are impure.
Immutable constants and verified pure callees are allowed.

Function-local allocation and mutation are allowed. Temporary allocations must be
reclaimed when the function exits; allocations transferred into the output are
exempt. Output equality concerns values, not allocation addresses. Referenced
input state is part of the input, rather than just the reference's address.
The resource-management mechanism and behavior on exceptional exits remain open.

The frontend must establish purity before accepting a call in an argument
expression. Implementation options include explicit checked `pure` declarations,
inferred effects, or inference with optional declarations that require purity as
an interface guarantee. An annotation alone is not proof. Callable and trait
contracts may need effect information, especially for callbacks and dynamic calls.

A checker can enforce a sound set of permitted operations without deciding whether
every arbitrary program is observationally pure. Conservative checking may reject
an effect in an unreachable branch or a call whose type lacks effect information.
Determining effect reachability in arbitrary code is not generally decidable.

Haskell separates I/O actions through types and uses `ST` to encapsulate local
mutation; OCaml permits impure functions without distinguishing them in ordinary
function signatures. These are useful comparisons, not implementation commitments.
Neither comparison establishes Xann's allocation-reclamation requirement.

## Memoization

I want to permit memoization of verified pure functions and, if practical, activate
it automatically. The relevant guarantee is referential transparency: substituting
a cached result must preserve observable behavior.

Memoization introduces an exception to the temporary-allocation rule: a cache can
persist beyond a call if it exists only to reuse equivalent output and does not
change observable results. A runtime-managed cache is one possible implementation;
user-managed persistent caches require separate rules and verification.

A memoized function may share the same returned reference only when it is immutable
at every callsite. Otherwise the result must be copyable or clonable and an
independent copy/clone is returned. The cache must not become mutable through a
caller. If the necessary isolation cannot be established, that sharing/copying
strategy cannot be used.

Immutability must extend through reachable state and aliases. Copy/clone operations
must preserve the result value without exposing shared mutable cache contents or
introducing external effects. Ordinary shallow cloning is insufficient when it
retains mutable shared children. Whether to require these guarantees through
reference/ownership types, special clone contracts, or another mechanism is open.

Cache keys must represent the actual inputs; an address alone is insufficient for
data that can change. Closures also bring function identity and captured state into
the key design. Cache bounds, eviction, concurrency, failure caching, and the cost
of comparison/copying remain open. Purity makes caching semantically possible,
not necessarily profitable.

Reusing intermediate work across different inputs, rather than caching each
input's output, may be useful but needs stronger invariants. An arbitrary mutable
cache must not influence the answer. This generalization is not solved merely by
marking the surrounding function pure.

Background references:

- [Haskell I/O](https://www.haskell.org/onlinereport/haskell2010/haskellch7.html)
- [Haskell local mutable state, `ST`](https://hackage.haskell.org/package/base/docs/Control-Monad-ST.html)
- [OCaml functions and effects](https://ocaml.org/docs/values-and-functions)

## Possible inheritance and implicit superclass construction

If inheritance is introduced, a subclass could omit an explicit superclass
constructor call when its ordinary, non-promoted parameters can be forwarded by
matching names. Forwarding should match superclass **constructor parameters**,
not fields: fields may be derived internally and are not necessarily inputs to
construction.

The implicit call might run at the beginning or end of the subclass constructor;
the ordering remains undecided. Calling it first would ensure inherited state
exists before the subclass body uses it.

Further questions include missing required arguments, defaults, type compatibility,
explicit superclass calls, and which parameters are eligible for forwarding.
`protected` has little purpose until subclass access is defined.

## Classes versus structs with implementations

I like two aspects of these approaches:

- Classes co-locate data and behavior in a unified declaration.
- Separate struct/type declarations and implementation blocks make it possible
  to add behavior separately, particularly trait implementations.

The alternatives are classes with interfaces, structs with separate `impl`
blocks and traits, or classes with ordinary methods plus separate trait
implementations. Traits need not imply inheritance or inherited state.

```text
trait FileReader {
    fn read(path: string): string;
}

impl FileReader for LocalFileReader {
    public fn read(path: string): string {
        // ...
    }
}
```

I also like the possibility of implementing traits for generic types. Generic
syntax, implementation-coherence rules, and the final object model remain open.

## Required customization and configuration

I want libraries to be able to require client-supplied customization or
configuration, with those obligations enforced by semantic checking.
Merely omitting an implementation does not establish such an obligation.

Possible ways to express these requirements:

- A constructor requires a value implementing a trait; callers must supply it.
- A typed configuration value has mandatory fields without defaults.
- A library explicitly declares dependencies that application assembly must bind.

The last option might use a declaration such as `requires storage: Storage;`.
That spelling is provisional. A library can be valid with declared obligations, while a complete
application must discharge them. Type checks can establish the presence and type
of configuration; runtime facts such as a directory's existence still require
runtime validation.

## Dependency injection

I am interested in language-level automatic wiring, inspired by Symfony's
service container: request a trait/interface and obtain an initialized
implementing instance automatically.

Two possible approaches:

- Discover any existing initialized instance implementing the requested contract.
- Explicitly register instances or providers in a container/scope, with automatic
  wiring confined to that scope.

Explicit registration would keep the creation of unrelated objects from changing
dependency resolution. Selection with zero/multiple providers, lifetimes,
ownership, construction failure, and dependency cycles all need explicit rules.
Static versus runtime resolution also remains open.

An incremental possibility is to prototype injection in the standard library,
then consider language syntax once its semantics are understood. Dependency
injection does not itself require class inheritance.

## Source dependencies and compiled libraries

I lean toward distributing dependencies as source and compiling them with the
application, similar to the usual Cargo workflow. Rust also supports compiled
libraries, so that comparison does not dictate Xann's artifact model.

With source dependencies, checking a library can validate its definitions and
declared requirements without producing an executable. Application building can
combine sources, resolve bindings, and emit an executable. This avoids designing
a distributable compiled-library format immediately.

Supporting compiled libraries later would require a distinct artifact containing
code and metadata for exported types, signatures, traits, and unresolved
requirements. It would be a valid library artifact intended for linking, not a
deliberately unrunnable executable. The checker would use the metadata without
needing original source.

Source distribution, separate compilation, cached build artifacts, and required
customization are separate decisions. Source-only distribution need not prevent
local caching. Module/package design and binary-library support remain open.

## Memory layout and foreign libraries

Reordering struct fields to improve packing is acceptable as an implementation
choice. Source declaration order must remain available to the frontend; physical
layout order is a separate backend decision. Layout changes must preserve named
field access and language-defined initialization/evaluation order.

I do not want FFI in the first version. I am unsure whether I want compatibility
with C libraries later. Rust library interoperability interests me more, since
the bootstrap implementation will be written in Rust.

An implementation language does not establish a foreign ABI. Rust's native ABI
does not promise stability, and its default struct representation does not promise
declaration-order layout. Possible later approaches include compiling Rust
integration code with the same toolchain, a defined foreign boundary (possibly
`extern "C"` wrappers despite implementation in Rust), or a custom plugin/runtime
protocol. None is selected; no Rust-library compatibility is promised.

If foreign interoperability or observable layouts are introduced, their types
would need explicit layout/ABI contracts. Ordinary Xann types could retain layout
freedom. This is future design work, not part of the lexer refactor.

## Revisiting these ideas

These ideas are separate from the identifier specification and current lexer
refactor. Before implementing any of them, settle the capability and its semantics,
then revise the examples and add the corresponding specification and tests.
