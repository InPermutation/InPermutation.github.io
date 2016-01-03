---
layout: post
title: "Predicates are sets"
date: "Sun, 03 Jan 2016 22:30:00-0500"
---
A **predicate** is a function that takes a value and computes a `true` or `false` response.

This type can be expressed easily in .NET. In C#, the 
[`Predicate<T>` Delegate](https://msdn.microsoft.com/en-us/library/bfcke1bz%28v=vs.110%29.aspx)
is declared as
{% highlight csharp %}public delegate bool Predicate<in T>(T obj){% endhighlight %}

A **set** is an abstract data type that stores values, in no particular order, without repetitions.

.NET defines many interfaces for sets:

* [`System.Collections.Generic.ISet<T>`](https://msdn.microsoft.com/library/dd412081%28v=vs.100%29.aspx), a generic mutable finite set
* [`System.Collections.Immutable.IImmutableSet<T>`](https://msdn.microsoft.com/en-us/library/dn467169%28v=vs.111%29.aspx), a generic immutable finite set
* [`Microsoft.SqlServer.Management.Sdk.Sfc.IReadonlySet`](https://msdn.microsoft.com/en-us/library/microsoft.sqlserver.management.sdk.sfc.ireadonlyset.aspx), an immutable non-generic finite set, for some reason in the `Microsoft.SqlServer` namespace.

All of these types have a `Contains` method, which can be stored as a `Predicate<T>`. This means that any of the set types in .NET can be used as predicates. Intuitively, this predicate answers the question "does `this` set contain the argument?"

Is the converse also true? Can we create a set from predicates? Let's declare a `PredicateSet` in C# 6:

{% highlight csharp %}{% raw %}
using NUnit.Framework;

public partial class PredicateSet<T>(Predicate<T> indicator)
{
    public bool Contains(T item) => indicator(item);
}

namespace Test
{
    [TestFixture]
    public partial class PredicateSetTest
    {
        [Test]
        public void ItemTest()
        {
            var ps = new PredicateSet<int>(i => i == 2);
            Assert.False(ps.Contains(0));
            Assert.True(ps.Contains(2));
            Assert.False(ps.Contains(100));
        }
    }
}
{% endraw %}{% endhighlight %}

That looks sort of like a set. It seems like we can define more interesting sets already, too:

{% highlight csharp %}{% raw %}
namespace Test
{
    partial class PredicateSetTest
    {
        [Test]
        public void OddsTest()
        {
            var odds = new PredicateSet<int>(i => (i & 1) == 1);
            Assert.False(ps.Contains(0));
            Assert.True(ps.Contains(1));
            Assert.False(ps.Contains(2));
            Assert.True(ps.Contains(3));
            Assert.False(ps.Contains(-157204));
            Assert.True(ps.Contains(-157203));
            Assert.False(ps.Contains(65432));
            Assert.True(ps.Contains(65431));
        }

        /// All the integers
        private static readonly IEnumerable<int> Integers =
            Enumerable.Range(int.MinValue, int.MaxValue).Concat(Enumerable.Range(0, int.MaxValue));

        [Test]
        public void SpecialSets()
        {
            var empty = new PredicateSet<int>(i => false);
            var universal = new PredicateSet<int>(i => true);
            foreach (var i in Integers)
            {
                Assert.True(universal.Contains(i));
                Assert.False(empty.Contains(i));
            }
        }
    }
}
{% endraw %}{% endhighlight %}

It's a neat trick. Can we add set algebra?

{% highlight csharp %}{% raw %}
public partial class PredicateSet<T>
{
    PredicateSet<T> Union(PredicateSet<T> other) =>
        new PredicateSet<T>(v => this.indicator(v) || other.indicator(v));

    PredicateSet<T> Intersection(PredicateSet<T> other) =>
        new PredicateSet<T>(v => this.indicator(v) && other.indicator(v));

    PredicateSet<T> Complement() => new PredicateSet<T>(v => !this.indicator(v));
}
namespace Test
{
    partial class PredicateSetTest
    {
        [Test]
        public void UnionTest()
        {
            var a = new PredicateSet<int>(i => i == 3);
            var b = new PredicateSet<int>(i => i > 5 && i < 8);
            var u1 = a.Union(b);
            var u2 = b.Union(a); // assert commutivity
            for (var u in new[] { u1, u2 })
            {
                Assert.False(u.Contains(2));
                Assert.True(u.Contains(3));
                Assert.False(u.Contains(4));
                Assert.False(u.Contains(5));
                Assert.True(u.Contains(6));
                Assert.True(u.Contains(7));
                Assert.False(u.Contains(8));
            }
        }

        [Test]
        public void 
    }
}
{% endraw %}{% endhighlight %}

