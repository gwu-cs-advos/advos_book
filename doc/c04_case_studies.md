# Design Case Studies

We've discussed the design of systems from a "first principles" perspective whereby hardware primitives are abstracted into more friendly abstractions that are used to compose higher-level systems.
Importantly, capability-based systems do this while maintaining fine-grained access control of system resources, thus building security into the core compositional facilities of the system.
Now, lets pivot and step through the underlying concepts behind the design of *existing, common systems*.
Lets start with UNIX, then we'll step through system evolutions motivated by its shortcomings.

## UNIX

UNIX consists of many different design components, including

- the goal of separating mechanism from policy, enabling composition of behaviors desirable to the user, by the user;
- the POSIX interface for interacting with the kernel and a sharable libraries for module reuse; and
- underlying resource abstractions focused on polymorphism to enable applications to have broad applicability (e.g. the VFS layer).

Lets start with the design [philosophy](https://danluu.com/mcilroy-unix/) underlying [UNIX](https://dl.acm.org/doi/10.1145/800009.808045)^[You should read the original UNIX paper. It is a great example of a paper that feels very *obvious* because it *became* the de-facto. But remember, at the time, it was not a given!].
Doug McIlroy was fundamental in the early design of the UNIX system, and well-captures the philosophy:

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

**Composition with pipes.**
This quote focuses our attention on how important pipes are to the composition properties of the system.
The [origin of pipes](http://doc.cat-v.org/unix/pipes/) and a broader discussion by [McIlroy](https://archive.org/details/DougMcIlroy_AncestryOfLinux_DLSLUG) further emphasize this fact.
The idea that we can compose complex functionality by simply directing the output of a program to the input of another, and "programming" the pipeline is amazingly simple.
This is an instantiation of the separation of mechanism (individual, small programs doing something simple) and policy (the composition of them to perform some goal).
It focuses on individual mechanisms of the system "doing one thing well", and enabling them to be composed to create application-specific behaviors.

Lets compare two means of programmatic composition:

- program $\to$ shell $\to$ pipelines $\to$ shell scripts
- functions $\to$ libraries $\to$ dependencies $\to$ scripting languages $\to$ applications

The shell is amazing as it bridges the gap between a programming interface, and a user-interface (UI).
In this way, it is similar to the spreadsheets: custom programming environments that provide UI specialized around the programming domain.
Just as spreadsheets enable the relatively simple programming incomputation over tabular data, shell programming enables text-based programming over the resources of the OS.
This has the significant impact of enabling the system user or administrator to efficiently, programmatically interact with system resources.
This places a lot of *responsibility* in the hands of the programmer: they must understand how to use the command line, the set of programs available to compose via pipes, and -- inevitably -- shell script syntax to preserve the created pipelines.

In contrast, modern programmers are more accustomed to the second form of composition.
Using our expressive programming languages to compose functionality provided by the large number of libraries.
There is less of a difference between this and pipeline composition that you might expect.
When you consider that UNIX programs such as `sed` and `grep` replace regular expression programming, `awk` replaces tabular computation and aggregation (often performed in contemporary infrastructures on json/xml), and even the likes of netcat (`nc`) enables the integration of networked services into pipelines.

Composition via pipelines relies on a few organizational principles:

1. dependencies (references to programs) are resolved using very simple, and adaptable, `PATH`-based mechanisms,
2. data shall be passed as streams of bytes, typically in ascii, textual formats to ensure a uniform interface for programs,
3. the fast-path for coordination is through asynchronous, kernel-based, statically-bound coordination, and
4. interaction between user, system, and pipelines through the filesystem namespace.

The trivial resolution of dependencies (1) is made much simpler when all input and output are in text, and compatibility across program versions must only maintain command-line options.
If you wrote a shell script to use program `x`, it is likely it will work on another system if `x` is standardized.
This is much more difficult for libraries (leading to ["dependency hell"](https://en.wikipedia.org/wiki/Dependency_hell)) given the need for binary compatibility since coordination is *typed*.

An implication of (2) is that each program requires parsing.
Many are traditionally written in languages with poor facilities for string manipulation (e.g. C), even more-so emphasizing the use of pipeline-based composition to leverage `sed` and `awk`.
The stream-of-bytes interface between computations in a pipe is rather limiting in a few ways.

- System services that control and manage resources (many of the "daemons" that clutter our `ps aux` output on modern Linux systems) don't interact with a streams-of-bytes, and require typed data.
	Untyped streams are quite useful when the main currency of data is text, but it is less useful when data is complex and better fits into typed tuples (think: arguments to functions).
- Complex applications (for example, browser, games, editors, ...) require richer interactions.
	It isn't enough to pass streams of text; typed interactions are required.
	For example, a browser tab must communicate its bitmap of its rendered webpage to the main browser tab controlling which tab is displayed and should receive interaction events back from it.
	Performance dictates that you really want to share this data without copying it, thus promoting the use of shared memory (instead of a stream of bytes).
	The bi-directional communication means that one-directional streams are not sufficient.
- Synchronous, call-return interfaces are necessary and pervasive.
	Streams are good for the transformation and filtering of data; they do *not* solve the problem of enabling a service to return appropriate responses to queries and operations from clients.
	Thus stream-based composition is not a good fit for *system construction* in which providing abstractions through call-return APIs is necessary.
	We see this complexity in UNIX's difficulty in adapting to the propagation of daemons.

The asynchronous communication model via kernel-based IPC (pipes) in (3) has a number of consequences.
The asynchrony enables pipelines to be able to accommodate system concurrency.
A stage in the pipeline can block on I/O while others continue processing.
It also enables the usage of parallelism as each pipeline stage can reside on a separate core.
These properties (along with the inherent fault isolation) mean that pipelined (or, more generally, DAG-based) composition is still rather common as it enables *distribution* of the processing.
See [spark](https://spark.apache.org/) as an interesting example of this composition style used for big-data processing.
A downside of (3) is that the kernel's IPC performance can matter quite a bit, and it has been difficult to implement efficient IPC in a monolithic OS.

To dive into (4), lets continue.

**Everything is a file^[Almost. Kinda. Not really. Read on.].**
It is notable that each program in a pipeline can access resources in the system's index of resources (the hierarchical file system namespace).
This opens up the possibility for system management and modification with potentially pipelined processes (often in shell scripts to enable conditional computation).
That docker and other container runtimes specify their automated system setup with shell scripts demonstrates the power provided to simple programs (e.g. `apt-get install`).

Everything is a file (EiaF) enables programs that operate not just on traditionally construed "files", but also on anything that can appear as a file.
For example, the `mkfs` commands can create a new file system image on a disk (as it appears as a file in `/dev/sda?`), or in a file, which can then later be mounted and managed at user-level using [FUSE](https://en.wikipedia.org/wiki/Filesystem_in_Userspace);
programs that operate on the contents of files, can transparently operate on dynamic content provided by other files using named pipes;
programs that process file data can introspect and modify programs as they execute using the `proc` virtual file system; and
performing system configuration uses simple command line utilities (e.g. setting your power management to ignore power via `echo performance > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor`) in the `sys` virtual file system.
Even programmatic data-structures, databases, and whatever your heart desires can be exported into the filesystem namespace using FUSE.

Lets place the effectiveness of EiaF into the context of the discussion of the class:
It is effective as the core system exposes system resources using a general namespace operated on using the same set of polymorphic operations.
System modules are broken into programs that are composed to define complex operations on system resources.
Each of the resources exposed by the FS namespace is a simple mechanism, and the composition of programs that operate on it defines the policy for using it.

**Shared libraries for module reuse.**
This is paired with a structured means of sharing code (shared and dynamically loadable libraries) to enable productive programming -- still the core of C programming.
A set of directories (that can be user-expanded) is used to lookup libraries, enabling their code to be shared between applications (thus saving aggregate system memory).
However, the dependency hell that can result from having various applications that all require different versions of libraries (that themselves require different versions of other libraries) is somewhat distressing.
Some systems have acknowledged that memory is cheap, and that we can statically compile most libraries into our applications without many ill effects (OSX's solution).
Alternatively, system configuration has been automated by containers (e.g. `docker`) acknowledging that most modern applications require many different carefully orchestrated frameworks, libraries, and programs, thus it is sane for an application to be specified and loaded as that collection.
Both of these solutions demonstrate the inherent simplicity (to a fault) of the default UNIX library mechanisms.

### Shortcomings of UNIX design

UNIX is quite old.
It is unreasonable to expect that when it was created (and matured) the designers understood what computers were going to be used for, and asked to do almost 50 years later.
In a very real sense, it is *amazingly impressive* that UNIX has remained relevant and useful across the decades as much as it has.
This is in part due to its adaptability, and its corresponding evolution over time.
In this section, we'll discuss some of the aspects of UNIX that did not age well, largely from the perspective of interface design.
These should be read not as an indictment of UNIX -- as by all accounts it is an immensely successful design -- and rather as a study in interface design, with an eye toward understanding system evolution over time.

**Lack of orthogonality.**
We see the consequence of poor orthogonality in UNIX APIs most prominently in the event management APIs of the system.
Event management and notification is a core system functionality made necessary by non-deterministic interactions due to concurrency and parallelism.
Our program wants to be notified when data is available, or actions have been taken that will impact us.
UNIX provides a number of event notification mechanisms:

- blocking system calls denote that data is available (the "event" has occured) when returning from the system call,
- signals emulate interrupts in software by causing a signal handler to execute *in the context of whatever thread is executing*,
- *event multiplexing APIs* such as `select`, `poll`, and `epoll` (`kqueues` on BSD) that enable a thread to await an event on any of a number of file descriptors,
- timers and time-based system calls provide means for the system to be notified when some specification of time has elapsed, and
- we want to be notified when our child programs have completed their computations (via `wait` or `waitpid`).

But what if we want to wait for when a client makes a request to a socket *and* to receive a notification that the user has pressed `Cntl-C` (a signal) *and* to await multiple file reads resulting in data *and* for some time to pass *and* for our children to `exit`?
Yikes.
This often required us to write multi-process/thread programs each using a separate event notification mechanism, and aggregate all events into the same API (similar to how `libuv` does it).

Linux has moved on, and attempted to solve this lack of orthogonality in an adhoc manner.
The key observation is that the different event notification mechanisms cannot easily co-exist, but that `file descriptors` are a ubiquitous abstraction upon which one key notification mechanism is based: `select`/`poll`/`epoll`.
If we can convert signals, timers, and process notifications into file descriptor actions, then we should only require the event multiplexing APIs that operate on `fd`s.
Thus, Linux has enabled attaching each of these event sources to file descriptors:

- `signalfd` as a solution for signals, but it carries around a [lot of legacy](https://ldpreload.com/blog/signalfd-is-useless) that make it very difficult to use,
- `pidfd_open` to get process termination notifications (but be careful of race conditions due to terminates before this function is called), and
- `timerfd` for timer notifications.

The goal of orthogonality is also violated in UNIX's proliferation of IPC mechanisms.
IPC mechanisms include: signals, pipes, System V Shared Memory, mapped files, shared files, named pipes, UNIX domain sockets, TCP/UDP sockets, DBus, Binder (on Android), and so on.
Every service needs to choose the mechanism it wants to use, and every client must use that mechanism.
If a client wants to interact with multiple services?
Multiple IPC mechanisms -- and you get to hope they compose (for example, can I wait for the reply from one service, while awaiting the reply via memory mapped buffers in another?...Not really).
In the best case, having this many mechanisms is simply confusing.
Similar orthogonality violations include process creation (`fork`, `vfork`, `clone`, `posix_spawn`), and system configuration (`sysctl` and `/sys/*`), but I think that you get the idea.

**Accidental complexity and composition challenges.**
As we look at the *interactions* between different operations, we see that UNIX had some trouble adapting to the changes that were required over its lifetime.
To start, see the discussion on reentrancy in the section on state management in interface design.
Summarizing here: signals interrupt whatever a thread is currently doing, and execute a signal handler.
If the computation interrupted holds a lock, the handler will fail (deadlock, or, at best a race condition) if it takes the same lock.
Because most APIs don't publish if they use locks, it becomes difficult to use *any* library in signal handlers, including `malloc`.
Thus, signals don't compose well with just about any library.

Signals get worse.
"Long" system calls are those that can block and include the likes of `read` and `write`.
Signals can cause a thread blocked on a long system call to wake up (before the underlying event they were waiting for is complete), and return setting `errno = EINTR`.
This means that every single time you call a long system call, you have to remember to add a check for this case, and *retry* the operation again after handling the signal.
This behavior can be disabled for specific signals (using `sigaction` using `SA_RESTART`), but libraries cannot generally assume this configuration, thus requiring them to all handle `EINTR`.
When one of your abstractions doesn't compose well with the operations of the system (`read` and `write`), that abstraction should be reevaluated.

Additionally, imagine how complex the kernel code must be to return out of any long system call due to signal handers.
Regardless how deep a system call is into a complicated call-stack (one of the traditionally most complicated is calling into the VFS, which calls the `dentry` caches which calls NFS, which calls the TCP/IP stack), you must include the logic for either cancelling the request or enabling its re-submission at a later point, and the ability to unwind the call-stack, returning from the system call.
Only to have the same call made immediately after the signal handler is executed.
This example is so prototypical of accidental complexity that it  might just accompany the definition.

Threads generally complicated most existing APIs.
They don't compose well with signals: it isn't clear which thread a signal should be sent to, thus requiring APIs to establish that.
The `errno` value was global, meaning that errors returned from system calls made by multiple threads could clobber each other's `errno` values.
This was later fixed by placing `errno` into thread-local storage (a global variable per-thread).

The last example I want to go over is [`fork`](https://www.microsoft.com/en-us/research/uploads/prod/2019/04/fork-hotos19.pdf).
`fork` generally complicates the implementation of many different abstractions.
To quickly name a few:

- Threads and locks -- as one must answer the question what to do with all of the threads in a forking process, especially those that hold locks!
- Signals -- what should the system do if you `fork` a process in a signal handler?
- `mmap` -- when we map in a file, then `fork`, the file transitions from being trivial to modify, to suddenly suffering from race conditions (as the file memory is *shared* not copied across the fork).
- Buffered I/O -- when we print out, it doesn't necessarily trigger the system call to output to `stdout`; instead it is often buffered in `libc` until a `\n` is output -- thus `fork`ing can dead to replicated output if it isn't `fflush(stdout)`ed pre-`fork`.

UNIX has aged quite well, in general, as we are still using it!
However, the composition challenges that afflict some of its core abstractions certainly emphasize how challenging it can be (and how much you must know) to write correct programs.

**System adaptation and added features.**
Linux have had to adapt quite a bit over the past 20 years.
Linux on Android does not resemble a typical (SysV) UNIX system.
Even typical Linux distributions are pushing their lineage well beyond initial intent.
What are some of the [tacked on and/or replaced mechanisms](https://roxanageambasu.github.io/publications/eurosys2016posix.pdf) in a modern Linux system?

- BSD sockets -- Internet connectivity is essential, and the BSD socket API is a significant and note-worthy departure from the UNIX EiaF philosophy as there is no file system namespace presence for sockets.
- Notification and communication -- desktop notifications (for wifi, power, applications, etc...), and application services (GPS, contacts, ...) all rely on `dbus` or Android's binder via `ioctl`.
	These use dynamic routing of messages between clients and services and couldn't be farther from pipeline processing.
- File storage -- file systems are somewhat obsolete in many modern systems.
	Applications, especially on mobile, have foregone hierarchical file systems and often instead rely on the default storage: a SQLite database.
	On Android the file system is no longer seen as the main means of identifying system resources, instead relying on system services to provide resources.
- Generally, the rise of `ioctl` -- This system call is often seen as a system call multiplexer.
	Device drivers, or other kernel models can implement their own `ioctl` implementation, and the user can use `ioctl` to effectively invoke one of any number of operations defined by that module.
	`ioctl` is an acknowledgement that the system calls provide by Linux are not enough.
	Kernel modules want ways to extend them, and `ioctl` enables that.a
	`ioctl`s are used to implement the functionality of graphics cards, IPC (Binder in Android), I/O out-of-band changes, and, generally, to provide "system call extensibility".
- Concurrency/parallelism frameworks -- The core of Linux's APIs to handle concurrency and parallelism are focused on low-level, general mechanisms.
	Frameworks have picked up the slack to provide higher-level abstractions such as thread pools (e.g. `libglib` with `GThreadPool`, OpenMP, and Intel Thread Building Blocks) and event multiplexing (e.g. `libuv`).
	The challenge is that because these are not standardized, it becomes difficult to compose libraries that depend on one framework, with an application that uses another.

This is not to only pick on Linux.
OSX is a hybrid fusion of the Mach microkernel with FreeBSD, and uses Mach's IPC facilities pervasively, and it uses GDC to provide parallelism and concurrency services.

**Coarse-grained, discretionary security.**
Though UNIX has not aged well with respect to security, I will delay that discussion till a later chapter.
For now, the key to understand is that much of UNIX's view on security is painted by the lens of *discretionary access control* and of *course-grained, user-centric access control*.
UNIX security is *discretionary* because each user has the ability to share their own data with any other user should they want to.
This is in contrast to a security policy which disallows information from *flowing* from places where it is protected to parts of the system (e.g. the network) where it should not be accessed.
UNIX security is rather *coarse-grained* because each *user* is the principle and focus of access control.
This is nowhere more clear than with the `setuid` mechanism for privilege delegation.
This mechanism enables a specific program, when executed by a user, to run with the user identity (`uid`) as the user that created the program.
Thus, while executing the program, it runs with heightened privilege, opening the program up to [confused deputy](https://en.wikipedia.org/wiki/Confused_deputy_problem) attacks, as well as traditional attacks based on the user controlling the input.
As discussed previously, capability-based models that enable delegating access to specific resources (not all that can be accessed by another user) go a long way to preventing problems here.

In lieu of many details here, I'll simply point out that the [Android Security Model](https://arxiv.org/pdf/1904.05572.pdf) (especially section 4.3) shows how far the security model of such a system has to depart from that of UNIX.

**Summary.**
The root of many of these problems is just that it is hard to merge a modern system's requirements into the UNIX shell of POSIX APIs.
One view of this tension can be that it is hard to merge the UNIX philosophy with the modern requirements of frameworks and programming language environments.
Illustrations of this tension:

- UNIX emphasizes that we should "do one thing well" but modern applications only care to find an answer, and don't mind if there are many options (look at the number of libraries to parse json in many languages).
- There has been an increasing practical pressure and movement away from single-purpose applications toward those moving closer to swiss army knife applications (despite arguments *against* [this movement](http://harmful.cat-v.org/cat-v/unix_prog_design.pdf)).
	Even veritable command-line utilities, traditionally the encapsulation of UNIX design, have been increasing in [commmand-line complexity and utility](https://danluu.com/cli-complexity/).

One final perspective indicating that UNIX philosophies might simply have a scaling problem:

> To many, the UNIX systems embody Schumacher's dictum, "Small is beautiful." On the other hand it has been argued by Brooks in The Mythical Man Month, for example, that small is unreal; the working of a handful of people doesn't extrapolate to the world of big jobs. We agree only in part, for the present volume demonstrates with unusual force another important factor: intelligently applied computing technology can compress jobs that used to be big to manageable size.

### Reading

- Architecture of [bash](http://aosabook.org/en/bash.html)
- `xv6` [shell](https://github.com/gwu-cs-os/gwu-xv6/blob/master/sh.c)

## UNIX Counterpoints

UNIX is clearly not the be-all, end-all answer to all our systems needs.
This section will assess the underlying concepts behind existing development and system composition models that have evolved beyond UNIX.

When software was not as uniformly complicated, it was more common to

1. focus on obsessing about how to structure the abstractions of a tight body of code, and
2. compose modules these together using OS facilities (see not just pipeline composition, but also the likes of [Component Object Model -- COM](https://en.wikipedia.org/wiki/Component_Object_Model)).

However, modern application development cannot happen as a self-contained code-base, and require many different libraries, and programs.
We'll treat each of these in turn.

### Modern Applications Focus on *Library Composition*

Applications commonly *require* integration with tens if not hundreds of libraries.
"DLL hell" has argued that this is a challenging problem.
How can the versions of all of those libraries be consistent, and predictably work well with each other?
What are the properties of library-based composition versus system composition from isolated modules?

Programming environments and programming language dependency resolution (`cargo`, `pip`, `npm`, ...) organize the download, compilation, and linking of libraries into a user's application.
This job is [quite](https://research.swtch.com/vgo-import) [complicated](https://research.swtch.com/vgo-intro), and is [NP-complete](https://research.swtch.com/version-sat).
As such, some dependency resolvers use full [SMT solvers](https://en.wikipedia.org/wiki/Satisfiability_modulo_theories).
Note that this is a difficult problem even if we download a separate set of libraries for each application (a common option): you want to compile with the highest version of each library, but each library might require specific ranges of other libraries.
However, if the programming environment aims to share libraries between applications, this is even more challenging.
If two applications require different versions of a library, which do you use?
This often results in DLL hell in which different applications require different versions of the same library, thus preventing them from both executing.

Programming language libraries are linked and loaded into the process of the application.
Modules structured as libraries have a number of implications, including both limitations and benefits:

- *Limitations.*
	Libraries must often be implicitly trusted.
	Libraries must abide, by default, by the types of the programming language, thus might only have access to arguments passed into the library, allocated by the library, and globally stored by the library.
	However, in practice, this offers little isolation.
	Libraries can often access the entire OS's system call layer, thus access any system resources the application code has access to.
	Further, most programming languages enable code to interact with C code (often for compatibility reasons), and some even focus on C inter-operation (Python).
	Even more, synchronous function calls mean that infinite, or long loops in libraries consume the CPU resources of applications.
	Clearly, libraries must be treated as being one-and-the-same as the application code, thus falling short of providing isolation.
- *Benefits.*
	Library functionalities are exported as typed sets of functions and data-types that the programming language understands.
	This means that applications use direct function calls to communicate with the libraries.
	This results in efficient function calls to the library's functions, even potentially inlining library code into the instruction stream of the application.
	Importantly, it also means that data passed from application to library is directly interpreted by the library.
	The application understands the type layout (think `struct`s) in the same way that the library does.
	This concretely avoids the serialization (marshalling) and deserialization (demarshalling) costs of packaging and unpackaging data passed between modules.
	There is clearly a huge benefit to integrating modules via the compiler which has full type information.

Systems have, in the past, attempted to wrestle isolation out of programming-language facilities to establish some form of isolation.
These systems include: [spin](TODO) and [tock](TODO) that enable the extension of OSes using programming language modules, [singularity](TODO) which uses asynchronous communication between software-provided processes along with an exchange heap for passing arguments, and, more recently, [redleaf](https://www.usenix.org/conference/osdi20/presentation/narayanan-vikram) that enables the dynamic loading, termination, and isolation of individual modules.
However, short of these research systems, library-based composition, despite its performance benefits, cannot be used as a *general system composition* technique.

This leaves us in an awkward position:
Modern software development requires composition of libraries to create the rich applications which are pervasive, but we cannot use this mechanism to compose system functionality.
The disparity here is what has led to research systems such as Composite that aim to compose systems in a manner similar to traditional library-based composition, while maintaining strong isolation.

> **An aside: thought exercises.**
> Lets go through a sequence of questions to make sure we understand the relationship between an application, a library, and the kernel.
>
> - In C, what can a library do to interfere with the proper functioning of the client program?
> - In a type safe language (e.g. Java, Python, Rust), what can a library do to interfere with the proper functioning of a client program?
> - What if we want to have the library and application be implemented in different languages?
>
>	- What is a [Foreign Function Interface (FFI)](https://en.wikipedia.org/wiki/Foreign_function_interface)?
>		What complexities does it present?
>		Hint: think about garbage collection, data-layout, liveness, etc...
>
> - How about protection of the library from the application?
> 	Why would we want this?
> - Why are some services isolated in a separate processes, rather than as a library (think: a database)?
> - To enable functionality typically implemented in the OS to be implemented instead in libraries, what assumptions would we need to make, and how would we construct a system?
> - Related to this, what are the properties of kernel functionality that might make it *impossible* to be implemented in a library?

In many ways, the aggregation of functionality into monolithic applications, is very un-UNIX-like.
It flies directly in contrast to the "do *one thing* well" ethos of UNIX.
This is *not to say* that modern development styles are bad, in any way.
However, it is important to understand the presented trade-offs.
Specifically, the UNIX philosophy focuses on placing the eventual *policy* close to the user.
Doing one thing well often implies that the building block is not self-sufficient, and must be combined toward some goal.
Placing the composition capabilities close to the user increases the chance that the system will serve the user's purpose.
However, it also tends to create *expert systems* that require that users (or logic closer to users) understand both the nuances of the building blocks, and of how to compose them.
Contrast the command line which enables pipelined composition, versus a spreadsheet -- you can do less with a spreadsheet, but it sure is easier to perform tabular computation.
Thus, the development style that creates monolithic applications generally provides a much stronger statement about policy -- it *fully encodes* the policy about how resources should be put to a purpose.
This is un-UNIX-like, but leads to more opinionated, and more targeted applications.

### Centralized Frameworks and Middleware

An important property of modern applications is that they are not a *single* process.
Put another way, the composition of applications from libraries is not sufficient to be able to deploy and execute an application.
Applications also require a set of middleware to execute on the system to function.
*Frameworks* often tie together libraries and a set of that middleware.
Examples of this include Ruby on Rails and Django that tie together webserver functionalities such as HTTP parsing, routing, html generation from templates, and database storage and access.
All of these functions are spread across many different processes and libraries.

We're going to focus on system daemons to understand how modern systems (i.e. most Linux distributions) have evolved.
Historically, many daemons, each doing one(-ish) thing, would provide system services in UNIX:

- bootup is always driven by `init` (which is generally executed from memory as file systems are not yet mounted at boot time), which does a number of things including mounting the initial filesystems, `wait`ing for zombie processes without parents whom should wait for them, and starting other daemons/services *sequentially* via `initrc` (isn't this a lot for something that should be following "do one thing well"?),
- `cron` would execute commands at a time in the future, optionally on a repetitive schedule,
- `update` periodically writes out file-system superblocks,
- [`getty`](https://en.wikipedia.org/wiki/Getty_(Unix)) provides terminal services (input/output) and is part of the traditional `init`$\to$`getty`$\to$`login`$\to$`shell` sequence, and
- `telnetd` and `sshd` to provide their corresponding remote terminals.

Linux has added a few on its own including:

- [`udev`](https://opensource.com/article/18/11/udev) which enables dynamic device management -- for example answer how does the system know which driver, and which `/dev/*` entry to use for a thumb drive, and
- the Network Manager that determines which wireless AP or wired connection to use.

At some point, it became apparent that running every network service on a system was a bad idea, and that they should instead potentially be started only when they were requested.
Thus, `inetd` was created to understand which programs should reply to network requests on different ports, and to `fork`/`exec` the new service on demand.
This was essentially a service who job was to define the semantics around starting and executing other services!
Taking this idea to its logical conclusion, [`systemd`](https://lwn.net/Articles/777595/) is a complete rethink of [pid `1`](http://0pointer.de/blog/projects/systemd.html) (traditionally `init`) and is in some sense `inetd` for local services as well.
It is highly influenced by OSX's [launchd](https://www.youtube.com/watch?v=SjrtySM9Dns).
`systemd` has been criticised for doing a great many things, thus running counter to UNIX philosophy.
Many applications and frameworks have come to depend on it.
About `systemd`:

- `systemd` requires that communication with services and between services uses a *new* messaging medium called `dbus`.
	D-Bus is used quite extensively as a message based transport and it supports both call/return, function-call style "invocation" of services (via remote procedure call or RPC), and [publisher/subscriber](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern)-based communication.
	The latter enables a service to define a channel on which it can publish information, and for other services and applications to subscribe to the channel, thus receive all messages published to it.
	There is no semblance of pipes here, but rather a typed API for inter-service, message-based coordination.
	For example, the network manager publishes to its channel any wifi events (e.g. your desktop is disconnected from an access point), and applications (i.e. the icon displaying wifi status in the status-bar) consume this information.
	Additional examples of these publication channels: energy events (battery status) and "file open" events ("the user wants to open a music file").

	- [D-Bus](https://en.wikipedia.org/wiki/D-Bus) enables addressing of services, addressing of application-specific values (a cell in a spreadsheet), and the *typing* of interactions on the operations on those objects.
	- It requires [types over bitstreams](https://pythonhosted.org/txdbus/dbus_overview.html), and [API specification](https://dbus.freedesktop.org/doc/dbus-api-design.html), thus ensuring that clients and servers understand the same arguments and return values.
	- D-Bus is implemented not as a kernel abstraction, and instead as a single process on the system that others chat with via sockets.
	- [`kdbus`](https://lwn.net/Articles/580194/) attempted to move the dbus communication logic into the kernel, but a [lot of](https://lwn.net/Articles/649111/) [concerns were raised](https://lwn.net/Articles/619068/) (see the  [documentation](https://lwn.net/Articles/619069/)), and development moved on to [Bus1](https://bus1.org/) which is...a kernel dbus implementation!
		The initial PR showed good performance numbers: going from [$200\mu s \to 8\mu s$](https://lwn.net/Articles/640360/) when switching from process-implemented D-Bus to the kernel-defined D-Bus protocol^[Note that synchronous invocation through a call-gate in Composite takes around 0.3$\mu s$.].
		It is hard to implement a communication service in a process.
		A well-articulated summary of the [complications](https://dvdhrm.github.io/rethinking-the-dbus-message-bus/) of doing `dbus` in user-level, and some of the historical bugs.
		That said, as of now (early 2021), D-Bus is still implemented as a user-level process.
	- Android uses `binder` and OSX uses Mach IPC for similar coordination and IPC.
- One of `systemd`'s original purposes was to enable services to only be created when they are leveraged and required in the system.
	D-Bus enables this as it enables clients to express their interest in specific channels.
	This enables `systemd` to start those services associated with those requested channels on-demand.
	This means that a Linux distribution can include many different services, but they don't take up system memory nor CPU unless they are needed.
	For example, a the wifi service won't start if a desktop doesn't have wifi, and power management (e.g. battery status) won't execute unless there is a battery.
	An additional benefit of this on-demand initialization of system services is that they can start up at boot time in parallel.
	`initrc` used to start up system services sequentially.
	This 1. doesn't use the many cores of the system, and 2. might result in services blocking on I/O, thus the system idling awaiting I/O instead of desperately racing to system bootup!
	We used to wait for 10s of seconds for our systems to boot up; now we have to wait for single-digit number of seconds.
	SSDs helped with this, but `systemd`'s inherent design to encourage parallelism and concurrency should get part of the thanks.
- `systemd` is actually a [collection of processes](https://www.linux.com/training-tutorials/understanding-and-using-systemd/), not a single monolithic process.

	- See a list of all services with  `systemctl` and find those units postfixed with `*.service`; those with `systemd-*` are from the systemd collection.
	- There's even a service to create [containers](https://blog.selectel.com/systemd-containers-introduction-systemd-nspawn/).
	- It will try and reboot services if they fail, which increases system resilience.
	- You can [write your own](https://medium.com/@benmorel/creating-a-linux-service-with-systemd-611b5c8b91d6) service!

	Despite the implementation as multiple processes, the `systemd` processes are meant to be used as a cohesive core, rather than as replacable or composable modules.
	They are not portable (they use Linux-specific APIs), and the lamentations of which get the reply from the FreeBSD dev, Benno Rice, ["Unix is dead"](https://www.youtube.com/watch?v=o_AIw9bGogo).

- So wait, D-Bus is used as a foundational organizing IPC mechanism for system coordination?
	Is Linux a micro-kernel now?
	No, right?
	That's crazy.
	But also, yes, right?

Other examples exist of modern systems consolidating more policy, further from the user.
For example, modern Linux windows systems are based on Wayland which replaced the venerable X.org.
Where X separates the compositing of many windows onto a shared screen and the rendering of the scene into separate, replaceable processes, Wayland [co-locates compositing with rendering](https://wayland.freedesktop.org/architecture.html).
This places more power and more intelligence into the Wayland server, and enables better usage of graphics hardware.

Modern systems have required more policy to be placed into the core of the system.
No longer do we only have simple daemons that each do a single thing, and simply interpret configuration files in `/etc/*`, thus enabling tight control by the user.
Instead, `systemd` says that we need the system to autonomously understand what services are required, and to make its own decision about how to execute them.
This does move away from UNIX's organizing philosophies, but also demonstrates the power of creating a focused collection of services that are all written to solve a concrete problem.
Almost more importantly, `systemd` shows that *pipes alone don't seem like a sufficient organizing principle*, instead requiring the likes of call/return-style RPC *and* streams of data and notifications based on publisher/subscriber interactions.

### Containers: Modern Applications as *Ecosystems*

Given the fact that applications are a collection of libraries, middleware, and frameworks, a fundamental concern is how an application can be deployed as a single atomic unit that includes all of the required (specific versions of) libraries and processes?
*Containers*, starting with [Zones](https://en.wikipedia.org/wiki/Solaris_Containers), and made popular by [Docker](https://en.wikipedia.org/wiki/Docker_(software)), are a description of all of this application context, and an infrastructure for efficiently deploying all of it as a unit.
A side-effect of this is that a single system can execute multiple sets of the applications and middleware on the same shared hardware.
In this way, its goals are similar to virtualization.

However, containers are different from VMs in that they all share the same kernel.
Thus, containers provide significantly less isolation than VMs as a compromise in the container-providing kernel will directly impact all containers.
Containers are implemented by separating the kernel *namespaces* between containers.
These include the process id, hierarchical file-system, and networking namespaces, among others (seven namespaces, on Linux).
The benefit of containers over VMs is that they provide much more efficient sharing between containers (e.g. via [union mounts](https://en.wikipedia.org/wiki/UnionFS)) at the granularity of individual system resources, as exposed by the abstractions of the kernel (e.g. files, directories, processes, shared memory segments, etc...).

### Whither UNIX?

Does the move away from "do only one thing" with users defining the composition policy mean that UNIX is dead?
No.
UNIX philosophies are still going strong, but the need to create more capable, monolithic applications is not going to go away, and in many domains, will dominate.

So this leaves us with the burning question: how should we think about designing complex software systems?
Remember, first and fore-most, that we not only *compose* the *mechanisms* provided by modules into *policy*, but we also create complex software out of *many layers of these abstractions* where policy at one level is the mechanism at the next.
Modern systems absolutely demonstrate a preference for policy often defined in monolithic applications, but that doesn't mean that the modules that constitute the system *below* that point doesn't focus on composition of simple modules.
What is most clear to me is that streaming via pipes is not a sufficient coordination mechanism to create these layers of abstraction.

So UNIX is certainly not dead, but it is being made more complex and more capable.
There is a lot more "grey area" between "do one thing well" and monolithic blobs that is being inserted into the design space, all to reflect the requirements of the inherent complexity of modern software ecosystems.
It is important to understand how to design interfaces designed for modularity, composition, orthogonality, and simplicity, but it is also important to understand when departing from these rules is necessary.

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
