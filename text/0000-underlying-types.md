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
UT ::= ('int' | 'uint') [':' IntegerConst] | '{' (UT Ident)* '}' | UT '[' ']' | UT
('|' UT)+
```

For the purposes of this RFC we consider only this simplified
language, though it is relatively easy to fully extend this.  Note,
however, that intersection and difference types are _not_ present in
the language of underlying types.  Likewise, unions are _tagged_ and,
hence, have a slightly different semantic where e.g. `int | int` is
permitted and `int|null` is not equivalent to `null|int`, etc.

# Terminology

* **Underlying Type.** The least over-approximation of a given Whiley
  type `T` which is used to guide the actual representation of that
  type on a backend target.

# Drawbacks and Limitations

None.

# Unresolved Issues

None.
