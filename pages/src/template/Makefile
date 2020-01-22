
MD = $(wildcard *.md)

HTML = $(patsubst %.md,%.html,$(MD))

DIR := $(notdir $(CURDIR))

all: $(HTML)

%.html: %.md
	cat ../pre.html $< ../post.html | pandoc -f gfm -o $@

	mkdir -p ../../$(DIR)
	mkdir -p ../../$(DIR)/media
	cp *.html ../../$(DIR) 
	$(cp ./media/* ../../$(DIR)media || :)
