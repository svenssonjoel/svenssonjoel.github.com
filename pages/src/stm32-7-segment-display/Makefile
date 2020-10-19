
MD = $(wildcard *.md)

HTML = $(patsubst %.md,%.html,$(MD))

DIR   = $(notdir $(CURDIR))
TITLE = Controlling a Seven Segment Display from an STM32 running ChibiOs
DESCRIPTION = How to interface a seven segment display with an STM32.
KEYWORDS = 7 segment display, Seven segment display, stm32, stm32f4, stm32f4-discovery, stm32f407-discovery, chibios, gpio, stm32 gpio, simple gpio, C programming, electronics
YEAR  = 2020
LINK  = https://svenssonjoel.github.io/pages/$(DIR)/index.html


all: $(HTML)

%.html: %.md

	(sed "s/###123###/$(TITLE)/g;s/###456###/$(DESCRIPTION)/g;s/###789###/$(KEYWORDS)/g;s'###LINK###'$(LINK)'g" ../pre.html; cat  $<; sed "s/###YEAR###/$(YEAR)/g" ../post.html) | pandoc -f gfm -o $@ 

	mkdir -p ../../$(DIR)
	mkdir -p ../../$(DIR)/media
	cp *.html ../../$(DIR) 
	cp ./media/* ../../$(DIR)/media || true

clean:
	rm *.html
