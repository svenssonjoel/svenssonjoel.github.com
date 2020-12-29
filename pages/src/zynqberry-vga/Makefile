
MD = $(wildcard *.md)

HTML = $(patsubst %.md,%.html,$(MD))

DIR   = $(notdir $(CURDIR))
TITLE = Zynqberry with VGA output using DIY DAC
DESCRIPTION = Zynqberry graphics output using a DIY VGA DAC
KEYWORDS =  Zynq Zynqberry zynq7000 zynq 7000 7010 VGA DAC graphics VHDL breadboard bread board experiment  
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
