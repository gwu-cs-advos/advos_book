\newpage

<!--
Copyright (c) 2021 by Gabe Parmer.

Redistribution of this file is permitted under the GNU General Public
License v2.
-->

# Security Topics

## Concepts

Reference Monitor (see OS security by Jaeger)

- Complete Mediation

	- Complexities: [TOCTTOU](http://www.watson.org/~robert/2007woot/) with an example in [dirtyCOW](https://dirtycow.ninja/)
	- Linux Security Modules ([LSM](http://www.cse.psu.edu/~trj1/cse544-s10/papers/lsm.pdf) and [its](https://www.kernel.org/doc/html/latest/security/lsm.html) [API](https://www.kernel.org/doc/html/latest/security/lsm-development.html))
	- Capability-based OSes

- Tamperproof
- Verifiable/Trustworthy

Principles:

- Complete Mediation
- Least Privilege
- Economy of Mechanism
- Least Common Mechanism
- Separation of Privilege
- Minimal Trusted Computing Base (TCB)
- Defense in Depth

## Mechanisms

Taxonomy

- Access control matrix
- Mandatory vs. discretionary AC
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

## References

- A great taxonomy and overview of security concepts: [The Protection of Information in Computer Systems](http://web.mit.edu/Saltzer/www/publications/protection/)
- Fantastic [OS security](https://pdfs.semanticscholar.org/3126/a5318ec8b478da08944b97358c7e3491c08d.pdf?_ga=2.180702311.1928750056.1609337029-963934370.1607645716) book by Trent Jaeger
- A decent [overview](http://dance.csc.ncsu.edu/papers/CSUR2016.pdf)
- Specific systems:

	- [Protection and the control of information sharing in multics](https://dl.acm.org/doi/10.1145/361011.361067)
	- [Flask](https://www.cs.cmu.edu/~dga/papers/flask-usenixsec99.pdf) & seLinux
	- [Asbestos](http://www.scs.stanford.edu/~dm/home/papers/efstathopoulos:asbestos.pdf)
