\newpage

# Frequently Asked Questions

## L1: Constraining Complexity

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

## C1: System Interfaces and Zircon

- What differentiates systems programs vs. application programs?
	A simple set of differences get at the main point.

	1. System services define APIs, while applications use them.
		Thus, services define the abstractions of the system, applications use them.
	2. This is confusing when you consider that libraries also define APIs but aren't often considered system services.
		This is because libraries don't need to not just define resources, but also multiplex and control access to them across multiple applications.
	3. As the system service cannot share its memory resources with an application (lest that application corrupt its data-structures), it cannot provide pointers to the client to refer to its resources.
		Thus, system services often provide handles (integers, or a level of indirection) to bind the clients references to the resource.
		Correspondingly, you will see this in many system designs.
- Why do different clients require different mapping functions from names to resources?
	Rephrasing: why do different processes have different virtual address spaces?
	They should only be able to address potentially different sets of resources (pages of memory).
	Containers are another good example of this: two containers have different views of the FS to increase isolation and customizability!
	Even without containers, directory access rights can prevent some users from seeing some resources.
- Do we care about system design concerns (naming, etc...) differently in systems with different system structures (e.g. a monolithic system vs. a micro-kernel)?
	These are pretty universal concerns.
	In monolithic systems, the handle-based bindings are exposed to user-level, while pointer-based bindings are used between different system modules.
	In micro-kernels, you typically require handle-based bindings between each service (as they cannot share pointers).

### Action Aggregation

- Action aggregation is terrifying; how can we securely implement it?
	Yes, it is terrifying.
	You're running user-code on your system!
	You always want to run user-code in a "sandbox" -- i.e. a controlled environment in which they can only access resources you've explicitly granted them.
	You need to pair that with a constrained API that prevents the user from doing "bad things".
	This is why most technologies for this use higher-level languages (like SQL for stored procedures, and lua for game policy programming), or carefully constructed sandboxes (ebpf and Webassembly).

	An example for which you can more easily understand the security story: if you're installing software on your system, the software server is conceptually uploading a shell script onto your system that contains the programmatic logic of installation.
	What do you have to do to make this secure?
	Make sure that the shell script cannot access parts of the FS outside of what you want, memory-limit the script, make sure that it is CPU-limited, thus cannot monopolize the cores, etc...

### Event notification

- How do event notification systems handle races on modifications and accesses to the resources?
	If we're told via an event notification API that a pipe has data, then before we can read from it someone else consumes the data, any further processing assuming that the event is still pending is misguided.
	Level-triggered APIs (that return the *current* state of the system -- i.e. if there is an event active *now*) help, but you can always have a race between when the event notification API returns, and you try and access the resource.
	There is no solution to this aside from tightly constraining who can access the objects (files) your waiting for events on.
- Are "handles" the "fds" of zircon?
	Yes.
	They are the dynamic bindings to the kernel resources.
- How does "liveness" work for kernel objects?
	Some kernel resources have access to other handles (e.g. thread/process).
	When you close a handle, the "kernel resources" are supposed to be cleaned up.
	How does this happen with these handles that have handles?
	All kernel objects are reference counted, so they are removed when there are no more references to them.
	In this "nested" case, removing the handle to the thread might mean that the contained handles are no longer referenced, thus will also be deallocated.

### Threads vs. Events for Concurrency Control

