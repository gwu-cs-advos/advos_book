# Frequently Asked Questions

## Constraining Complexity

Section on "Complexity Management", and lecture [video](https://youtu.be/a8V2d33KvaE).

### Simplicity

- If you avoid generalizing your code, don't you pay for it later, and add more redundancy and complexity?
	You want to avoid generalizing immediately when you approach a problem.
	If you do start to have redundancy in your code, and it would lower complexity to generalize, then you should.
	Take this prescription as a proxy for "implement a concrete version of functionality before even considering generalizing; once you understand the system's complexities, generalize only where appropriate".
- How can over-generalizing increase complexity?
	Lets compare the code for an "[embedded radix trie](https://github.com/gwsystems/ps/blob/master/ps_ertrie.h)" in `parsec`.
	That code enables a radix trie of a configurable number of levels, with each node being of a configurable size, enables configuring how the pointers to each level is resolved (e.g. page-tables contain pointers to physical addresses, thus require translation to virtual addresses before dereferencing), etc...
	If you just need a radix trie with two levels that contains `void *`s, the code is simple:

	```c
	#define SZ_BITS 10
	#define SZ (1 << (SZ_BITS-1))
	struct simple_rt {
		void *ptrs[SZ];
	};
	void *lookup(struct simple_rt *r, uint32_t id) {
		struct simple_rt *level2;
		uint32_t lvl1_id = id >> SZ_BITS;
		uint32_t lvl2_id = id & (SZ-1);

		level2 = r->ptrs[lvl1_id];
		if (!level2) return NULL;
		return level2->ptrs[lvl2_id];
	}
	```

	This is much simpler, but not as general^[Note that `ps` is meant to be a library for supporting much more than just radix tries, so the complexity is likely warranted from the requirements, but the *first implementation* was certainly much simpler.].

### Modularity

- Are applications, modules?
	From a system-designer's perspective, Yes!
	We might think about an application as a simple `.c` file we load into the overall system's binary.
	This is the typical case for Real-Time OSes.
	In contrast, we can isolate, and dynamically load application "modules".
	From a system's perspective, we need to determine how we want to execute the modules we happen to call applications.
- A DAG model of system architecture seems too complex, and like it would push out development timelines; is this OK?
	Modern development practices certainly make a DAG structure seem complex.
	There is a beauty to being able to simply create a new application using the shell, rather than a web of connected modules.
	There is simplicity in having applications be layered above a large system call layer to all of the kernel's functionality.
	But there are costs WRT security for these decisions.
- In layered systems, how do you interact with lower levels that aren't directly below you?
	In a strict layered system, you can only interact with the layer below you.
	If a child, $c$, must leverage functionality below that (in layer $g$ -- grandparent), then your parent $p$, needs to provide an interface that calls $g$ for $c$.
	If a system does allow $c$ to directly leverage abstractions of $g$, this is called $p$-"bypass".
	Modern systems (like DPDK) enable device drivers at user-level which provide kernel-bypass -- you can send packets without kernel involvement!
- Is there any way to modify interfaces without causing required changes in modules?
	Yes.
	There are many changes that you can make in a "backwards compatible" manner.
	Adding new functions that only use existing functions, for instance, requires no changes in surrounding modules.
	However, these changes are limited in their power.
	The addition of functions requires changes in the implementing modules, but not those that depend on the system.
	Changes to *existing* functions require changes in both implementing and dependent modules.
	This yields a significant design space in which you can try and make changes that minimally impact surrounding modules, but the required functionality will also determine which option is available.
	Read more about interface versioning to understand how people wrap these decisions into a software design process, and semantic versioning to see how these options are encoded into version numbers.
- How do languages fit into the abstraction landscape?
	Languages are an abstraction that bridges the gap between human cognition, and computational models.
	Higher-level languages like Java, Haskell, or SQL disallow the human to express many things, but those constraints make it easier to write code that you can express.
	In contrast, C lets you express just about anything, but is a horrible language for complex application-level programming.
	Languages are interesting as they have a different *dimension* of trade-offs in the *logic* that underlies the language.
	Rust attempts to give you the ability to express the same effects as C, and constrains what you can do with the language using the *type system*.
	As such, the trade-off you make is in programming logical model complexity -- developers must come to understand affine types, borrowing/ownership, interior mutability, etc... before leveraging the language.

	I would not put languages into the same "class" as the modules and interfaces we're discussing in the class.
	The interface with human cognition makes them into a different beast.
	However, they are not all-together dissimilar from the concepts of abstraction we'll focus on in systems.
- How do layering and hierarchy fit into the separation/isolation we know about in systems (e.g. kernel/user)?
	They are, in many ways orthogonal concepts.
	Modularity is a statement about handling system complexity.
	However, pairing modularity with isolation is, of course, a key system concept, and how we consider fault isolation and security constraints in addition to addressing complexity.
- How does a component relate to a module?
	For the purposes of this class (and most other purposes), they are equivalent.
	I use the term "module" here as it is the term used in the referenced books.
- How is modularity different from abstraction?
	How I introduce them, abstraction is about constraining complexity, and structuring interactions with resources.
	Modularity is about information hiding, and replaceability -- implementing abstractions in software.
	At the highest-level, abstraction is the conceptual idea, and modularity is about making it real.

### Technical Debt and Accidental Complexity

- When do you go back and pay off the technical debt?
	This is always a function of three things:

	1. what the development "velocity" of the current system is with the technical debt (think: the average productivity per day of each developer),
	2. what would the development velocity be if you removed technical debt, and
	3. how long will it take to remove the technical debt?

	Often the velocity doesn't matter as a deadline makes 3. intractable.
	However, a team's architect should make the call about when to work down technical debt based on the long-term ability of the team to function.

	Example: we used to only support a "fork-style" creation of new protection domains in Composite (as it was easy to implement).
	It was creating a great many problems as time went on (fork is not composable with many different abstractions).
	Thus, we spent some time to implement a `posix_spawn`-like API, and it sped up our research significantly.
	I noticed that we still weren't as agile as we should be, so we implemented the Composite Runtime (`crt`) library to remove a lot of accidental complexity in the system APIs.
- Is there a certain amount of technical debt that's allowed?
	Absolutely.
	It is always an engineering decision.
	Do you have the developer time to spare to remove it, or not?
	If so, improve the code base.
	This is a very nuanced decision.
- Is there a way to quantify technical debt?
	Not that I know of.
	In some sense, it is quite subjective.
	Until we can quantify "proper design", I'm bearish on quantifying technical debt.
- Is it not better to plan your software development to minimize technical debt?
	Absolutely.
	However, this is not a sufficient solution due to two factors:

	1. You cannot preemptively understand all impacts of planned features.
		There are always complexities that you simply didn't "see" at design time.
		It is not uncommon for these to provide awkward "contortions" in the code to fit them into the planned design.
	2. Software is not designed once, implemented, then static.
		We receive new design requirements and feature requests *after* you've spent the time on the design.
		These new requirements might require significant changes in the existing design to avoid technical debt.
		Your team might not have time for this restructuring.
- What do we do when we find our self introducing unexpected accidental complexity?
	You have to make the call: do you go back and redesign to remove the friction, or do you continue on and accept the debt.
	We're engineers, and this is a trade-off we have to make frequently -- better to make it consciously than not.

### Semantic Gap

- What types of semantic gaps exist?
	There are those that are due to simply a large gap between the abstractions a system provides, and those required to implement a functionality.
	For this type, one could easily install or implement new layers of abstractions to bridge the gap.
	There are those that are due to a gap between the abstraction provided and that of those required where there is a fundamental incompatibility between the two.
	Abstractions *are opinionated*.
	They disallow certain ways of using the system.
	If one of these disallowed uses is required by the new module, then there isn't a means to bridge the semantic gap.
	The underlying system is simply fundamentally inappropriate for the task at hand.
