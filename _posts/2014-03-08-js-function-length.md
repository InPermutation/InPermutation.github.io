---
layout: copyablepost
title: Using function.length to implement a stack language
date: "Sat, 08 Mar 2014 23:15:00-0500"
---

/\* `Function`s in JavaScript have a `length` property, measuring
the number of named formal parameters. This means that a stack-based language
can tell how many items to pop off the stack to pass to the function.

Here is a very simple stack language, implemented in quasi-literate JavaScript.

(That is, you can copy this post out of your browser into a `.js` file and
run it in Node.) \*/
{% highlight js %}
var interpreter = {
{% endhighlight %}

/\* The "Standard Library" is implemented as a
dictionary of plain JavaScript functions. \*/
{% highlight js %}
{% raw %}
    '+': function(a, b) { return [a + b]; },
    '-': function(a, b) { return [a - b]; },
    'dup': function(a) { return [a, a]; },
    'swap': function(a, b) { return [b, a]; },
    /* Must be wrapped. `console.log.length` === 0 */
    '.': function(i) { console.log(i); },
{% endraw %}
{% endhighlight %}

/\* **`run`** is the entry point to the interpreter.
It resets the `inputBuffer` and `stack` fields. \*/
{% highlight js %}
{% raw %}
    'run': function(program) {
        this.inputBuffer = program;
        this.stack = [];

        while (this.inputBuffer) {
            // read one word from inputBuffer & interpret it
            this.interpret(this.word());
        }
    },
{% endraw %}
{% endhighlight %}

/\* **`word`** removes one word from the `inputBuffer` and returns it. \*/
{% highlight js %}
{% raw %}
    'word': function(/* operates on inputBuffer, not stack */) {
        var word = '';
        while (this.inputBuffer) {
            var ch = this.inputBuffer.charAt(0);
            this.inputBuffer = this.inputBuffer.substr(1);
            if (/^\s?$/.test(ch)) {
                if (word) {
                    break;
                }
            } else {
                word += ch;
            }
        }
        return word;
    },
{% endraw %}
{% endhighlight %}

/\* **`interpret`** decides how to handle each word.

* If the word is in the dictionary, execute it.
* If it is a number, push the number onto the stack.
* Otherwise throw an exception.

\*/
{% highlight js %}
{% raw %}
    'interpret': function(word) {
        if (!word) return;
        if (word in this) {
            // it's defined, execute it.
            this.execute(this[word]);
        } else if (isFinite(word)) {
            // it's a number
            this.stack.push(parseFloat(word));
        } else throw new Error('unknown word `' + word + '`');
    },
{% endraw %}
{% endhighlight %}

/\*
**`execute`** provides the interoperability with JavaScript `Function`s.
It provides the top `fn.length`
items from the stack as arguments to the function.

If `fn` returns a value, the value is concatenated
back on top of the stack.

Multiple values can be returned in an array;
`Array.prototype.concat` will correctly concatenate
all of the values to `this.stack` in the intended order.
\*/
{% highlight js %}
{% raw %}
    'execute': function(fn) {
        var len = fn.length;
        // pop the top `len` items off the stack, in order
        var applies = this.stack.splice(-len, len);
        // apply the items to the Function
        var results = fn.apply(this, applies);
        if (results) {
            // put any results back onto the stack
            this.stack = this.stack.concat(results);
        }
    },
{% endraw %}
{% endhighlight %}

{% highlight js %}
};
{% endhighlight %}

/\* Unit tests to demonstrate usage \*/
{% highlight js %}
var assert = require('assert');
{% endhighlight %}

/\* An empty program leaves an empty stack. \*/
{% highlight js %}
interpreter.run('');
assert.deepEqual(interpreter.stack, []);
{% endhighlight %}

/\* The standard library: `dup`, `-`, `+`, `swap` \*/
{% highlight js %}
interpreter.run('3 dup');
assert.deepEqual(interpreter.stack, [3, 3]);
interpreter.run("2 6 -");
assert.deepEqual(interpreter.stack, [-4]);
interpreter.run('5 8 +');
assert.deepEqual(interpreter.stack, [13]);
interpreter.run('10 20');
assert.deepEqual(interpreter.stack, [10, 20]);
interpreter.run('10 20 swap');
assert.deepEqual(interpreter.stack, [20, 10]);
{% endhighlight %}

/\* We can reflect on just about any part of the execution of the interpreter. \*/
{% highlight js %}
interpreter.run('12 23');
assert.deepEqual(interpreter.stack, [12, 23]);
// continues...
{% endhighlight %}

/\* For example, calling `interpret` without `run` or `word`
can be used to single-step: \*/
{% highlight js %}
// ...continued
interpreter.interpret('swap');
assert.deepEqual(interpreter.stack, [23, 12]);
// continues...
{% endhighlight %}

/\* We can `execute` a function that isn't even in the dictionary: \*/
{% highlight js %}
// ...continued
interpreter.execute(function(a, b) {
    assert.deepEqual([a, b], [23, 12]);
    return ['whoa', 'nelly'];
});
assert.deepEqual(interpreter.stack, ['whoa', 'nelly']);
{% endhighlight %}

/\* Interpreting the word `'execute'` pops a function off the stack \*/
{% highlight js %}
interpreter.stack = [function() { return ['WOW'] }];
interpreter.interpret('execute');
assert.deepEqual(interpreter.stack, ['WOW']);
{% endhighlight %}

/\* Use the `word` tokenizer function to put single-word `String`s on the stack \*/
{% highlight js %}
interpreter.run('word foo');
assert.deepEqual(interpreter.stack, ['foo']);
{% endhighlight %}

/\* The preexisting `+` operator can work with this new `String` data type \*/
{% highlight js %}
interpreter.run('word Hello, word World! +');
assert.deepEqual(interpreter.stack, ['Hello,World!']);
{% endhighlight %}

/\* And `interpret` can be called on `String`s on the stack \*/
{% highlight js %}
interpreter.run('5 5 word + interpret');
assert.deepEqual(interpreter.stack, [10]);
{% endhighlight %}

/\* Even if the `String` is a result of some other computation! \*/
{% highlight js %}
interpreter.run('8 9 word sw word ap + interpret');
assert.deepEqual(interpreter.stack, [9, 8]);
{% endhighlight %}


