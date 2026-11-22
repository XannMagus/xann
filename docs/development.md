# Development

Exploratory alternatives for constructors, traits, dependency injection, and
library distribution are recorded in [design musings](design-musings.md).
Those notes are not a definitive language target.

## Workspace

| Crate | Responsibility |
| --- | --- |
| `xann-core` | Shared language definitions and the future checked program representation. |
| `xann-frontend` | Lexing, raw AST construction, and future semantic analysis producing a checked program. |
| `xann-interpreter` | Future execution of the checked program. |
| `xann-compiler` | Future LLVM code generation, object emission, and linking into a native binary. |
| `xann-cli` | The `xann` executable: source input, orchestration, and displaying results/errors. |

The front end and both execution backends depend on `xann-core`. The interpreter
and compiler do not depend on front-end internals. The CLI wires the stages
together. LLVM dependencies and linker invocation will belong to `xann-compiler`.

The checked program is not implemented yet. When introduced, it will express
resolved names, types, and validated operations and will be the same input to both
backends. A raw parsed AST is not evidence of semantic validity.

### Frontend validation boundary

Parsing produces a raw AST or syntax diagnostics. Semantic analysis validates
that AST and produces a checked program or semantic diagnostics. Invalid source
may remain available for inspection and diagnostics, but must not reach either
execution backend as executable input.

Interpreter and compiler entry points will accept the checked representation,
not a raw AST with a validity flag. Its construction must be controlled so callers
cannot bypass validation. The frontend must establish all specified static
language rules before producing that representation and explain validation
failures through diagnostics. This does not guarantee the absence of runtime
failures, such as unavailable files or division by zero.

The complete program must pass frontend validation before any runtime execution
begins. A statically valid prefix must not execute if another part of the program
is invalid. Interpretation means executing the checked representation directly;
native compilation means generating machine code. Both obey the same validation
boundary. Checking first does not itself prove that the checker is sound.

The frontend owns language meaning: name resolution, type and operation checking,
and any specified initialization, ownership, or access rules. The compiler owns
target-specific lowering, ABI/layout choices permitted by that meaning, LLVM code
generation, object emission, and linking. It can reject an unsupported target or
report a backend/toolchain failure even for a valid checked program.

Memory rules visible to programs must be specified upstream; physical packing
and target-specific optimizations belong downstream. Representation-neutral
transformations may eventually be shared if both backends benefit, but are not
required for frontend validity.

## Versioning

All workspace crates share a version following [SemVer 2.0.0](https://semver.org/spec/v2.0.0.html).
The language is currently an alpha with no stable public API. Its version sequence
starts with the first commit unique to `custom-lang` (`52932af`), excluding the
earlier calculator/shunting-yard development from the version count.

For this initial alpha series, each commit is a revision toward `0.0.1`:
`0.0.1-alpha.1`, `0.0.1-alpha.2`, and so on. The historical branch head
`b3d828e` is revision 17. These are retrospective version labels, not published
releases or Git tags. The initial workspace version is `0.0.1-alpha.17`.
Alpha, beta, and release-candidate revisions use `alpha.N`, `beta.N`, and `rc.N`.
Stage transitions depend on the milestones below, not commit count. Later release
targets get their own revision series.

## Release milestones

| Stage completed | Milestone |
| --- | --- |
| Alpha | A complete vertical slice from source through parsing and semantic analysis to interpreter execution and a compiler-produced native binary. |
| Beta | A stable core syntax, grammar, and language semantics; a complete Rust implementation; and the language capable of building its own implementation written in Xann. |
| Migration before `1.0.0` | Replace the Rust implementation with Xann. The final prerelease removes the last Rust implementation components. |
| `1.0.0` | The language implementation is entirely written in Xann and can build itself. |

The alpha milestone requires an executable subset, not the complete language.
The beta milestone completes the Rust implementation and establishes the language
needed for self-hosting. The subsequent migration replaces that implementation
with Xann before the first stable release.

Self-hosting concerns Xann's implementation. It does not require LLVM, the system
linker, or other external build dependencies to be rewritten in Xann.

## Checks

```sh
cargo fmt --all --check
cargo check --workspace --locked
cargo test --workspace --locked
```

The inherited implementation has no automated behavioral tests yet.

## Planned lexer refactor

The [lexer refactor target](lexer-refactor.md) records the agreed scope, state
transition guarantees, regression coverage, and exclusions. Implementation is
still pending.

## Playground limitations

AST display is debug-only. The class/function example describes planned syntax.
Known lexer failures and parser bugs remain; EOF currently does not exit the
playground.
