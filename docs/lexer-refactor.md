# Lexer refactor target

Status: planned; the current lexer has not been refactored. This records the
scope agreed in October 2026. The implementation is a learning exercise led by
Xann, with assistant review, explanations, and debugging help when requested.

## Goals

- Make legal state transitions explicit in Rust types. Each concrete state
  consumes itself and returns only its permitted outcomes. A single enum that
  every state can freely return does not establish this guarantee.
- Keep partially read token data in the state that owns it, rather than a shared
  bag of token fields and flags. Keep construction controlled so callers cannot
  manufacture invalid intermediate states.
- Keep the runtime driver responsible for dispatching states. Make character
  consumption and termination clear: processing must advance, make a bounded
  transition toward completion, finish, or return an error.
- Fix the lexer shortcomings identified during review, preserving supported
  syntax except where the identifier specification intentionally changes it.
- Implement the identifier policy in [syntax.md](syntax.md): Unicode 17.0
  XID_Start/XID_Continue, underscore restrictions, case sensitivity, and NFC
  normalization before keyword recognition and symbol emission. Preserve original
  spelling for diagnostics; do not normalize strings or comments.

The exact state types and their transition graph are an implementation design
exercise, not settled by this document. Sketch the states, owned data and legal
successors before committing to the design.

## Correctness and acceptance

Reproduce the review findings against the current lexer before fixing them;
observations from earlier branches are regression candidates, not fresh test
results. Cover:

- Unsupported characters and Unicode input: no table-index panic or repeated
  empty tokens without progress; reject invalid characters explicitly.
- End of input in each relevant reading state, including unfinished strings and
  comments: terminate or diagnose rather than hang or silently lose input.
- Zero at delimiters, numeric prefixes, malformed radix literals and overflow:
  accepted numbers retain their value; invalid numbers never silently become zero.
- Delimiter handling: no counter underflow/overflow or silently accepted crossed
  delimiters where the lexer currently checks nesting. Decide explicitly whether
  nesting checks remain here or belong to a later parser change.
- Comment opening and closing, including the previously observed `/*/1` case.
- Existing operators and their boundaries, strings, whitespace and supported
  ASCII names; Unicode names and canonically equivalent NFC spellings.

Use behavioral regression tests for these cases and compile-fail examples or
tests demonstrating that forbidden state transitions cannot be expressed through
its public transition interface. Tests should prove the guarantees rather than
mirror every implementation detail.

Run the workspace checks from [development.md](development.md). Any changed
behavior should be explained by a regression fix or the identifier policy.

## Outside this refactor

Parser precedence and associativity, assignment validation, AST/struct ordering,
CLI EOF behavior, semantic analysis, interpreter/compiler implementation, and
future language features are separate work. Do not expand this learning exercise
into fixing every issue found in the repository review.

## Current branch

The existing unpublished commits document identifier rules and future language
ideas and reorganize the README and development notes. They prepare context for
this work; they do not implement the state-machine refactor or Unicode support.
