---
layout: post
title: "Binary Boolean Operator: The Lost Levels"
date: "Wed, 05 Mar 2014 21:52:00-0500"
---
There are sixteen possible binary operations on Boolean inputs.
The operations can be numbered based on the truth table used to generate them.
For example, the `AND` function is defined by the following truth table:

<pre>
P   Q   |  P AND Q
--------+---------
1   1   |     1
1   0   |     0
0   1   |     0
0   0   |     0
</pre>
By reading down the result column, we determine that `AND` is function number `1000b`, or **8** in decimal.

Here are all 16 of the binary Boolean operations:

<pre>
#   |  11   10   01   00  |   Common name
----+---------------------+--------------
0   |  0    0    0    0   |   False
1   |  0    0    0    1   |   NOR
2   |  0    0    1    0   |
3   |  0    0    1    1   |   Not P
4   |  0    1    0    0   |
5   |  0    1    0    1   |   Not Q
6   |  0    1    1    0   |   XOR / Not Equal
7   |  0    1    1    1   |   NAND
8   |  1    0    0    0   |   AND
9   |  1    0    0    1   |   Equal
10  |  1    0    1    0   |   Q
11  |  1    0    1    1   |
12  |  1    1    0    0   |   P
13  |  1    1    0    1   |
14  |  1    1    1    0   |   OR
15  |  1    1    1    1   |   True
</pre>
Almost all of these operators are familiar, but there are four less-familiar operators.
11 and 13 are the same operation, but with their input order reversed.
4 and 2 are the inverses of those two, respectively.

The most widely known of these four siblings is operator number 11.
This operator is called the "material conditional".
It is used to test if a statement fits the logical pattern "P implies Q".
It is equivalent to `!P || Q` by the [material implication][2].

I only know one language that implementes this operation: [VBScript][1].

Instead of writing a test like this:

{% highlight python %}
if needs_val:
    assert(has_val())
{% endhighlight %}
or the logically identical, but difficult to read:
{% highlight python %}
assert(not needs_val or has_val()) # BAD! Precedence is difficult to reason about.
{% endhighlight %}
you could write it like this:
{% highlight python %}
assert(needs_val Imp has_val()) # AWESOME! (If `Imp` exists!)
{% endhighlight %}
Once you get the hang of this operator, you start wishing you could use it everywhere.
I wish more languages supported `Imp`.

[1]: https://msdn.microsoft.com/en-us/library/57eas59d%28v=vs.84%29.aspx "VBScript Imp Operator"
[2]: https://en.wikipedia.org/wiki/Material_implication_%28rule_of_inference%29
