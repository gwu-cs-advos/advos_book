- Add "On the Criteria to be used in Decomposing Systems into Modules" by Parnas
- Update design section with a discussion about decomposition
- Add information/thoughts from "A Philosophy of Software Design" by John Ousterhout
- Example about perf vs. complexity in Java file I/O (3 classes) vs. python vs. UNIX (5 functions)
- Remove errors: Remove a file? Can remove from the namespace while refs still exist (refcnt)

    - capability access (bounds check vs. wrapping)
	- memory allocation (panic on `NULL == malloc(...)`) e.g. `xmalloc`
	- library vs. application for error APIs
