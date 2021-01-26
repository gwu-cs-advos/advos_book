---
title: "Advanced Operating Systems: A Deeper Dive"
author: [Gabriel Parmer ©]
keywords: [Operating Systems, Educational, Trade-offs]
subtitle: "Version 2021-01-11"
titlepage: true
titlepage-color: "1034A6"
titlepage-text-color: "FFFFFF"
titlepage-rule-color: "FFFFFF"
titlepage-rule-height: 4
toc-own-page: true
listing-no-page-break: true
header-includes:
  - \hypersetup{colorlinks=false,
            allbordercolors={0 0 0},
            pdfborderstyle={/S/U/W 1}}
  - \usepackage{tikzsymbols}
---

<!--
Copyright (c) 2021 by Gabe Parmer.

Redistribution of this file is permitted under the GNU General Public
License v2.
-->

# Overview

This document includes some of the materials for the Spring 2021 (remote) version of GWU's Advanced Operating Systems class (scoped for undergraduates who have taken the OS class, and MS/PhD students who have taken architecture) created and taught by [Gabriel Parmer](www.seas.gwu.edu/~gparmer).
The course consists of

- **asynchronous, recorded lectures**
- questions that accompany those lectures
- **an FAQ that the instructor synthesizes from three open questions that are part of the questions provided for each lecture**
- recorded (private) in-class lectures that often generate a **visual blackboard discussion**
- **in-class exercises and discussions to emphasize group work and practical application of class concepts**
- and homeworks which are a focus of the class

This document includes -- or includes links to -- the **bolded** items above.
The class' [syllabus and policies](https://www2.seas.gwu.edu/~gparmer/classes/2021-01-11-Advanced-Operating-Systems.html) includes the syllabus and class policies, and the class' [github org](https://github.com/gwu-cs-advos/organization) includes a large collection of resources.

This document should be used as a guide to the class.
It is not comprehensive enough to be a book, and is instead a painfully detailed outline.
It is written to read more like (technically detailed) blog posts more-so than a book.
We focus on providing a large number of references to enable the independent investigation of class issues.

## Class Organization

The class is separated into three broad categorical topics:

1. **System Design and Abstraction** --
	Here we discuss a cohesive understanding of how and why we design systems, and the goals toward which they should be implemented.
	We discuss strategies for reading code, writing code for reading, design goals, and case studies of the trade-offs that multiple systems make.
1. **System Security** --
	We focus on security as a means to understand a cross-cutting, non-functional aspect of the system can figure prominently into the construction of systems.
1. **System Parallelism and Concurrency** --
	We focus on parallelism as a means to understand system design made more complex by extreme optimizations.
	Thus, this give us broader insight into the complexities of the push-and-pull between a clean design, and an efficient optimization.

There are many references throughout this document.
I choose to link to Wikipedia not because it is the highest-quality source, but because it is a decent source to get a high-level understanding.
I use it because the URLs are more persistent than referencing slides or data on academic webpages.

## Reference Code

The class is very heavy on reading code.
Many of these repos have been replicated into the [class organization](https://github.com/concurrencykit/ck).
Systems we'll read include:

- [xv6](https://github.com/gwu-cs-os/gwu-xv6)
- [plan 9](https://github.com/gwu-cs-advos/plan9)
- [systemd](https://github.com/systemd/systemd)
- [Nova](https://github.com/udosteinberg/NOVA)
- [Fuchsia](https://fuchsia.googlesource.com/fuchsia/+/refs/heads/master) ([download instructions](https://fuchsia.dev/fuchsia-src/get-started/get_fuchsia_source#download-fuchsia-source), [concepts](https://fuchsia.dev/fuchsia-src/concepts/kernel/concepts))
- [Parallel Sections](https://github.com/gwsystems/ps/)
- [Concurrency Kit](https://github.com/concurrencykit/ck)
- [memcached](https://github.com/memcached/memcached)
- [libkvm](https://github.com/ceph/qemu-kvm/blob/master/kvm/libkvm/libkvm.c) (and the [rust](https://github.com/rust-vmm/kvm-ioctls/tree/master/src/ioctls) [variant](https://github.com/alexpilotti/libkvm))
