

SUBDIRS := $(wildcard */.)

all: $(SUBDIRS)
	echo $(SUBDIRS)
	echo $@
	echo $<
	$(MAKE) -C $<

.PHONY: all $(SUBDIRS)
