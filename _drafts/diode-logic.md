In Factorio, conveyor belts do not require external power.
Can this be exploited to create logic gates?
Is this subset of the game Turing complete?

## Step 0: Which way are signals flowing?

If you're thinking "the stuff on the belt is the signal,"
the problem is that stuff on belts travels fairly slowly, so you're limited in the speed of the circuit you can create.

But, if you fill up the belts with stuff, and use "jammed" as a 1 and "moving" as a 0, signals can propagate effectively immediately.

## Step 1: Create gates

We can exploit the splitters to create binary logic gates. 
This gets briefly confusing, because there are two kinds of inputs and outputs.

A logic gate has two inputs and one output. A Factorio splitter has two inputs and two outputs.

We can provide the second splitter input with an always-full belt.

OR is simple: the outputs are normal, the second input is always-on, and the prioritized input is the signal input.

AND is also simple: the outputs are normal, the second input is always-on and the prioritized input.
Only if both outputs are moving does the signal input move.

But... can we do anything else? It seems like splitters can't make any other logical signals.

## Step 2: Invert it
I can't come up with a way to create a NOT: one belt flowing when the output is jammed. And I also can't think of NOR/NAND equivalents.
What else can we do?

Well, Augustus De Morgan figured it out in the XIXth century: the duality principle.

Let's assume our input signals can have their inverse provided.
Then, following De Morgan's laws, we can also find the complements of the outputs.
Negation is as simple as switching the belts' positions. With that, we can construct a NAND or a NOR.
NAND and NOR are both universal gates, which means they can be used to create any other Boolean circuit.

## Step 3: Practical applications??

Diodes can create AND and OR gates.

### Aside: semiconductors and vacuum tubes.

* First semiconductor device was the diode, discovered by Ferdinand Braun in 1874.
* Thermionic emission 1873 Frederick Guthrie  / 1883 Thomas Edison
* First vacuum tube was also a diode, John Ambrose Fleming, 1904.
* Triode 1907
* 1925 Lilienfeld FET???
* 1939-1940 WWII need for radar amplifiers / Russell Ohl's cat's whisker / found a junction.
* Shockley and Brattain / surface science / 23 December 1947 
* John Bardeen, Walter Houser Brattain, and William Bradford Shockley were awarded the 1956 Nobel Prize in physics for their work.
* Maybe this aside is too much?

Diode-resistor logic ("DRL") - heavily used in the D-17B computer in 1962,
which was part of the Minuteman I NS-1OQ missile guidance system.
Transistors were expensive, so DRL reduced the number of transistors used.

Why did they use any transistors? Forward voltage drop and source resistance. Can't make huge circuits with real physical diodes.
