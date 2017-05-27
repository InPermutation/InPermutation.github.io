---
layout: post
title: Holes in the C# type system
---
{% highlight cs %}
class Thing {
    public Thing(object o) { }
}

static class Program {
    public static void Main(string[] args) {
        var lazy = new Lazy<Thing>();
    }
}{% endhighlight %}

crashes at runtime.

{% highlight cs %}
static class Program2  {
    static Lazy<T> DefaultLazy<T>() where T : new() {
        return new Lazy<T>();
    }

    public static void Main(string[] args) {
        var lazy = DefaultLazy<Thing>();
    }
}{% endhighlight %}

doesn't compile.


