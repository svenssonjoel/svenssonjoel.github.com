
MD = $(wildcard *.md)

HTML = $(patsubst %.md,%.html,$(MD))

DIR   = $(notdir $(CURDIR))
TITLE = Assembler programming of ARM Cortex-M microcontrollers - Part 0 - Tools and basics
DESCRIPTION = Assembler programming of ARM Cortex-M microcontrollers 
KEYWORDS =  tools, basics, assembler, arm, cortex-m, cortex, cortex-m4, stm32, stm32f4, discovery, stm32f4-discovery
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
