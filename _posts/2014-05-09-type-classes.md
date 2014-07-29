---
title: Type classes
layout: post
---

Welcome to the inaugural *Functional Friday* post. I intend to explore, as
far as is possible in C#, what functional programming brings to the table:
types, laws, and equational reasoning. Example code is in the C# project at
the root of this repository.

First up: parametricity and type classes.


## Parametricity

The type signature of a [parametrically polymorphic][pp] function informs
about what the function *cannot* do. For example, from the function type
signature, `IEnumerable<T> Mystery<T>(T x)` it can be deduced that `Mystery`
will not instantiate a new `T` and will not call any methods of `T`. In
fact, because `T` is a type parameter without constraints, `Mystery` cannot
touch `x` in any way, which means that it operates the same regardless of
the type of `T`. Here are a few of the possible implementations of
`Mystery`.

[pp]: http://en.wikipedia.org/wiki/Parametric_polymorphism

~~~cs
IEnumerable<T> One<T>(T x)
{
    yield return t;
}

IEnumerable<T> Two<T>(T x)
{
    yield return t;
    yield return t;
}

IEnumerable<T> Infinite(T x)
{
    while (true) { yield return x; }
}
~~~

Take `One` for example: `One(3)` and `One("a")` operate exactly the same. If
`Mystery` were not parametrically polymorphic the same guarantees would not
apply. The number of possible implementations of `IEnumerable<int>
Mystery2(int x)` are much greater than `Mystery` because `Mystery2` can
instantiate a new `int` and call any of the methods on `x`. Here are a
couple of possible `Mystery2` implementations.

~~~cs
IEnumerable<int> Multiply(int x)
{
    yield return x * 5;
}

IEnumerable<int> OfLength(int x)
{
    for (var i = 0; i < x; ++i)
    {
        yield return i;
    }
}
~~~

[Parametricity][] is the name for this restrictive yet helpful property of
parametrically polymorphic functions.

[Parametricity]: http://en.wikipedia.org/wiki/Parametricity


## Parametricity is not enough

What if what is needed is not a fully parametric function but one that
allows *some* interaction with objects of a parametric type?