- Thread vs. event driven architecture trade-offs?
	Simply put, we need threads if we want to execute on multiple cores.
	We often want events (and event loops) when we want high scalability with controlled memory consumption (no thread stacks).
	Unfortunately naive event programming is not fun and requires "stack ripping" or nested closures.
	Modern languages have gotten smart and are able to turn functions into state-machines that look like threaded code, but are, behind the scenes, event-driven code (see `async/await` in javascript and Rust).
	There are arguments that [events are better](https://web.stanford.edu/~ouster/cgi-bin/papers/threads.pdf), and that [threads are better](http://capriccio.cs.berkeley.edu/pubs/threads-hotos-2003.pdf), but I think that the world has shown that they are just [multiple tools for the job](https://lwn.net/Articles/223980/).
- Why do event-driven APIs require non-blocking?
	If we block on a system call, we cannot service any other pending events.
	Thus blocking is bad when you want a single thread to service multiple event sources.
	But if `select`/`epoll` told you that there was data on a file, why do you care if the system call is blocking or not?
	You shouldn't block as you know data is available on it, right?
	Event notification APIs tell you that there is data available, not *how much* data is available.
	With blocking APIs, it is really hard to know when you've read all available data.
- Do systems provide both threaded and event-driven?
	Yes!
	They provide both thread APIs, and event notification APIs with the option of asynchronous APIs.


## L2: Interface Design and Properties

### State Management

- The less our design leverages global state, the better?
	Generally, yes.
	It is much easier to test, and implement functions that are isolated from the effects of other functions -- in some sense, avoiding global state massively increases compositionality.
	However, it is impossible to provide many system services with this property.
	We always have some global structures as they track the state of system resources.
- Is idempotency equivalent to deterministic functions in math?
	No.
	Deterministic functions (if I understand the referenced concept) return output based solely on the input.
	These are *pure* functions or (as I said in the lecture) "stateless".
	Idempotent functions can have output based on "hidden" state (think, a cache, a disk, etc...).
	If some of those operations can *change* the state, then another otherwise idempotent functions retrieve that state, the retrieval functions are not idempotent with respect to the update operations.
	However, the retrieval and update operations are likely idempotent with respect to themselves.
	A simplistic (and common) view on idempotency ignores the relationship of these functions to each other.
- Is it possible to have a thread-safe function be re-entrant?
	Yes.
	A few examples (I'm sure there are more, but these are the ones that popped into my head):

	1. You can use recursive/re-entrant locks (see the book), and track if your own thread is already processing in a critical section, and figure out some way to proceed without touching the shared structures (e.g. return some failure mode).
	2. You can use wait-free algorithms that only use atomic instructions, and avoid `cas` loops, thus can proceed in the signal handler regardless where the preemption happened in the main thread's execution.
- Which functions actually *are* re-entrant?
	`strtok_r` stores the state in the arguments passed in, thus will execute re-entrantly for different sets of arguments; similar arguments apply to many of the `str*` class of `string.h` functions; and, generally, any API that depend only on arguments to compute the results.
	Careful implementation in stateful functions can also be re-entrant, but this requires a lot of consideration and work; most modern libraries don't consider this, and instead require that signal handlers are simple and restricted in scope.

### Trading Complexity for Performance

- Which of these aspects of interface design do we prioritize if we're on a deadline, and can't do it all?
	Depends on the goals of the system.
	You really don't want to bake into the API, designs that support many clients, thus you have to life with the API forever-more.
	If it is a "public" API, you really want to get it right.
	API design is usually worth the design time spent on it.
	You can fix a simplistic or faulty implementation, but backwards compatibility makes you always have to live with API mistakes.
- Should we always choose performance over the "nice-to-have" interface properties?
	No.
	We often design systems and ask "what operations *must* be fast".
	We define a "fast-path" in the system (or multiple), and focus optimizations around that.
	Once we do that, we want to make the API as easy to use as possible in general, and depart from that ease of use to make the fast-path efficient.
	Linux doesn't really have this luxury as the "fast paths" differ across the vast number of applications.
	However, most systems that we implement are much more specialized, and do have the luxury to be more opinionated about what we optimize for.

### Designing for Composability and Orthogonality

- Are there mechanisms to avoid breaking orthogonality?
	Not that I know of.
	That's why we're discussing it in the class: if you're aware of the goals, you can consider them in interface creation.
	If you break then, then you can do so intentionally, and document why.
- Is there a general way to create composable APIs for shared resources?
	Not that I know of.
	A common technique, if we can call it that, is to make the API more complicated to handle changes that span multiple API calls.
	They are conceptually similar to the `cas` atomic instructions:

	1. one of the API functions returns a "token" that records the state of the system,
	2. each subsequent call takes that token, and will make modifications contingent on not conflicting with other parallel modifications.

	The general idea is to "make modifications only if the state is as expected", and this is a composable function that can be tied together into higher-level operations.
	Many web caches provide an API to set a key to a value *only if* the value was previously some specific value.
	This is similar and generalized to [Optimistic Concurrency Control (OCC)](https://en.wikipedia.org/wiki/Optimistic_concurrency_control) in data-bases.

	An additional technique is to provide a "token" as part of the API that specifies which "portion" of the backing state is being modified and queried.
	A good example of this is [session tokens](https://en.wikipedia.org/wiki/Session_(computer_science)) that are used to identify a specific user that is interacting with a webpage over multiple requests (think the "paging" required by the "next" links to query subsequent data items).

	[Software transactional memory](https://en.wikipedia.org/wiki/Software_transactional_memory) (STM) in Haskell/Clojure implements APIs essentially based on the "make modifications only if state is as expected", and are able to hide the complexity of the API behind a coherent library abstraction.
	They both rely on the functional behavior of the underlying system to make this practical, thus this technique is not generally applicable to systems -- that are *not* functional (e.g. stateless or pure).

	However, these are examples.
	I don't know of any general -- or prescribed -- techniques to create composable APIs.
- Is the API composable if some of the functions, or the resources in state, can compose, but others cannot?
	Within a given API, the composability constraints can be documented.
	We often define a protocol (or state-machine) which can define the state of the resources being modified, and only specific functions can apply to resources in specific states.
	There is an inherent protocol in networking connection creation that flows through `socket`, `bind`, `listen`, `accept`, and `read`.
	These functions are *not* arbitrarily composable, instead they can only operate on a resource (the socket) when it is in a specific state (as defined by the previous functions used to operate on it).

	That said, we often worry more about the composability of different APIs together.
	Does `fork` compose with `pthread`s?
	Do `pthread`s compose with `errno`?
	Do signals compose with blocking ("long") system calls?
	No, no, and no.
	We must focus not only on "self composability" (and document constraints on that), but also on the composability of different, potentially "interfering" with each -- demonstrating a lack of composability.

### Commutativity

- How can `open` not commute -- two calls, regardless the order seem identical for all functionality that matters, right?
	For a user's perspective, it *feels* like they are identical.
	From a shell's perspective, they are *not*.
	For example, `sh` in `xv6` *requires* that the order of opens translates into *specific* file descriptors being allocated (see the [careful dance](https://github.com/gwu-cs-os/gwu-xv6/blob/master/sh.c#L100-L112) with `close` and `dup` when creating pipes).
	More importantly, the *specification* of how *open* and *dup* behave is what we assess when judging commutativity.
	Human perceived importance of various behaviors in APIs are not very relevant, only the specification.
	POSIX's *specification* requiring allocation of the lowest-free file descriptor ensures that two `open` (or `dup`) calls do *not* commute.
	This means that an implementation that adheres to this specification cannot be scalable.
	This hopefully emphasizes why the *specification*, not a *sense* about what part of APIs are important, is the relevant detail to focus on.

### Moar Interfaces, MOAR

- With respect to liveness: if a user opens a file, thus in some sense owns the reference to the file, and is responsible for `close`ing it, what happens if the user doesn't?
	The reference will stick around!!!
	When a process `exit`s, all of its referenced are cleaned up, and at that point the file will be released (and potentially freed if all other references are removed from it).
	There is a very deep, worrisome problem here: we are using up kernel resources (memory and potentially disk) to store the file.
	Are we "charging" (i.e. *accounting* for) those resources properly?
	Are they counted against the total amount of memory/disk that the process/user is using?
	This is a very difficult problem to solve.
	To make proper resource *allocation* decisions, we often want to consider how much of a resource some user/process is already using, and allocate more to those that are using fewer (as an example policy).
	Without proper accounting, a potential attack is opened up where the user can use many more resources than it should be able to by consuming service's resources on their behalf.
	This is made all the more complicated when those resources are "shared" between processes.
	Do we proportionally "charge" each process?
	If so, one process removing a reference will *increase* the amount that others are charged.
	This is odd: your resource consumption accounting goes *up* through no action of your own.
	Hard stuff.
- Do we ever remove API functions in systems -- for example, system calls in Linux?
	Not really.
	Certainly Linux doesn't remove system calls.
	So long as there are users of the API, to remove it you have to either 1. be willing to break existing applications (see Python's transition to 3.0), or 2. provide a reasonable transition plan to support the old functions in libraries (e.g. support the old APIs in a library by using underlying composable APIs of the other functions of the interface).

## C2: A Study in Event Models: Demikernel and `libuv`

### Demikernel Questions

- Why have all of the complexity of merge/map/filter/sort?
	These are convenient APIs that are meant to generate a higher-level programming API encouraging varies forms of composition.
	As this is research, it absolutely is *not clear* if this complexity pays off.
	We'll have to look at a variety of applications to judge this.
- What is a scatter/gather array?
	Instead of passing in an array of data, you pass in an array of references to separate arrays of data.
	If you're "gathering" you asking for data to be sent out from one of these arrays of references by accessing each of the separate data arrays, in order.
	If you're "scattering", you're asking to receive data that will be split across all of the subsequently referenced data arrays.
	See `writev` and `readv` for more documentation.
- Why are there so many different types of queues?
	DK is using queues polymorphically here, so we can "speak" the queue API, and it can talk to disks, networks, or pipes, depending on the backing resource.
	This is similar to the VFS API in UNIX.
- Does kernel-bypass bring more security issues?
	Yes.
	As there is no isolation between I/O subsystems, and the application, so a bug in either will impact both.
	That said, there is effectively a single application executing on the system, so at least such a bug will not span the data of multiple principals.
- Can DK work with multiple applications?
	Unlikely.
	They would generally have to be in the same address space, and the "event loop" of DK would have to change to enable one application to block while another computes.
	These goals are challenging when using kernel bypass.
- Is avoiding blocking really that beneficial?
	Blocking means that you have the overhead of interrupts, and mode transitions, which *are too much* in many data-center operations.
	Polling avoids all of those overheads and enables the application to get the lowest-latency access to data at the lowest overhead.
	The downside is that we're burning CPU cycles on polling, and preventing other applications for executing.
- Instead of doing kernel bypass, could the kernel just export queues to user-level; would it still be fast?
	Yes!
	See the work by Jens Axboe on [`io_ring`](https://lwn.net/Articles/776703/) support that has [matured](https://lwn.net/Articles/810414/) quite a bit to do exactly this.
	Writing a DK backend to `io_ring`s wouldn't be that challenging.
	There would be a performance hit, for sure.
	Additional overheads would include interrupts (that DK avoids using DPDK), copying data across user/kernel boundary, and mode transitions.
	The latter would be minimized, most likely due to buffering (remember "data aggregation" as an optimization technique).
- Generally, do we really get much better performance by doing kernel bypass?
	If you're doing fast computations in response to packets, **yes**.
	Networking interfaces can push data fast enough that at a maximum packet rate, you don't have enough time to service a single cache miss.
	100Gb/s is an extraordinary data rate that is twice the throughput of the fastest current DRAM (see DDR5 [here](https://en.wikipedia.org/wiki/Double_data_rate#Relation_of_bandwidth_and_frequency))!
	It is necessary to avoid mode transition and interrupt costs for such systems.

	If your application does a fair amount of computation for a request (e.g. a webserver), then you'll get diminishing returns for saving the overheads of interrupts and mode transitions.
	Consequently, the benefit is quite application-specific.
	One domain that clearly benefits: network function virtualization -- where we push many of the router/smart network functionalities into software.

### libuv Questions

- Why all the wrappers around functions like thread operations?
	Portability across systems.
	You want to define a common API that doesn't have too large a semantic gap to the APIs exposed by all OSes you want to support.
- Documentation specifies that you can run an event loop per thread.
	Why?
	Each event loop often implies blocking waiting for one of the events that has been registered.
	If you're blocking waiting in "one of the event loops", you might never wake up.
	If you don't when are you going to handle events from the "other loop"?
	Really the more fundamental issue is that you have two different sets of resources that are separately blocked on.
- Why does libuv support two separate callbacks for `uv_read_cb` and `uv_udp_recv_cb`?
	Likely because UDP is a datagram-based protocol, so each packet has "framing".
	Each packet is meant to received a single unit, rather than as part of a larger stream of all packets (as in TCP).
	Thus two callbacks: the latter understands that the data passed in is a single packet.
	If this hypothesis is correct, then it would be better named `uv_dgram_recv_cb` (note the `DGRAM` option for sockets that motivates the shortening of the word).

## L3: Composite Resources

- Does a component in which a thread executes have to have access to the thread's capability?
	No.
	A sane system will only let the scheduler have access to capabilities to threads; only the scheduler should have thread dispatching privileges!
- How can scheduling work when it is more of a free-for-all of context switches?
	Most components should not have access to the thread capabilities.
	Only the scheduler should.
	Thus the only context switches derive from the scheduler's context-switches (on thread capabilities), interrupt activations of threads on asynchronous activation end-points, and timer interrupts firing and switching back to the scheduler thread.

	The key thing to understand is this:
	we've discussed the *mechanisms* for access control (capability-based access), but *still must discuss the policy*.
	For example, a necessary policy to have sanity in scheduler implementation is that the scheduler is *the only component with capabilities to the threads it schedules*, and to have all asynchronous activation end-points for the threads it schedules have the "parent" set to the scheduler's thread's end-point.
	The mechanisms are not alone sufficient to implement a sane system.
	Abstractions and organization must be *layered on top of the raw mechanisms* to derive predictable behavior.
- It seems like we have a comparable number of resources now exposed to user-level rather than raw hardware mechanisms; have we made progress?
	Is this better?
	Yes, it is better.
	Once we have provided the ability to harness and guide the raw hardware resources with *user-level, safe interfaces*, we have now enabled:

	1. multiple user-level components to define how to use and orchestrate the resources in different ways;
	2. system specialization by providing developers with the tools to manage resources in application-specific ways;
	3. increased isolation in make the system more fault resilient and secure.

- Do synchronous invocations avoid the kernel?
	No!
	They end up making effectively two system calls, thus go through the kernel twice -- once for invocation, and once for return.
- How could you support virtualization on this system?
	See the Nova system we'll go over quite a bit in this class.
- How is isolation provided with synchronous invocations?
	What if the thread crashes or goes into an infinite loop in the server?
	The execution in the server cannot access the memory of the client.
	A failure in the server does not necessarily cause a crash in the client, and vice-versa a client crash.
	If there is a fault (or a perceived fault) in the server, it is possible to implement an exception model in which a "return" happens to the client, and executes and error path (e.g. a `catch` block).

	Note that if the server goes into an infinite loop, the client will be impacted.
	However, this is a limitation of *all synchronous APIs*.

	There are other complications.
	If we make a synchronous call while holding a lock in the client, the client is opening itself up to failures.
	Such an action is tying the progress of the threads in the client, to progress in the server.
	As such, critical sections should really not make synchronous invocations, unless you're willing to tie their progress together.
- How does any of this work on multiple cores?
	The scheduler thread must now consider multiple cores?
	Synchronous invocations into the same component on multiple cores seems complicated?
	There is a scheduler thread per-core, and schedulers can only switch to threads that are bound to that core.
	There are other APIs to change which core a thread is bound to.
	Scheduler threads can coordinate through shared memory, or by using inter-processor interrupts.
	This situation is quite similar to how Linux implements its scheduling queues.

	Synchronous invocations imply that servers must be concurrent and parallel.
	They must use locks and consider races at all times.
	This is a downside of synchronous invocations.
- The kernel should not trust user-level but if you move the scheduler to user-level it must now trust it for proper scheduling; how can this be safe?
	Microkernels general export most abstractions to user-level.
	If a user-level file system doesn't work, then those applications the depend on it might experience faulty service.
	This system is simply treating scheduling the same way -- put it in user-level and any service that depends on it might fail if the scheduler fails.
	A big difference with the FS analogy is that not all components must depend on the FS, but all of them must depend on a scheduler.
	Not discussed here is a mechanism (Temporal Capabilities -- i.e. capabilities for time) that enables there to be multiple schedulers in the system, and a fault in one only imples failures in the components the depend in it, but the other components can keep on running uninhibited.
- Why do synchronous invocations, and why not just have a server thread that is communicated with?
	We'll talk about this for about two weeks, but for now you can watch the [gory-level-of-details-preview](https://www.youtube.com/watch?v=_qrP8jQ-eoc).

## C3: Composite Runtime
