---
title: Interface Properties
---

# Interface Properties

<div class="center">

**Gabe Parmer**

© Gabe Parmer, 2025, All rights reserved

</div>

---

## Interface Design

How can we think about interface design?

---

## Generality vs. Specificity

General, thin interfaces can be polymorphic
- Multiple implementations
- Component dependencies are on *interfaces*

Techniques
- Identify resources with (hierarchical) strings
- Decouple resource resolution from access (path vs. fd)
- Operations: access/modify/delete/create resource

-v-

- The [Liskov substitution principal](https://en.wikipedia.org/wiki/Liskov_substitution_principle) provides the foundation for how we might think about polymorphism independent of any specific language.


---

## General Interface Examples

Virtual File System API (VFS) and/or [9P](http://man.cat-v.org/plan_9/5/intro)
- OPs: open, read, write, close, creat, unlink, lseek, ...
- Polymorphic: pipe, file, socket, ...

[CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete) (create, read, update, delete)/REST
- OPs: GET, PUT, DELETE, POST, PATCH
- Polymorphic: cached, static webpage, dynamic webpage

---

# Function Composition

---

## Function Composition

Given functions $f$ and $g$, and arguments $a$ and $a'$:
- Each function has an expected behavior
- Is that expected behavior maintained, when executed in sequence?

$f(a); g(a')$?

---

Does `strtok` compose with itself?

```c [1-3|5-9|10-17|10-25]
/* Find strings that are space-delimited*/
s1 = strtok(str, " ");
s2 = strtok(NULL, " ");

/* What about if we add another operation in-between? */
s1 = strtok(str, " ");
s3 = strtok(str2, " ");
s2 = strtok(NULL, " ");

/* How could this happen? */
void
parse(void)
{
	s1 = strtok(str, " ");
	foo();
	s2 = strtok(NULL, " ");
}

void
foo(void)
{
	/* ... */
	s3 = strtok(str2, " ");
	/* ... */
}
```

---

Contrast this with `strtok_r`

```c []
char *saved_position = NULL;
s1 = strtok_r(str, " ", &saved_position);
s3 = strtok(str2, " ");
s2 = strtok_r(NULL, " ", &saved_position);
```

---

## Composition: Locks Don't Compose

```c [1-19|22-23|26-35|37-41|43-51|53-66]
void
dec(struct account *a, int amnt)
{
	lock_take(&a->l);
	a->balance -= amnt;
	lock_release(&a->l);
}

void
inc(struct account *a, int amnt)
{
	lock_take(&a->l);
	a->balance += amnt;
	lock_release(&a->l);
}

inc(a, 10); /* correct */
/* ... */
dec(a, 10); /* correct */

/* transfer between accounts */
dec(a, 10);
inc(b, 10);
/* not necessarily correct */

void
transfer(struct account *from, *to, int amnt)
{
	lock_take(&from->l);
	lock_take(&to->l);
	to->balance += amnt;
	from->balance -= amnt;
	lock_release(&to->l);
	lock_release(&from->l);
}

/* thread 1 */
transfer(a, b, 10);
/* thread 2 */
transfer(b, a, 10);
/* potential deadlock! */

void
transfer_not_scalable(struct account *from, *to, int amnt)
{
	/* Not scalable! Single coarse-grained global lock, used everywhere */
	lock_take(&accounts->l);
	to->balance += amnt;
	from->balance -= amnt;
	lock_release(&accounts->l);
}

void
transfer_complex(struct account *from, *to, int amnt)
{
	/* lock ordering by address */
	struct account *min = MIN(from, to);
	struct account *max = MAX(from, to);

	lock_take(&min->l);
	lock_take(&max->l);
	to->balance += amnt;
	from->balance -= amnt;
	lock_release(&max->l);
	lock_release(&min->l);
}





```

-v-

Locks are not composable, and transactional memory was proposed as a solution. [Beautiful concurrency](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/beautiful.pdf) in Haskell discusses a software transactional memory approach.

---

## Composability vs. "The Right Thing"

Recursive locks have an interface that self-composes:
- Can you take a taken lock?
- `lock(&l); lock(&l); cs(); unlock(&l); unlock(&l);`

Easier to use:
- Functions can take whatever lock is relevant

Interfaces shouldn't encourage "bad behavior":
- There are [many](http://www.fieryrobot.com/blog/2008/10/14/recursive-locks-will-kill-you/) [arguments](http://www.zaval.org/resources/library/butenhof1.html) [against](https://blog.stephencleary.com/2013/04/recursive-re-entrant-locks.html) [them](https://inessential.com/2013/09/24/recursive_locks).

---

## Composition: `fork`

`fork` - create new process, copying current process' state

`fork` doesn't compose with the following...*why*?
- `pthread_create`/`pthread_mutex_lock` - thread creation and locking
- `printf` - without a `\n` in the string, nor a `fflush`
- `exec` - does the composition have strong performance?

fork exposes leaky abstractions! <!-- .element: class="fragment" data-fragment-index="3" -->

Notes:
- If a thread is accessing a locked data-structure, and another thread calls `fork`, the data-structure will be left inconsistent.
- Buffered I/O suffers from the buffered output being replicated in `fork`
- `exec` must be called immediately, and all memory is replaced immediately after being copied

-v-

`fork` is [really not great](https://dl.acm.org/doi/10.1145/3317550.3321435). `posix_spawn` and the like are cleaner APIs.

---

## Considerations for Composability

*Thread safety*
- Does the interface allow two threads to call functions in an interface?
- In other words: are locks and other facilities used correctly?

*Reentrancy*
- Will an implementation provide proper service if a signal triggers during its execution?
- What if the signal handler calls the component?

---

## Considerations for Composability

Reentrancy problem examples:
- What if we call `malloc` in a signal handler?
- Call buffered I/O (e.g. `printf`)?

If your interface is not thread safe and reentrant, you are *not* composable with threads nor signals.
- I'd argue to *not use signals* and "give up" on that game (there are alternatives).

Notes:
`malloc` requires locks, so if we get a signal handler while holding one of these locks, then call `malloc`, we'll take the lock a second time and, more importantly, try and access data-structures currently in a critical section.
Not good.

Similar, what if are in the middle of flushing the buffered I/O for `printf` when a signal triggers, and calls `printf` -- thus trying to flush the buffer that is already being flushed.
This will likely result in redundant output.
Note, this applies to any buffered I/O, not just to stdout.

Signal alternatives: use signals, but just have them write to a FD hooked to a pipe to send an event to the main event loop.
Or in Linux, use `signalfd`.

---

## Goal: Composability

Simple systems, complex behavior:
- Independent functions, composed to create complex behavior

UNIX commands:
- Programs with text-based input/output, composed with `|`
- `$ cat classlist | grep "gwu.edu" | wc -l` - how many students in class?

---

## Namespaces and Composability

If we have $N$ different namespaces, then interfaces speak different languages
- It is harder for functions to be composable
- They don't have the ability to consistently name resources

---

## Composability via VFS

Linux unifies interfaces around `fd`s:
- [`pidfd`](https://man7.org/linux/man-pages/man2/pidfd_open.2.html)
- [`timerfd`](https://man7.org/linux/man-pages/man2/timerfd_create.2.html)
- [`signalfd`](https://man7.org/linux/man-pages/man2/signalfd.2.html)
- [`eventfd`](https://man7.org/linux/man-pages/man2/eventfd.2.html)
- [`memfd`](https://man7.org/linux/man-pages/man2/memfd_create.2.html) (vs. tmpfs)
- [`userfaultfd`](https://man7.org/linux/man-pages/man2/userfaultfd.2.html)
- [`setns`](https://man7.org/linux/man-pages/man2/setns.2.html) to update the namespaces

Why do we want all of these that used to be different interfaces?

Notes:

- `pidfd`? How do we wait for a file to have activity and for a child process to terminate?
- Integrate all of these fds into the event loop of the application

Namespace manipulations
- Update root w/ [pivot_root](https://man7.org/linux/man-pages/man2/pivot_root.2.html)
- [`clone`](https://man7.org/linux/man-pages/man2/clone.2.html) to control what is shared upon creating a new process
- [`unshare`](https://man7.org/linux/man-pages/man2/unshare.2.html) to remove shared resources

References many resources via fds
- [`pidfd`](https://man7.org/linux/man-pages/man2/pidfd_open.2.html) Why? How do we wait for a file to have activity and for a child process to terminate?
- [`timerfd`](https://man7.org/linux/man-pages/man2/timerfd_create.2.html)
- [`signalfd`](https://man7.org/linux/man-pages/man2/signalfd.2.html)
- [`eventfd`](https://man7.org/linux/man-pages/man2/eventfd.2.html)
- [`memfd`](https://man7.org/linux/man-pages/man2/memfd_create.2.html) (vs tmpfs in /tmp/) memfd’s transient use without an entry in the FS namespace means it can be better for [storing secrets](https://benjamintoll.com/2022/08/21/on-memfd_create/)
- [`userfaultfd`](https://man7.org/linux/man-pages/man2/userfaultfd.2.html)
- Even the function to set the namespace to be used [`setns`](https://man7.org/linux/man-pages/man2/setns.2.html) does so using fds that reference namespace files in a processes’ /proc entries…

---

## Composability via VFS

Unifying the namespace of kernel resources
- ensures consistent access to all resources
- ...but often uses bit-formats (i.e. `struct`s) that are `read`/`written`.

In contrast, plan9 uses a uniform textual representation
- no `socket`, `bind`, `connect`, ... instead:
  - `echo "connect 2048" /net/tcp/2/ctl` to set who can connect to the second process' socket
  - `cat /net/tcp/2/data` to read packets

---

## Composability via Hierarchical Namespace

- `/proc/*/*` - normal programs uniformly access kernel process state
- `/sys/*` - normal programs uniformly access kernel system configuration
- [`setns`](https://man7.org/linux/man-pages/man2/setns.2.html) updates the namespace of a process, and often requires passing a `fd` to a namespace file in the `/proc` FS.

Using hierarchical FS namespace as a *program abstraction*:
- the [`acme`](https://www.youtube.com/watch?v=dP1xVpMPn8M) editor
- the [plumber](http://9p.io/sys/doc/plumb.html) for expressive system messaging

---

## Everything is a File

In Plan 9, this is very much true.
- uniform representation of resources in the hierarchical namespace
- accessed via the small set of VFS functions
- Example: plan9 networking

---

```[]
cpu% cd /net/tcp/2

cpu% ls -l
--rw-rw---- I 0 ehg    bootes 0 Jul 13 21:14 ctl
--rw-rw---- I 0 ehg    bootes 0 Jul 13 21:14 data
--rw-rw---- I 0 ehg    bootes 0 Jul 13 21:14 listen
--r--r--r-- I 0 bootes bootes 0 Jul 13 21:14 local
--r--r--r-- I 0 bootes bootes 0 Jul 13 21:14 remote
--r--r--r-- I 0 bootes bootes 0 Jul 13 21:14 status

cpu% cat local remote status

135.104.9.31 5012

135.104.53.11 564

tcp/2 1 Established connect
```

-v-

- [Plan 9 videos](https://www.youtube.com/@adventuresin9/videos)
- [Namespaces in Plan9](https://doc.cat-v.org/plan_9/4th_edition/papers/names)
- [Networking in plan9](https://doc.cat-v.org/plan_9/4th_edition/papers/net/)
- [Implementation and namespaces in plan9](https://doc.cat-v.org/plan_9/4th_edition/papers/9)
---

## Separation of Concern/Orthogonality

> Do one thing well.
> - Doug McIlroy

-v-

- [Separation of concerns](https://en.wikipedia.org/wiki/Separation_of_concerns) (SoC)
- [Orthogonality](http://www.catb.org/~esr/writings/taoup/html/ch04s02.html#orthogonality)

---

## Separation of Mechanism and Policy

> The identification of a piece of software as "policy" or "mechanism" is a relative one. The implementor of a (virtual) resource establishes policies for the use of that resource; such policies are implemented with mechanisms provided by external software.
> Thus a hierarchy exists (under the relation "is implemented using") in which higher-level software views higher-level facilities as mechanisms.
> - Levin et al.

-v-

- [Mechanism/Policy Separation in Hydra](https://dl.acm.org/doi/10.1145/800213.806531) by Levin et al.

---

## Separation of Mechanism and Policy

A perspective on dependencies

- What mechanisms is my abstraction providing to its clients?
  - often: what is the resource and its properites?
- What are the policies it wishes to enable?
  - how are resources to be used?
- Policies determine how to deploy and use the resource

---

## Virtualization

Virtualizing an interface
- Providing an interface $A$ similar to another $A'$
- ...where $A'$ depends on $A$
- ...thus providing a higher-level abstraction
- Example: VMs vs. hardware

Virtualizing a resource
- Provide a more abstract version of that resource
- CPU $\to$ thread
- physical memory$\to$virtual memory
- interrupt$\to$signal

---

## Case Study: Scheduling

Four systems that handle scheduling differently:

1. Linux
2. Hydra
3. Exokernel
4. Composite microkernel

Questions
- What are the trade-offs for each?
- What is difficult or impossible?
- What is unclear in the design?

---

1. *Linux* - fixed set of policies (general-purpose, EDF, fixed priority), interfaces to set per-thread parameters (priority, etc...)
   - `cgroups` can assign rate-limits to collections of threads, either weighted-fair, or a limit of N units over M
2. *Hydra Scheduling* - a scheduling policy process can start/stop their processes, and alter their parameters
   - Kernel mechanism selects the policy process, and schedules its threads according to their parameters
   - parameters include timeslice and priority
   - execute until timeslice expires, or process blocks
   - kernel has fairness policies between policy processes, each w/ N out of M time units

---

3. *Exokernel Scheduling* - The kernel maintains a timeline of timeslices, processes are given a proportion of time, and they address which *slots* in the timeline in the future they want to use.
   - The kernel simply switches to the corresponding process at each timeslice.
4. *Composite Microkernel* - A scheduler component has the ability to dispatch directly to its threads.
   - When a thread wants to block, it uses IPC to invoke the scheduler, which can dispatch to another thread.
   - The scheduler, when dispatching can program the timer interrupt to fire at $c$ cycles in the future.
   - There can be multiple schedulers.

---

# Algebraic Properties

-v-

- Apache Beam requires some operators to be [commutative and associative](https://beam.apache.org/documentation/programming-guide/#combine)
- The [*scalable commutativity rule*](https://dl.acm.org/doi/10.1145/2699681) guides us to understand if an operation can be implemented scalably
- [Conflict-free replicated datatypes (CRDTs)](https://hydro.run/papers/keep-calm-and-crdt-on.pdf) enable many parallel operations on a core data-structure (multiple people modifying google docs at the same time), and guarantee eventual consistency. The “merge” operator is commutative, associative, and idempotent, and operations must be monotonic to be reasonably viewable.

---

## The Maths

Properties over *functions*
| Property | | Constraints |
|----------|--:| -------------|
| Commutativity | $\forall f, g,$ | $f \circ g = g \circ f$ |
| Associativity | $\forall f, g, h,$ | $(f \circ (g \circ h)) = ((f \circ g) \circ h)$ |
| Idempotency | $\forall f,$ | $ f \circ f = f$ |

Potentially for only specific $f$, $g$, and $h$

---

## What Does it Mean?

Assumptions

- $\circ$ = component execution
- $f$ = interface function (e.g. syscall) w/ specific set of arguments
- $f \circ g$ = execute $f$,then $g$

---

## Commutativity

Are operations *order invariant* and can be implemented scalably?

<div class="multicolumn"><div>

| core 0        | core 1        |
|---------------|---------------|
| 1: $\textsf{op}(...)$ | 1: $\textsf{op}(...)$ |
| | |
|  | 1: $\textsf{op}(...)$ |
| 2: $\textsf{op}(...)$ | |
| | |
|1: $\textsf{op}(...)$ | |
| | 2: $\textsf{op}(...)$|

</div><div>

- Operations aren't strictly related
- Any order of execution of operations is permissible

</div>

---

## Commutativity: Implications

**Scalability** -
An operation's logic doesn't impact another's.

- They modify not shared state -- no locks!
- They should be able to execute independently -- scalably.

---

## Associativity

Can operations be *parallelized*?

$f \circ g \circ h \circ i = (f \circ g) \circ (h \circ i)$
- parallel execution of $f \circ g$ and $h \circ i$

Examples:
- $\textsf{block_read}(b); \textsf{block_read}(b'); \textsf{block_read}(b'')$
- Same for $\textsf{block_write}$
- Same for $\textsf{block_read}$ & $\textsf{block_write}$ where $b \neq b' \neq b''$
- $(\textsf{block_read}(b); \textsf{block_read}(b');) \textsf{block_write}(b) \neq$
  $\textsf{block_read}(b); (\textsf{block_read}(b'); \textsf{block_write}(b))$

---

## Associativity: Implications

**Parallel execution**
- If the result is the same regardless how we "split up" execution, we can choose the execution order.
- Can enable *distributed execution* of operations

**Separate batch processing**
- Collections of operations can be executed together, then the output merged.
- This enables flexible *batch processing* of operations.

---

## Idempotency

Can operations be *replayed*?

Example:
- $\textsf{read}(fd, b, 1); \textsf{read}(fd, b, 1); \neq \textsf{read}(fd, b, 1, 0);$
- $\textsf{pread}(fd, b, 1, 0); \textsf{pread}(fd, b, 1, 0); = \textsf{pread}(fd, b, 1, 0);$
- $\textsf{select}(...); \textsf{select}(...); = \textsf{select}(...); $ <!-- .element: class="fragment" data-fragment-index="1" -->
- $\textsf{dup}(1); \textsf{dup}(1); \neq \textsf{dup}(1)$ <!-- .element: class="fragment" data-fragment-index="2" -->
- $\textsf{dup2}(1, 10); \textsf{dup2}(1, 10); \neq \textsf{dup2}(1, 10); $ <!-- .element: class="fragment" data-fragment-index="2" -->
- REST GET operations <!-- .element: class="fragment" data-fragment-index="3" -->
- SQL SELECT queries <!-- .element: class="fragment" data-fragment-index="3" -->

---

## Idempotency: System Implications

**Cacheability**

- No idempotency? $\to$ something being updated
- Nothing being updated? $\to$ can *cache* the output (rather than compute it)

**Burden of client tracking**

- Does the client need to track data, or can the server just be asked for it?
- Idempotent abstractions can lead to less state tracked in client ($\textsf{select}/\textsf{poll}$ vs. $\textsf{epoll}$)

---

# Implementation Optimizations

---

## Data Aggregation

- $t$ interface cost
- $c$ communication cost
- $N(t + c)$ send N messages
- $Nt + c$ batch N messages

> Trades *Latency* for *Throughput*

---

## Data Aggregation: Batching Policies

- Batch $N$:
- Batch based on concurrency:
- Time-bounded batching

---

## Data Aggregation: Examples

- Buffered I/O
- UNIX pipes
-
-

---

## Action Aggregation

---

## Action Aggregation: Examples

---

## Combining Optimizations

[Zygote](https://source.android.com/docs/core/runtime/zygote) processes in Android
- cache the output of
- aggregated actions
- to quickly create pre-initialized app startup
