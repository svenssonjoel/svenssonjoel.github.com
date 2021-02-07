
MD = $(wildcard *.md)

HTML = $(patsubst %.md,%.html,$(MD))

DIR   = $(notdir $(CURDIR))
TITLE = Assembler programming of ARM Cortex-M microcontrollers - Part 3 - Loops, arrays and functions
DESCRIPTION = Loop over a set of values and output them to a set LEDs. Will also implement a basic delay function.
KEYWORDS = STM32, STM32F4, buttons, gpio, LED, assembler, discovery, cortex, cortex-m, cortex-m4, loops, arrays, functions 
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
