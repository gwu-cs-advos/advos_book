\newpage

<!--
Copyright (c) 2021 by Gabe Parmer.

Redistribution of this file is permitted under the GNU General Public
License v2.
-->

# Design for Parallelism

First, you want to understand the relative costs of different CPU operations.
Take the [quiz](http://computers-are-fast.github.io/) and see how well-trained your intuition is!

## Latency Comparison Numbers

These numbers are from ~2012 and are replicated from [here](https://gist.github.com/jboner/2841832).
See the FAQ for a more visual representation.
```
L1 cache reference                        0.5 ns
Branch mispredict                         5 ns
L2 cache reference                        7 ns
                                                    14x L1 cache
Mutex lock/unlock                        25 ns
Main memory reference                   100 ns
                                           20x L2, 200x L1 cache
Compress 1K bytes with Zippy          3,000 ns       3 us
Send 1K bytes over 1 Gbps network    10,000 ns      10 us
Read 4K randomly from SSD*          150,000 ns     150 us
                                                    ~1GB/sec SSD
Read 1 MB sequentially from memory  250,000 ns     250 us
Round trip within same datacenter   500,000 ns     500 us
Read 1 MB sequentially from SSD*  1,000,000 ns   1,000 us   1 ms
                                         ~1GB/sec SSD, 4X memory
Disk seek                        10,000,000 ns  10,000 us  10 ms
                                        20x datacenter roundtrip
Read 1 MB sequentially from disk 20,000,000 ns  20,000 us  20 ms
                                             80x memory, 20X SSD
Send packet CA->Netherlands->CA 150,000,000 ns 150,000 us 150 ms
```

Notes
```
1 ns = 10^-9 seconds
1 us = 10^-6 seconds = 1,000 ns
1 ms = 10^-3 seconds = 1,000 us = 1,000,000 ns
```

Credit

- By Jeff Dean:               http://research.google.com/people/jeff/
- Originally by Peter Norvig: http://norvig.com/21-days.html#answers

Contributions

- 'Humanized' comparison:  https://gist.github.com/hellerbarde/2843375
- Visual comparison chart: http://i.imgur.com/k0t1e.png

## Micro-optimization

This chapter briefly discusses the role of low-level optimization in system design.
We use this as a counterpoint to the high-level abstractions investigated previously, and attempt to understand the trade-offs between abstraction, simplicity, and optimization.

- Micro-architecture, memory models, coherency (TSO)

	- Pipeline - branches, ALU operations (add/sub vs. mult/div)
	- Memory - sequential vs. random, working set WRT cache size

- Coding for optimization

	- static information -> compiler
	- function inlining, constant propagation, dead-code elimination, loop unrolling
	- fast-path identification and customization (see nova vs. composite)
	- optimization vs. readability

- Linux [performance introspection tools](http://www.brendangregg.com/linuxperf.html)
- Applications [working set size](http://www.brendangregg.com/wss.html)
- Introductory overview of [architecture details](https://medium.com/swlh/what-does-risc-and-cisc-mean-in-2020-7b4d42c9a9de), and an interesting treatment of the [Apple M1](https://debugger.medium.com/why-is-apples-m1-chip-so-fast-3262b158cba2)

## Macro-optimization

Cross-layer puncturing of abstractions

- Checksum calculation and checking in Linux networking.
	We need to checksum the contents data that we're going to send in a packet.
	The checksum needs to be in the packet header to detect packet corruption (on the wire).
	Iterating through the packet is not cheap, but is necessary for computing the checksum.
	Key observation: the `write` system call requires that we effectively do a `memcpy` from the user-level buffer, into a kernel buffer used to construct the packet.
	Why should we iterate through the packet in `memcpy` **and** also do so when computing the checksum?
	Linux collapses the layers, and enables the copying of the packet from the user **and** the checksum to be done at the [same time](https://elixir.bootlin.com/linux/latest/C/ident/csum_and_copy_from_user).
- Checksum checking in hardware.
	Checksums are usually associated with a specific layer of the network processing (IP, UDP/TCP, etc...), yet hardware supports checksuming packets at all levels.
	In essence, it enables the dropping of corrupted packets *before* they arrive at the system, despite networking cards traditional role to simply treat packets as a collection of bits to be sent to the OS via DMA.
	This enables the OS to avoid the $O(n)$ costs of iterating through the packet to do the checksum upon arrival!
	Similar optimizations exist for TCP Segment Offload (TSO), and RSS.

### Reading

- [middleware optimization](http://aosabook.org/en/posa/applying-optimization-principle-patterns-to-component-deployment-and-configuration-tools.html)
- [event-driven processing in `nginx`](http://aosabook.org/en/nginx.html)
- building an [IPC infrastructure](http://aosabook.org/en/zeromq.html) (batching, memory allocations, concurrency, and lock-free algorithms)
- high-level walkthrough of the [Linux networking subsystems](https://www.cs.unh.edu/cnrg/people/gherrin/linux-net.html)

## Parallelism Concepts

- Atomicity (all-or-nothing)
- Coherency, memory barriers, atomic instructions (see 3.1 and 3.2 of [McKenney's book](https://mirrors.edge.kernel.org/pub/linux/kernel/people/paulmck/perfbook/perfbook.html))

	- Model of costs (3.2)
	- Amdahl's law + coherency overheads - linked list vs. ring buffer (ck) vs. LF stack
	- Locks -- TAS vs. ticket vs. MCS
	- TLB invalidation
	- IPIs

- Abstraction Concepts

	- Commutative scalability rule
	- MCS?

- References

	- Gory details about [memory](https://people.freebsd.org/~lstewart/articles/cpumemory.pdf) (an online ["summary"](https://lwn.net/Articles/250967/))
	- [Concurrent Data Structures](https://www.cs.tau.ac.il/~shanir/concurrent-data-structures.pdf) by Mark Moir and Nir Shavit
	- [The Art of Multiprocessor Programming](https://dl.acm.org/doi/pdf/10.5555/2385452) ([replicated here](http://cs.ipm.ac.ir/asoc2016/Resources/Theartofmulticore.pdf))
	- [Intro to parallelism](https://mirrors.edge.kernel.org/pub/linux/kernel/people/paulmck/perfbook/perfbook.html) by McKenney

- Case studies:

	- the idea behind a partitioned memory allocators, in chapter [6.4.3](https://mirrors.edge.kernel.org/pub/linux/kernel/people/paulmck/perfbook/perfbook-1c.2019.12.22a.pdf)
	- [jemalloc](https://github.com/jemalloc/jemalloc): decent [somewhat recent design](https://www.facebook.com/notes/facebook-engineering/scalable-memory-allocation-using-jemalloc/480222803919/), an [old design document](https://people.freebsd.org/~jasone/jemalloc/bsdcan2006/jemalloc.pdf), [phrack's take](http://phrack.org/issues/68/10.html), and a [video](https://www.youtube.com/watch?list=PLn0nrSd4xjjZoaFwsTnmS1UFj3ob7gf7s&v=RcWp5vwGlYU&feature=youtu.be)
	- reference counts (e.g. [section 4.2](https://pdos.csail.mit.edu/6.828/2017/readings/rcu-decade-later.pdf))
	- [memcached](https://github.com/memcached/memcached)
	- `pid` table implementation in `xv6` vs Linux ([section 4.5](https://pdos.csail.mit.edu/6.828/2017/readings/rcu-decade-later.pdf))
	- [parsec](https://github.com/gwsystems/ps/)
	- [concurrency toolkit](http://concurrencykit.org/)
	- Composite kernel
	- [Berkeley DB](http://aosabook.org/en/bdb.html)
