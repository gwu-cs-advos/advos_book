all: build

install:
	sh install.sh

build:
	mkslides build slides
# fix bug in mkdocs (see issue 27 in mkslides)
	cp slides/resources/* site/resources/

serve:
	mkslides serve slides
