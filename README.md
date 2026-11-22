# Xann

Xann is an experimental custom programming language and a personal project for
learning language design and implementation. It is implemented in Rust. Its
syntax and specification are still taking shape, and it is not ready for
production use.

The project began as a Rust follow-along to OneLoneCoder's DIY language videos,
then grew beyond the original calculator and lexer into a language of its own.
The tutorial-era project remains in [olc-diy](https://github.com/XannMagus/olc-diy).
This repository is the home of continued Xann development and preserves that history.

## What works today

The terminal playground turns source text into tokens, builds a raw syntax tree,
and displays it. It supports numeric and string literals, names, binary
expressions, unary minus, grouping, assignments, variable declarations with
optional types, array types, and struct declarations.

Semantic analysis, interpretation and native compilation are not implemented.
The lexer and parser have known correctness issues; accepting source does not
establish that it is a valid executable program. Some examples explore syntax
that the parser does not support yet.

## Design goals

- **Validate before executing.** Check the complete program before any runtime
  execution. A valid prefix must not run if another part is statically invalid.
  Runtime failures can still occur after successful checking.
- **One language, two execution paths.** Have the frontend establish language
  meaning and produce a checked representation shared by an interpreter and a
  native compiler. The native backend is intended to use LLVM and a system linker.
- **Express guarantees in types.** Keep raw syntax separate from checked programs,
  control construction of validated representations, and use state types to
  enforce legal lexer transitions.
- **Make mutability explicit.** Bindings are intended to be immutable by default,
  with explicit mutable declarations. Final declaration spelling and constant
  evaluation rules remain open; existing examples use an earlier convention.
- **Support Unicode names deliberately.** Adopt Unicode identifier properties and
  NFC normalization while preserving source spelling for diagnostics. This policy
  is specified but not implemented yet.
- **Eventually host the language in itself.** Build a working Rust implementation,
  then use Xann to implement and build its own language tools. This is a long-term
  goal, not a current capability.

The [syntax reference](docs/syntax.md) records specified rules and implementation
status. [Development notes](docs/development.md) describe architecture, validation
boundaries and milestones. [Design musings](docs/design-musings.md) collect open
ideas and alternatives, rather than settled language requirements.

## Try the parser playground

With a Rust toolchain installed, run:

```sh
cargo run -p xann-cli
```

Enter source to inspect its parsed tree:

```text
let x = 1 + 2;
```

Use `file1` for the typed-declaration example or `file2` for the struct example.
Enter `quit` or `exit` to leave. Run without `--release` to see tree output.

## Current development focus

The next implementation task is the [lexer refactor](docs/lexer-refactor.md):
typed legal transitions, lexer correctness fixes, and Unicode identifiers with
NFC normalization. Parser, AST and CLI changes are outside that task's scope.
Implementation is pending.
