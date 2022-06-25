
MD = $(wildcard *.md)

HTML = $(patsubst %.md,%.html,$(MD))

DIR   = $(notdir $(CURDIR))
TITLE = Defunctionalization of a continuation-passing style evaluator
DESCRIPTION = The LispBM evaluator is implemented in C in a defunctionalized continuation-passing style.
KEYWORDS = lisp, lispbm, scheme, defunctionalization, continuation-passing, evaluation, evaluator 
YEAR  = 2022
LINK  = https://svenssonjoel.github.io/pages-2022/$(DIR)/index.html


all: $(HTML)

%.html: %.md

	(sed "s/###123###/$(TITLE)/g;s/###456###/$(DESCRIPTION)/g;s/###789###/$(KEYWORDS)/g;s'###LINK###'$(LINK)'g" ../pre.html; cat  $<; sed "s/###YEAR###/$(YEAR)/g" ../post.html) | pandoc -f gfm -o $@ 

	mkdir -p ../../$(DIR)
	mkdir -p ../../$(DIR)/media
	cp *.html ../../$(DIR) 
	cp ./media/* ../../$(DIR)/media || true

clean:
	rm *.html
