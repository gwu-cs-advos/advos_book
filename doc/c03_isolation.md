# OS Design and Isolation

How do the previously discussed software engineering concepts apply to systems?
Now that we understand the design goals and constraints of systems, we'll move on to understanding how OSes apply these designs to hardware.
At the highest level, OSes provide the following major functions:

1. *sharing* of hardware between different principals,
2. providing *functionality, abstractions, and services* for principals to harness, and
3. ensuring *protection* between principals.

## Definitions and Background Concepts

First, lets go over some high-level, necessary background concepts.
We've discussed how systems, from a software engineering perspective, must consider modularity and abstraction, and be careful in the interfaces that they provide to manipulate their resources.
Now we're going to pivot into the question of **what this means for system construction**.
We've used a lot of terminology and referenced a lot of ideas, but now we're going to be a little more concrete.
First, some definitions:

1. *Resources* are the means by which we can access and modify the system.
2. An *abstraction* is a set of resources and set of operations defined by an *interface* (which might include not just functions, but also `load`/`store`, etc...).
3. A *module* is a self-contained body of software that uses encapsulation to hide its implementation, while exporting an abstraction with which other modules can harness its functionality.
4. We describe the module that provides an abstraction to another through an interface as the *server* and the one that harnesses the interface as the *client*.
5. A *principal* is the part of the system over which *trust* relationships are parameterized.
	Does module $A$ trust module $B$ to not be faulty? to not be malicous? to not have any compromises? to use only its share of resources?
	If the answer to any of these is "no", then $A$ and $B$ should be treated as separate principals, and should be mutually isolated -- thus motivating inter-principal isolation.
	Note that different application modules rarely trust each other to be bug-free (thus the "process" abstraction in monolithic systems).
6. A *service* is a module that provides an abstraction to potentially *multiple* other modules.
	If the client modules are for separate principals, then to maintain inter-principal isolation, the service must be isolated from the clients.
7. A *library* is a module that executes on behalf of the same principal as a module, and that provides an abstraction to that single module.
	Thus the library module is loaded directly into the client module's protection domain.
8. An *application* is a module that does *not* provide abstractions (interfaces) to other modules, and is associated with some principal.
	It defines the policy that composes all of the abstractions it has access to from other modules (services and libraries).
9. A *component* is a module that makes the exported interface, and its *dependencies* on other component interfaces explicit.
	This enables a "component-based design" development style that involves putting together a jig-saw puzzle of interface exports and dependencies.
10. A *protection domain* is the subset of resources that are accessible and/or modifiable to the modules within the protection domain.
	Protection domains form the foundation of isolation in the system.
11. A *process* is a set of modules in a protection domain.
12. The *access control mechanism* of the system ensures modules can only access resources within their protection domain.
13. The *access control policies* control how the protection domains of the system are formed, and can change.
	This enables resources to not be shared in a manner that could weaken the security properties of the system (e.g. passwords being accessed by another user).
14. If client protection domains compose server resources into higher-level abstractions, they might require passing the server's resources to their respective clients.
	For example, the pages underlying files might be derived from a device driver module, and be passed to a file system; at that point the FS might want to share the page with an application (due to a `mmap` call).
	This requires a mechanism to pass resources between modules called *delegation*.
	When delegating access to a resource, the ability to further *delegate* the resource enables even the client to manage the resource (and pass it out to its clients).
15. If resources can be delegated between modules, then the converse operation, *revocation* must be supported.
	*Revocation* removes previously delegated access to a resource from a client.
	If a delegated resource was delegated with the ability to delegate to other modules, a *tree* of delegations in created, and revocation must recursively revoke the entire subtree of delegations.
16. The kernel is special in that it is a set of modules that have access to all system resources as provided by the (potentially virtual) hardware.
	Thus, the kernel *must* define at least a subset of the access control mechanisms (e.g. page-tables), and must abstract the raw hardware mechanisms.

In monolithic systems, the various services (modules) are compiled together, and resource sharing is encoded in the net of pointers that constitutes the kernel's data-structures.
Revocation is much more of a communal process in which the different modules orchestrate through `free` APIs and reference counts to release resources.
However, in micro-kernels, the structure of the protection domains can mimic that of modules.
This raises the questions of how *delegation* and *revocation* can be supported, but also if this the benefit is worth the additional complexity.
How can we evaluate which structure is a good fit for our system's requirements?

