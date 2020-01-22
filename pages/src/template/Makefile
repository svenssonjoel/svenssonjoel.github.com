
MD = $(wildcard *.md)

HTML = $(patsubst %.md,%.html,$(MD))

DIR   = $(notdir $(CURDIR))
TITLE = Template Page

all: $(HTML)

%.html: %.md

	(sed "s/###123###/$(TITLE)/g" ../pre.html; cat $< < ../post.html) | pandoc -f gfm -o $@ 

	mkdir -p ../../$(DIR)
	mkdir -p ../../$(DIR)/media
	cp *.html ../../$(DIR) 
	$(cp ./media/* ../../$(DIR)media || :)

clean:
	rm *.html
