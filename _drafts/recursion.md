---
title: Recursion
layout: post
---

Functions on a recursive [ADT](2014-05-23-adts) like `List` from the previous
post lend themselves to a recursive implementation. Consider a `Map` function:
`List<B> Map<A, B>(List<A>, Func<A, B>)`.

### Iterative Map

Note these things about `IterativeMap`:

1. It requires the definition of helper functions;

2. Two of those are partial functions, meaning they do not return a value for
   all possible inputs; and

3. `IterativeMap` itself mutates two variables.


~~~~~~~~~~~~~~~~~~~~csharp
public static bool IsNil<A>(this List<A> list)
{
    return list.Fold(() => true, (h, t) => false);
}

pubilc static A UnsafeHead<A>(this List<A> list)
{
    return list.Fold(
        () => { throw new InvalidOperationException("Head of empty list."); },
        (head, tail) => head);
}

pubilc static List<A> UnsafeTail<A>(this List<A> list)
{
    return list.Fold(
        () => { throw new InvalidOperationException("Tail of empty list."); },
        (head, tail) => tail);
}

public static List<B> IterativeMap<A, B>(this List<A> list, Func<A, B> f)
{
    var result = List.Nil<B>();
    while (!list.IsNil())
    {
        result = List.Cons(f(list.UnsafeHead()), result);
        list = list.UnsafeTail();
    }
    return result;
}
~~~~~~~~~~~~~~~~~~~~


### Recursive Map

The recursive implementation is shorter, clearer, requires no additional
functions, and mutates nothing.

~~~~~~~~~~~~~~~~~~~~csharp
public static List<B> RecursiveMap<A, B>(this List<A> list, Func<A, B> f)
{
    return list.Fold(
        () => List.Nil<B>(),
        (head, tail) => List.Cons(f(head), tail.RecursiveMap(f)));
}
~~~~~~~~~~~~~~~~~~~~

The recursive implementation has one problem though. For large lists it will run
out of stack space. If C# guaranteed tail-call elimination, the problem could be
solved by rewriting `RecursiveMap` with the recursive call in tail position.
Since C# does not guarantee tail-call elimination another workaround is needed.

~~~~~~~~~~~~~~~~~~~~csharp
public static List<B> TrampolinedMap<A, B>(this List<A> list, Func<A, B> f)
{
    var mapFn = Recur.Func<List<A>, List<B>>(
        map => ll => ll.Fold(
            () => Recur.Done(List.Nil<B>()),
            (head, tail) => map(tail).Map(
                tail1 => List.Cons(f(head), tail1))));
    return mapFn(list);
}
~~~~~~~~~~~~~~~~~~~~
