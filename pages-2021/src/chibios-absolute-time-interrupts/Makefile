
MD = $(wildcard *.md)

HTML = $(patsubst %.md,%.html,$(MD))

DIR   = $(notdir $(CURDIR))
TITLE = Absolute time interrupts, mailboxes and memorypools in ChibiOS
DESCRIPTION = An experiment with chibios and timer interrupt at an absolutely specified count. 
KEYWORDS =  Chibios, STM32F4, Counter, Timer, Interrupt, Memorypool, Mailbox
YEAR  = 2021
LINK  = https://svenssonjoel.github.io/pages-2021/$(DIR)/index.html


all: $(HTML)

%.html: %.md

	(sed "s/###123###/$(TITLE)/g;s/###456###/$(DESCRIPTION)/g;s/###789###/$(KEYWORDS)/g;s'###LINK###'$(LINK)'g" ../pre.html; cat  $<; sed "s/###YEAR###/$(YEAR)/g" ../post.html) | pandoc -f gfm -o $@ 

	mkdir -p ../../$(DIR)
	mkdir -p ../../$(DIR)/media
	cp *.html ../../$(DIR) 
	cp ./media/* ../../$(DIR)/media || true

clean:
	rm *.html
