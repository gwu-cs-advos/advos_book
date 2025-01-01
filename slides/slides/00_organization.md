---
title: Advanced Operating Systems
---

# Advanced Operating Systems

<div class="center">

**Gabe Parmer**

© Gabe Parmer, 2024, All rights reserved

</div>

---

<div class="center">

# Gabe :wave:

</div>

---

## Concepts

- Complexity, system structure, and interfaces
- Interface and system design
- Isolation and security
- Concurrency and parallelism

---

## Skills

- Reading & understanding code
- Practicing and applying system design
- Understand the design of a microkernel
- Design and implement OS features

---

## Prerequisites

- :bangbang: Comfortable & experienced with one of C/C++/Rust
  - **Reading large C codebases**
  - **Writing C**
- Understand core ideas of OSes
  - Interrupts/exceptions/preemption
  - System calls
  - Scheduling
  - Parallelism and multi-core
  - Page-tables & memory virtualization/isolation

---

## Systems & Software

Systems:

- Linux, Windows, Android, `systemd`/`initrc`, containers

Software:

- UNIX & beyond: `xv6`, `plan9`
- Performance & concurrency: `demikernel`, `go`, `libuv`
- Microkernels: `nova`, **`composite`**
- Specialization: `unikraft`, `composite`
- Separation kernel: `nickel`

---

| Topic | Systems |
|-------|---------|
| Reading Code                       | `xv6`                   |
| System Complexity                  | `plan9`                 |
| System Structure - layering, hierarchy, components (DAG) | unix, `plan9`, windows  |
| Interfaces and Properties          |                       |
| Isolation - namespaces, resources, and binding | `plan9`, `composite`      |
| Isolation - horizontal vs. vertical, design principles | `nova`, `composite`       |
| Concurrency                 | `go`, `libuv`, `demikernel` |
| Parallelism            | `go`, `libuv`, `nova`       |
| Specialization | `composite`, `unikraft`, firecracker, FreeRTOS |
| Security | `nickel`, `composite` |
| Android | android, fuchsia |
| Resource management | cgroups |

---

```mermaid
mindmap
	root((Advanced OS))
		Reading code
		Software Complexity
			Concepts
				Accidental
				Essential
				Semantic gap
				Optimization
				Generality
			Specialization
				misra C
				RTOSes
				CBOS
			Systems
				plan9
				xv6
				unikraft
		System Structure
			Abstraction & Interfaces
				Properties
				Design
			Organization
				Layers
				Hierarchy
				Components
				DAG
			Systems
				xv6
				plan9
	    Concurrency & Parallelism
			Interfaces
				Properties
				Examples
					epoll
					io_uring
					poll
					non-blocking
					threads
	        Mechanisms
	        Systems
				libuv
				demikernel
				go
	    Isolation
			Examples
				Containers
				VMs
				Separation kernels
			Systems
				Nova
				Nickel
			Vertical
			Horizontal
			Namespace
			Resource
			Security
				Interfaces
					Properties
					Non-interference
```

---

## Class Organization

:one:: Understand concepts $\to$ read existing systems $\to$ understand concepts++

| Day                   | Class             | Course Work               |
|-----------------------------|------------------------|-------------------------|
| Thurs                    | *Gabe*: concepts/systems | -             |
| Thurs $\to$ Tues: | -             | *You*: understand code, do design |
| Tues                     | *You*: discuss studied system in group, All: discuss together | -              |

:two:: Second half of class - project applying concepts into a system.

---

## Class Organization

Tests? Graded homeworks? :-1: :poop:

Incentive management - peer evaluation:

- Peer evaluation
- If you aren't prepared for class, your group will suffer.
- You'll get out of this class what you put into it :shrug:

Last week of class: Quick demos

> Self motivation is required to get **anything** from this class
