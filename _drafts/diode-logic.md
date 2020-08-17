---
layout: post
title: Diode logic in Factorio
date: 2020-08-17T03:32Z
---
With the release of Factorio 1.0 this week, lots of folks are writing articles about how 
[Factorio](https://blog.nindalf.com/posts/factorio-and-software-engineering/)
is
[just](https://www.reddit.com/r/factorio/comments/i8wwmp/factorio_is_similar_to_coding/)
like
[programming](https://www.reddit.com/r/factorio/comments/hnqchq/if_you_like_factorio_you_should_try_programming/).

I started playing Factorio seriously last year.
Since the beginning of this year, I've been playing with digital electronics.
Let's combine these two indoor enthusiast hobbies.

In Factorio, conveyor belts do not require external power.
Can this be exploited to create logic gates?
Is this subset of the game Turing complete?

## Step 0: Which way are signals flowing?

If you're thinking "the stuff on the belt is the signal,"
the problem is that stuff on belts travels fairly slowly, so you're limited in the speed of the circuit you can create.

But, if you fill up the belts with stuff, and use "jammed" as a 1 and "moving" as a 0, signals can propagate effectively immediately.

## Step 1: Create gates

We can exploit the splitters to create binary logic gates.
(This gets briefly confusing, because there are two kinds of inputs and outputs.)

A logic gate has two inputs and one output. A Factorio splitter has two inputs and two outputs.

We can provide the second splitter input with an always-full belt.

OR is simple: the outputs are normal, the second input is always-on, and the prioritized input is the signal input.

AND is also simple: the outputs are normal, the second input is always-on and the prioritized input.
Only if both outputs are moving does the signal input move.

But... can we do anything else? It seems like splitters can't make any other logical signals.

## Step 2: Invert it
I can't come up with a way to create a NOT: one belt flowing when the output is jammed. And I also can't think of NOR/NAND equivalents.
What else can we do?

Well, [Augustus De Morgan](https://en.wikipedia.org/wiki/De_Morgan%27s_laws)
figured it out in the XIXth century: the duality principle.

Let's assume our input signals can have their inverse provided.
Then, following De Morgan's laws, we can also find the complements of the outputs.
Negation is as simple as switching the belts' positions. With that, we can construct a NAND or a NOR.
NAND and NOR are both universal gates, which means they can be used to create any other Boolean circuit.

## Step 3: Practical applications??

If diodes can create AND and OR gates, can we use them to build a computer?

Sure - Mike Sutton shows
[how diode logic works on a breadboard](http://bread80.com/2019/09/13/breadboarding-with-diode-logic/).

This kind of [diode-resistor logic](https://en.wikipedia.org/wiki/Diode_logic)
("DRL") was heavily used in the 
[D-17B computer](https://en.wikipedia.org/wiki/D-17B) in 1962,
which was part of the Minuteman I NS-1OQ missile guidance system.
At that time, transistors were expensive and less reliable than passive components like diodes and resistors,
so the designers leaned heavily on DRL to minimize the number of transistors used.

Why did they use any transistors at all?

Unlike Factorio splitters, diodes have a "forward voltage drop" -
there is about a 0.6V lower voltage on the output than the input. So, there's a limit to the number of logic
gates that can be chained together. A circuit running at 3.3V could support a single stage of gates,
while 5V could marginally support 3 stages of gates.

There's also source resistance - the supply line has a resistor,
which becomes part of a voltage divider in the circuit.

So, real world diode computers would need to add transistors every few layers of gates, by implementing
[diode-transistor logic](https://en.wikipedia.org/wiki/Diode%E2%80%93transistor_logic) ("DTL").
