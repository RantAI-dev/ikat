# Build the manuscript.
#
# Tectonic is used rather than a TeX Live install: it fetches exactly the
# packages the document asks for and nothing else, so the build is the same on
# a laptop and in CI without a multi-gigabyte base image.
#
#   make          build main.pdf
#   make watch    rebuild on save
#   make clean    remove build output

TECTONIC ?= tectonic

main.pdf: main.tex
	$(TECTONIC) -X compile main.tex

.PHONY: watch clean
watch:
	$(TECTONIC) -X watch

clean:
	rm -f main.pdf main.aux main.log main.out main.bbl main.blg