One important aspect of system design is understanding the risk of erroneous behavior in each service's protection domain.
If a service misbehaves (i.e. does not behave in accordance with specification and expected behavior), it will impact all of its clients, and potentially all of their clients, and so on.
What increases the risk of this happening?

- Code that has higher inherent complexity (e.g kernels) is more difficult to get right.
- Code not written by seasoned developers is more likely to have latent issues.
- A larger body of code (many modules, or large modules) has a higher likelihood of having bugs.
	History has shown that even highly-trained kernel developers have trouble keeping the massive body of code underlying modern monolithic kernels error free and reliable.
- The list goes on and on.

A healthy sense of humbleness might draw one to the conclusion that it is better to have many protection domains to minimize the scope of impact of any errant behaviors.
However, this can increase the complexity of the system as the access control mechanisms, and the delegation and the revocation mechanisms become quite central to system design.

## Atomic Elements of Hardware: Resources

As we construct higher and higher-level resources, we must be cognizant of the lowest-level resources provided by the hardware.
They are the starting point as we construct the rest of the system.

Being concrete about resources, we can start by talking about those at the lowest-level:

- cores and processing allocation over time

	- each core's power level (frequency and voltage scaling)
	- Inter-processor interrupts (IPIs) for inter-core event notification
	- Timers and timer interrupts

- Physical frames of memory
- Interrupt lines

	- Associating interrupt lines with devices (PIC programming)

- Devices and busses

	- PCIe/USB/SPI/I$^2$C/... mapping and communication
	- virtual access to specific physical addresses -- e.g. through memory-mapped registers
	- DMA transfers

- Exception handlers

	- page-faults, divide by zero, invalid instruction, etc...

