
MD = $(wildcard *.md)

HTML = $(patsubst %.md,%.html,$(MD))

DIR   = $(notdir $(CURDIR))
TITLE = Implementation of Quasiquotation in LispBM
DESCRIPTION = A first go at adding quasiquotation to lispBM 
KEYWORDS = lisp quasiquotation quotation mcu microcontrollers interprerter stm32 nrf52 esp32  
YEAR  = 2020

all: $(HTML)

%.html: %.md

	(sed "s/###123###/$(TITLE)/g;s/###456###/$(DESCRIPTION)/g;s/###789###/$(KEYWORDS)/g" ../pre.html; cat  $<; sed "s/###YEAR###/$(YEAR)/g" ../post.html) | pandoc -f gfm -o $@ 

	mkdir -p ../../$(DIR)
	mkdir -p ../../$(DIR)/media
	cp *.html ../../$(DIR) 
	cp ./media/* ../../$(DIR)/media || true

clean:
	rm *.html
