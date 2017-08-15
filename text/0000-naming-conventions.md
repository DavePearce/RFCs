- Feature Name: `naming_conventions`
- Start Date: `15-08-2017`
- RFC PR:

# Summary

This proposal is to lay out the recommended style guidelines for
writing Whiley programs.

# Motivation

Providing style guidelines for new developers is important to help
ensure consistency across different Whiley programs.  This document is
the first attempt to provide a concrete set of guidelines.  These
conventions loosely follow the Rust style guidelines (see [RFC#430](https://github.com/rust-lang/rfcs/blob/master/text/0430-finalizing-naming-conventions.md)])

# Technical Details

In general, Whiley employs `snake_case` for named declarations
(e.g. `type`, `function`, `method`), but uses `CamelCase` for compound
types (i.e. records).

| Item | Convention |
-------|-------------
| modules | `snake_case` |
| Functions | `snake_case` |
| Methods | `snake_case` |
| Types | `snake_case` (for primitives) or `CamelCase` (for compounds) |
| Local Variables | `snake_case` |
| Static Variables | `SCREAMING_SNAKE_CASE` |

The only convention which requires judgement is the distinction
between `snake_case` and `CamelCase` for type declarations.  This
differs from Rust where `CamelCase` is recommended for all type
declarations.  This distinction is used because Whiley, unlike many
languages, supports _type aliasing_.  Furthermore, primitive types are
already lowercase (e.g. `int`, `bool`, etc) and aliases of primitive
types should retain this.  As an example:

```
type nat is (int x) where x >= 0
```

The convention is for `nat` over `Nat` to signal that this is a
constrained primitive type.  Note that, for the purposes of this RFC,
arrays should be considered primitive types.  Thus, we have `set`
over `Set`:

```
type set is (int[] items)
where all { i in 0..|items|, j in 0..|items| | (i != j) ==> (items[i]
!= items[j])
```

Finally, the preference of `CamelCase` for compounds applies
primarily to records and unions involving records.  For example:

```
type Point is { int x, int y}
```

This follows convention as any record type should follow `CamelCase`.
Likewise, a more subtle example would be:

```
type LinkedList is null | { LinkedList next, int data }
```

This is adjudicated to be sufficiently "complex" as to warrant
`CamelCase`.

Finally, we note the reason for choosing `CamelCase` for records
essentially arises from C++ and, later, Rust.  In particular, in C++,
class names generally follow `CamelCase`.

# Terminology

In changing the language, it is important to develop a suitable
"vernacular" for talking about the new aspects of the language.  For
example, if a new piece of syntax is proposed, then a standard way of
referring to this should be proposed.  Likewise, if a new phase of the
compiler is introduced, then suitable terminology for discussing the
various aspects of this should be developed.

# Drawbacks and Limitations

None.

# Unresolved Issues

None.
