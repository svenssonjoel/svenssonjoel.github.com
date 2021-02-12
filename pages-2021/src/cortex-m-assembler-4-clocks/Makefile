
MD = $(wildcard *.md)

HTML = $(patsubst %.md,%.html,$(MD))

DIR   = $(notdir $(CURDIR))
TITLE = Assembler programming of ARM Cortex-M microcontrollers - Part 4 - Clock configuration
DESCRIPTION = Configuring the stm32f4 to run at 168MHz
KEYWORDS = STM32, STM32F4, buttons, gpio, LED, assembler, discovery, cortex, cortex-m, cortex-m4, loops, arrays, functions, baremetal, bare metal, bare-metal
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
