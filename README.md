# Xann

Xann is an experimental language implemented in Rust, originally inspired by
OneLoneCoder's DIY language series. The current playground parses source into a
raw AST; semantic analysis, interpretation, and native compilation are future work.

## Quickstart

With a Rust toolchain installed, run:

```sh
cargo run -p xann-cli
```

Enter source to inspect its parsed AST:

```text
let x = 1 + 2;
```

Use `file1` for the typed-declaration example or `file2` for the struct example.
Enter `quit` or `exit` to leave. Run without `--release` to see AST output.

See the [syntax reference](docs/syntax.md) for language rules and implementation
status, and [development notes](docs/development.md) for the workspace, checks,
and versioning policy.
