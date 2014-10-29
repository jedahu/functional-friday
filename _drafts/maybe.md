---
title: Goodbye null
layout: post
---

Null pointer exceptions can be prevented at compile time using a simple data
type borrowed from Haskell.

~~~~~~~~~~~~~~~~~~~~haskell
data Maybe a = Nothing | Just a
~~~~~~~~~~~~~~~~~~~~


### The Maybe type

`Maybe` either holds a value or it doesn’t. Here it is in C#. An instance of
`Maybe<A>` is either `Nothing` or `Just` (empty or has just one item). There are
two constructors, `Nothing` and `Just`, and a method `Match()` which pattern
matches on those states.

~~~~~~~~~~~~~~~~~~~~csharp
public struct Maybe<A>
{
    private readonly bool isJust;
    private readonly A value;

    internal Maybe(A value)
    {
        isJust = true;
        this.value = value;
    }

    public B Match<B>(Func<B> nothing, Func<A, B> some)
    {
        return isJust ? some(value) : nothing();
    }
}

public static class Maybe
{
    public static Maybe<A> Nothing<A>()
    {
        return new Maybe<A>();
    }

    public static Maybe<A> Just<A>(A a)
    {
        return new Maybe<A>(a);
    }
}
~~~~~~~~~~~~~~~~~~~~

The presence of `Match()` and absence of members like `Value` or `Get()` is what
makes `Maybe<A>` superior to `Nullable<A>`. It is possible to attempt to access
the value of an empty `Nullable` (e.g. `new int?().Value`), which will fail at
runtime. The equivalent is not possible with `Maybe`. The only access is through
`Match()` which requires the caller to handle both `Nothing` and `Just` states.

Unlike `Nullable`, `Maybe` works with both reference types and value types.


### Maybe is useful

`IEnumerable` has a `First()` method. However, in the case of an empty
enumerable, this method throws an `InvalidOperationException`. Worse, exceptions
in C# are not checked so the compiler doesn’t bark if you don’t handle it. The
type signature of `First()` is `IEnumerable<T> => T`, but that is true only some
of the time.

There is another method `FirstOrDefault()` which returns `null` for the empty
case (for class types). This suffers from the same problem, except the exception
is a `NullReferenceException` which will be thrown when the null pointer is
dereferenced, which could be further up the call stack. Yuck. The type signature
of `FirstOrDefault()` is also `IEnumerable<T> => T`, and also only true some of
the time.

Enter `MaybeFirst()`.

~~~~~~~~~~~~~~~~~~~~csharp
public static Maybe<T> MaybeFirst<T>(this IEnumerable<T> e)
{
    try
    {
        return Maybe.Just(e.First());
    }
    catch (InvalidOperationException)
    {
        return Maybe.Nothing<T>();
    }
}
~~~~~~~~~~~~~~~~~~~~

The type signature of `MaybeFirst()` is `IEnumerable<T> => Maybe<T>`. In
contrast with `First()` and `FirstOrDefault()`, this is an ‘honest’
signature. Every time `MaybeFirst()` is called on an enumerable it will return a
`Maybe<T>`. No thrown exeptions, no nulls.


### Calling legacy null-using code

The .NET standard library is riddled with null pointers as is much other library
code. `Maybe` makes using them safer.

#### Converting between null, Nullable, and Maybe

~~~~~~~~~~~~~~~~~~~~csharp
public static Maybe<A> ToMaybe<A>(this A a) where A : class
{
    return ReferenceEquals(null, a) ? Maybe.Nothing<A>() : Maybe.Just(a);
}

public static Maybe<A> ToMaybe<A>(this A? a) where A : struct
{
    return a.HasValue ? Maybe.Just(a.Value) : Maybe.Nothing<A>();
}

public static A ToUnsafeNull<A>(this Maybe<A> m) where A : class
{
    return m.Match(() => null, a => a);
}

public static A? ToUnsafeNullable<A>(this Maybe<A> m) where A : struct
{
    return m.Match(() => new a?(), a => a);
}
~~~~~~~~~~~~~~~~~~~~

#### Calling exception throwing code safely

~~~~~~~~~~~~~~~~~~~~csharp
public static Maybe<A> Catch<A, E>(Func<A> f) where E : Exception
{
    try
    {
        return Maybe.Just(f());
    }
    catch (E)
    {
        return Maybe.Nothing<A>();
    }
}

public static Maybe<A> CatchAll<A>(Func<A> f)
{
    return Catch<A, Exception>(f);
}
~~~~~~~~~~~~~~~~~~~~
