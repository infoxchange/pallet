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

serve:
	python3 -m http.server &
	xdg-open http://localhost:8000/

.PHONY: all install serve
