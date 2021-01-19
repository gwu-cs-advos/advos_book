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

- non-uniform event notification (fds vs. signals vs. timers vs. process termination)
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
	- `signalfd` as a solution, but it carries around a [lot of legacy](https://ldpreload.com/blog/signalfd-is-useless)

Unifying Event notification around `fd`s:

- `signalfd` as a solution, but it carries around a [lot of legacy](https://ldpreload.com/blog/signalfd-is-useless)
- `pidfd_open` to get process terminate requests, but what about processes that terminate before this call?
- `timerfd`
- ...all tied together with `fd`s mainly to fit into the same event loop

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
