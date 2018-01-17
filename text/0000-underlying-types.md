- Feature Name: `underlying_types`
- Start Date: `07-01-2018`
- RFC PR: (leave this empty)
- See also:
[RFC#0021](https://github.com/Whiley/RFCs/blob/master/text/0021-wyll.md)

# Summary

The connection between a source-level type and its "underlying type"
is important, particularly for backend code generators.  This RFC aims
to clarify exactly the meaning of an underlying type.

# Motivation

When compiling a Whiley program to a backend target, many challenges
arise in the mapping of Whiley types to those of the backend target.
The following illustrates:

```
type pos is (int x) where x >= 0
type neg is (int x) where x <= 0
```

This raises the question as to how the type `pos|neg` should be
represented.  For example, should it be a true union or just be
represented as an `int`?  This seems sensible in this case since the
test to distinguish them is efficient.  Then, what about
`(pos[])|(neg[])`?  Representing this as `int[]` makes the type test
very expensive.

Let us assume `pos|neg` should be implemented as a true union (as this
provides by far the simplest general rule).  We might say it's
_underlying type_ is `int|int`.  But, now we are in a pickle.
Consider this example:

```
function f(pos|neg x) -> int:
   if x is pos:
      return x
   else:
      return 0
```

The type of `x` on the true branch becomes `(pos|neg)&pos`.  The
problem is that, if we expand this to `(pos&pos)|(pos&neg)` we end up
with an underlying type of `int|int` (i.e. because `pos&neg` cannot be
reduced to `void` without reasoning about integer arithmetic).  This
is disappointing as, from `x is pos`, we would expect `x` to have an
underlying type of `int` in the true branch.

**The key challenge here is that the type `(pos|neg)&pos` is perfectly
  manageable at the source level, but ambiguous at the underlying level**.

Another interesting illustration of an ambiguous underlying type is
the following:

```
function f({int x, int y}|null x) -> {int y, int x}:
  if x is {int y, int x}:
    return x
  else:
    return {y:0,x:0}
```

Here, the type of `x` on the true branch is `{int x,int y}&{int y, int
x}`.  _Therefore, what is its underlying type?_  Either `{int x,int
y}` or `{int y,int x}` would seem equally viable candidates.

# Technical Details

The language of _underlying types_ is given by the following
(simplified) grammar:

```
UT ::= `null` | ('int' | 'uint') [':' IntegerConst] | '{' UT Ident (',' UT Ident)+ '}' | UT '[' ']' | UT
('|' UT)+ | Ident '<' UT '>' | Ident
```

For the purposes of this RFC we consider only this simplified
language, though it is relatively easy to fully extend this.  Note,
however, that intersection and difference types are _not_ present in
the language of underlying types.  Also, unions are _tagged_ and,
hence, have a slightly different semantic where e.g. `int | int` is
permitted and `int|null` is not equivalent to `null|int`, etc.
Finally, records have a slightly different semantic where e.g. `{int
x, int y}` is not equivalent to `{int y, int x}`.  Finally, a
recursive type looks something like this: `X<null|{int data, X
next}>`.

The language of _source-level types_ is given by the following
(simplified) grammar:

```
T ::= `null` | ('int' | 'uint') [':' IntegerConst] | '{' T Ident (',' T Ident)+ '}' | T '[' ']' | T
('|' T)+ | T1 'is' T2 | T1 'isnt' T2 | Ident ['?'] '<' T '>' | Ident
```

The main difference between the source-level and underlying types two
is the presence of `is` and `isnt` types.  Note, we replace
intersections with `is` types and differences with `isnt` types.  This
is needed to give the necessary direction.

**GOAL:** The goal, then, of this RFC is to enable a clear mapping
  from the Whiley source type of a given variable to its underlying
  type.

The key issue here that the underlying types determine exactly how
data will be represented on a machine.  For example, the type
`int|int` will be represented as a tagged union with two elements.  We
can think of the source-level types as being "abstract" in some sense,
whilst the underlying types are "concrete".

## Baseline

The simplest (yet safe) method for translating between source-level
types and underlying types is two apply these two rules:

```
T1 is T2 ==> T3, where T1 ==> T3


T1 isnt T2 ==> T3, where T1 ==> T3
```

This approach simply drops type refinements alogether.  This is safe
but, of course, not efficient.  For example, consider this program:

```
function add(int|null lhs, int|null rhs) -> int|null:
   if lhs is int && rhs is int:
      lhs = lhs + rhs
   return lhs
```

Using our two rules above, the translation of this to low-level
machine representation (e.g. C) would look something like this:

```
type tagged_t is { int tag, int|null data }

function add(tagged_t lhs, tagged_t rhs) -> tagged_t:
   if lhs is int && rhs is int:
      lhs.data = lhs.data + rhs.data
   return lhs
```

They key is that the declared type of `lhs` is its type throughout,
and the type tests do not change this.  The effect is that we must
constantly cast `lhs` as we attempt to use it.

###
# Terminology

* **Ambiguous Type.** An ambiguous type is a type at the Whiley source
  level which cannot be reduced to eliminate all occurrences of
  intersection or difference types.  For example, `({int|null f, int
  g}|{int f, int|null f}) is {int f, int g}` is ambiguous.  This is
  because the intersection represents a _type selector_, but we cannot
  determine which type is being selected.

* **Underlying Type.** The least over-approximation of a given Whiley
  type `T` which is used to guide the actual representation of that
  type on a backend target.

# Drawbacks and Limitations

None.

# Unresolved Issues

### Recursive Types

The hard case seems to be how to correctly deal with recursive types.
The typical example would be something like this:

```
type IN_List is null|{ IN_List next, int|null data }
type I_List is null|{ I_List next, int data }

function check(IN_List l) -> bool:
	return l is I_List
```

The type we are trying to reduce would probably be expressed like so:

```
IN_List<null|{IN_List next, int|null data}> is I_List<null|{I_List next, int data}>
```
To handle this, we might formulate a rule such as the following:
```
X<T1> is Y<T2> ==> Z<T3>, where T1 is T2 ==> T3 assuming X is Y ==> Z
```

Here, `Z` is an arbitrary introduced name for simplicity.  In our
previous example, we'd have something like this:

```
==> null|{IN_List next, int|null data} is null|{I_List next, int data}
==> (null is null|{I_List next, int data}) | ({IN_List next, int|null
data} is null|{I_List next, int data})
==> null | { IN_List is I_List next, int|null is int data}
==> null | { Z next, int data}
```

Here, `Z` is some introduced name.  This kinda works, though am not
really sure how to go about implementing it.
