
MD = $(wildcard *.md)

HTML = $(patsubst %.md,%.html,$(MD))

DIR   = $(notdir $(CURDIR))
TITLE = Control an 8x8 LED Matrix from a STM32 and ChibiOS
DESCRIPTION = 8x8 LED Matrix control from STM32 and ChibiOS
KEYWORDS =  8x8 LED MATRIX STM32 STM32F4 STM32F407 STM32F407G DISCOVERY STM32-DISCOVERY STM32F407G-DISC1 CHIBIOS SCROLLING TEXT TEXTSCROLLER
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