C# allows this through type constraints. An example function is, `bool
Mystery3<T>(IEnumerable<T> xs) where T : IEquatable<T>` where `T` is
constrained to be a type that implements the `IEquatable<T>` interface. The
type of `Mystery3` constrains its implementation such that it cannot touch
objects of type `T` *except* through the methods of `IEquatable<T>`. A
possible implementation follows.

~~~cs
bool AllEqual<T>(IEnumerable<T> xs) where T : IEquatable<T>
{
    return xs.Any()
        ? xs.All(x => x.Equals(xs.First()))
        : true;
}
~~~

The downside to this method of constraining type variables is that it
requires ahead-of-time implementation of an interface. For example, objects
of a type `Entity` which does not implement `IEquatable` and is defined in a
non-modifiable library cannot be used with `AllEqual` unless they are
wrapped in a conforming class.

~~~cs
public sealed class EntityWrapper
    : IEquatable<EntityWrapper>
{
    private readonly Entity entity;

    public EntityWrapper(Entity entity)
    {
        this.entity = entity;
    }

    public Entity WrappedValue
    {
        get { return entity; }
    }

    public bool Equals(Entity other)
    {
        ....;
    }
}
~~~

IEnumerable<Entity> entities = ...;
var allAreEqual = AllEqual(entities.Select(e => new EntityWrapper(e)));

But wrapping and unwrapping object is not space or time efficient. There
ought to be a better way to constraint type variables, one that does not
require wrapping or ahead-of-time inheritance.

Enter, type classes...


## What type classes do

Type classes add constraints to type variables in parametrically polymorphic
types and functions. They enable constraints on type variables that are
checked at compile time and are [ad-hoc][ap] (I.e. they do not require the
constrained types to be subtypes of an interface or base class). They were
pioneered in Haskell but have a simple encoding in C#.

The .NET System library already uses parts of the encoding, but not always
to best type-safe effect. The second example in this post will demonstrate
this along with improvements that can be made for greater type safety.

Further information on type classes can be found on [Wikipedia][w] and in
[‘A Gentle Introduction to Haskell’][intro].

[ap]: http://en.wikipedia.org/wiki/Ad_hoc_polymorphism
[w]: http://en.wikipedia.org/wiki/Type_class
[intro]: http://www.haskell.org/tutorial/classes.html


## A C# encoding

The type-class encoding in C# consists of three things:

1. An interface.
2. Implementation(s) of the interface for a type and a behaviour.
3. Use of the interface to constrain type parameters in type declarations.

The aim of type classes and of this encoding is to enable constraints on
type variables that are visible in type signatures and are ad-hoc (I.e. do
not require the constrained types to implement an interface or inherit from
a base class). For these properties to hold in C#, the encoding requires
that the following rules be obeyed:

1. Type class interfaces must inherit only from other type class interfaces.
2. Type class implementations must not implement any interfaces other than
   type-class interfaces or inherit from any classes. In addition they must
   be sealed.
3. Implementations of a type class must have only a public nullary
   constructor and be free of mutable fields and properties.

If these rules are abided by, different type-class behaviour will always be
visible in the type system.


## Example 1: the JsonShow type class

**Task:** Show objects as JSON strings. Show only objects that it makes
sense to show (no DB access objects or other non-data types) and fail all
others at compile time. Show primitive objects and others whose definitions
cannot be opened for modification.

**Solution**: A JsonShow type class.

~~~cs
public interface IJsonShow<in T>
{
    string ShowJson(T x);
}
~~~

With implementations for some primitives.

~~~cs
public sealed class JsonShowInt
    : IJsonShow<int>
{
    public string ShowJson(int x)
    {
        return x.ToString();
    }

    public static JsonShowInt Instance = new JsonShowInt();
}

public sealed class JsonShowString
    : IJsonShow<string>
{
    public string ShowJson(string x)
    {
        return string.Concat(
            "\"",
            x.Replace("\"", "\\\""),
            "\"");
    }

    public static JsonShowString Instance = new JsonShowString();
}
~~~

And a complex type.

~~~cs
public sealed class JsonShowEnumerable<T, TJsonShow>
    : IJsonShow<IEnumerable<T>>
    where TJsonShow : IJsonShow<T>, new()
{
    private readonly TJsonShow tJsonShow = new TJsonShow();

    public string ShowJson(IEnumerable<T> x)
    {
        return string.Concat(
            "[",
            string.Join(",", x.Select(tJsonShow.ShowJson)),
            "]");
    }
}
~~~

An example of call-site use.

~~~cs
public void PrintDelimitedJson<T, TJsonShow>(
    T obj,
    TJsonShow show)
    where TJsonShow : IJsonShow<T>
{
    Console.Write("<");
    Console.Write(show.ShowJson(obj));
    Console.Write(">");
    Console.WriteLine();
}

PrintDelimitedJson(3, JsonShowInt.Instance); // => <3>
PrintDelimitedJson("foo", JsonShowString.Instance); // => <"foo">
PrintDelimitedJson(
    new[] {1,2,3,4},
    new JsonShowEnumerable<int, JsonShowString>());
    // => <[1,2,3,4]>
~~~

An alternative call-site formulation where the type class instance is
created rather than passed in. Note the `new()` constraint on `TJsonShow`.

~~~cs
public void PrintDelimitedJson2<T, TJsonShow>(T obj)
    where TJsonShow : IJsonShow<T>, new()
{
    Console.Write("<");
    Console.Write(new TJsonShow().ShowJson(obj));
    Console.Write(">");
    Console.WriteLine();
}

PrintDelimitedJson2<int, JsonShowInt>(3); // => <3>
PrintDelimitedJson2<string, JsonShowString>("foo"); // => <"foo">
PrintDelimitedJson2<
    IEnumerable<int>,
    JsonShowEnumerable<int, JsonShowInt>>(
        new[] {1,2,3,4});
    // => <[1,2,3,4]>
~~~

It is enforced at compile time that types without an `IJsonShow` type class
instance cannot be shown.

~~~cs
PrintDelimitedJson2<StringBuilder, ???>(new StringBuilder());
// No IJsonShow<StringBuilder> instance type to put in place of ???.
~~~


## Example 2: improving on the .NET System library

Here is the equality type class declaration in C#. It might look familiar;
it is the `System.Collections.Generic.IEqualityComparer`.

~~~cs
public interface IEqualityComparer<in T>
{
    bool Equals(T x1, T x2);
    int GetHashCode(T x);
}
~~~

Here’s how `IEqualityComparer` is currently used in
`System.Collections.Generic.Dictionary`.

~~~cs
public class Dictionary<TKey, TValue>
{
    public Dictionary();

    public Dictionary(IEqualityComparer<TKey> comparer);

    public Dictionary(int capacity, IEqualityComparer<TKey> comparer);

    public Dictionary(
        IDictionary<TKey, TValue> dictionary,
        IEqualityComparer<TKey> comparer);
}
~~~

There is a problem with this usage as demonstrated by the functions,
`ReadFileBytes` and `ResponseContentLength`.

~~~cs
// assuming a case sensitive file system
public void WriteToFile(
    string fname,
    byte[] content,
    IDictionary<string, FileStream> files)
{
    if (files.ContainsKey(fname))
    {
        files[fname].Write(content, 0, content.Length);
    }
}

public int? ResponseContentLength(IDictionary<string, string> headers)
{
    if (headers.ContainsKey("content-length"))
    {
        try
        {
            return int.Parse(headers["content-length"]);
        }
        catch (Exception)
        {
            return new int?();
        }
    }
    return new int?();
}
~~~

Because it is intended to work with a case sensitive file system,
`ReadFileBytes` requires that the passed-in dictionary make case sensitive
comparisons when finding a key. `ResponseContentLength` on the other hand is
intended to operate over a collection of HTTP response headers, which are
case insensitive.

However, nothing prevents the following bad code from compiling and
operating incorrectly.

~~~cs
var files = new Dictionary<string, FileStream>(
    "CaMel.txt",
    StringComparer.InvariantCultureIgnoreCase)
    {
        { "valuablefile", ... },
        ...
    };
string content = ...;
WriteFileText("ValuableFile", content, "files);
// OH NO! Unintentionally overwrote valuablefile with content intended
// for ValuableFile.

var headers = new Dictionary<string, string>(
    StringComparer.InvariantCulture)
    {
        { "Status", "200 OK" },
        { "Content-Length", "256" }
    };
var length = ResponseContentLength(headers);
// OOPS! Got a length of int?() instead of int?(256).
~~~

This hole in type safety is easily fixed if the type of the equality comparer
is added to the dictionary’s type signature. Here is a `Dict` type that
wraps `Dictionary` to achieve this.

~~~cs
public interface IDict<TKey, TValue, TEq>
    : IDictionary<TKey, TValue>
    where TEq : IEqualityComparer<TKey>, new()
{
}

public sealed class Dict<TKey, TValue, TEq>
    : Dictionary<TKey, TValue>
    , IDict<TKey, TValue, TEq>
    where TEq : IEqualityComparer<TKey>
{
    public Dict(TEq comparer)
        : base(comparer) { }

    public Dict(int capacity, TEq comparer)
        : base(capacity, comparer) { }

    public Dict(IDictionary<TKey, TValue> dictionary, TEq comparer)
        : base(dictionary, comparer) { }
}
~~~

To make our example safe the two string comparers need to be reimplemented
as rule abiding type class instances.

~~~cs
public sealed class StringEq
    : IEqualityComparer<string>
{
    public bool Equals(string x1, string x2)
    {
        return StringComparer.InvariantCulture.Equals(x1, x2);
    }

    public int GetHashCode(string x)
    {
        return StringComparer.InvariantCulture.GetHashCode(x);
    }

    public static StringEq Instance = new StringEq();
}

public sealed class IgnoreCaseEq
    : IEqualityComparer<string>
{
    public bool Equals(string x1, string x2)
    {
        return StringComparer
            .InvariantCultureIgnoreCase.Equals(x1, x2);
    }

    public int GetHashCode(string x)
    {
        return StringComparer
            .InvariantCultureIgnoreCase.GetHashCode(x);
    }

    public static IgnoreCaseEq Instance = new IgnoreCaseEq();
}
~~~

Now the example functions can be rewritten safely. Code that calls these
safe functions with dictionaries of the wrong equality comparer will fail to
compile.

~~~cs
// assuming a case sensitive file system
public void WriteToFile(
    string fname,
    byte[] content,
    IDict<string, FileStream, StringEq> files)
{
    if (files.ContainsKey(fname))
    {
        files[fname].Write(content, 0, content.Length);
    }
}

public int? ResponseContentLength(
    IDict<string, string, IgnoreCaseEq> headers)
{
    if (headers.ContainsKey("content-length"))
    {
        try
        {
            return int.Parse(headers["content-length"]);
        }
        catch (Exception)
        {
            return new int?();
        }
    }
    return new int?();
}
~~~


## Example 1 in Haskell

For comparison, here is the JsonShow example in Haskell.

~~~hs
class JsonShow a where
    showJson :: a -> String

instance JsonShow Int where
    showJson x = show x

instance JsonShow String where
    showJson x = "\"" ++ map replace x ++ "\""
      where
        replace '"'  = '\\'
        replace char = char

instance (JsonShow a) => JsonShow [a] where
    showJson x = "[" ++ join "," (map showJson x) ++ "]"

printDelimitedJson :: (JsonShow a) => a -> IO ()
printDelimitedJson x = putStr ("<" ++ showJson x ++ ">")

main :: IO ()
main = do
    printDelimitedJson 1
    printDelimitedJson "abc"
    printDelimitedJson [1, 2, 3, 4]
~~~

Note that the type-class instances are anonymous and are passed implicitly
to `printDelimitedJson`. This is possible because Haskell allows only one
type-class instance per type. Alternate instances require the use of a
`newtype` wrapper, a syntactically light way of wrapping a type that has
zero runtime overhead. To demonstrate, here are the two string equality type
class instances from example 2 (sans hash code method).

~~~hs
class Eq a where
    (==) :: a -> a -> Bool

instance Eq String where
    s1 == s2 = stringEqual s1 s2

newtype IgnoreCase = IgnoreCase String

instance Eq IgnoreCase where
    (IgnoreCase s1) == (IgnoreCase s2) =
        stringEqual (downCase s1) (downCase s2)
~~~

`String` and `IgnoreCase` are identical at runtime, but are treated as
different types by the compiler, and therefore can have an Eq instance each.

Implicit instance passing is not possible in C#, so it makes no sense to
restrict type-class instances to one per type like Haskell does, especially
since following the ‘rules’ means alternate instances will still be
distinguishable by type, as example 2 demonstrates.
