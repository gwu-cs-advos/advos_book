# Some notes and retrospects on user-level threads

https://www.open-std.org/JTC1/SC22/WG21/docs/papers/2018/p1364r0.pdf

...and the response @ https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p0866r0.pdf

https://devblogs.microsoft.com/oldnewthing/20191011-00/?p=102989

## Docs

https://learn.microsoft.com/en-us/windows/win32/procthread/fibers
https://learn.microsoft.com/en-us/windows/win32/procthread/user-mode-scheduling

# PicoProcesses and Pico Providers

[Drawbridge](https://www.microsoft.com/en-us/research/project/drawbridge/) [rethinks](https://www.microsoft.com/en-us/research/publication/rethinking-the-library-os-from-the-top-down/) how Windows services interact with processes.
At its core, this is a refactoring of Windows to have the kernel focus on a "Security Monitor" that is lower-level than OS APIs.
Libraries and services can use this API to implement personalities.
Interesting parallels between these systems (and these APIs) and Plan 9.
Focus is mainly on supporting library OS functionalities for specific processes (personalities to determine all of a process' services), than on namespacing and composition of services.
APIs are small, including memory management, thread management, stream management, process management, and time/hardware management.
The I/O APIs share many UNIX philosophies around polymorphic behavior around streams.
A notable difference is that the namespace doesn't identify the type of object, rather a URI argument used to `DkStreamOpen` the resource.
