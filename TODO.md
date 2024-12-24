- Add "On the Criteria to be used in Decomposing Systems into Modules" by Parnas
- Update design section with a discussion about decomposition
- Add information/thoughts from "A Philosophy of Software Design" by John Ousterhout
- Example about perf vs. complexity in Java file I/O (3 classes) vs. python vs. UNIX (5 functions)
- Remove errors: Remove a file? Can remove from the namespace while refs still exist (refcnt)

    - capability access (bounds check vs. wrapping)
	- memory allocation (panic on `NULL == malloc(...)`) e.g. `xmalloc`
	- library vs. application for error APIs
- Add articles on the [semantic gap](https://www.riverphillips.dev/blog/go-cfs/)

- Examples for simplicity

	- Tinygrad for ML
	- Alan Kay about [complexity vs compliated](https://tinlizzie.org/IA/index.php/Alan_Kay_at_Qualcomm:_Is_it_really_%22Complex%22%3F_Or_did_we_just_make_it_%22Complicated%22%3F) and modern [complexity](https://tinlizzie.org/IA/index.php/How_Complex_is_%22Personal_Computing%22%3F_(2009)) and his institute's view on how to make a [simple OS](https://tinlizzie.org/VPRIPapers/rn2006002_nsfprop.pdf)
	- [Ian Piumarta's](https://www.piumarta.com/software/) projects ooze simplicity

- More systems reference

	- [MIT class](https://6826.csail.mit.edu/2020/)
	- [Lampson's links](https://bwlampson.site/) including updated [sys design notes](https://bwlampson.site/Slides/Hints%20and%20principles%20(HLF%202015)%20abstract.htm)

- [Rigorous System Design](https://www-verimag.imag.fr/PEOPLE/Joseph.Sifakis/papers_pdfs/Rigorous%20System%20Design.pdf) - decent guides (sep of concerns, component-based design, coherency, and correct by construction), but unclear how scale

- Argument for simplicity, and with a ToC that is out of software design: [What made Apollo a Success](https://ntrs.nasa.gov/api/citations/19720005243/downloads/19720005243.pdf)
- Add link to a Binder [video](https://www.youtube.com/watch?v=Fb4UoqXPEtI&list=TLPQMDUwODIwMjSWs7PT98Ul8Q) when discussing android
