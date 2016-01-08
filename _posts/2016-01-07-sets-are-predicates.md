---
layout: post
title: "Predicates are sets"
date: "Thu, 03 Jan 2016 23:11:00-0500"
---
A **predicate** is a function that takes a value and computes a `true` or `false` response.

This type can be expressed easily in .NET. In C#, the 
[`Predicate<T>` Delegate](https://msdn.microsoft.com/en-us/library/bfcke1bz%28v=vs.110%29.aspx)
is declared as
{% highlight csharp %}public delegate bool Predicate<in T>(T obj){% endhighlight %}

----

A **set** is an abstract data type that stores values, in no particular order, without repetitions.

.NET defines many interfaces for sets:

* [`System.Collections.Generic.ISet<T>`](https://msdn.microsoft.com/library/dd412081%28v=vs.100%29.aspx), a generic mutable finite set
* [`System.Collections.Immutable.IImmutableSet<T>`](https://msdn.microsoft.com/en-us/library/dn467169%28v=vs.111%29.aspx), a generic immutable finite set
* [`Microsoft.SqlServer.Management.Sdk.Sfc.IReadonlySet`](https://msdn.microsoft.com/en-us/library/microsoft.sqlserver.management.sdk.sfc.ireadonlyset.aspx), an immutable non-generic finite set, for some reason in the `Microsoft.SqlServer` namespace.

All of these types have a `Contains` method, which can be cast to `Predicate<T>`. This means that **any of the set types in .NET can be used as predicates!** Intuitively, this predicate answers the question "does `this` set contain the argument?"

----

Is the converse also true? Can we create a set from predicates? Let's declare a `PredicateSet` in C# 6:

{% highlight csharp %}{% raw %}
using System;
using Microsoft.VisualStudio.TestTools.UnitTesting;
using System.Collections.Generic;
using System.Linq;

public partial class PredicateSet<T> {
    private readonly Predicate<T> indicator;
    public PredicateSet(Predicate<T> indicator) {
        this.indicator = indicator;
    }

    public bool Contains(T item) => indicator(item);
}

namespace Test {
    [TestClass]
    public partial class PredicateSetTest {
        [TestMethod]
        public void ItemTest() {
            var ps = new PredicateSet<int>(i => i == 2);
            Assert.IsFalse(ps.Contains(0));
            Assert.IsTrue(ps.Contains(2));
            Assert.IsFalse(ps.Contains(100));
        }
    }
}
{% endraw %}{% endhighlight %}

That looks sort of like a set. It seems like we can define more interesting sets already, too:

{% highlight csharp %}{% raw %}
namespace Test {
    partial class PredicateSetTest {
        [TestMethod]
        public void OddsTest() {
            var odds = new PredicateSet<int>(i => (i & 1) == 1);
            Assert.IsFalse(odds.Contains(0));
            Assert.IsTrue(odds.Contains(1));
            Assert.IsFalse(odds.Contains(2));
            Assert.IsTrue(odds.Contains(3));
            Assert.IsFalse(odds.Contains(-157204));
            Assert.IsTrue(odds.Contains(-157203));
            Assert.IsFalse(odds.Contains(65432));
            Assert.IsTrue(odds.Contains(65431));
        }

        private static readonly Random Random = new Random();
        /// It takes too long to run our tests on all the integers,
        /// so let's just try some random ones
        private static IEnumerable<int> SomeIntegers() {
            yield return int.MinValue;
            yield return int.MinValue + 1;
            yield return int.MaxValue;
            yield return int.MaxValue - 1;
            foreach (var i in Enumerable.Range(100, 200)) {
                yield return i;
            }
            foreach (var i in Enumerable.Range(1, 32768)) {
                yield return Random.Next(int.MinValue, int.MaxValue);
            }
        }

        [TestMethod]
        public void SpecialSets() {
            var empty = new PredicateSet<int>(i => false);
            var universal = new PredicateSet<int>(i => true);
            foreach (var i in SomeIntegers()) {
                Assert.IsTrue(universal.Contains(i));
                Assert.IsFalse(empty.Contains(i));
            }
        }
    }
}
{% endraw %}{% endhighlight %}

It's a neat trick. Can we add set algebra?

{% highlight csharp %}{% raw %}
public partial class PredicateSet<T> {
    public PredicateSet<T> Union(PredicateSet<T> other) =>
        new PredicateSet<T>(
            v => this.indicator(v) || other.indicator(v));

    public PredicateSet<T> Intersection(PredicateSet<T> other) =>
        new PredicateSet<T>(
            v => this.indicator(v) && other.indicator(v));

    public PredicateSet<T> Complement() =>
        new PredicateSet<T>(
            v => !this.indicator(v));
}
namespace Test {
    partial class PredicateSetTest {
        [TestMethod]
        public void UnionTest() {
            var a = new PredicateSet<int>(i => i == 3);
            var b = new PredicateSet<int>(i => i > 5 && i < 8);
            var u1 = a.Union(b);
            var u2 = b.Union(a); // assert commutivity
            foreach (var u in new[] { u1, u2 }) {
                Assert.IsFalse(u.Contains(2));
                Assert.IsTrue(u.Contains(3));
                Assert.IsFalse(u.Contains(4));
                Assert.IsFalse(u.Contains(5));
                Assert.IsTrue(u.Contains(6));
                Assert.IsTrue(u.Contains(7));
                Assert.IsFalse(u.Contains(8));
            }
        }

        [TestMethod]
        public void IntersectionTest() {
            var odd = new PredicateSet<int>(i => (i & 1) == 1);
            var b = new PredicateSet<int>(i => i > 5 && i < 8);
            var i1 = odd.Intersection(b);
            var i2 = b.Intersection(odd); // assert commutivity
            foreach (var u in new[] { i1, i2 }) {
                foreach (var i in SomeIntegers()) {
                    bool shouldContain = (i == 7);
                    Assert.AreEqual(shouldContain, i1.Contains(i));
                    Assert.AreEqual(shouldContain, i2.Contains(i));
                }
            }
        }

        [TestMethod]
        public void ComplementTest() {
            var odds = new PredicateSet<int>(i => (i & 1) == 1);
            var evens = odds.Complement();
            foreach (var i in SomeIntegers()) {
                Assert.AreNotEqual(odds.Contains(i), evens.Contains(i));
            }
        }
    }
}
{% endraw %}{% endhighlight %}

Well, that's a very powerful way to define sets. Why isn't it used more?

Unfortunately, `PredicateSet` can't implement any of the existing interfaces for sets. They all want to be `IEnumerable<T>`. There's no way for a `PredicateSet` to be able to generate all of the elements that are contained in it, or even count them.

{% highlight csharp %}{% raw %}
Predicate<string> isUpper = s => s.ToUpper() == s;
var allUppercaseStrings = new PredicateSet<string>(isUpper);

allUppercaseStrings.Count(); // ???
for (string s in allUppercaseStrings) {
    // ???
}
{% endraw %}{% endhighlight %}

Similarly, set relations like `Subset`, `Superset`, and `Equals` are impossible to calculate.

Here's the entire combined source listing for the `PredicateSet` class:

PredicateSet.cs
===============

{% highlight csharp %}{% raw %}
using System;

public partial class PredicateSet<T> {
    private readonly Predicate<T> indicator;
    public PredicateSet(Predicate<T> indicator) {
        this.indicator = indicator;
    }

    public bool Contains(T item) => indicator(item);
    public PredicateSet<T> Union(PredicateSet<T> other) =>
        new PredicateSet<T>(
            v => this.indicator(v) || other.indicator(v));

    public PredicateSet<T> Intersection(PredicateSet<T> other) =>
        new PredicateSet<T>(
            v => this.indicator(v) && other.indicator(v));

    public PredicateSet<T> Complement() =>
        new PredicateSet<T>(
            v => !this.indicator(v));
}
{% endraw %}{% endhighlight %}

Isn't that interesting? Set algebra and Boolean algebra seem to be incredibly closely related.
