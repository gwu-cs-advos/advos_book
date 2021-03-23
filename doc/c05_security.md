\newpage

<!--
Copyright (c) 2021 by Gabe Parmer.

Redistribution of this file is permitted under the GNU General Public
License v2.
-->

# Operating Systems Security

We've discussed system abstraction and design.
We've identified the separation of software into modules, and orchestrating the coordination and isolation between modules as fundamental in providing inter-client isolation in spite of having shared services that can manage system resources.
We've also discussed capability based systems as a means to provide access control on primitive system resources, and Plan 9 namespaces as a means to control which resources (and which virtual resources provided by user-level 9P services) each process has access to.

However, we haven't thoroughly discussed how to think about *system security*.
This chapter will focus on this essential topic.

## References

- A foundational taxonomy and overview of security concepts: [The Protection of Information in Computer Systems](http://web.mit.edu/Saltzer/www/publications/protection/).
- Fantastic [OS security](https://pdfs.semanticscholar.org/3126/a5318ec8b478da08944b97358c7e3491c08d.pdf?_ga=2.180702311.1928750056.1609337029-963934370.1607645716) textbook by Trent Jaeger.
- Great OS textbook *overview* of security: an [intro](https://pages.cs.wisc.edu/~remzi/OSTEP/security-intro.pdf), [authentication](https://pages.cs.wisc.edu/~remzi/OSTEP/security-authentication.pdf), [access control](https://pages.cs.wisc.edu/~remzi/OSTEP/security-access.pdf), [crypto](https://pages.cs.wisc.edu/~remzi/OSTEP/security-crypto.pdf), and [distributed](https://pages.cs.wisc.edu/~remzi/OSTEP/security-distributed.pdf).
- A decent [overview](http://dance.csc.ncsu.edu/papers/CSUR2016.pdf) of more modern OS security techniques.
- Specific systems:

	- [Protection and the control of information sharing in multics](https://dl.acm.org/doi/10.1145/361011.361067)
	- [Flask](https://www.cs.cmu.edu/~dga/papers/flask-usenixsec99.pdf) which is the foundation for seLinux
	- [Asbestos](http://www.scs.stanford.edu/~dm/home/papers/efstathopoulos:asbestos.pdf)

## Concepts

System security revolves around defining and constraining which principals can access which data and resources.
We often intuitively think about cryptography when we discuss security.
Cryptography is often used to algorithmically limit which principals can access encrypted data.
We think about OS security as focusing around abstractions to provide isolation, and limit access to system resources.
We've already seen now *namespaces* are integral in this.
Namespaces prevent resources from even being addressed by a process when they aren't present.
*Capabilities* in a CBOS mediate access to primitive resources in the system (pages and other kernel resources).
*Hierarchical namespaces* define the set of file-like resources accessible to a process in Plan 9, and, importantly, abstract resources can be defined in user-level services, thus the system can be arbitrarily extended.
These are the *mechanisms* by which we provide isolation and prevent unconstrained resource accesses.

The next important question is how we determine *which resources are available in a principal's namespace*.
In Plan 9, a parent process that `rfork`s a child has the ability to use `unmount` to remove portions of the namespace (i.e. remove access to resources), and `bind` to craft the namespace.
We also have ways in which this access is organized and distributed using `exportfs` which chooses which subtree of the namespace it is serving up to an `import`.
In a CBOS, delegation and revocation define a set of mechanisms for resource namespace modification, and components use those mechanisms to enable access from other components to a subset of their resources.
In Composite, the capability manager can define its own policies for how resources can made accessible across components.

### CIA

A good "first cut" at what we're trying to accomplish in secure systems is that we're trying to enforce CIA:

- **Confidentiality.**
	This is the act of ensuring that only principals that should be able to see some data, can actually *read* the data.
	It is easy to see why encryption is often an important part of providing confidentiality, especially when communication crosses a network.
	It is important to realize that confidentiality is also one of the primary consideration when removing read access to files, or when removing files from a principal's namespace.
	When encryption keys or your password are held in a processes, we rely on memory isolation to ensure confidentiality.
- **Integrity.**
	Integrity's goal is to prevent principals from potentially corrupting information or data-structures that they should not have access to.
	Module protection domains also prevent *writes* to sensitive data from other principals, thus ensuring integrity.
	Integrity is also protected for files when principals are *not* given write access to them.
	Note that leaking out passwords, encryption keys, or contact lists are not threats to integrity.
- **Availability.**
	The ability of the system to provide service is its availability.
	If a principal is able to manipulate the system such that it cannot provide service (or takes longer to provide service than is useful) for other principals, then the system fails to provide availability.
	A Denial of Service (DoS) attack that continuously crashes a web-server, or triggers some execution path that takes too long and prevents servicing other clients, are both attacks on availability.

### Reference Monitor

The core of OS security is the **reference monitor** (*refmon*).
When a principal requests access to a resource, the refmon validates that it has the *privilege* to access the resource.

- *Complete Mediation* -
	All requests to access resources in the system must be checked, and allowed or denied based on the refmon's mechanisms and policies.

	A CBOS's kernel is mainly focused on being the system's refmon (especially in Composite where scheduling is moved to user-level).
	Whenever a capability is activated, the kernel determines 1. if that capability exists and references a resource, and 2. how the resource can be operated on.
	The hardware plays a necessary part of the refmon as well.
	For example, each load and store is translated via the page-tables to the physical memory resource, which means that the hardware page-table walker, the TLB, and the interaction between the kernel and those hardware facilities are all part of the refmon.

	- The complexities in refmon implementation are many.
		Beyond traditional software correctness, one needs to worry about Time of Check to Time of Use ([TOCTTOU](http://www.watson.org/~robert/2007woot/)) bugs that open up small windows of refmon incorrectness.
		An amazing example: [dirtyCOW](https://dirtycow.ninja/).
	- Linux Security Modules ([LSM](http://www.cse.psu.edu/~trj1/cse544-s10/papers/lsm.pdf) and [its](https://www.kernel.org/doc/html/latest/security/lsm.html) [API](https://www.kernel.org/doc/html/latest/security/lsm-development.html)) try to simplify the reasoning behind the refmon in Linux.

- *Tamperproof* -
	The resource access decisions made by the refmon must be determined solely by the refmon's own state, and its intended logic.
	Principals, and principal's actions must not be able to corrupt the decision making process of the refmon.
- *Verifiable/Trustworthy* -
	As the refmon is the core decision procedure for all resource access, we *must trust it*.
	Independent if a principal can tamper with it, it might have *bugs* which cause it to crash or make incorrect resource access decisions.
	Thus, we must take steps to ensure that it is maximally trustworthy.
	The concept of "trustworthiness" is hard to put your finger on, but it is clear that you trust code to perform its required logic if it is *formally verified*.
	For software that has been formally verified, the software's code is mathematically proven to correspond to a *specification* of the software's desired functionality^[Technically, we aim to prove that the specification is a refinement of the code.].
	There are other characteristics of software that might make it more trustworthy, but verification is one of the strongest ways.

In the end, of the refmon is compromised in any way (i.e. its decisions are coerced by a principal), then security at higher-levels in the system is generally not possible.
Its mechanisms and policies are the foundations for providing CIA in the system.

### Principles

To generate a secure system we typically follow a number of principles.
These are the modus operandi that we should follow by default, and depart from only with great care.

- **Complete Mediation.**
	As discussed above, we must ensure that all resource accesses must be properly authorized by the refmon.
	If the refmon is "out of the loop" of some resource access decisions, it can challenging to maintain strong CIA.
	Note, that this implies that there are foolproof means for the refmon to know which principal is making the request, and to identify the requested resource.
	For example, a CBOS kernel must know which protection domain is requesting capability access, look up the access in the proper, associated capability table, and indexing into it to find the resource.
- **Least Privilege.**
	Each protection domain should have access to the minimal set of resources required to complete its task.
	There's no reason that a spreadsheet application should have access to the network, or a webserver to the ability to install software.
	Most processes in Linux should not have access to all of the almost 400 system calls (nor the almost 2000 in [Windows](https://github.com/j00ru/windows-syscalls)).
	Further, most processes should not have access to all of a user's files!
	This principle is important 1. to minimize the scope of integrity and confidentiality violations if a program is compromised by an attacker, and 2. to minimize the processes that must be inspected if a resource is found to be corrupted (this is related to the concept of "audits").
- **Economy of Mechanism** or "Keep it Simple Stupid" (KISS).
	Keep the design and implementation as simple and small as possible.
	There is a correspondence between system complexity, and the incidence of bugs.
	There seems to be a correspondence between system complexity (and lines of code), and effort to formally verify it (seL4 is 10K LoC in C, and 200K LoC in Isabelle/Hol proof language).
	The more edge-cases a design has, the harder it is to test, and the easier it is to gain false confidence in it -- "it works in all of the situations I tested!".
	The fewer bugs an implementation has, the less easily it will be compromised, and the easier it will be to fix.
- **Least Common Mechanism.**
	Minimize the number of services that are shared between two principals.
	Where a service is leveraged by multiple principals, it represents a potential means for one principal to negatively impact another.
	One might leverage a bug in the shared service to corrupt the other's data (integrity), to access its data (confidentiality), or to send the service into an infinite loop while it is servicing the other principal (availability).
	Each shared service requires significant scrutiny.
	Thus, such shared services should be minimized where possible.

	Note that this goal is often in opposition to efficient resource usage!
	If we follow the least common mechanism, we might have two components that manage separate pools of memory for two principals.
	What if one of the principals needs more memory than is in its pool, and the other is under-utilized?
	In contrast, having a single memory management component with a single pool that it can distribute among principals enables efficient memory allocations in response to dynamic demands, but is a single server whose compromise can impact all principals.
- **Separation of Privilege.**
	If the privilege to accomplish some task can be split up among different isolated protection domains such that they can coordinate to accomplish the task, system security will be increased.
	If one of the protection domains is compromised, an attacker will not have unfettered access to resources, and will still have to coordinate through well-defined APIs with the other protection domain.
	This will limit the extent to which it can access and manipulate resources.
	A system's file system can be isolated from the SSD device driver (in separate components) such that a fault in the driver will not corrupt all "in flight" I/O requests stored in the FS.
- **Minimal Trusted Computing Base (TCB).**
	The TCB is the set of all software and hardware that a given computation requires to be correct to accomplish its goals.
	In Linux, this includes all programs run as the same user as the current process, all those running as `root`, all of the `systemd` services whose function is leveraged by the process (or by libraries and programs it leverages), the kernel, and all aspects of the hardware.
	A bug in any one of those functionalities can prevent the proper functioning of the process.
	In contrast, in Composite, those component services depended on by a given component, and transitively depended on by them, the kernel (which is <10K LoC), and all hardware accessed by any of those components constitute the TCB.
	As only those component services that are required for the functioning of the component are depended on (principle of least privilege in action).
	Hopefully this analysis alone answers the question why microkernels are common in security sensitive domains.
- **Defense in Depth.**
	This is concerned with, intuitively, encouraging the construction of as many barriers as possible, over which attackers need to jump.
	Modern browsers have been great examples of this.
	Different tabs are separated out into separate processes to prevent compromised webpages from accessing other webpage data (barrier 1).
	The code executing in these processes executes within a software-provided sandbox that prevent access outside of valid ranges of memory (barrier 2).
	The javascript VMs prevent programming practices that access memory outside of the allowed objects (barrier 3).
	The [same-origin policy](https://en.wikipedia.org/wiki/Same-origin_policy) prevents requesting web resources outside of the original domain (barrier 4).
	The set of system calls that a browser tab can access is (or should be) restricted (barrier 5).
	The list goes on and on.
	Any one of these layers can (and will) have bugs that will be exploited.
	However, the likelihood that *all* of them will be exploited is quite unlikely.
	Outside of state actors, it is rare to have exploits that span many different layers of defense in depth.

	Note that defense in depth should inspire some sadness.
	It is a strong acknowledgement that there *will be compromises* of our code.
	It is inevitable.
	Given this, lets make it harder to effect widespread threats to CIA by adding more layers of checking and security.
	Put another way, there is an active *nuclear escalation* between those implementing the multiple layers of security, and those attempting to compromise each successive layer.

## Mechanisms

Taxonomy

- Confinement
- Access control matrix
- Mandatory vs. discretionary access control
- Information flow control

- LSM "hooks" (what are "hooks"?) for reference monitor implementation

	- [pathname](https://lwn.net/Articles/277833/)-based security in [Tomoyo](http://tomoyo.osdn.jp/2.6/index.html.en),
	- [AppArmor](https://wiki.ubuntu.com/AppArmor) and see an example of an AppArmor [webserver access policy](https://gitlab.com/apparmor/apparmor/-/blob/master/profiles/apparmor.d/usr.sbin.apache2), and
	- [Smack](http://schaufler-ca.com/description_from_the_linux_source_tree) and other [related](https://lwn.net/Articles/244531/) documentation

- Access Cntl Matrix protection vs. Information flow control

    - IFC: Flume, seLinux [policies](https://github.com/SELinuxProject/refpolicy/wiki)
	- ACM: Capability-based OSes, [Capsicum](https://www.cl.cam.ac.uk/research/security/capsicum/)
	- Nexus

Kernel minimality and design

- core of the security requirements
- separation of mechanism and policy ([chapter 11](https://ocw.mit.edu/resources/res-6-004-principles-of-computer-system-design-an-introduction-spring-2009/online-textbook/part_ii_open_5_0.pdf))
- The extent of complications for [complex kernels](https://lwn.net/Articles/749707/)

## Case studies

- [KPTI](http://www.brendangregg.com/blog/2018-02-09/kpti-kaiser-meltdown-performance.html) analysis

	- Linux vs. cos w/o KPTI vs. cos w/ vs. cos with per-component mappings

- VMs and related tech

	- hypervisor implementation via `libvirt`& [firecracker](https://www.youtube.com/watch?v=cwruf1ERAKM&list=TLPQMjYxMjIwMjCdrIWpLWTXvg)
	- Analyse the concrete [protection provided by gvisor](https://gvisor.dev/docs/) (read the entries in the "Architecture Guide" on the left)
	- [Nova](http://hypervisor.org/eurosys2010.pdf) ([github](https://github.com/udosteinberg/NOVA))

- Capability-based OSes

	- Composite
	- [Grant](https://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.130.9896&rep=rep1&type=pdf)-[take](http://courses.cs.vt.edu/~cs5204/fall99/distributedSys/groener/takegrnt.html) security model
	- [seL4](http://sigops.org/s/conferences/sosp/2013/papers/p133-elphinstone.pdf)
	- [Nova](http://hypervisor.org/eurosys2010.pdf)

- Adapting existing systems

	- seLinux
	- containers
	- dbus $\to$ kdbus [concerns](https://lwn.net/Articles/649111/) and [benefits](https://lwn.net/Articles/640360/) clients close buffers, clients cause server to block, memory consumption for published messages, "By moving broadcast-transmission into the kernel, we can use the time-slices of the sender to perform heavy operations", ...
