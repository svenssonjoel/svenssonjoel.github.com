
MD = $(wildcard *.md)

HTML = $(patsubst %.md,%.html,$(MD))

DIR   = $(notdir $(CURDIR))
TITLE = Zynq 7010 VHDL Blinky 
DESCRIPTION = How to get to blinky using Vivado, VHDL and a zynq 7010 (Zynqberry)
KEYWORDS =  Zynq Zynq7000 7000 7010 Zynq7010 Zynqberry Vivado VHDL Blinky getting to blinky walkthrough tutorial RTL
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
