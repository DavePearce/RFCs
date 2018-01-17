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
('|' UT)+ | X '<' UT '>'
```

For the purposes of this RFC we consider only this simplified
language, though it is relatively easy to fully extend this.  Note,
however, that intersection and difference types are _not_ present in
the language of underlying types.  Also, unions are _tagged_ and,
hence, have a slightly different semantic where e.g. `int | int` is
permitted and `int|null` is not equivalent to `null|int`, etc.
Finally, records have a slightly different semantic where e.g. `{int
x, int y}` is not equivalent to `{int y, int x}`.

The language of _source-level types_ is given by the following
(simplified) grammar:

```
T ::= `null` | ('int' | 'uint') [':' IntegerConst] | '{' T Ident (',' T Ident)+ '}' | T '[' ']' | T
('|' T)+ | T1 'is' T2 | T1 'isnt' T2 | X ['?'] '<' T '>'
```

The main difference between the source-level and underlying types two
is the presence of `is` and `isnt` types.  Note, we replace
intersections with `is` types and differences with `isnt` types.  This
is needed to give the necessary direction.

**GOAL:** The goal, then, of this RFC is to enable a clear mapping
  from the Whiley source type of a given variable to its underlying
  type.

## Mapping

The rules for converting a source-level type into an underlying type
are now presented.

### Inductive Cases

These cases describe how changes in an element of some type are
propagated outwards.

```
T[] ==> S[], where T ==> S

{T1 f1, ... Tn fn} ==> {S1 f1, ... Sn fn}, where T1 ==> S1, ..., Tn
==> Sn

{T1 f1, ..., void fi, ... Tn fn} ==> void
```

The above simply reduce a compound type when one or more of its
children reduces.  The second case for a record simply eliminates
records which end up with a `void` field.  Unions are similar:

```
T1 | ... | Tn ==> S1 | ... | Sn, where T1 ==> S1, ..., Tn ==> Sn

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
T1 is T2 ==> T2
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

### Isnt Rule

This is the more complex rule, as it must eliminate types where
possible.  We begin with the easy primitive cases:

```
int isnt int ==> void

int isnt null ==> null

int isnt { ... } ==> int

int isnt T[] ==> int

null isnt null ==> void

null isnt int ==> null

null isnt { ... } ==> null

null isnt T[] ==> null

{ ... } isnt int ==> { ... }

{ ... } isnt null ==> { ... }

{ ... } isnt T[] ==> { ... }

T[] isnt int ==> T[]

T[] isnt null ==> T[]

T[] isnt { ... } ==> T[]
```

Arrays

Records

Nominals

These cases are more complex and require reasonably clarification.

```
T1[] is T2[]   ==> S[], where (T1 is T2) ==> S
```

The above enables, for example, `(int|null)[] is int[]` to reduce to `int[]`.

```
T is T1|T2    ==> (T is T1)|(T is T2)
```

Likewise, the above enables `int is int|null` to reduce first to
`int|void` and then `int`.

```
T1|T2 is T    ==> (T1 is T)|(T2 is T)
```

This, unfortunately, implies that `pos|neg is pos` reduces to `int|int`.

```
(T1 is T2) is T3 ==> S is T3, where (T1 is T2) ==> S
```

This final case handles nested `is` types in a relatively expected
fashion.  For example, `(int|int[]|null is int|null) is int` reduces
to `int`.

## Isnot Types

The language of Whiley types is updated so that, instead of difference
types (e.g. `A-B`), we have _isnot_ types (e.g. `A isnot B`).  We can
then provide a simple mapping from Whiley types to underlying types.

### Primitive Cases

These cases are straightforward and don't require much clarification.

```
int isnot int     ==> void

int isnot { ... } ==> int

{ ... } isnot int ==> { ... }

T[] isnot int     ==> T[]

int isnot T[]     ==> int

{ ... } isnot T[] ==> { ... }

T[] isnot { ... } ==> T[]
```


# Terminology

* **Ambiguous Type.** An ambiguous type is a type at the Whiley source
  level which cannot be reduced to eliminate all occurrences of
  intersection or difference types.  For example, `({int|null f, int
  g}|{int f, int|null f})&{int f, int g}` is ambiguous.  This is
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