- Protection

	- Page-tables and MMU programming
	- Dual mode hardware and protection (sensitive/privileged instructions, kernel vs. user-level memory, and mode transitions -- [system calls](https://www.youtube.com/watch?v=xHu7qI1gDPA&list=TLPQMjIwNTIwMjA1MAN_H1P-DA))
	- I/O MMUs and page-table association with devices
	- Hardware virtualization acceleration

In the end, we're mapping these resources up to the higher-level resources we're accustomed to.
These include (but are, of course, not limited to).

- processing: threads, processes
- persistence and resource naming: files and the hierarchical namespace
- communication: network sockets, IPC (e.g. pipes), shared memory
- protection: processes, containers, hierarchical namespace access rights

## Discussion: General-Purpose vs. Specialized Systems

Before we dive into a set of OS designs, I want to understand how engineering constraints complicate system design.
Engineering is fundamentally about applying finite resources to solve a specific set of goals.
When *designing* a system, it is important to understand the goals.
Engineering decisions will make some goals impossible, and will add friction to achieve others.
Specialized systems typically have very refined and specific goals and are often accepting of extreme designs, whereas general purpose systems try to be the jack of all trades.

In more general-purpose systems, a huge amount of engineering goes in to optimizing the system.
The analogy for this is in the increased fuel savings we see in passenger jets.
An immense amount of engineering went into the [737 Dreamliner](https://en.wikipedia.org/wiki/Boeing_787_Dreamliner) with a primary goal of fuel consumption reduction to reduce the total cost of ownership.
Analogously, the Linux kernel and the OS service above have been re-architected multiple times to better accommodate the general goals of efficiency and flexibility.
We'll discuss the move from `initrc`-based system boot and service control systems to `systemd` as one example.
These approaches do not fundamentally redesign the core structure of the system, yet they can achieve impressive gains through careful engineering.

Specialized systems are often quite important for pushing the boundaries on cost and capability.
Two examples, and some computational analogies:

- The [Lockheed U-2](https://en.wikipedia.org/wiki/Lockheed_U-2) spy plane is quite unique.
	It has a jet engine, yet glider-like wings.
	It has around a 70,000 feet altitude ceiling^[Mt. Everest's peak is at 29,000 feet, and normal commercial airlines typically cruise between 31,000 and 38,000 feet.].
	Systems start to become quite specialized when they require those altitudes.
	The density of air is only around 5.8% that at sea-level, whereas air density for commercial airlines (at 35,000 feet) is around 31% that at sea-level.
	As lift requires air density, this means that every pound saved in the aircraft results in a [10-foot increase in altitude ceiling](http://www.barryschiff.com/high_flight.htm).
	Demonstrating how specialized this airplane is, it only has two landing gear wheels to minimize the required weight.
	As you might imagine, airplanes don't do strong bicycle impressions.
	When taking off, two additional wheels are attached to the wings that simply fall of when the plane takes off.
	To land, the pilot would carefully balance the plane until humans could grab the wings to prevent them from hitting the ground.

	The aggressive specialization to remove and innovate away a conventional design (e.g. landing gear) has analogues in systems.
	Embedded systems must be comparably specialized around providing predictable response times while using a fixed amount of memory reliably.
	As we saw before, Misra C and JPL standards specify that there be no dynamically allocated memory in the system (past boot time).
	This is not a useful policy for many computing systems, but helps reach the extremes in memory consumption and predictability required for these small systems.
	A similar example: The Composite kernel can [execute VMs](https://www2.seas.gwu.edu/~gparmer/publications/rtas18mpu.pdf) on microcontrollers with only 128KiB of memory by statically layout out all VMs memory to make optimal use of the MPU-based memory isolation.
	All of these examples show the benefits in being able to *scale down* by adding unconventional simplicity.
- The [SR-71 Blackbird](https://en.wikipedia.org/wiki/Lockheed_SR-71_Blackbird) is another spy airplane that is an impressive feat of engineering.
	Like the U-2, it was created to go to extreme altitudes, but also to be extremely fast so as to effectively outrun ground-to-air missles.
	It didn't have large glider wings to compensate for low air density, and instead had turbojet engines that could convert into ramjets (using only the pressure of air to drive the controlled explosion) to push it up to Mach 3.2.
	As such it could ascend up to 85,000 feet.
	It also was concerned with weight, thus it had no internal fuel blatter, instead relying on the fuselage panels to hold in the fuel.
	Famously, it would leak fuel on the runway as the fuselage panels fit loosely until the air friction at high speeds would heat them up (and expanded them) to make a tight fit.
	In this way, much of the plane was constructed to withstand high temperatures: the leading tips of the SR-71 experienced enough friction to heat them up to 300 degress C, and jet fuel was chosen that had a high flash point so that it wouldn't spontaneously combust.
	The aerodynamics of Mach 3.2 made maintaining lift at low speeds difficult, thus it would take off with only enough fuel (thus weight) to immediately rendezvous with a refueling tanker, and would land at 200 miles per hour using a parachute to slow itself down on reasonably lengthed runways.
	Like the U-2, the SR-71 required unconventional trade-offs that would not be tenable in more general-purpose airplanes.

	High-performance computer systems are also engineered in such a way that they cast off traditional designs.
	For example, [DPDK](https://www.dpdk.org/) enables the user-level, direct access to networking devices (called "kernel-bypass").
	This casts off the traditional roles of hardware multiplexing, and multi-application access to hardware that underlie traditional kernel design.
	Taken a step further, more logic is being pushed into the networking cards so that they can directly reply to requests (see [RDMA](https://en.wikipedia.org/wiki/Remote_direct_memory_access)).
	This breaks many of the "layering" abstractions that are the core of existing systems.
	Another example: to provide lightweight protection domains that are quickly created and destroyed, specialized [approaches](https://www2.seas.gwu.edu/~gparmer/publications/atc20edgeos.pdf) are [required](https://www2.seas.gwu.edu/~gparmer/publications/middleware20sledge.pdf).

General purpose systems like Linux do a fantastic job at applying well enough to many domains.
However, at the low-end when Size, Weight, and Power (SWaP) and cost are concerns, Linux is challenging.
At the high-end, getting the most out of every single cycle often requires bypassing OS services.
However, across all sizes, security tends to be a concern that requires simplicity and is incompatible with monolithic structures.
So when we think about OS design, it is useful to understand the goals.

## Example OS Design: Composite

This chapter on OS design motivates the need to consider

1. how we methodically construct a hierarchy of services that provide subsequently higher- and higher-level abstractions by composing the mechanisms of those below it;
2. how can the kernel's abstractions and resources avoid creating a significant semantic gap for specialized systems;
3. how we can provide protection domains at a relatively fine-granularity to minimize the impact of errant or compromised service logic; and
4. how delegation and revocation can be effectively implemented for the lowest-level hardware resources to be utilized throughout the system?

This motivates a micro-kernel-style system design.
A few direct consequences of these considerations:

- services isolated in separate protection domains require fast IPC facilities for modules to coordinate in a manner that mimics interface function invocations, and
- the kernel should provide the *lowest-level*, *safe* abstractions to protect access to hardware resources possible to give services the ability to define the higher-level resources.

In this section, we'll lay out a set of low-level abstractions for hardware resources that attempt to enable the necessary safety and security.
Lower-level abstractions are close to the hardware resources, but the challenge is how to both provide low-level access, while enabling strong access control.

### Why microkernels?

Lets take a quick aside to justify why we're going to be talking so much about microkernels.
Microkernels are generally used in domains that require heightened trust in the computational infrastructure.
If things go wrong, then there is a loss of equipment or life, or there is a threat to the security of the important information.
A few brief examples:

- Microkernels are the core of many autonomous vehicles through the QNX microkernel at the core of NVIDIA's "DriveOS".
- It was (is?) the OS in the security co-processors on iPhones to conduct authentication (fingerprint reader), payment processing (iPay), key management, and other sensitive operations.
- Modern Intel chips run [Minix in a Management Engine](https://www.zdnet.com/article/minix-intels-hidden-in-chip-operating-system/) (think of it like a hypervisor's hypervisor), meaning microkernels on servers/desktops/laptops are likely as common as monolithic systems.

What's the common denominator?
Microkernels excel at being specializable, enabling powerful abstractions with little code (thus minimizing the risks of compromise), and, generally, being secure.
For example, seL4 is a *formally verified* OS, which means that we know (via mathematical proof) that it is bug-free, a powerful property.
We tend to simply not know that they are as common as they are simply because they aren't client/user-facing.

Note that many of these domains will use a microkernel along-side a monolithic OS (e.g. Linux, OSX).
We'll look at a microkernel later that acts primarily as a hypervisor.

### Kernel Resources

The [Composite component-based OS](http://composite.seas.gwu.edu) is a micro-kernel and provides a set of these abstractions.
The resources that abstract hardware:

- **Resource tables** act as the access control mechanism that provide protection domains
	Resource tables include page-tables and capability-tables.
	Both are radix tries that take an index, and lookup the corresponding resource associated with that index.
	It might be strange to think about page-tables in this way, but at their core, they are simply enabling lookup for load/store instructions to find the appropriate memory resources.
	I think about page-tables as simply a form of hardware-accelerated resource table lookups (e.g. the hardware page-table walker and TLB simply speed up access control checks).
	In contrast, capability-tables track the abstractions created by the logic of the kernel.
	*Capabilities* are integer handles that are used to uniquely identify a resource tracked in the capability-table.
	The kernel resources constitute this list!
	Note that means that resource tables track resource tables.

	Put abstractly, resource tables provide the access control mechanisms for system resources via:

	- Each protection domain has a page-table that enables memory resource access via $\text{pgtbl}(\text{addr}) \to \text{memory}$.
	- Each protection domain has a capability-table that enables system calls to perform operations on kernel resources via $\text{captbl}(\text{capability}) \to \text{kern\_res<T>}$.

	> *Resource tables* abstract
	>
	> - page-tables (MMU)
	>
	> ...and provide the abstractions of
	>
	> - *all* kernel resource access through capability lookup in capability tables,
	> - including abstract operations to update and maintain page-tables.
	>
	> This enables the necessary modularity required to isolate bodies of software in the system.

- **Components** are a simple collection of a page-table, capability-table (e.g. a protection domain), and an instruction pointer for the entry point for new threads in the component (think like `_start` in Linux programs).

	> *Components* provide the abstraction of
	>
	> - protection domains that track all resources that are possibly accessible to a computation, and
	> - the valid execution entry points for new computations.

- **Typed memory** is the abstraction for the physical memory of the system.
	This can get complicated so I'm only going to cover the highest-level concepts here.
	The core concept is that physical pages can be directly addressed through resource tables, and the operations on them enable them to be used as virtual memory, but also as kernel data-structures.
	The latter is quite unintuitive: it leads to the *user-level management of kernel memory*.
	It will take some build up to understand how this can be safe, so we'll ignore it for now (see "Memory Management" below).

	This is surprisingly versatile.
	Most device drivers use *memory mapped I/O*, which means that we can both support device drivers, and provide access control for them using only^[Ignoring, for now I/O MMU protection.] memory protection facilities via resource tables.

	> *Typed memory* abstracts
	>
	> - the physical memory of the system into logical chunks that can be used either for kernel data-structures or virtual memory, and
	> - device access, programming, and protection (when paired with an I/O MMU)
	>
	> ...to put the memory of the system to useful work.

- **Threads** are the abstraction for sequential processing.
	Each thread executes within a component (starting at the entry point).

	> *Threads* abstract
	>
	> - per-core computation
	> - within a protection domain (thus sub-setting the computation's accessible resources)
	>
	> ...to enable the concurrency of many threads executing on each core.

	One of the main operations on threads is their **asynchronous activation**.
	If a component has a capability to a thread, it can context switch to it.
	In this way, schedulers are implemented at user-level
- **Asynchronous activation end-points** are a resource that a thread is associated with that

	1. threads use to cooperatively suspend themselves, and
	2. is used to attempt to activate the associated thread.

	This is also used to abstract interrupts.
	An interrupt service routine (in the kernel) can be configured to perform one of these activations to the associated thread.
	Hardware interrupt semantics share the stack of the currently executing thread and are hard to get right (see "re-entrancy" discussions in previous sections).
	Thus asynchronous activation end-points abstract these challenging interrupt semantics into simple thread activation.
	These "interrupt threads" are used to effectively handle interrupts while executing in a user-level component.

	> *Asynchronous activation end-points* abstract
	>
	> - inter-computation coordination (IPIs at the hardware level)
	>
	> ...to enable concurrency via many threads.

- A **Virtual Interrupt Controller** (VIC) that abstracts the actual interrupt sources.
	Hardware provides a fixed, small (e.g. 32) number of interrupt lines that devices can be attached to.
	Those devices will ([setup](https://github.com/gwu-cs-os/gwu-xv6/blob/master/trap.c#L22) and) [trigger](https://github.com/gwu-cs-os/gwu-xv6/blob/master/trap.c#L36) ISR execution.
	Composite enables components that have access to the VIC to "attach" to an asynchronous activation end-point to a specific interrupt line, thus causing the ISR to activate (see "asynchronous activation" above) the corresponding thread.

	> The *VIC* abstracts
	>
	> - system device interrupts
	>
	> ...into component, threaded execution in response to device actions.

- **Synchronous invocation call-gates** that enable inter-component communication^[We'll refer to this as IPC for consistency reasons].
	This is the main mechanism used to communicate between components.
	Holding a capability to one of these call-gates enables a thread in a protection domain to *invoke* a function in another.
	A thread the "invokes" one of these call-gates will resume its execution *inside the server component* at an instruction pointer associated with the call-gate.
	After computing in that component, it can then "return" at which point it resumes execution where it left off in the client.

	As the same schedulable thread executes in *both* the client and server, this IPC uses "thread migration", as opposed to the more conventional rendezvous between separate threads, one per component.
	Regardless, it still maintains memory isolation between protection domains as invocation and return are mediated by the kernel, and they switch protection domains.
	As such, execution in the client and server has to use separate, isolated C execution stacks.

	> *Synchronous invocation call-gates* abstract
	>
	> - system calls
	>
	> ...to enable cross-component calls, thus composition across multiple coordinating modules to create component-defined, higher-level abstractions while separating mechanism and policy.

- **Exception invocation call-gates** are similar to synchronous invocations, but are triggered due to processor exceptions.
	Processor exceptions trigger a constant and specific capability that should hold the corresponding exception invocation call-gate.
	These include page-faults, general protection faults (e.g. accessing kernel memory from user-level), divide-by-zero, illegal instruction, and other such operations for which the CPU does not have a valid response.
	In such cases, a full register set is saved in the thread in which the exception triggered, and the exception handler function activated in the server component.
	This means that the handling of such exceptions is exported to components, strengthening their ability to provide high-level abstractions, for example, copy-on-write fork, and demand paging.

	> *Exception invocation call-gates* abstract
	>
	> - CPU exceptions
	>
	> ...so that they can be handled at in user-level components, thus promoting modularity, and the component-definition of mechanisms.

### Scheduling

Surprisingly, with this set of abstractions alone, you do not need a scheduler in the kernel.
Instead you can implement scheduling logic in a user-level component.
This uses capabilities to threads to context switch to threads.
Schedulers, then, are components that have references to threads in their capability-table.

If the scheduler activates a capability referencing a thread, it will context switch to that thread, saving the state of the current thread, and restoring that of the target thread.
If the thread being context switched was previously preempted (by an interrupt), when switching to it, we might switch components (thus page-tables) as well.
This fact gives scheduler components the power to not just schedule the threads that are executing in their own component, but threads that are executing across the system.

Thread-migration-based IPC is an abstraction that composes well with scheduler component ability to context switch between threads.
Should a client component want to leverage a scheduler's logic to block and later wakeup a thread, the scheduler component must only:

- export an interface including functions to block and wakeup a thread, and
- implement policy that will track when a thread is blocked, and not switch to it in such cases, and when it is runnable.

A client thread can call the `sched_block` function, which (via synchronous call-gate invocation) executes the function in the scheduler component, thus blocking the thread until later another thread calls `sched_wakeup`.
At that point, the scheduler might (depending on its scheduling policy) context switch to the now woken thread, or might continue executing in the thread that woke it.

Up till now, we have the mechanisms necessary for *cooperative* (non-preemptive) scheduling.
To graduate to preemptive scheduling, we must somehow provide an abstraction for hardware timer interrupts, and to detail how asynchronous end-point activations interact with scheduler components.

- *Coordinating between asynchronous activations and schedulers.*
	A *scheduler thread* is a thread in the scheduler component that is associated with an asynchronous activation end-point.
	Whenever an asynchronous activation end-point is created, we must specify its "parent", and provide the scheduler thread's end-point.
	The scheduler thread will be send notifications (in a queue tracked with the end-point) when threads are activated and when they suspend themselves on their own end-points.
	This enables the thread in the scheduler component to process thread activations in response to, for example, interrupts.
	The scheduler uses these notifications to correspondingly track which threads are blocked, and which are runnable.
	Note that *any thread* in the scheduler component (thus executing trusted code of the scheduler component itself) can make scheduling decisions (e.g. in response to the thread asking to block itself), not just scheduler threads.
- *Abstracting timer interrupts.*
	An argument passed to the capability operation to switch to another thread is a timeout into the future.
	Before the thread is context switched to, the hardware timer (LAPIC) is programmed.
	Thus, schedulers can control when the next timer interrupt will fire.
	When the timer interrupt is triggered, it will simply switch to the corresponding scheduler thread, thus enabling it to make a scheduling decision based on the passage of time.

There are some important details that I'm omitting here for simplicity.
For example:

- How do we make priority-based decisions about if we should execute a thread activated by an interrupt, or continue executing the pre-interrupt thread?
- How do we orchestrate the management of a CPU across *multiple* component schedulers?

Access control for time through [temporal capabilities](https://www2.seas.gwu.edu/~gparmer/publications/rtss17tcaps.pdf) (or Tcaps) was invented to solve these challenges.
For coordination between different schedulers, Tcaps attempt to make the following guarantees:

- Each scheduler can maintain its own definition of inter-thread priority (e.g. fixed priority scheduler versus "nice"-based UNIX scheduling versus dynamic priority systems like Earliest-Deadline First).
- Each scheduler is allocated spans of time that they can expend on thread execution, or delegate sub-spans to other schedulers.
- Schedulers only have time to use for thread execution if they are delegated that time.
- A scheduler, when delegating a span of time to another scheduler, strictly limits the interference for that span of time.

In this context, *interference* has two aspects:

1. the ability of a scheduler to *preempt* another^[Technically, this should read "the ability of a thread associated with a scheduler thread or an asynchronous activation end-point associated with that scheduler is able to preempt a similar thread associated with another scheduler", but I'm simplifying the wording here.] scheduler, thus *delay* its execution, and
2. the amount of execution (i.e. the span of execution) a scheduler can expend before a timer forces a switch to another scheduler.

For details, see the [RTSS paper](https://www2.seas.gwu.edu/~gparmer/publications/rtss17tcaps.pdf).


### Capability-based Protection and Delegation and Revocation

We've discussed how resource tables define a protection domain for a component by limiting the scope of resource access to only those accessible within the tables.
This is a specific implementation of Capability-Based Access Control (CBAC).
The **only** *namespaces* used to reference resources are the *capabilites* used to index capability-tables, and *addresses* used to index page-tables.
This leads to a very *simple* system on which we construct more complex namespaces and higher-level resources through component-provided abstractions.

However, we must answer how components can provide access of their resources to clients -- delegation, and how to later remove that access -- revocation.
Composite focuses on the simplest possible solution to this problem, on which components can implement their own policies.
We enable the *modification* of resource tables *only if* a component has a capability to a resource table!
A component's resource tables are simply used to look up resources on which to perform operations, and a class of those resources are the resource tables themselves.
If we have a capability to a resource table, we can perform a number of general capability operations:

- The *copy* operation duplicates a capability into a new capability id which effectively creates another reference to the underlying resource.
- The *remove* operation removes a capability, though not the underlying resource.

Copy is used by a server component that has capabilities to the resource tables of two other components to implement *delegate*.
One component can ask to transfer a reference to its resource (e.g. a page of virtual memory) to another, and the server can *copy* the reference to the page into the page-table of the other component, while tracking the delegation (tree) in its own data-structures.
Later, if the first component wishes to revoke access to that page, it will invoke the server which will *remove* the reference in the other component (and all other recursive delegations made by that component, and tracked in the server's data-structures).
In this case, the server is implementing and defining the delegation and revocation logic of the system.
We [have implemented](https://github.com/gwsystems/composite/blob/loader/src/components/implementation/capmgr/simple/capmgr.c) a policy for this that is much simpler than a tree, and is appropriate for systems with limited sharing.

When a component has a capability to a resource table, it also designates the ability to perform a number of operations on the resource tables themselves:

- *Construct* is used to take a resource trie node for level $n$ (remember they are radix tries) and hook into it a node at level $n+1$.
	This is the *only way* that resource tables are constructed, and it is driven by logic in the user-level components.
- *Deconstruct* will remove the pointer to a node at level $n+1$ from a node at level $n$.
	This is the mechanism to help tearing down resource tables (e.g. because a component terminated).
- *Activate<T>* will active one of the other resource types, *T* to be referenced at a specific capability index (e.g. a thread, or a resource table node).
	This is the main mechanism to create kernel resources.
	Often to activate a resource, additional memory is required, in which case a reference to kernel-typed memory is provided (see next section on "Composite Memory Management").
- *Deactivate* performs the converse operation and destroys a kernel resource (e.g. a thread, or a resource table node).

These six operations, and the memory retyping to be discussed in the next section, constitute the only means to modify kernel data-structures.

### Memory Management

One of the most conceptually challenging aspects of the Composite kernel API has to do with the *typing of memory* which is strongly inspired by [similar support in sel4](http://sigops.org/s/conferences/sosp/2013/papers/p133-elphinstone.pdf).
Memory pages in page-tables can be of three different *types*: **UnTyped Memory** (UTM), **Kernel Memory** (KM), or **User Virtual Memory** (UVM).
The goal of typing memory is to guarantee that each page of memory can be *either* accessible through direct `load`/`store` instructions from user-level *or* as a kernel data-structure, but *never as both*.
This protects the integrity of kernel memory.
Thus, the retype operation will fail, for example, if there shared memory mappings of virtual memory, and memory is retyped into UTM.
The *retype* kernel operation -- that operates only on resources accessible in a component's page-tables -- only allows transitions from KM $\to$ UTM, UVM $\to$ UTM,$UTM $\to$ UVM, and UTM $\to$ KM.
UVM is mapped into page-tables for component access.
KM is used when kernel data-structures are *activate*d.
KM must be provided to those operations to back the kernel structure (e.g. to back a thread's in-kernel memory).

### System Initialization

At boot time, the kernel creates a single component, the initial [constructor component](https://github.com/gwsystems/composite/blob/loader/src/components/implementation/no_interface/llbooter/llbooter.c).
The resource tables of that component are [statically configured](https://github.com/gwsystems/composite/blob/loader/src/kernel/include/shared/cos_types.h#L243-L268) to have references to its *own capability-table and page-table* so that it can modify itself, and create other components.
All untyped memory is provided in a separate page-table at a [constant offset](https://github.com/gwsystems/composite/blob/loader/src/kernel/include/shared/cos_types.h#L251) in its capability-table.
This UTM is used as the building blocks to create other components and all kernel data-structures.

### Abstraction Summary

So far, I've presented a relatively minimal set of resources that are used to abstract all of the hardware resources.
These resources are "close" in abstraction level to the hardware abstractions, but deviate in significant ways (e.g. interrupts are abstracted into threads) to enable more reasonable, safer APIs.
A valuable perspective on these resources is that they abstract the raw hardware operations into three broad areas:

1. *Memory* - how do we enable physical memory to be put to work either as kernel data-structures, or as component's virtual memory?
	Untyped memory enables component management of system memory.
2. *Protection* - how do we create protection domains to enable access only to a subset of system resources, and control the instruction entry pointers for threads into them to provide a component abstraction?
	Resource tables limit the extent of accessible resources for each component, and they provide limited access to be able to define delegation and revocation policies.
3. *Control flow* - how do we provide concurrency and parallelism in the system, and multiplex between principals.
	Threads, asynchronous end-points, and synchronous call-gates enable components to control CPU allocations, and enable component-based exporting of resources to other

## Component-based Design on Composite

We've discussed the system's resources, and many of their core operations, but many questions remain.
The system's core mechanisms, alone, don't ensure some notion of system safety.

> The goal of the kernel's resource abstractions is to provide the sufficient semantics to be able to
>
> 1. construct systems that provide the safety and performance requirements of a specific system, while
> 2. using capability-based access control to limit the scope of access to those resources.

This means that we must also consider *how to create the components, and the access rights for each of those components* in the system to provide some notion of safety.
An example that clearly demonstrates the need for component-based organization of resources is the use of thread capabilities and scheduling.
If thread capabilities are spread throughout the system, how could a scheduler know which thread is execution at any point in time?
This implies that we need to be careful about which components have access to thread capabilities.
For a component scheduler to accurately control thread execution, it must be *the only component* with a capability to each of its threads^[We'll see this is not strictly true. If we are using [hierarchical scheduling](https://www2.seas.gwu.edu/~gparmer/publications/rtas11_hires.pdf) which creates scheduler relationships similar to virtual machines (i.e. a scheduler-per-VM, and also a system scheduler)].

Put simply, the kernel's mechanisms require a *policy* defined by the higher-level abstractions to use them safely (and, indeed, to define what "safe" means).
This re-emphasizes the importance of the system being able to *compose* the kernel's access control mechanisms -- capability-controlled access to resources, including resource-tables -- into higher-level access-control policies such as delegation and revocation, and to control which components can access which resources.
Importantly, it also emphasizes the core tenant of modularity when combined with isolation to provide abstraction:

> If you require some functionality, but don't have access to the resources to provide it, you must use a synchronous invocation call-gate to request the controlled logic of a more privileged component -- that has access to the required resources -- to provide that functionality for us.
> This is the core of abstraction in systems.

This emphasizes the question, "how can we orchestrate ensuring that the proper resources are available only to those components that need them"?
Composite answers this question in two ways:

1. A component-graph specification for which components should be in the system, what interfaces they export, which interfaces they depend on, which clients are dependent on which servers (related by an exported and depended-on interface), and what the component's core function is (scheduler, manager of the resource tables of other components, etc...).
	This enables the *static configuration* of a system as a graph of coordinating components.
	See "System Initialization" above.
2. A system created with the previous static specification can provide all of the functionality to enable the *dynamic* creation of processes, and the abstractions to support them.
	You could, for example, implement POSIX as a set of these components that enables the dynamic behavior of POSIX.

It is perhaps quite surprising, but our focus is almost entirely on the former.
In the cloud, technologies like the firecracker VMM, unikernels, and containers minimization are increasingly moving cloud services toward static configuration.
In the embedded space, the static configuration of the system to minimize memory overhead is a long and popular tradition.
All that is to say that when resource constraints and optimizations are the most important and constrained, static configuration is quite important.
As that corresponds well to the main domains in which microkernels find the most utility, so it is no surprise that Composite focuses on static configuration.

## Intuition: Vertical vs. Horizontal Isolation

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

## Detailed Isolation Model

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
