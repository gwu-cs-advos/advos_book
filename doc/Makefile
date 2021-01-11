CHAPTERS     =$(sort $(wildcard c??_*.md))
PANDOC=pandoc
RES=resources
# --highlight-style pygments another option vs listings
PANDOC_FLAGS=--toc --number-sections --listings --template=$(RES)/eisvogel.tex --default-image-extension=pdf
VERSION=$(shell grep subtitle c00_*.md | awk -F 'Version ' '{print $$2}' | sed 's/"//')

# These are separated so that we can make a lot of changes without modifying the viewable .pdf in git.
NAME        =../gwu_aos
PDFOUTPUT   =$(NAME).pdf
PDFVERSIONED=$(NAME)_v$(VERSION).pdf


.PHONY: all pdf version
all: pdf
version: pdf
	cp $(PDFOUTPUT) $(PDFVERSIONED)

$(PDFOUTPUT): $(CHAPTERS) $(MANUAL_SRC)
	$(PANDOC) $(CHAPTERS) -o $@ $(PANDOC_FLAGS)

# use pandoc to generate the book
pdf:  $(PDFOUTPUT)

clean:
	rm -f $(PDFOUTPUT)
