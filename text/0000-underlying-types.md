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

## Mapping

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

### Inductive Cases

These cases describe how changes in an element of some type are
propagated outwards.

```
T[] ==> UT[], where T ==> UT

{T1 f1, ... Tn fn} ==> {UT1 f1, ... UTn fn}, where T1 ==> UT1, ..., Tn
==> UTn

{T1 f1, ..., void fi, ... Tn fn} ==> void
```

The above simply reduce a compound type when one or more of its
children reduces.  The second case for a record simply eliminates
records which end up with a `void` field.  Unions are similar:

```
T1 | ... | Tn ==> UT1 | ... | UTn, where T1 ==> UT1, ..., Tn ==> UTn

void | ... | T | ... | void ==> T

void | ... | void ==> void
```

The latter two rules handle `void` in different ways.  For a union
containing exactly one non-void type, then it reduces to that type.
Likewise, a union containing only `void` types is reduced to `void`.
There are some interesting implications of this, such as:

```
(int|pos[]|neg[]) is int[]
==> (int is int[]) | (pos[] is int[]) | (neg[] is int[])
==> void | int[] | int[]
```

The latter is considered a well-formed underlying union type.  The key
feature here is that, starting from a union, we are left still with a
union.  The presence of `void` then ensures that this union has the
same tag layout.

### Is Rule

The case for handling types of the form `T1 is T2` is very easy, as
follows:

```
T1 is T2 ==> UT, where T2 ==> UT
```

The benefit of this rule is that the following compiles as expected:

```
function f(pos|neg x) -> int:
   if x is pos:
      return x
   else:
      return 0
```

Here, `x` has type `pos` on the true branch.

### Isnt Rules

These are the more complex rules, as we must actually eliminate types
where possible.  We begin with the easy primitive cases:

```
int isnt int ==> void

int isnt null ==> null

int isnt { ... } ==> int

int isnt T[] ==> int

null isnt null ==> void

null isnt int ==> null

null isnt { ... } ==> null

null isnt T[] ==> null

{ T1 f1, ..., Tn fn } isnt int ==> { UT1 f1, ..., UTn fn }, where T1 ==> UT1, ..., Tn ==> UTn

{ T1 f1, ..., Tn fn } isnt null ==> { UT1 f1, ..., UTn fn }, where T1 ==> UT1, ..., Tn ==> UTn

{ T1 f1, ..., Tn fn } isnt T[] ==> { UT1 f1, ..., UTn fn }, where T1 ==> UT1, ..., Tn ==> UTn

T[] isnt int ==> T[]

T[] isnt null ==> T[]

T[] isnt { ... } ==> T[]
```

**Arrays.**  The rules for arrays are surprisingly simple:

```
T1[] isnt T2[] ==> UT[], if T1 :> T2 and T1 ==> UT

T1[] isnt T2[] ==> void, otherwise
```

By the first rule, we have `(int|null)[] isnt int[]` refines to
`(int|null)[]` which follows as `[1,null] isnt int[]` holds.
Likewise, by the second rule, we have `int[] isnt (int|null)[]`
refines to `null`.

**Records.** The rule for records is more involved:

```
{ T1 f1, ..., Tn fn } isnt { S1 g1, ..., Sm gm } ==> { UT1 f1, ...,
UTn fn }, where {f1,...fn} != {g1,...,gm} and T1 ==> UT1, ..., Tn ==> UTn

{ T1 f1, ..., Tn fn } isnt { S1 g1, ..., Sn gn } ==> ?????
```

{ int|null x, int|null y} isnt { int x, int y }
==> { int|null isnt int x, int|null y }|{ int|null x, int|null isnt
int y }
==> { null x, int|null y}|{ int|null x, null y}

**General Rule.**

```
T1 isnt T2 ==> void, if T1 <: T2

T1 isnt T2 ==> T1, otherwise
```

**Nominals.**

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

None.
