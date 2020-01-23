
MD = $(wildcard *.md)

HTML = $(patsubst %.md,%.html,$(MD))

DIR   = $(notdir $(CURDIR))
TITLE = An Overview of the Four Motor Robot Controller (FMRC) Project
YEAR  = 2020

all: $(HTML)

%.html: %.md

	(sed "s/###123###/$(TITLE)/g" ../pre.html; cat $<; sed "s/###YEAR###/$(YEAR)/g" ../post.html) | pandoc -f gfm -o $@ 

	mkdir -p ../../$(DIR)
	mkdir -p ../../$(DIR)/media
	cp *.html ../../$(DIR) 
	cp ./media/* ../../$(DIR)media || true

clean:
	rm *.html
