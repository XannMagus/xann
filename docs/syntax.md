# Xann syntax

This reference defines the language's syntax as decisions are made. Each section
states whether the current implementation enforces its rules. Accepted syntax is
separate from semantic validity: parsing produces a raw AST, and semantic analysis
will resolve names and check their use.

## Bindings and mutability

**Specified rule; declaration syntax is being revised.**

Everything is immutable by default. Mutability must be explicitly requested at
variable declaration. Initialization establishes a value; it does not itself
make the binding mutable.

An immutable runtime binding and a compile-time constant are distinct concepts.
A runtime binding may be initialized from a runtime computation. A compile-time
constant must be evaluable during compilation. Their declaration keywords and
the rules for compile-time evaluation are not yet specified here.

The existing examples use the earlier `let`/`const` convention and will need to
be updated when declaration syntax is settled.

## Identifiers

**Specified; Unicode support and NFC normalization are not implemented yet.**
The current lexer accepts ASCII names and may fail on other input. The rules below
are the target for the lexer refactor.

Xann adopts Rust's Unicode identifier character policy and NFC normalization,
using Unicode 17.0 character properties. This does not adopt Rust's keywords,
raw-identifier syntax, or rules for external/module names.

### Characters

An identifier has this form:

```text
identifier := (XID_Start | "_") XID_Continue*
```

`XID_Start` and `XID_Continue` are Unicode properties defined by Unicode Standard
Annex #31. They distinguish characters allowed at the beginning of a name from
those allowed later. Combining marks and digits may continue a name but cannot
start one. Underscores can begin or continue a name; a lone `_` is not a name.
Zero-width joiner (U+200D) and zero-width non-joiner (U+200C) are excluded.

Examples of valid names:

```text
width
_width
width2
café
π
变量
Москва
```

Spaces, punctuation such as `-` and `.`, and emoji do not belong to identifiers.
For example, `first-name` is separate names with a subtraction operator, not one
name; `object.field` contains a dot between names. Neither `2width` nor a name
starting with a combining accent is a valid identifier.

### Normalization and equality

Identifiers are case-sensitive and normalized to NFC before keyword recognition
and emission as symbol tokens. Two names are equal exactly when their NFC forms
are equal. The original source spelling must remain available for diagnostics;
normalization does not rewrite source text.

For example, these Unicode sequences denote the same name:

| Source characters | NFC name |
| --- | --- |
| `c a f é`, ending with U+00E9 | `café` |
| `c a f e ◌́`, ending with U+0065 U+0301 | `café` |

Here `◌` only makes the combining accent visible in the table; it is not part of
the name. This is Unicode notation, not a specification of escape syntax in Xann.

Normalization does not fold case: `Width` and `width` are different names. It does
not equate visually similar letters from different scripts, such as Latin `a`
(U+0061) and Cyrillic `а` (U+0430). String literals and comments retain their text
without identifier normalization.

### Keywords

The following spellings are reserved and cannot be ordinary identifiers:

```text
true false null let const class new import from fn if else
foreach while for export typeof in struct static
```

Keywords are case-sensitive: `let` is a keyword, while `Let` is a valid name.
Reservation does not imply that a keyword's language feature is implemented.
No keyword-escape syntax is specified yet.

### References

- [Rust Reference: identifiers](https://doc.rust-lang.org/reference/identifiers.html)
- [Unicode 17.0 identifier specification, UAX #31 revision 43](https://www.unicode.org/reports/tr31/tr31-43.html)
- [Unicode normalization, UAX #15](https://www.unicode.org/reports/tr15/)
