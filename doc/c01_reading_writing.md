\newpage

<!--
Copyright (c) 2021 by Gabe Parmer.

Redistribution of this file is permitted under the GNU General Public
License v2.
-->

# Interfaces and Design

This section lays the foundation for understanding not only how to design systems, but also some of the many constraints that go into that design.
These include a base-level understanding of software complexity on the micro- and macro-scale.
It then pivots into thoughts, goals, and guidelines for system design.
Finally, it discusses a few existing systems with a philosophical lens.
These include:

- UNIX with counterpoints in programming environments, and frameworks,
- Plan 9 as pushing the UNIX philosophy to the limit, and gleaning a deeper understanding of the power and limitations of naming, and
- Capability-based OSes which have a similarly principled take on systems design, but focus on construction at a much lower level of abstraction.

## Micro-Design: reading code, and writing for reading

Understanding complexity

- How can we think about the complexity of code?
    One (flawed) version is automatic metrics like [cyclomatic complexity](https://en.wikipedia.org/wiki/Cyclomatic_complexity) -- approximately, that complexity is the sum of the number of expressions, the number of edges (each `if` and `while` creates two), and the number of connected components (a proxy for this is one for each loop).
	This has been applied to [understand](https://yurichev.com/blog/cyclomatic/) it for Linux.
	There are many [arguments against](https://www.cqse.eu/en/news/blog/mccabe-cyclomatic-complexity/) it and [alternatives](https://carlalexander.ca/what-is-software-complexity/).
	I do find this metric flawed as it doesn't consider the fact that we have to *reason* about the *paths* in the program, but the number of paths is exponential with respect to (WRT) the number of conditions and loops.
-  What are some high-level prescriptions to constrain this type of complexity?
	A number of programming guidelines for high-confidence systems all seek to constrain this complexity.
	Some examples:

    - [jpl](https://yurichev.com/mirrors/C/JPL_Coding_Standard_C.pdf) coding standard for c,
	- the quite common [misra](https://en.wikipedia.org/wiki/MISRA_C) [c](http://caxapa.ru/thumbs/468328/misra-c-2004.pdf) c, and
	- the [composite style philosophy](https://github.com/gwsystems/composite/blob/ppos/doc/style_guide/composite_coding_style.pdf).

### Reading Code

The main question we're going to investigate is how to effectively gain *knowledge* from a code base.
This is one of the most important skills for a programmer as reading a lot of code gives us a lot of context and knowledge for how to productive write our own programs.
In the world of pervasive open-source software, where "the specification is the code", it is also quite necessary when stack overflow falls short.

The high-level act of *reading* code is often viewed as a somewhat passive activity.
Parsing through syntax, reading variables, and watching data flow between them guided by conditional gates.
These are all things can make us *feel* like we're being productive when going through code.
That is a trap, and we must be honest about how much knowledge we're acquiring while going through code.
To avoid this, we'll discuss means of productively read code guided by the following:

- Have a set of overall **goals** -- knowledge you want to extract from the code.
- Be **active** in exploring the code -- ensure it is a conversation between you and the code and that your mind is engaged.
- Integrate intentional **feedback loops** in the exploration of the code.
- Be cognizant of **human short term memory limitations** when diving into code -- we can only hold $7 \pm 2$ specific concepts in our meat brains.

Lets go through some options for algorithms for reading code.

Depth-First Search:
```python
def dfs(fn):
	for line in inspect_function():
		ponder(line)
		if is_function_call(line):
			dfs(fn_name(line))
```

This does *not work*.

1. When you are $N$ functions deep in the call graph, will you remember where you started?
	Do you remember the questions you had?
	This approach doesn't consider the limitations of human short term memory.

Breadth-first search fails for similar reasons.

1. Without being able to push into the implementation of key functions to understand how they impact the current code, it can be impossible to gain understanding.
	This might be a reasonable algorithm only in a hypothetical (impossible) code base that has perfect naming, thus descending into functions isn't necessary.
2. Without the ability to go *back* to a function after seeing some of the functions it calls means that you often can't remember the impacts that functions make on the surrounding code.

![Left: a graphical depiction of our relationship to code. All of the orange is *unknown*, and the red is *understanding*. At the start, we only understand the surface-level elements of the code-base, for example the design. Right: a DFS generates what is essentially a random walk through the code-base. The "spaghetti-like" representation of that walk emphasizes that this will not provide a cohesive understanding of targeted questions.](resources/dfs.pdf)

**Scientific method for reading code.**
It may sound strange to use an approach similar to the scientific to investigate code, but it is a general technique that we use to gain knowledge from initially not understood phenomena; which is identical to an intractable code base that you don't yet understand.
The idea behind the approach:

1. Before you start looking at code, come up with a set of *goals*.
	Once you have satisfactory answers to these goals, you'll leave the code base, with your accomplishments in brain.
2. Additionally, use the documentation and what you know of the design of the software to enable you to make some tentative hypothesis about how the code works, about code invariants, or about the naming.
3. Our current understanding of the system is codified in a set of *hypothesis* about system.
	These hypothesis might be as mundane as what a variable is used for and what service a function provides, but also should approach high-level proclamations about the public API's abstractions.
	It is essential that we actively create hypothesis, even if sometimes they feel like *guesses* about what is happening.
	For example, we might make a weak hypothesis (i.e. a guess) for what a function does as we don't believe it is that important, and we want to continue moving on in the current function.
4. As we generate hypothesis, we will often eventually have to *confirm* or *refute* them.
	If the hypothesis is instrumental in building an understanding toward achieving a knowledge goal, then it is necessary to confirm or refute it.

Importantly, these hypothesis depart from details about specific lines of code, and thus capture *knowledge* about what the system is doing.
Viewed this way, as we interpret the code, we're refining our understanding through an operation $f$, which simply takes our current understanding of the code, the code for continued inspection, and synthesizes a set of hypothesis $H$ while working toward a set of goals $G$:

$H = f(G, \text{code}, \text{understanding})$

What is our understanding?
An approximation of this is that it is the set of hypothesis that we have currently!
*Before* we dive into the codebase, our hypothesis are entirely based on what we know before we started reading: documentation.

$H_1 = f(G, \text{code}, H_0^{\text{doc}})\\$
$H_{n+1} = f(G, \text{code}, H_n)$

Note that often $H_n \subset H_{n+1}$ as we are able to make more and stronger hypothesis over time.
However, as some hypothesis are *disproven* as we go through more code, it is not always the case.
Thus any change $H$ represents acquired knowledge.

This model, this fails to capture a very important relationship that isn't necessarily obvious until you *feel* it in action a few times: as you start to develop high-confidence hypothesis about the code, it becomes easier to interpret the code, which makes it easier to develop the next hypothesis!
This is the *feedback* that is necessary to *productively* go through code.
We have to now understand that our brain is in the loop, and it is *empowered* by stronger hypothesis: a [virtuous cycle](https://en.wikipedia.org/wiki/Virtuous_circle_and_vicious_circle).

$H_{n+1} = f(G, \text{brain}(\text{code}, H_n))$

Here the $\text{brain}(\ldots)$ function simply becomes more capable of digesting the code more, stronger, and more abstract hypothesis.
You might think that the practical "runtime" ($R$) of your brain is inversely, proportional to the number of strong hypothesis you have (roughly $R(brain, H) = \frac{1}{|H|}$).
This makes it proportionately faster to consume the next code, which leads to faster hypothesis formation, which leads to faster code consumption, ...
A virtuous cycle at work, creating super-linear increases in your ability to understand a codebase.

Another important note is the relationship between the hypothesis you start with, and those that you end with.
In general, $H_n$ will have higher-level hypothesis than $H_m | m < n$.
That is, the higher-level hypothesis depart from the specifics of the code, and starts to approach more abstract states closer to the level of your goals.
As such, hypothesis are *abstractions for reading code*.
Given this, we essentially evaluate $f$ until $G \subseteq H$, giving us our base-case.

$f(G, H, \text{code}) = \text{if}\,\, G \subseteq H \,\,\text{then}\,\, H \,\,\text{else}\,\, f(G, \text{brain}(\text{code}, H), \text{code})$

How do we confirm or refute hypothesis as we digest more code?
I have little guidance here, but that it is important to become somewhat comfortable with uncertainty.
Though hypothesis can be fully and unambiguously confirmed, it is more common that you don't want to put in the effort to do so.
As such, it is more productive to view yourself as simply *gaining confidence* in hypothesis.
You want to be able make progress through the code base, and coming up with more confidence in hypothesis enables you to better digest additional code.
Practically, this means that you often watch how a function is used and observe its name, make guesses for what it does, and as you see it called again and again, you assess if it is used consistently to your hypothesis for what it does.
Sometimes you'll jump into a function to quickly scan for confirmations, but don't digest the code with enough confidence to full confirm it.
If you find out that a hypothesis is wrong, you'll have to *backtrack* in the code and reassess your hypothesis that are impacted by the new information.
I'll assert, without evidence aside from personal experience, that it is must more efficient to make a lot of hypothesis (that often amount to simple guesses), and have to backtrack through the code, than to strive to perfectly understand each line.
It forces you to think about the code at a more abstract level that better matches how our brains understand code.

The equation model above does not discriminate between hypothesis in which we have low versus high confidence.
Thus, we make $f$ return a tuple of $C$, the set of confirmed or high confidence hypothesis, and the rest, $H$.

$f(G, C, H, \text{code}) = \text{if}\,\, G \subseteq C \,\,\text{then}\,\, (C, H) \,\,\text{else}\,\, \text{let}\,\, (c, h) = \text{brain}(\text{code}, C, H) \,\,\text{in}\,\, f(G, c, h, \text{code})$

Of course, we're pushing off the interesting work into $\text{brain}(\ldots)$ here.
Read on for some specifics about a method for consuming code, module at a time.

Lets revisit our original goals to see how the scientific method for reading code stacks up:

- *Goals* are fundamental to the model.
- The process of creating, confirming, and refuting hypothesis is engaging the reader in *active* exploration and reasoning.
- The *feedback loop* of increasing knowledge with stronger hypothesis is implicit.
- The focus on creating hypothesis replaces rote memorization with high-level ideas, thus better accommodates *human short term memory limitations*.

![Left: We do not read code for reading's sake. To productively read code, we focus on understanding some set of *goals*. Conceptually, the question is how we read code productively to mentally get to the goal. Center: The arrows represent the hypothesis that we use to create understanding of clusters of code. We are always attempting to create hypothesis in the *direction* of the goal. Right: Using the onion model (next section) we bound our investigation strategically to emphasize understanding the abstractions the code is structured around, modules.](resources/sci_method.pdf)

**The code reading scientific method as mental model refinement.**
A mental model is a high-level abstraction about the properties, behavior, and invariants on a body of code.
In some way, it is *predictive*, and we use it to decide when to deploy systems, and in what situations.
As such, it is reasonable to talk about refining your mental model as you go through code.
The mapping is simple: your mental model is encoded as the $(C, H)$ pair.

**The onion approach to understanding code.**
How does this translate into an actionable plan for how to read code?
First and foremost, it gives you intention when reading code.
You're goal is not to scan syntax, and rather to generate knowledge through hypothesis.

But when you sit in front of a screen-full of code, how do you proceed?
The onion model attempts to help you guide your search through code.
The core idea is to try and understand the code's abstraction levels (or modules) at a time, only with excursions into surrounding modules when necessary.
This enables the creation of strong hypothesis in cohesive sets of code, thus focusing the mind on a restricted set of concerns.
Well-designed code is clustered into models or abstractions, and any method to investigate that code should optimize to understanding those abstractions, not simply a chain of functions that cross abstraction boundaries.

As we read code, we often must ask "what does this function do", or (if there are global variables) "what does this global variable represent"?
We must decide if we should jump down the rabbit hole of that function, or where the global variable is modified.
If the answer is uniformly "yes", we have DFS.
So the main question is when we bound that search.
The onion model, then, use a "peel-first search" (PFS) with a focus aggressively *bounding* our walk between abstraction layers.
This decision is impacted (as a first-order approximation) by asking the following questions.

1. Can we form a hypothesis that we don't care about what the function does (e.g. if it is only executed in the error-path), thus can avoid diving in,
2. Do we believe that the function/variable are part of the *current* abstraction layer (onion peel) or module? If so:

	- if we have a firm hypothesis about what it does, keep moving on in the current function, and otherwise
	- dive in with a set of hypothesis from the calling context about what it does to seed your investigation.

3. Otherwise,

	- if we make a (decent, if low confidence) hypothesis about what the function/variable does from the name or surrounding context, then file away that hypothesis and continue reading through the current function keeping an eye out for inconsistencies that might arise between the code you're reading and the hypothesis, and otherwise
	- dive in to the function to better understand this abstraction layer.

The onion model focuses on studying the system peel-by-peel (module at a time).
It makes a strong statement: prioritize not diving into functions that you deems not being part of this abstraction layer unless you have to understand it to make sense of the current code.
It *does* rely on our set of hypothesis to do some heavy lifting: we have to assess if a function is in this module, or in the next.
The best hint for this is usually vested in the naming conventions (for C, that typically uses module-specific prefixes as in `module_*`, and class-based organizations for C++/Rust).
This emphasizes the strong importance of naming in making your code tractable to readers.

*Within* an onion layer or module, how do you guide your attention to understand the salient abstractions?
In systems code (not algorithms), you should prioritize understanding the following (highest priority, to lower).

1. *Module interfaces.*
	The interfaces to a module (to be used by code outside of the module) encode the *purpose* of the code.
	You hope that these are well documented, but quite often they are not.
2. *Data-structures.*
	These form the skeleton of most systems code, and provide a rough map.
	The relationships between data-structures is often quite indicative of overall code structure and functionality.
	The *hypothesis* at this level should be able data-structure relationships, what is tracked by what, and how the data-structures are related to the high-level operations of the system.
3. *Data-structure operations.*
	You can go through the operations on those data-structures (think: class methods) with the main intention of simply confirming or refuting your understanding of the data-structures, and their relationships.
	These are effectively the muscles of the system.
4. *Higher-level operations.*
	The core of the complicated logic ties together the data-structures and entails most of the semantics of the system.
	This is the equivalent of the nervous system to guide the rest of the system in the desired direction.

There is one pending question that is likely on the mind of everyone staring down a new code-base: "where do I start"?
Unfortunately, there isn't a great answer to this.
In my experience, understanding the *design* of the system helps, especially if design documents has mappings of that design back to files/functions.
Understanding the naming schemes in systems often helps as well.
For example, knowing that all Linux system calls are prefixed by `sys_*` enables you to know some version of "where to start".

> An aside for *Composite*: as the system has a strong component-based focus (synonymous with "module" above), the modules are explicitly delimited in the codebase.
> The interfaces to components are in `src/components/interfaces/*`, and the library interfaces are in `src/components/lib/foo/foo.h` (for a library "foo").
> The implementations are either part of the libraries, or are in `src/components/implementation/bar/*` for components that implement the `bar` interface.


**The guaranteed method for effectively reading code.**
Reading code, forming hypothesis, and extracting high-level designs from code, are all skills that can be trained.
As you go through a code-base, you have stronger hypothesis (a more refined mental model), which enable you to more quickly extract knowledge from the code.
However, as you read code you also learn how to more effectively extract reasonable hypothesis and when to bound your search based on experience, in ways that transcend any specific code-base or programming language.
Thus, the guaranteed method to effectively read code is to *read a lot of code*.

### Writing Readable Code

How that we have a reasonable way to think about understanding code, the next important question is how to *write readable code*.
I want to be clear: the goal of writing code is *not* to write code cleverly, tersely, concisely, nor elegantly.
Those things might be nice, but the *goal* must always be to write *readable* code.
Many more people will read code (including the original author), but very few will write it.
The success of any future code modifications, is completely contingent on being able to understand the existing code, thus design and implement additions.
This section will dive into a means of thinking about how to write readable code.
If the knee-jerk intuition is that this means "write more comments", then a deeper examination is required.
The [Composite Style Guide](https://github.com/gwsystems/composite/blob/loader/doc/style_guide/composite_coding_style.pdf) (CSG) goes into this topic in gory detail, so please see that more for information.

Most strategies for writing readable code focus on making strong hypothesis formation easy for readers.
What mechanisms do we have for this?

**Naming.**
Choosing the names of functions, types, and variables is one of the most important aspects of writing readable code.
It is rarely excusable to not think deeply about these issues (especially the types and functions that are part of a module's API).
A few perspectives on this:

- *Self-documenting code.*
	Writing code that is relatively easy to digest is possible with intention.
	If aspects of a function are a little "in the weeds", it can be pulled into a separate function, with an appropriately chosen name.
	This has two effects: 1. it separates abstraction levels across two functions, easing an onion-model investigation, and 2. it associates a specific (and, of course, well-chosen) name with the functionality.
	Both of these enable better hypothesis forming.
	Similarly, choosing useful variable and type names enables much stronger hypothesis formation.
- *Avoid committing mental registers.*
	We store $7 \pm 2$ items in our short-term memory -- our mental "registers".
	There is not very much information.
	Each variable commits a register, *unless* it is well named, thus itself serves as a reminder of what it holds.

**Code structure.**
The structure of the code itself can have a significant impact on our ability to read it.

- *Function size.*
	See the bullet above about self-documenting code.
	There is no rule, "keep your functions small".
	Instead, in each function try and capture a level of abstraction.
	If the complexity rises to a point where it makes sense to pull logic out of the function, choose a name well for the resulting function.
- *Avoiding redundancy.*
	If you manually write the same code many times (of course with small permutations) throughout your code, a reader must read through and digest that code multiple times.
	However, there is inherently an abstract operation being performed, and choosing a function name for that abstraction, so that a reader needs only digest that function once, regardless how many times it is used.
- *Indentation and short-term memory.*
	In languages like C and Java that don't use closures extensively have an interesting property: the level of indentation is often a proxy not just for complexity, but also for how many mental registers are required.
	While in the body of each condition and loop, you must remember the conditions that got you there.
	Each level of nesting simply means an additional set of conditions you must store in short-term memory.

	How do you avoid tons of nesting?

	1. Carefully construct your conditionals to separate error handling (and paths that will `return`, from the rest).
		This enables a resumption after that condition without increasing indentation (i.e. you don't need the `else` if the `if` returns).
	2. Simply be careful about how you nest.
		Craft your code carefully to simplify the logic, especially in nested cases.
	3. Move functionality to a separate function!
		It is not uncommon to see long functions in many professional code-bases.
		That can be OK if all the logic is at the same abstraction level.
		However, often see code pulled into a separate function if the nesting level gets too deep.

	To encourage good behavior, our coding style discourages nesting beyond 4 levels, and requires 8-space nesting to visually emphasize the cognitive cost.

**Error handling.**
An important way to enable a reader to make hypothesis about what code they *don't need to focus on* is to make your error checking code easily discernible.
You can often pay less attention to the error cases when trying to understand a general design.

- *Isolating error cases.*
	Error cases should (where possible) follow these guidelines:

	1. They should appear as early in the logic as possible.
		There is no reason to do any useless operations if you're just going to return an error.
		This informs hypothesis formation by enabling a reader to more strongly assume that error-like code at the start of functions can likely be skimmed through quickly.
	2. Error conditions almost always return, thus you can avoid indentation of non-error cases (by avoiding the `else`).
		This enables a reader to effectively not spend a mental register on error-checking, thus enables the focus on the key logic.
	3. Organizing clean-up code is necessary.
		As errors can occur mid-way through a function's execution, cleanup is required (releasing locks, deallocating freed memory, etc...).
		If you have multiple such error paths, you must remember to execute the correct cleanup each time.
		It is very easy to get this wrong.
		As such, it is important to write readable code that uses language features such as go's `defer`, Rust's `?`, C++'s RAII, and C's local exception handling via `goto` (see the CSG).

- *Pervasive error checking.*
	It is easy to undervalue error checking for all possible conditions.
	Missing error checking is *very* confusing for readers.
	Is there some condition hidden in the logic of the code that means that the error doesn't need to be checked?
	If error checking is lacking to validate input arguments, does that mean that the function *can* be used for all inputs that aren't checked?
	This can make it very difficult to digest the logic.

**Comments.**
Comments are important, but they are *not* sufficient.
In some cases they can be counterproductive.

- An unfortunate truth: comments can go out of sync with code.
	As comments are not compile-checked, nor unit tested, they are in a precarious position.
- Comments should not be used for "business as usual" aspects of the code that are coded in accordance with the design.
	The design should be documented, but the use of the design in the prescribed way is normal.
	Comments should absolutely be used in all places in the code in which something clever, something obtuse, or something out of the ordinary happens.
- The interfaces between modules need to be well commented.
	No exception.
	Readers following the onion model of reading code, need to be enabled to *not* dive into the next level of abstraction, and understanding what functions do (at a high-level) is a key enabler of that.

**Understanding the culture** of the developer community that works on the code base is important and factors into how you write readable code.
This is key in a few different areas, three of which I'll emphasize:

- *Comments.*
	Many students feel very self-assured about where comments are required, and how detailed they should be.
	One need only look at "well documented" code generated by new programmers to understand how wrong this perspective is.
	Beginner programmers, when asked to comment their code, will have comments like `x = x + 1; // add one to x`, which cause an abundance of sorry to more experienced programmers.
	Students in an undergraduate OS class do not so egregiously comment trivial aspects of the system, but they *will* over-comment any interactions with the surrounding system.
	A developer who is used to that code-base would not comment the same code as those interactions are normal, and consistent with the system design.

	What's the point?
	What should be commented, and when should comments be omitted are *subjective* questions relative to the *culture* of the code-base.
	What is the expected level of comfort with the language for a developer?
	What is the expected understanding of the overall design?
	Which APIs are part of the "programmer lexicon" for the project?

	What is clear is that any usage of the programming infrastructure that is *clever*, *departs from the prescribed, normal usage*, or that has *unexpected results* should *always* be commented.
	Module interfaces are key to a system's design, thus it is almost always the case that they should be well implemented to promote the desired culture of understanding key APIs.
- *Naming.*
	Naming iterators after `i`, `j`, and `k` in C-like languages has a long history.
	Using any other variables might raise eyebrows, and cause confusion.
	In many `c` code bases, APIs are prefixed by the equivalent of the "class" to which they belong, and the first argument is often the data-structure corresponding to the object, for example `int brain_process(struct brain *b, char *code, ...)`.
	When you see this naming pattern, you don't need to process the details, they are obvious as they are part of the culture.

	Sometimes the culture changes in response to technology.
	A good example of changing cultures is the fading relevance of [hungarian notation](https://en.wikipedia.org/wiki/Hungarian_notation).
	When we didn't have editors that could trivially provide type information for variables and functions, it made some sense to encode those types into the variable/function names^[Note: I'm not arguing for this notation. But I think that there was a reasonable *justification* for a code-base to use it in the past. I did not use that justification in my own projects.].
	But now we have [Language Server Protocol](https://langserver.org/) (LSP) implementations for most languages, and LSP integrations into most editors that bring type annotations a simple cursor focus away.
	Now hungarian notation only has the effect of adding distracting names that require mental filtering.
- *Interfaces.*
	There are many factors that must be considered when implementing APIs.
	These include not just the types, and the function signatures.
	How is object naming performed -- are objects referenced directly by pointer, or hidden behind an opaque, indirect reference (e.g. file descriptors)?
	What are conventions around object ownership and liveness -- is an object passed in to a function referenceable after the function returns?
	How does the API treat object initialization and allocation -- they can be decoupled to enable stack or global allocation?
	All of these are part of the *personality* of an API that you should understanding to be able to make more effective hypothesis.

**Conventions for writing readable C.**
It is very easy to write hard to read code in many languages, and C is absolutely no exception.
"Seasoned" C developer often simply write boring code.
There is a specific way to write C code that avoids the hard to think about parts of C.
Pointers can be confusing; don't write code that depends on complicated pointer patterns.
C lets you do anything you want memory; don't write that style of code, and when you need to, make it minimal, and well documented.
There are many ways to write any given code; stick to one way, and be consistent.
Many other examples exist.
Please see the [CSG](https://github.com/gwsystems/composite/blob/loader/doc/style_guide/composite_coding_style.pdf) for a detailed treatment of how to write C code that is boring and unlikely to have problems.
