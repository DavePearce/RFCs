- Feature Name: `property_syntax`
- Start Date: `13-01-2021`
- RFC PR:

# Summary

The syntax for `property` declarations is awkward because it only
permits returning boolean values.  This propals will extend the syntax
to support other return types.

# Motivation

Here is an example `property` declaration from the standard library:

```
// Check if two arrays equal for a given subrange
public property equals<T>(T[] lhs, T[] rhs, int start, int end)
// Arrays must be big enough to hold subrange
where |lhs| >= end && |rhs| >= end
// All items in subrange match
where all { i in start..end | lhs[i] == rhs[i] }
```

The essential point of properties (as it stands) are that they receive special status within the theorem prover.  Specifically, properties can be _inlined_ during the verification process. This provides significant additional expressivity over functions which are treated as _uninterpreted_ during verification.

Unfortunately, unlike `function` or `method` declarations, properties
have no explicit return type and, hence, they are restricted to
returning only `bool` values.

# Technical Details

The proposal is fairly straightforward, we simply require that
`property` declarations additionally provide a _return type_ and a
_body_ consisting of an expression `e`.  Our example from above
becomes:

```
public property equals<T>(T[] lhs, T[] rhs, int start, int end) -> bool:
   // Arrays must be big enough to hold subrange
   |lhs| >= end && |rhs| >= end &&
   // All items in subrange match
   all { i in start..end | lhs[i] == rhs[i] }
```

Observe here that properties don't need an explicit `return` statement
as they simply return the value generated from their body `e`.  Currently, this is fairly limiting since expressions do not include any statement forms.  However, in the future, one might imagine that some statement forms will be supported as expressions.  For example:

```
property sum(int[] arr, int i) -> int:
    if i >= |arr|:
        0
    else:
        arr[i] + sum(arr,i+1)
```

Here, we have a simple _conditional expression_ which makes up the body of the `property`.

## Purity

An import question arises around _purity_.  Since we want to use
properties in specifications, they need to be pure.  However, when
specifying a `method`, we might want to permit them to accessing the
heap as the following illustrates:

```
type Link is { int data, LinkedList next }
type LinkedList is null | &Link

property disjoint(LinkstList l1, LinkedList l2):
    if l1 is null || l2 is null:
       true
    else:
       l1 != l2 && disjoint(l1->next, l2)
```

In other words, properties are really just macro's and their context
should determine their required level of purity.

# Terminology

# Drawbacks and Limitations

# Unresolved Issues
