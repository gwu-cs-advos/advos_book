SLIDEDIR=~/repos/advos/organization/docs/

all: build

install:
	sh install.sh

build:
	mkslides build slides
# fix bug in mkdocs (see issue 27 in mkslides)
	cp slides/resources/* site/resources/
	cp -r site/* $(SLIDEDIR)

serve:
	mkslides serve slides
