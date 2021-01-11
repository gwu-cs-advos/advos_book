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

- Code complexity:
    [cyclomatic complexity](https://en.wikipedia.org/wiki/Cyclomatic_complexity), [understanding](https://yurichev.com/blog/cyclomatic/) it for Linux, and [arguments against](https://www.cqse.eu/en/news/blog/mccabe-cyclomatic-complexity/) it and [alternatives](https://carlalexander.ca/what-is-software-complexity/).
-  Constraining complexity:

    - [jpl](https://yurichev.com/mirrors/C/JPL_Coding_Standard_C.pdf)
	- [misra](https://en.wikipedia.org/wiki/MISRA_C) [c](http://caxapa.ru/thumbs/468328/misra-c-2004.pdf)
	- [composite style philosophy](https://github.com/gwsystems/composite/blob/ppos/doc/style_guide/composite_coding_style.pdf)

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

## Macro-level, System Design

- Design complexity:
	[Brooks](http://worrydream.com/refs/Brooks-NoSilverBullet.pdf) observed that much software complexity is essential, but some is also accidental

	**Essential complexity:**

	> The essence of a software entity is a construct of interlocking concepts: data sets, relationships among data items, algorithms, and invocations of functions. This essence is abstract, in that the conceptual construct is the same under many different representations. It is nonetheless highly precise and richly detailed.
	>
	> I believe the hard part of building software to be the specification, design, and testing of this conceptual construct, not the labor of representing it and testing the fidelity of the representation
	>
	> If this is true, building software will always be hard. There is inherently no silver bullet
	>
	> - Fred Brooks, The Mythical Man Month

	Essential complexity is a property of the *problem you want to solve*, and is independent of the developer, team, or technology.

	**Accidental complexity:**
	This can include system acrobatics motivated by working on the given system architecture and abstraction (e.g. using the wrong tools), complexity added due to [technical debt](https://en.wikipedia.org/wiki/Technical_debt), etc...
	Writing in high level languages raises the level of abstraction (e.g. garbage collection), thus removes the programmer's consideration for "unnecessary" details...until performance or predictability become part of the specification.
	"Hacking in" features, or maintaining backwards compatibility in spite of new architectures also significantly add to the accidental complexity of the system.

	When the system abstractions work *against* your goals, there is a *semantic gap* between what the system architecture makes easy or prioritizes, and what you need from the system.

    - [accidental vs. essential](https://en.wikipedia.org/wiki/No_Silver_Bullet) complexity
	- there are [many](https://pressupinc.com/blog/2014/05/root-causes-software-complexity/) [takes](https://medium.com/background-thread/accidental-and-essential-complexity-programming-word-of-the-day-b4db4d2600d4)
	- an [argument](https://danluu.com/essential-complexity/) that technology *does* decrease accidental complexity, e.g. displaying data with `ggplot` includes downsampling, rendering, image output, etc...

### Complexity Management

How do we design systems to manage complexity?
Code has a tenancy to become pretty tightly coupled.
Adding a new function codifies a number of assumptions for the data-structures and functions accessed.
When this happens on the large scale, logic becomes quite intertwined, thus any change impacts more code.
This makes any future change much more difficult.
Yet we have massive systems such as kernels, browsers, and video games.
How is this?

- *Abstraction* -
	We don't write high-level code that uses low-level mechanisms.
	Your application isn't programming interrupt handlers.
	Web applications don't program layout engines and bitmap creation, instead focusing on DOM manipulation.
	Abstraction is the foundation of computer science, and is particularly important in system design.
	Key components of abstraction include:

	1. providing abstract interfaces for resource/object access, and
	2. mechanisms to *contain the entropy* expressed by more permissive interfaces that we use to provide a higher-level, more constrained functionality.

	Each of these is inter-related, and essential.
	We wish to take the chaos of the system on which the abstraction is implemented, and organize it into a representation that is *less capable*, but *more useful* (for specific goals).

	*Example:*
	A car with steering wheels for each wheel.
	This is very powerful, but very easy to use incorrectly.
	Abstraction over this system can be used to tie the front wheel's steering together, and locking the back wheels.
	We provide an interface of a *single steering wheel* to actuate the car.
	We contain the complexity of the underlying system, while providing an abstraction that is more useful in many situations (unskilled drivers).
- *Modularity* -
	Abstraction is important, but *structuring the abstractions* with respect to each other is also necessary.
	Modularity requires that abstractions are implemented with unfettered access to the functionality in their module, but that inter-module dependencies are restricted to module interfaces.
	This requires the encapsulation and information hiding of the details for how the abstraction is provided.
	Inter-module *interfaces* provide important abstractions that tie modules together.
	Modules have the effect of constraining complexity by disallowing unrestrained code access outside of modules, and focuses a lot of design on inter-module abstractions.
	Much of the power of modularity comes from the ability to *replace different modules that share the same interface*, thus alter system behavior without writing additional code.
	The [Liskov substitution principal](https://en.wikipedia.org/wiki/Liskov_substitution_principle) provides an intuition as to when this is possible, but we've internalized this notion as *polymorphism*.
- *Layering and Hierarchy* -
	In systems that are large enough that the complexity of inter-module interactions becomes intractable, strategies for organizing modules are required.
	If there are $N$ modules, there are a potential $N^2$ interactions between them.
	For a large $N$, this can be challenging.
	A change in one module can impact all $N-1$ other modules, which makes changes quite challenging.

	Empirical studies have shown that complexity in complex subsystems is dependent on the *coupling* between components (inter-module connections).
	For example, some have found a correlation between some [coupling](https://www.hbs.edu/faculty/Publication%20Files/18-031_78569c7d-fc0c-4e1e-bd79-143bc507112a.pdf) [metrics](https://link.springer.com/chapter/10.1007/978-3-319-62105-0_4) in Chrome and security issues.

	The simplest way to constrain this complexity is to use a *layered* architecture in which modules can only leverage other modules at their level, and at one lower level.
	They can provide services to modules at the next higher level.
	This can be quite limiting, but constraints interactions to modules at the same level (on average $N/L$ modules for $L$ levels), and those at the next lower level.
	This creates $N/L + I$ interactions where $I << N/L$ is the number of modules at the next lower level that have public interfaces to the higher level.

	Compilers are typically implemented as a sequence of *passes* that happen on the core code representation data-structures.
	They tend to go from high-level representations of the code, down to more and more detailed.
	A similar example is the *Composite* linker that parses through many components, performing a set of [transitions](https://github.com/gwsystems/composite/blob/loader/src/composer/src/main.rs#L50-L68) that represent layers of computation, taking the system from general states to move specific, and finally, executable states.
	Note that pipelined-composition of modules (as shown here, and similar to pipes in UNIX) are similar to layered system construction as they restrict the interactions between modules, but they focus mainly on the processing of data between modules.

	The [*total ordering*](https://en.wikipedia.org/wiki/Total_order) on modules can be quite limiting.
	For example, they often require that modules be created at a level that simply pass a request from a higher-level downward to a lower level.
	As such, some systems instead use a hierarchical organization for modules.
	Systems form a *tree* of modules in which, like a layered system, they only communicate with modules in the same "branch" of the hierarchy, and with the interface of the "parent" in the hierarchy.
	This is a *very* common structuring technique and is used in monolithic systems, VM-based systems, and even distributed systems such as DNS.

	Hierarchical module organization is still limiting.
	You cannot communicate easily between branches, and you still can only communicate one level below you.
	So we can think about modules as generally interacting with each other in a structured directed acyclic graph ([DAG](https://en.wikipedia.org/wiki/Directed_acyclic_graph)).
	Question: why a DAG instead of a general graph?
	We'll discuss this case quite a bit, so I won't go into more details here.

**Importance of Interfaces.**
One important implication of a module- and interface-based structuring of system is that the interfaces are very important.

1. The interfaces encode the core of the abstractions in the system -- the usability of your abstractions will be tangibly experienced through your interface.
2. Specifications are stated in terms of the interface -- they encode the semantics (behavior) of the system at a high-level.
3. Multiple modules can implement the same interface, and many modules will depend on the same interface, thus any changes to the interface impact a potentially large amount of functionality -- it is hard to update interfaces.
4. Interfaces encode many of the assumptions that implementations are allowed to make, and how complex the error checking will need to be -- thus the simplicity of implementations and of the error handling in clients is heavily impacted by the interface.
5. Unit tests should be written to the *interfaces* -- the automated means of helping you maintain some consistency across changes (made by many people in a team) are coupled with interfaces.

When designing a system, the highest priority is to get the interfaces right^[In our own research, we've made the error again and again to work with hacked-together interfaces under the pressure of deadlines. In the medium term, this has wasted a lot of developer effort due to confusions and accidental complexity due to the weaknesses of the interfaces.].
If you have strong interfaces, implementations can be weak, and replaced.
If you're interfaces are not good, the system will collapse under complexity and confusion.

A number of other factors can combat complexity.
They all fall into a simple motto:

**Keep it simple.**
Instances of this include the following:

- *Avoid over-generalization.*
	Attempting to solve all problems runs the risk of generally solving none.
	This can apply to the implementation: you don't always need to make it generic across all future changes.
	A simple solution, to a well-stated problem, is often sufficient.

	Over-generalization can also apply an interface.
	Locks that enable recursive (nested) locking are more general than those that do not, but there are [many](http://www.fieryrobot.com/blog/2008/10/14/recursive-locks-will-kill-you/) [arguments](http://www.zaval.org/resources/library/butenhof1.html) [against](https://blog.stephencleary.com/2013/04/recursive-re-entrant-locks.html) [them](https://inessential.com/2013/09/24/recursive_locks), simplicity included.
- *Avoid optimization in general*.
	Optimization usually involves rewriting code-paths in ways that make them less terse, and less direct.
	Optimization almost always makes code more difficult to read, more difficult to debug, and more challenging to modify.
	Systems require optimization, but this can usually be done in targeted parts of the system.

**Include unit tests**.
This may be a strange item to discuss at this point, but it is quite important.
Unit tests are most useful (and near essential) on the interfaces between levels, and for public module interfaces.
When you *change* a system, it is immensely valuable to have a *baseline* for correct functionality.
A unit testing suite enables more aggressive design, as changing a module can leverage automatic checking of the intended semantics (assuming you have a decent test suite).
It also constrains complexity as module interfaces are better documented with use-cases, and auto-checked for some approximation of correctness.

### System Interface Design Considerations

When designing system interfaces, you must often consider more factors than if you're simply implementing a library, or a module in an application.
These are all motivated by system's responsibility to manage system resources and multiplex them.
The general question include how you should *name* system resources, how you can *access those resources*, and how you interact with *concurrent* resources.

- *Naming.*
	When modules are separated, they require structured means of communicating with each other.
	Where information hiding is taken very seriously, and where isolation is required, passing pointers to data-structures is not appropriate.
	Thus the question of *naming* becomes very important: how do we name the resources/objects that are passed into the interface on the client side, and manipulated on the server?

    - Text-based hierarchical namespaces (UNIX, URL/REST, D-Bus, plan9)
	- Per-principal hierarchical namespaces (containers, plan9)
	- Isolation via namespace hiding (containers, plan9, composite)
	- General routing facilities via programmable namespaces (D-Bus, plan9, systemd)
	- Serializable means of addressing in namespaces, and interacting (9P, HTTP)
	- names often include a service *and* an object/API selector

	When a client *references* an object by name, it is associated in the namespace with the object by *binding* the name to the object.
	This might be done using OS-provided abstractions such as file descriptor tables and the Virtual File System (VFS) layer in UNIX to dynamically *bind* a file descriptor to a *file* named using the hierarchical namespace.
	This form of binding often uses "handles", or opaque references (integers) that somehow *map* to the objects (e.g. through the file descriptor table).
	Alternatively, binding might be done *statically* during *linking* to tie function calls, and data accesses to their implementations.
	This form of binding often uses "references".
	Techniques such as lazy binding are somewhere in-between in that they use [Procedure Linkage Tables](https://www.technovelty.org/linux/plt-and-got-the-key-to-code-sharing-and-dynamic-libraries.html) (PLT) to dynamically bind client and library code (for dynamically loadable libraries,`.dll`s or `.so`s).
	A level of indirection in binding is powerful as it enables the replaceability of the module's implementation.

	Naming is the *foundation of isolation*.
	If a client cannot *name a resource*, then it *cannot access or modify that resource*.
	Lets go over a couple of examples:

	- Containers are a means in an existing monolithic OS to provide a custom user-level personality to a set of principals.
		As such, the file system, process id, network (and on and on) namespaces can be made *disjoint* across different principals.
		This effectively enabling those principals are isolated from each other (assuming the kernel itself is not compromised), and enables different OS and programming environments for each principal.
	- Capability-based systems associate a single namespace with each protection domain such the only accessible resources are those defined by a flat function $f(\text{capid}) \to \text{res}$ where $\text{capid} \in \mathbb{N}_0$ (the system call on such systems is essentially a wrapper around $f$).
		This simple namespace limits all resources available to a protection domain (to $R = \cup_{\text{id} \in \mathbb{N}_0} f(\text{id})$).
		Note that this looks a lot like a *file descriptor table*, but is used for *all* naming of resources.
		Two protection domains $p_i$ and $p_j$ (with corresponding capability lookups using $f_0, R_0$ and $f_1, R_1$)  are completely isolated from each other (modulo CPU time) if and only if $R_0 \cap R_1 = \emptyset$.
		One of the benefit of capability-based systems is the simplicity in understanding resource access availability in the system.
- Uniform means of *interacting with named objects*.

	- VFS, 9P, [CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete)

		| Op     | VFS    | [9P](http://man.cat-v.org/plan_9/5/intro) | SQL    | HTTP           |
		| ---    | ---    | ---                                       | ---    | ---            |
		| create | creat  | create                                    | INSERT | PUT            |
		| delete | unlink | remove                                    | DELETE | DELETE         |
		| read   | read   | read                                      | SELECT | GET            |
		| update | write  | write                                     | UPDATE | PUT[^RESTPOST] |

	    [^RESTPOST]: Note that `POST` is closer to a method invocation to perform some operation with semantics that go beyond "update please".

	    Note that UNIX and 9P focus on optimizing for iterative reads, and repetitive updates, but also uses `open` and `close` (`clunk` in 9P) to maintain a record of continuing read an update operations.
		This simplifies a number of things, including race conditions between accessing data in an object and deleting that object (the equivalent problem in HTTP is *paging* while deleting), and doing expensive access control checks once on `open` to avoid having to do them on read/update operations.

- *Data representation* for each object.

	- Untyped streams of bytes: data passed as text streams (pipes, files, etc...)
	- Typed functions: language ecosystems, [dbus](https://pythonhosted.org/txdbus/dbus_overview.html), [protobuf](https://developers.google.com/protocol-buffers)s, [capnproto](https://capnproto.org/)

- *Action and data aggregation for optimization.*
	Action aggregation across communication (move computation to data) moves some set of actions to where the data is to avoid communication in between each action.
	Adds complexity into the system by enabling aggregate operations to execute in the system service.

	- Database [stored procedures](https://en.wikipedia.org/wiki/Stored_procedure).
	- OpenGL [display lists](https://en.wikipedia.org/wiki/Display_list) (or command buffers/command lists).
	- Linux dynamically uploadable procedures via [ebpf](https://ebpf.io/).
	- The common use of [lua](http://www.lua.org/) as a configuration and policy engine embedded in C++ games.
	- [Graphql](https://graphql.org/)[[2]](https://en.wikipedia.org/wiki/GraphQL) vs [REST](https://en.wikipedia.org/wiki/Representational_state_transfer) w/ [CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete) operations.
	- HTTP 1.0 vs. [pipelining in HTTP/2](https://en.wikipedia.org/wiki/HTTP_pipelining).

	Data aggregation across communication is most often see as the batching of data.
	When we print (e.g. with `printf`), we don't usually call `putc` on each character.
	Instead, batch up data to be printed out, and only call `write` to output all of the data at once when a newline, `\n` is printed.
	This generally *trades higher throughput for increased latency* by delaying data movement, thus amortizing data movement costs.

	- async communication via pipes with finite buffering to batch sends/receives
	- buffering/*batching* of service requests (printing!) -- downsides WRT throughput vs. latency & `fflush(...)`

- *Concurrency.*
	We must consider asynchrony and multiplexing in our interactions with named resources/objects: how can interactions hide the latency of communication (e.g. network/disk latency), and compensate for non-deterministic concurrency

	The main challenges here are that if we have a blocking API, we're forcing clients to use multiple threads, and to couple their thread's execute to our blocking and wakeup logic.
	If we have solely non-blocking APIs, then we're forcing the client into polling all resources (which wastes CPU).
	The solution is to provide *event multiplexing* functionality.
	These are APIs that enable a thread to block waiting for *any one of the resources that might have an event*, and to return when *any of them have an event*.
	Conceptually, we pass in a set of resource handles, and the function only returns when any of them have events, which are returned as a set of events.
	As such, the client can iterate through the set, and process each of the resources (e.g. by `read`ing data from them that we now know is available).
	POSIX defines `poll` and `select` to do event multiplexing, and Linux/FreeBSD extend that with `epoll`/`kqueues`.
	A key challenge when you have many resources provided by many different modules is how they can coordinate to multiplex their events.

	- blocking considered generally dangerous -- ties communicating principals together in execution (how?)
	- asynchronous functions complicate APIs as they require event notification functions such as `select`, `poll`, and `epoll` (idempotentcy & edge vs. level triggering)
	- [go](https://golang.org) is a programming language and environment at encourages using synchronous and asynchronous channels, along with lightweight computations (goroutines) to handle concurrency.
	- [libuv](https://libuv.org/) is a library that attempts to use an *event-based* programming model (versus a *thread-based* programming model) based on *callbacks*.
		It is the framework underlying `node.js`, and mimics the closure-based programming model common in javascript.
- *Parallelization.*
	For the most part, you want to be able to hide any parallelism behind an interface.
	However, it is often the case that the parallelism constraints span interfaces.
	For example, the specification of a function should publicise if the API is concurrency/parallelism-safe, or if locks need to be taken before calling the function.

	- Partition/split - data parallelism
	- Stream - pipelined parallelism
	- Commutativity indicates that an API *can* be implemented in a way that it can scale across increasing numbers of cores, but not necessary that it *is* implemented that way.
	- Details later!

An additional important aspect of system design is an understanding of reality.
There is a common trade-off between *perfection and shipping*.
An important question is what features and aspects of design don't need to be perfect?
Where can the design be weak?
If you don't have time to prevent all accidental complexity, try and engineer where you're OK with it sneaking in.

Taking this consideration further, some harsh reality: "[worse](https://en.wikipedia.org/wiki/Worse_is_better) is [better](https://dreamsongs.com/WorseIsBetter.html)".
The best technology doesn't always win.
The first mover effect matters.

### Interface Design Properties

Different functions in system interfaces have many high-level properties.
It is important to consider many of these factors when reading or implementing an interface.
Some of the most important properties that you should consider when assessing, and designing interfaces include *composability*, *state management*, and *commutativity*.

- *Composability.*
	Are there traps in interactions between *different functions* in the API?
	How about between this API and others?
	There are two key questions related to composability:

	1. If I call *f(...)*, is it possible for it to fail -- despite its specification giving us confidence it should work -- given the context of previous abstractions I've used up till now?
	2. If so, how can the abstraction I'm trying to use, be impacted by the context?
		Is it possible to control the impact of the non-composability, and how much accidental complexity does it add?

	Some examples:

	- [`fork`](https://www.microsoft.com/en-us/research/uploads/prod/2019/04/fork-hotos19.pdf) vs. `posix_spawn` and `CreateProcessA`
	<!-- - The massive complexity Signals w/ re-entrancy & locks vs. blocking & event notification -->
	- Global data and state can cause a function to not compose with itself (`strtok` vs. `strtok_r`)!
	- Abstractions that have global and non-deterministic effects: interrupts vs. polling.
	- Locks (no enforced nesting) + blocking vs. lock-free/transactional memory.
	- Multithreading vs. data-structures fundamentally don't compose!!
		However, sometimes a lack of composability is worth it, but demands additional abstractions.

- *State management.*
	Interface implementations might abstract away an immense amount of state (like a file system), or they might take a lighter touch.

	- *Stateless and immutable APIs* -
		Stateless, or *pure* functions are those in which output depends *only* on the input, and the input is left unchanged.
		Specification and testing of pure functions is often much simpler than for those that have state.
		Specification need only be in terms of the input arguments, and not on the state of the underlying system.
		Testing need only consider permutations on input arguments, and not also on the various permutations on hiddlen system state.

		That said, functions that are stateless are often not very useful in systems.
		If a function can be computed only on the inputs, then why call down to the system to perform the operation -- instead just implement the question in a library?

		We might consider functions that access only *logically immutable structures* as a relaxation on stateless functions.
		Their functionality depends on hidden state beyond the arguments passed in, but if that state is not modified, we can get some testing and specification benefits.
		An example of this is system calls that are made only on subsets of the file-system namespace that can be read, but not written.
	- [*Idempotent APIs*](https://en.wikipedia.org/wiki/Idempotence#Computer_science_examples) -
		Idempotency is more realistic for system APIs.
		Multiple calls to the same API function from a client does not result in a change the return value nor the visible state of the system.
		Intuitively: Can we make the same call twice, and both times it has the same impact?

	    Idempotency is expected in many situations.
		For example, we expect that two executions of `$ ls` will print out the same values.
		Even `echo` is idempotent (with used with the same argument): `$ echo "hi!" > file` results in `file` holding the expected value; as does `$ echo "hi!" > file; echo "hi!" > file`.
		For webpages, we expect that two `GET` requests (to retrieve some resource on a webpage) will return the same value.
		Loads to the same address, and stores to the same address are, on their own idempotent.

		Note that idempotency is not typically seen to apply across different API calls (it does not apply to function compositions).
		`read` is not idemponent as every time we call it we get a non-overlapping section of the file (the "next" part of the file).
		However, we can write a `readfile` function that reads the entire contents of the file, and that function will be idempotent, so long as no other thread does a `read` on the backing file-descriptor (thus causing `readfile` to skip data).
		The [fetch and add (`faa`) instruction](https://en.wikipedia.org/wiki/Fetch-and-add) is also not idempotent when asked to add non-zero values as every time it is called it will return the (updated) value of the counter, then add to it.

		What are the *implications of idempotency* of a function in an API?
		Generally, idempotency is a strong indication that the backing data can be *cached*.
		`read`s on a pipe have very different properties than on a file.
		There is no way to retrieve the same data out of the pipe as once it is read once, it is gone.
		Thus pipe APIs aren't idempotent, thus pipe data cannot generally be cached.
	    We'll revisit idempotency when we discuss event management.

- *Commutative APIs* -
	Functions are commutative if they can be reordered and still result in a semantically identical result.
	We've seen this mathematically in that `a + b = b + a`.
	In a CS setting, examples are closer to

	```c
	fd1 = open("file1", ...);
	fd2 = open("file2", ...);
	```
	versus

	```c
	fd2 = open("file2", ...);
	fd1 = open("file1", ...);
	```
	How can we assess if these commute?
	The only observable effect of opening the file is the return values.
	So the question is, do these two options commute with respect to (WRT) the specification?
	You may be surprised to know that they *do not* on POSIX systems.
	The file descriptor returned from any system call that returns a new file descriptor must be the lowest numerical `fd` that is not currently in use.
	Thus the previous two operations do *not* commute as the functions are *guaranteed* to return different values depending on the order of their execution.
	Changed semantics that simply specify that the `fd` returned can be any unused `fd` make `open` commutative with respect to itself.
	This is somewhat confusing, but what matters the most here is commutativity with respect to the *specification*.

	API commutativity has a large implication on parallel scalability of the system, a correlation we'll discuss later.

- *[Simplicity](https://en.wikipedia.org/wiki/KISS_principle).*
	It is a trope to "keep your code simple", but it is very rarely executed.
	Avoiding unnecessary optimizations and generality helps, but simplicity can be encouraged in design.
	Simple code often comes from asking what assumptions you can engineer into your code.
	If you can statically allocate all data-structures, and avoid `malloc` (and, more importantly, `free`).
	If you can ensure that data-structures or data adhere to constraints (e.g. circular doubly linked lists can avoid conditionals), you can avoid superfluous edge-cases.
	Generally, one need be willing to *rewrite* your code and *redesign* when you find that there is a way to simplify logic.
	This is a higher-order optimization that should span your focus from the minutia of the code, up to module and inter-module design.

- *[Separation of concerns](https://en.wikipedia.org/wiki/Separation_of_concerns) (SoC)* and *[Orthogonality](http://www.catb.org/~esr/writings/taoup/html/ch04s02.html#orthogonality).*
	When attempting to figure out how to break down the interfaces of the system into different "clusters" that are implemented by modules, the SoC and orthogonality are guides.
	Different functions in an interface -- and, indeed, different interfaces -- should each perform a specific function that does not overlap with others.
	This lack of overlapping functionality, and unintended modifications, produces an orthogonal design.

	> Do one thing well.
	> - Doug McIlroy

	This quote is not about simplicity, instead about orthogonality.
	Orthogonality in code can be seen as avoiding code redundancy: a single function is responsible for some functionality.
	This follows from the "don't repeat yourself" rule.

	Imagine a system that replaces the file system code in the kernel with a database (see an attempt in [WinFS](https://en.wikipedia.org/wiki/WinFS), and a simpler hack in [joinFS](https://www2.seas.gwu.edu/~gparmer/publications/esa11.pdf)).
	However, for power users it enables them to directly use SQL for operations like aggregation that couldn't be easily leveraged through the FS interface.
	Now there are *two* interfaces the provide two subtlety different APIs to the same state.
	But the SQL interface can be used to do things that you cannot do in the FS, for example, to modify `..` and `.` to reference other directories.
	What does the FS code do when it finds these inconsistent states?
	Does the FS now need to include code to consider *all possible* configurations of the database?
	Doe the SQL processor have to include code to consider only operations the FS can perform?
	This example demonstrates the danger of non-orthogonality: either the implementations of different modules need to become increasingly coupled -- which makes it impossible to replace any of them, and more difficult to debug them, or they remain separate and have undefined, surprising, or bad interactions in edge-cases.

	The ability to `mmap` a file, and access its contents using load/store instructions *and* to *also* access the file using `read`/`write`/`seek` clearly is not orthogonal.
	However, the benefit of being able to directly access file data using memory operations has enough benefit that `mmap` won out over orthogonality.

	A high-level way to think about SoC and orthogonality is to ask: given the set of *effects* that a function or an interface has on encapsulated state, the extent to which two functions or two interfaces are coupled (thus not following SoC/orthogonality) is commensurate to how much those effects impact state visible to the other.
	We want our interfaces to be orthogonal *and* composable.
	Orthogonal interfaces do separate, useful things, and composing them together leads to a system that is greater than the two parts.

- *[Separation of Mechanism and Policy](https://dl.acm.org/doi/10.1145/1067629.806531).*
	Policy is a decision about *how* to use a provided *mechanism*.
	How does your application use the mechanisms for accessing files in the file system to create its functionality?
	How does the shell

	The X-windows server provides primitive mechanisms around compositing and rasterizing (displaying overlapping windows), and relies on the clients to provide their own styling and ["chrome"](https://en.wikipedia.org/wiki/Graphical_user_interface#User_interface_and_interaction_design).
	This moves the policy of how to define the chrome to the applications.
	This has the massive benefit that defines the chrome, is the application which likely knows best how to use the chrome.

	Policies tend to become *mechanisms* at the next higher abstraction level, and are used by even higher level policies as such.
	At the lowest level, atomic instructions are mechanisms provided by the hardware.
	We define policies on top of that to define locks and semaphores.
	We use those locks and semaphores as mechanisms to define the policy of full condition variables.
	This emphasizes the power of this separation: policy is enabled to use the abstractions in specific ways, and those policies that provide useful abstractions can be further built on.

	What's the downside?

	> the cost of the mechanism-not-policy approach is that when the user can set policy, the user must set policy.
	> - Eric Raymond

### Reading

- Chapters 1 & 2 in "Principles of Computer System Design: An Introduction" by Jerome H. Saltzer and Frans Kaashoek the latter chapters of which are [free online](https://ocw.mit.edu/resources/res-6-004-principles-of-computer-system-design-an-introduction-spring-2009/online-textbook/).
- [Lamport](https://www.dropbox.com/sh/4cex542zznbjh7b/AADM59pqAb9YBy4eeT1uw0t8a?dl=0)
- [The Art of Unix Programming](http://www.catb.org/~esr/writings/taoup/html/index.html)

## OS Design and Trade-offs

Necessary background concepts:

- Resources need to be controlled with policies, and those polices must be expressed in code.
- The kernel is only special as it has de-facto access to all resources.
- If the kernel provides a means for accessing resources to processes, and a subset of resources is given to a process, then that process has great power (over those resources), but great responsibility (to manage those resources well).
- If the system provides a means to *delegate* resources from one process to another, then access to the resource can be controlled *and extended* by the original resource holder; and if resource delegation can be delegated, then the ability to *manage* resources can be extended to other processes.
- In both cases, the question how to revoke access must be answered.
- Being concrete about resources, we can start by talking about those at the lowest-level:

	- CPU
	- Physical pages of memory
	- Interrupts
	- IPIs
	- Timers and timer interrupts
	- Devices (memory mapped) -- virtual access to specific physical addresses

	...or we might think about this at a higher-level

	- [system calls](https://www.youtube.com/watch?v=xHu7qI1gDPA&list=TLPQMjIwNTIwMjA1MAN_H1P-DA)
	- files
	- network sockets
	- pipes
	- shared memory
	- threads
	- processes
	- containers

- Case in point: We have $N$ pages of memory, who should be able to access them? The system must have a function $owner(n) \to \{ (p_0, va_i), (p_1, va_j), \ldots \}$, which is to say, that for frame $n \leq N$, it is mapped into a set of processes at specific virtual addresses. What in the system defines this function, $owner$?

At the highest level, OSes provide the following major functions:

1. sharing of hardware between different principals,
2. providing functionality, abstractions, and services for principals to harness, and
3. ensuring protection between principals.


### Example Design Issues

- Composability and tightening designs: recursive/[reentrant](https://en.wikipedia.org/wiki/Reentrant_mutex) locks/mutexes and why many [believe](http://www.fieryrobot.com/blog/2008/10/14/recursive-locks-will-kill-you/) they [should](http://www.zaval.org/resources/library/butenhof1.html) be [considered](https://blog.stephencleary.com/2013/04/recursive-re-entrant-locks.html) [harmful](https://inessential.com/2013/09/24/recursive_locks)

	- Code should, to the maximum extent possible, understand the state of the system necessary for it to properly execute.
		Not understanding if a lock is taken or not, and having that be dynamically determined is hiding important details from the programmer.
		Thus, it encourages the programmer & maintainer to no think hard about the locking structure of the system.
	- Generally holding a lock should be as short as possible.
		Recursive locks encourage you to have two separate understandings of the critical section length, conflated into the same code.
	- This encourages a notion of rentrancy: you're in a critical section, and start executing in a *new* critical section that accesses the same internal data-structures.
		This means that you need to understand that the critical section you're half way executing through can be interfered with by the recursive call.
		Thus, it doesn't prevent something that you don't want to happen.
	- Condition variables (`cv`s) interact horribly with this.
		If you wait on a `cv`, you release the lock...which might release it in an outer critical section not designed to be preempted via the `cv` API, or not release the lock at all (thus deadlocking)!
		Thus, these don't compose with condition variables.

	> Recursive mutexes are a hack. There's nothing wrong with using them, but they're a crutch. Got a broken leg or library? Fine, use the crutch. But at least be aware that you're using a crutch, and why...
	> - Dave Butenhof

	>  ...a thread tries to lock a lock that’s already, well, locked, you’ll deadlock. This is good. I like this, just as I like crashes. I know, you think I’m crazy, but if you deadlock on your own thread, then you know you just did something bad.
	> - [FieryRobot](http://www.fieryrobot.com/)

	> Recursive locks let you avoid design. If you're trying to avoid system design, just put the lock down, and step away.
   	> - Gabe

- Needing to weaken an abstraction for performance: [access times](https://en.wikipedia.org/wiki/Stat_%28system_call%29#Criticism_of_atime) on files (`relatime` as default in Linux, see `man mount`)
- Non-composable APIs: [file locks](https://gavv.github.io/articles/file-locks/) [considered](http://0pointer.de/blog/projects/locking.html) [harmful](https://www.samba.org/samba/news/articles/low_point/tale_two_stds_os2.html)

	- `fcntl(F_SET_LK...)`, `lockf(...)` (the POSIX options)-

		- locks associated with *process* thus don't help with multi-threaded access (first unlock, will unlock for all threads, and "retaking" will not count or prevent access), thus it must be paired with in-proc locks; the process-centric view is not "future proof" -- the intent was for the lock to have a "context", and instead of specify it as such, they chose the only current existing context, a process, which subsequently breaks when we now have threads
		- the lock is released on the `close `of *any* fd referencing that file, so, e.g. doesn't compose with `getpwuid` which accesses `/etc/passwd` -- no composability;
	- `flock` - does not enable record (byte range in file) blocking, all or nothing!
	- ...and these different APIs often don't interact! So do they actually provide utility?
	- All approaches: what about untrusted processes not releasing your file's lock? Permissions impact liveness! Not composable.


<!--
### Specification

- services as state & state transitions
- mathematical notation and description
- files and directories
- tighter or looser designs/specifications
- Hyrum's law, which states:

	> With a sufficient number of users of an API, it does not matter what you promise in the contract. All observable behaviors of your system will be depended on by somebody.
-->

### Layering and Structure

- Layering as a fundamental system structuring technique ([THE](https://dl.acm.org/doi/10.1145/363095.363143) OS)

	0. interrupts + scheduling
	1. memory allocation
	2. console
	3. I/O (paging/buffering)
	4. user programs
	5. "the user"

	Motivated by multiple desirable properties:

	- testing via layers
	- restricted reasoning about 1. current layer, and 2. the *abstraction* of lower layers (which have already been reasoned about).
	- isolation is cross cutting

	Limitations:

	- Debugging of lvl 0 & 1 without printing?
	- No memory allocation in scheduling?
	- Where is a "thread" structure implemented?
		Needed for scheduling, but also "associated" with memory allocations and I/O.

- [Microkernels](http://people.cs.uchicago.edu/~shanlu/teaching/33100_fa15/papers/nucleus.pdf) as a structuring technique
- Design via the separation of [mechanism and policy](https://dl.acm.org/doi/10.1145/1067629.806531)
- Concurrency-driven event notification

### Intuition: Vertical vs. Horizontal Isolation

We add isolation into systems to decouple different bodies of software (data and functionality).
We do this to

1. ensure that two bodies of software that should have no trust in each other, can only impact each other to a controlled extent (we are sharing a system's resources, so it is difficult to remove all impact), and
2. to concentrate privileged access to resources between different services and applications -- thus enabling a file system to have enhanced access to all file data, and to provide the necessary logic to *mediate* client access to that data.

An intuitive way to understand isolation properties is by considering both *vertical* and *horizontal* isolation.

**Vertical** isolation implies a logical *dependency* between a client, $p^c$, and a server $p^s$ that entails some amount of client trust in the server's logic.
Dependencies can take many forms.
These include:

- a *synchronous functional dependency* in which a client invokes a function provided by a server -- examples include system calls to the kernel from applications, hypercalls in hypervisors, and synchronous IPC to servers in $\mu$-kernels,
- a *resource dependency* whereby a resource is consumed by a, or on behalf of a client, while that resource is managed by a server -- examples include CPU time managed by a scheduler, and networking throughput consumed for a client.

Isolation of this type is quite complicated and must consider all of the bad things clients can try to do to servers, and all of the foreseeable bugs -- and their impact -- in servers.
We have to consider mundane abuse by passing all configurations of parameters (for functional dependencies), resource accounting/consumption attacks in which the client attempts to use resources in the server without them being properly accounted to that client, and denial-of-service attacks.
The focus is very much on isolating the server from the client (think: the kernel from applications), but also on providing a maximum isolation of the client from the server (i.e. limiting the *trust* the client must have in the server).

Properties of this isolation:

- It is often highly asymmetric: the client depends on the proper functioning of the server, but not vice-versa.
	Different technologies might change this trust relationship.
- As such, the focus is on ensuring that server memory and data-structures are not accessible from clients.
- It is not uncommon for the server execution to occur within the scheduling context of the client (e.g. a kernel system call is accounted to the client requesting that service).
	This does require some amount of trust as a server that goes into an infinite loop does expend the client's cycles.
- It is very important that the server maintains control over how it begins execution, and maintains that control (i.e. system calls trap *only* to kernel instructions configured by the kernel itself).
- The key is that the client and server (by their very nature) interact in controlled ways.

**Horizontal** isolation involves any two clients which each both depend on sets of servers.
The strength of horizontal isolation depends on the strength of the vertical isolation for each server.
For example, if a client can compromise one of these servers through a functional dependency, all other clients of that server might suffer decreased or errant service.
A focus of horizontal isolation is on strong inter-client isolation.
The focus of the system is often of implementing the shared server in such a way that it can provide its service to all of its clients correctly, while also ensuring that client's have limited interference on each other.

Properties of this isolation:

- The main goal is to simply prevent interactions between horizontally isolated clients, by default.

**Isolation intuition examples.**

- Monolithic OSes provide vertical isolation (via dual-mode hardware) of the kernel from user-level applications, and require strong trust of applications on the kernel.
	They focus on horizontal isolation of processes from each other.
- Virtual Machines focus on vertical isolation of the hypervisor, but prioritize horizontal isolation of the majority of the system's software in VMs.
	Within the VM is almost always a monolithic system.
- Containers modify a monolithic system but actually *don't change the structural isolation of the system*.
	This points to the fact that this model of thinking about isolation is not sufficient.
- $\mu$-kernels must be *designed* to discriminate between horizontal and vertical isolation (see [Figure 1](http://www.minix3.org/docs/login-2010.pdf) for an example).
	Clearly, applications are isolated horizontally, but services that they leverage have components of being (of course) vertically isolated from clients, but *also* horizontally isolated as they might leverage other system services.

### Detailed Isolation Model

Horizontal and vertical isolation provide some intuition about what types of isolation are desirable.
However, more fidelity is necessary to capture more complicated relationships.
When you need to describe something with high specificity, you kind of have to math it.

- $p_i \in P$ is one of the protection domains in the system.
- $d^{\text{op}}(p_i) = \{d_{j \neq i}, \ldots\}$ is the set of protection domains that $p_i$ depends on for dependencies of operation type $\text{op}$.
- Dependency types (not an exhaustive list) including $\text{op} \in \{ \text{sync, async, strm, cpu, mem, boot}\}$ which are (respectively)

	- *synchronous function invocation* from $p_i$ to $p_{\text{sync}} \in d^{\text{sync}}(p_i)$,
	- *asynchronous request and response* from with request made from $p_i$ to $p_{\text{async}} \in d^{\text{async}}(p_i)$, and the an eventual reply from $p_{async}$,
	- *data is streamed* from $p_{\text{strm}} \in d^{\text{strm}}(p_i)$ to $p_i$ (this odd ordering is simply saying that the "dependency" of the consumer on the producer),
	- *management of resources* by $p_{\text{cpu|mem}} \in d^{\text{cpu|mem}}(p_i)$ for $p_i$ -- these are schedulers, or memory managers, and
	- *a booter* $p_{\text{boot}} \in d^{\text{boot}}(p_i)$ that is responsible for creating the $p_i$ process

- $o(p_j) = \{o_x, \ldots\}$ is the set of abstract objects *exported* to clients from service $p_j$ and accessible through synchronous function invocations.
	These objects might include files, pages of memory, network sockets, IPC end-points, etc... and $p_j$ is the system service managing those abstractions.
- $n(p_i, p_j) = \{o_x, \ldots\} \subseteq o(p_j)$ is the set of objects accessible via synchronous invocations from $p_i$ and provided by $p_j$ (where as we'll see soon, $p_j \in d^{\text{sync}}(p_i)$).

Note that the dependency relation is *transitive* for a subset of the dependency types, $\{\text{mem, cpu, strm}\}$ such that $\forall_{o \in \text{\{mem, cpu, strm\}}} p_j \in d^o(p_i) \wedge p_k \in d^o(p_j) \to p_k \in d^o(p_i)$.
All protection domains in the system that provide memory and CPU time must be depended on continuously to provide those services.
For streaming dependencies, all "upstream" protection domains must continue to do their job for a process to do useful work.

The rest of the dependency relations are more local, thus are not transitive (including $\{sync, async, boot\}$).
If $p_c$ can make a synchronous invocation to $p_s$, it does *not* mean that $p_c$ can  make synchronous invocations to $p_{s'}$, despite $p_{s'} \in d^{\text{sync}}(p_s)$.
Note that this is *essential* as it supports information hiding and separation of privileges -- a user application should *not* be able to ask a driver to directly write to disk; it *must* go through the file system to validate the request!
Similar arguments can be made for $\text{async}$ and $\text{boot}$.

Lets revisit the examples:

- Monolithic system is simply a set of processes $A = P \textbackslash p_{\text{kern}}$ (note that $A\textbackslash a$ means the set resulting from removing $a$ from set $A$), and the kernel $p_{\text{kern}}$, where $\forall_{p_i \in A, o \in \text{op}} d^o(p_i) = p_{\text{kern}}$.
	This shows the centrality and concentration of trust in the large kernel.
- VMs simply add a new layer to this: a set of VMs, $V$, with each VM, $V_x \in V$, consisting of a set of processes and kernel, $V_x = \{p_{i,x}, \ldots\} \cup p_{\text{kern}_x}$, with similar trust relationships $\forall_{V_x \in V, p_{i,x} \in V_x, o \in \text{op}} p_{\text{kern},x} \in d^o(p_{i,x})$.
	Each VM kernel and application has specific dependencies on the hypervisor, $p_{\text{hv}}$: $\forall_{V_x \in V, o \in \text{op}} d^o(p_{\text{kern},x}) = \{p_{\text{hv}}\}$ and $\forall_{V_x \in V, p_{i \neq \text{kern},x},  o \in \{\text{cpu, mem}\}} p_{\text{hv}} \in d^o(p_i)$.
	It is necessary for proper (horizontal) inter-application protection that applications themselves cannot directly make requests of the hypervisor, and instead must go through their guest kernel.
	This structure also demonstrates that for applications in *different* VMs to interact they must do so by making requests through their kernel, and through the hypervisor.
	As such, VMs form a highly *hierarchical* structure.
- $\mu$-kernels are complicated here, and I'll delay their discussion till we talk about component-based systems.
	The key insight is that the protection domains in the system define if we are using vertical or horizontal isolation by controlling the inter-protection-domain relations, and which protection domains are in charge of what resources.

### Questions

The OS must include services that provide functionality to multiple application principals.
This raises more questions than you'd likely expect:

- **Q1.** Do we consider protection in spite of compromises and faults in OS services?
- **Q2.** What failures do we try and recover from, and which do we consider terminal?
- **Q3.** The implementation of a service requires committing some resources (CPU processing in the service, and memory `malloc`ed).
	How do we provide protection between principals, given this?

- **Q4.**
	What is the *core* of many of the issues in the [comparison between POSIX and Win32](https://www.samba.org/samba/news/articles/low_point/tale_two_stds_os2.html)?
- **Q5.**
	Imagine we create a `int close_unlink(int fd, char *path)` that combines close and unlink.
	Is this useful?
	When?
	How would you go about assessing the utility of the call?
	When could it fail?
- **Q6.**
	API question for locks: do you

	1. publicize that a failure (`assert`) will happen if you recursively take a non-recursive lock,
	2. deadlock on your own contention, or
	3. do you return an error value?

- **Q7.**
	Do you implement malloc that is thread safe, or not?
	How about signal-safe?
	Or do you publish that it is not thread safe, and rely on the user to use it safely (e.g. with a lock)?

How we think about system design matters (i.e. our mental model for the system design).
We might think of a "flat" system as a microkernel, that is often visualized with servers all at user-level, on the same "level".
The emphasis is "servers" aren't special, and are user-level protection domains, just the same.
On the other hand, we think of layered system structures with a total order on software subsystem dependencies, like THE.
All systems must be layered to some degree if they use dual-mode hardware (the kernel layered below the rest).
However, this mental model is overly-simplistic.

**Q8.**
List the differences between two different relationships between protection domains, $p_0$ and $p_1$:

1. $p_0$ uses IPC to harness some functionality in $p_1$, thus $p_1$ provides a *service* to $p_1$ which is a client, and
2. $p_0$ and $p_1$ are siblings (or "peers") in that they both use IPC (or generally some communication mechanism) to harness the functionality of some services, with some intersection of those harnessed services, $\{p_2, p_3, \ldots, p_n\}$.

Start with some examples of both structures in systems your familiar with.
Specifically focus on where attacks are, where "*trust*" is required (and trust in what dimension -- from full trust, "they can crash me at any time, and can access all of my secrets", to more nuanced trust, "they can access none of my resources, but IPC is synchronous, thus I trust them to return in a predictable amount of time").

## UNIX

[UNIX](https://dl.acm.org/doi/10.1145/800009.808045) design [philosophy](https://danluu.com/mcilroy-unix/)

> A number of maxims have gained currency among the builders and users of the UNIX system to explain and promote its characteristic style:
>
> Make each program do one thing well. To do a new job, build afresh rather than complicate old programs by adding new "features."
>
> Expect the output of every program to become the input to another, as yet unknown, program. Don't clutter output with extraneous information. Avoid stringently columnar or binary input formats. Don't insist on interactive input.
>
> Design and build software, even operating systems, to be tried early, ideally within weeks. Don't hesitate to throw away the clumsy parts and rebuild them.
>
> Use tools in preference to unskilled help to lighten a programming task, even if you have to detour to build the tools and expect to throw some of them out after you've finished using them.
>
> - McIlroy, "UNIX Time-Sharing System: Forward"

- The [origin of pipes](http://doc.cat-v.org/unix/pipes/) and a broader discussion by [McIlroy](https://archive.org/details/DougMcIlroy_AncestryOfLinux_DLSLUG)
- Pipes for computation composition, hierarchical namespace for resource addressing
- First, lets take on composition: compare

	- program -> shell -> pipelines -> shell scripts
	- functions -> libraries -> dependencies -> scripting languages -> applications

- UNIX composition properties

	- shell resolves dependencies -> pipes fast-paths unstructured, text-based interactions
	- stream-of-byte interactions require parsing
	- fast path still uses IPC through kernel

		- overhead of kernel IPC
		- provides isolation, concurrency, and parallelism by decoupling pipeline stages

	- Stream-of-bytes interface is limiting

		- system services that control and manage resources don't interact with a stream-of-byte
		- Complex applications (browser, games, editors, ...) require richer interactions

	- Streaming interface is limiting

		- *Not* synchronous, not call/response; how can we make a request, change the system, and get a corresponding response?
		- Can we implement OS services using pipe-based composition? Why/why not?
		- How do errors percolate back through a pipeline?

### Shortcomings of UNIX design

Scaling abstractions across features:

- non-uniform event notification (fds vs. signals vs. timers)
- [fork](https://www.microsoft.com/en-us/research/uploads/prod/2019/04/fork-hotos19.pdf) (vs. `posix_spawn` and [`CreateProcessA`](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessa))

	- threads
	- signals
	- locks
	- mmap
	- buffered I/O (flush before fork)

- reentrancy/threads/`errno`
- signals

	- reentrancy (`strtok`)
	- `errno = EINTR` for *most* system calls; when is the last time you checked for this?
	- signals don't compose with stateful APIs, nor with **system calls**!
	- Should use sigaction to turn off this behavior in most cases using `SA_RESTART`

[Tacked on and/or replaced mechanisms](https://roxanageambasu.github.io/publications/eurosys2016posix.pdf) -- discussion on the self-consistency of design

- BSD sockets
- semantic notification (dbus via `kdbus` and binder via `ioctl`)
- file storage (SQLite in Android)
- `ioctl` - Graphics, IPC (Binder), I/O out-of-band changes, and "system call extensibility" (see binder)
- concurrency/parallelism frameworks -- thread pools (`libglib` (e.g. GThreadPool) & event loops (`libuv`))
- inter-processes memory control ([memfd](https://man7.org/linux/man-pages/man2/memfd_create.2.html))
- `setuid` for privilege + delegation

Hard to merge UNIX philosophy and frameworks/programming languages?

- "Do one thing well" vs. aggregating functionality and becoming increasingly self-contained
- There are trade-offs between the two, and there is an argument *against* [mixing them](http://harmful.cat-v.org/cat-v/unix_prog_design.pdf) (remember, consistency)
- study of [commmand-line complexity](https://danluu.com/cli-complexity/)

> To many, the UNIX systems embody Schumacher's dictum, "Small is beautiful." On the other hand it has been argued by Brooks in The Mythical Man Month, for example, that small is unreal; the working of a handful of people doesn't extrapolate to the world of big jobs. We agree only in part, for the present volume demonstrates with unusual force another important factor: intelligently applied computing technology can compress jobs that used to be big to manageable size.

### Reading

- Architecture of [bash](http://aosabook.org/en/bash.html)
- `xv6` [shell](https://github.com/gwu-cs-os/gwu-xv6/blob/master/sh.c)

## UNIX Counterpoints

### Modern software development properties

- programming environment resolves dependencies (`cargo`, `pip`, `npm`, `docker`, ...), which is [quite](https://research.swtch.com/vgo-import) [complicated](https://research.swtch.com/vgo-intro), actually [NP-complete](https://research.swtch.com/version-sat) often using full on SMT solvers, linking fast-paths typed interactions
- libraries implicitly trusted
- synchronous, typed interactions that avoid parsing with shared arguments via reference
- no isolation means that this cannot be used as a *general system composition* technique
- full breadth of programming language semantics and typed argument formats

**Thought exercises.**

Lets go through a sequence of questions to make sure we understand the relationship between an application, a library, and the kernel.

- In C, what can a library do to interfere with the proper functioning of the client program?
- In a type safe language (e.g. Java, Python, Rust), what can a library do to interfere with the proper functioning of a client program?
- What if we want to have the library and application be implemented in different languages?

	- What is FFI?
		What complexities does it present? (Hint: garbage collection, data-layout, liveness, etc...)

- How about protection of the library from the application? (why?)

	- In the previous cases, we should consider both *buggy* and *malicious* libraries.
	- Consider memory and temporal safety: core issue is synchrony, and bitwise corruption.

- Why are some services isolated in a separate processes, rather than as a library (think a database)?
- What assumptions would need to be made, and how would a system need to be constructed, to enable functionality typically resident in the OS, in libraries instead?
- What are the properties of kernel functionality that might make it *impossible* to be implemented in a library?

### Centralized frameworks (or middleware)

Historically, many daemons would provide system services in UNIX:

- bootup is always driven by `init` (generally executed from memory), which does a number of things including mounting the initial filesystems, and `wait`ing for zombie processes without parents whom should wait for them, and starting other daemons/services *sequentially* via `initrc` (isn't this a lot for something that should be following "do one thing well"?),
- `cron` would execute commands at a time in the future (perhaps repetitively),
- `update` (to periodically write out FS superblocks),
- [`getty`](https://en.wikipedia.org/wiki/Getty_(Unix)) to provide terminal services (input/output) and is part of the traditional `init`$\to$`getty`$\to$`login`$\to$`shell` sequence,
- `telnetd` and `sshd` provide their corresponding terminals,
- `inetd` that understands which programs should reply to network requests on different ports, and will `fork` a new service for each request (expensive!!!)
- and Linux has added a few on its own:

	- [`udev`](https://opensource.com/article/18/11/udev) enables dynamic device management -- for example how does the system know which driver, and which `/dev/*` entry to use for a thumb drive -- and
	- the Network Manager to determine which wireless AP or wired connection to use

[`systemd`](https://lwn.net/Articles/777595/) is a rethink of [pid `1`](http://0pointer.de/blog/projects/systemd.html) and is in some sense `inetd` for local services as well and is highly influenced by OSX's [launchd](https://www.youtube.com/watch?v=SjrtySM9Dns)

- It is actually a [collection of processes](https://www.linux.com/training-tutorials/understanding-and-using-systemd/)

	- See a list of all services with  `systemctl` and find those units postfixed with `*.service`; those with `systemd-*` are from the systemd collection
	- There's even a service to create [containers](https://blog.selectel.com/systemd-containers-introduction-systemd-nspawn/)
	- It will try and reboot services if they fail, and you can [write your own](https://medium.com/@benmorel/creating-a-linux-service-with-systemd-611b5c8b91d6)
	- Inter-service dependencies are specific similar to programming language environments (which service should we start, and startup ordering)

- They are meant to be used as a cohesive core (rather than replacable/composable), and are not portable (using Linux-specific APIs), lamentations of which get the reply from the FreeBSD dev, Benno Rice, ["Unix is dead"](https://www.youtube.com/watch?v=o_AIw9bGogo).
- Uses D-Bus extensively as a message based transport that can be used for RPC. There is no semblance of pipes here, but rather a typed API for service-based coordination

	- [D-Bus](https://en.wikipedia.org/wiki/D-Bus) enables addressing of services, addressing of application-specific values (a cell in a spreadsheet), and the *typing* of interactions on the operations on those objects
	- Initially implemented as a single process on the system that others chat with via sockets; what are the trade-offs
	- pub/sub notifications: can publish events on known names -- energy events, wifi changes, opening a music file, etc... and applications/services can await those events, and handle them
	- name = bus name (unit location) + object path (location within an application) - similar to URLs/REST where bus name is routed via dbus, and object path is chosen by the service library/app
	- requires [types over bitstreams](https://pythonhosted.org/txdbus/dbus_overview.html), and [API specification](https://dbus.freedesktop.org/doc/dbus-api-design.html)
	- [`kdbus`](https://lwn.net/Articles/580194/) attempted to move the dbus communication logic into the kernel, but a [lot of](https://lwn.net/Articles/649111/) [details were raised](https://lwn.net/Articles/619068/) and (see the  [documentation](https://lwn.net/Articles/619069/)), and development moved on to [Bus1](https://bus1.org/) which is...a kernel dbus implementation!
		The initial PR showed good performance numbers: going from [$200\mu s \to 8\mu s$](https://lwn.net/Articles/640360/).
		A well-articulated summary of the [complications](https://dvdhrm.github.io/rethinking-the-dbus-message-bus/) of doing `dbus` in user-level, and some of the historical bugs.

- So wait, is Linux a micro-kernel?

### Questions

**Q1.**
Go through the [dbus example](http://www.matthew.ath.cx/misc/dbus) (cleaner code [exists](https://github.com/diwic/dbus-rs/tree/master/dbus/examples) for higher level languages).

- How much of this is accidental complexity?
- Where is the type-checking being done?
- Explain the [bugs](https://dvdhrm.github.io/rethinking-the-dbus-message-bus/) that are motivating creating a new `dbus` broker.

**Q2.**
Lets get a sense for how to implement a nameserver on Linux.
UNIX domain sockets can be used to [pass file descriptors](https://copyconstruct.medium.com/file-descriptor-transfer-over-unix-domain-sockets-dcbbf5b3b6ec) between processes!
Implement a simple `nameserver` that enables other "server" processes to associate themselves with a simple string that serves as a *name*.
"Clients" can ask the `nameserver` to allow them to communicate with a server associated with a given *name*.

1. Add RPC (synchronous "call-return") style calls to your `nameserver`.
	Implement this such that the client and server can communicate via directly-connected sockets (read the previous link to see how `fd`s can be passed between processes).
2. Add publisher-subscribe to your `nameserver` such that services can publish messages on a *name*, and the `nameserver` will send them to all clients subscribed to the *name*.

## Plan 9

[Plan 9](https://9p.io/plan9/) [design](https://9p.io/sys/doc/9.pdf) and [related info](https://en.wikipedia.org/wiki/Plan_9_from_Bell_Labs)

Fundamental question: Can we expand most aspects of system implementation to the "UNIX philosophy"? The idea:

1. Address *all* resources via the hierarchical namespace (think: are BSD sockets part of the Linux namespace? how about GUIs? how about program execution through text interpretation? how about remote hells, e.g. ssh? how about webpages?).
2. Enable per-process namespaces (can generalize `setroot` and container namespacing functionalities) and union mounts (binds).
3. Provide facilities for virtual resources (built on those provided by the kernel) that is latched into the namespace, and leveraged via IPC.
4. The protocol for namespaces and IPC has a serial representation, thus can span over a network.

This is pushed to a logical conclusion: execution, visualization and human interaction, and the semantic behavior of text (the bitstream), are all defined as user-level servers, hooked into the namespace, and leveraged via IPC.

- A set of [servers](http://man.cat-v.org/plan_9/4/) that all speak [the](http://doc.cat-v.org/plan_9/misc/ubiquitous_fileserver/ubiquitous_fileserver.pdf) [9p](https://en.wikipedia.org/wiki/9P_(protocol)) [protocol](http://man.cat-v.org/plan_9/5/intro) by accessing an manipulating their [namespace](http://doc.cat-v.org/plan_9/4th_edition/papers/names), and are accessible via general and programmable [IPC](http://doc.cat-v.org/plan_9/4th_edition/papers/plumb).
	Some [namespace](https://blog.darknedgy.net/technology/2014/07/31/0-9ns/) and [networking](https://blog.darknedgy.net/technology/2014/11/22/1-9networking/) examples.

**Remember: worse is [better](https://dreamsongs.com/WorseIsBetter.html).**
Plan 9 had an even more up-hill battle vs UNIX: coming out (significantly) later *and* requiring software migration.

> [I]t looks like Plan 9 failed simply because it fell short of being a compelling enough improvement on Unix to displace its ancestor. Compared to Plan 9, Unix creaks and clanks and has obvious rust spots, but it gets the job done well enough to hold its position. There is a lesson here for ambitious system architects: the most dangerous enemy of a better solution is an existing codebase that is just good enough.
> - Eric S. Raymond

### Questions

- How does the `plumber` compare to `dbus`?
	Similarities/differences?
	How do the APIs differ significantly?
- How does `9P` compare to `dbus`?
	Is there any analogy that fits best to `9P`?

## Capability-based OS

- Fundamental question: Can we enable the program-definition of *all* resource management policies, while maintaining performance, and security.
- Accidental complexity can create a *semantic gap*.
	When the system abstractions work *against* your goals, there is a *semantic gap* between what the system architecture makes easy or prioritizes, and what you need from the system.

## Case studies

- Asynchronous IPC between threads, considering resource accounting, data movement, parallelism, and implications on IPC interface and [zeromq](http://aosabook.org/en/zeromq.html)
- Synchronous IPC, in Nova
- [Demikernel -- see Fig 3](https://irenezhang.net/papers/demikernel-hotos19.pdf) ([code](https://github.com/demikernel/demikernel/blob/master/API.md)) vs. application [message passing](http://aosabook.org/en/zeromq.html) vs. POSIX
- Linux [vfs](https://www.kernel.org/doc/html/latest/filesystems/vfs.html) through the [ramfs](https://elixir.bootlin.com/linux/latest/source/fs/ramfs)
- [WASI](https://github.com/bytecodealliance/wasmtime/blob/main/docs/WASI-overview.md) [interface](https://github.com/WebAssembly/WASI/blob/master/phases/snapshot/docs.md)
- [FreeRTOS](http://aosabook.org/en/freertos.html) vs. patina

## Recommended Reading

- "Principles of Computer System Design: An Introduction" by Jerome H. Saltzer and Frans Kaashoek the latter chapters of which are [free online](https://ocw.mit.edu/resources/res-6-004-principles-of-computer-system-design-an-introduction-spring-2009/online-textbook/).
- [Lamport](https://www.dropbox.com/sh/4cex542zznbjh7b/AADM59pqAb9YBy4eeT1uw0t8a?dl=0)
- The [Little Manual of API Design](https://people.mpi-inf.mpg.de/~jblanche/api-design.pdf)
- The Architecture of Open Source Applications ([aosa](http://aosabook.org/en/index.html))
