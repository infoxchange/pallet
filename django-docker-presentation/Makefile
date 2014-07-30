NPM_DEPS = \
		bower \
		less \
		$(NULL)

NPM_BIN = $(shell npm bin)

all: build

install:
	npm install $(NPM_DEPS)
	$(NPM_BIN)/bower install

%.css: %.less
	$(NPM_BIN)/lessc $< > $@

build: danni.css

.PHONY: all install
