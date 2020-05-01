

# Memory management

[LispBM](http://www.github.com/svenssonjoel/lispbm) is meant to be run
on resource constrained platforms like the STM32, NRF52, ESP32 or
similar. The dev-boards I have been running lispBM on so far range
between 192 and and 520KB of RAM memory (It would be nice to also try
something with a 64KB memory). Currently I don't really know at all how
much memory a running instance of lispBM is occupying, and this
is something I want to get a better idea of.

There are a number of things that use `malloc`ed areas of memory
currently. Examples are continuation stacks, symbol table, arrays and
strings.

So, currently I am working on a module for memory management within a
predefined range of memory addresses without using malloc. If this ends up
working, a next step is to try to move, heap, stacks and symbol table
to this managed area of memory and then to use it's allocation
features to dynamically allocate (up to a predefined limit) the memory
needed for arrays and such. I hope this helps somewhat in narrowing
down how much memory is being used. 

As usual, all hints and tips are much appreciated. 


## Implementation idea

My first idea was to look at some Operating Systems books and see if
they described any details of how `malloc` is implemented. This did
not reveal very much information that I thought was applicable to this
case, so I went for something that felt simple instead. 

The approach taken depends on having an area of memory for a bitmap
that contains memory usage information and another area of memory for
data.  For each 4 bytes within the data area there are 2 bits in the
bitmap for storage information. This means that If a data-storage area
of 8KB is desired, an extra 512Bytes of data are required for the bitmap. 

---
	8KB / 4B per Word = 2KWords 
	2K  / 4Bitpatterns per Byte = 512B
---

The 2 bit patterns in the bitmap have the following meanings: 
```
00 - Free or used word of memory 
01 - End of a range of allocated words
10 - Start of a range of allocated words 
11 - Starting and endpoint of a range of allocated words (1 allocated word)
``` 
These are defined as follows in the source code: 
```
/* Status bit patterns */
#define FREE_OR_USED  0  //00b
#define END           1  //01b
#define START         2  //10b
#define START_END     3  //11b
```

The state needed for the memory subsystem is defined as follows. 

```
uint32_t *bitmap = NULL;
uint32_t *memory = NULL;
uint32_t memory_size;  // in 4 byte words
uint32_t bitmap_size;  // in 4 byte words
unsigned int memory_base_address = 0;
```
The state variables are initialized using `memory_init` defined in `memory.c`. 


``` 
int memory_init(unsigned char *data, uint32_t data_size,
                unsigned char *bits, uint32_t bits_size) {

  if (data == NULL || bits == NULL) return 0;

  if (((unsigned int)data % 4 != 0) || data_size != 16 * bits_size || data_size % 4 != 0 ||
      ((unsigned int)bits % 4 != 0) || bits_size < 1 || bits_size % 4 != 0) {
    // data is not 4 byte aligned
    // size is too small
    // or size is not a multiple of 4
    return 0;
  }

  bitmap = (uint32_t *) bits;
  bitmap_size = bits_size / 4;

  for (uint32_t i = 0; i < bitmap_size; i ++) {
    bitmap[i] = 0;
  }

  memory = (uint32_t *) data;
  memory_base_address = (unsigned int)data;

  return 1;
}
```

`memory_init` starts out by checking that the pointers passed in for
the start of the data-storage area and bitmap are aligned to a
multiple of 4 bytes and that their size is a multiple of 4 bytes.
Those restrictions make it a lot easier to write the rest of the
code. A 4 byte bitmap contains information about 16 x 4 byte
data-words (that is 64 bytes). There is also a check that the size of
the bitmap is not 0 and that data-storage is 16 times the size of the
bitmap (in bytes). 

Next, the state is set and the bitmap memory is cleared. 


To convert from an address within the data-storage area and an index
into the bitmap and the other way around, two functions are defined.  
To convert an address to a bitmap index the `memory_base_address` is
subtracted from the address and the result of that is divided by 4
(using a shift instruction - shift left by 2). This results in an
index to a pair of bits in the bitmap and to access the specific bits they are 
located at position `bitmap_ix * 2` and `bitmap_ix * 2 + 1`. 

```
static inline unsigned int address_to_bitmap_ix(uint32_t *ptr) {
  return ((unsigned int)ptr - memory_base_address) >> 2;
}

static inline uint32_t *bitmap_ix_to_address(unsigned int ix) {
  return (uint32_t*)(memory_base_address + (ix << 2));
}
```

Conversion in the other direction is then done by multiplying the
bitmap index by 2 and adding the `memory_base_address`.


There are two functions for checking and setting the status of a word
in the bitmap. The function that looks up the status is called
`status` and the setter is called `set_status`. The `status` function
takes an index to a bit-pair in the bitmap and returns the status code
those bits represent. Likewise, `set_status` takes a bitmap index and
a status code to set at that index. Given the requirements on alignment
on the bitmap and the data-storage area these functions end up being
quite small. 

```
static inline unsigned int status(unsigned int i) {

  unsigned int ix = i << 1;          // * 2
  unsigned int word_ix = ix >> 5;    // / 32
  unsigned int bit_ix  = ix & 0x1F;  // % 32

  uint32_t mask = 3 << bit_ix;       // 000110..0
  return (bitmap[word_ix] & mask) >> bit_ix;
}

static inline void set_status(unsigned int i, uint32_t status) {
  unsigned int ix = i << 1;          // * 2
  unsigned int word_ix = ix >> 5;    // / 32
  unsigned int bit_ix  = ix & 0x1F;  // % 32

  uint32_t clr_mask = ~(3 << bit_ix);
  uint32_t mask = status << bit_ix;
  bitmap[word_ix] &= clr_mask;
  bitmap[word_ix] |= mask;
}
```

Given an index into the bitmap the actual bit-position of the first
bit in that bit-pair is obtained by multiplying the index by 2. Then
to figure out which 32Bit word that bit-pair sits in, the bit-position
is divided by 32. How far into that 32Bit word that the bit-pair is
located can be found as the remainder of a division by 32. Here, all
these operations are expressed using bitwise logic. 

After computing which 32bit word of the bitmap is of interest, a mask
is applied to obtain only the bits of interest. 



The function for allocating memory in the data-area is called
`memory_allocate` and takes a single parameter indicating the number
of 4 byte words. This function is a state machine performing a search
over the bitmap.

```
/* States */
#define INIT                 0
#define FREE_LENGTH_CHECK    1
#define SKIP                 2
#define ALLOC_DONE           0xF00DF00D
#define ALLOC_FAILED         0xDEADBEAF
```

The `memory_allocate` function loops over the bitmap checking the
status at each index as it moves along. In case a `FREE_OR_USED` is
found, the current state of the search is checked. If in the `INIT`
state the `FREE_OR_USED` word is actually free and a potential
starting index candidate is set. The state is changed to
`FREE_LENGTH_CHECK` and the loop continues.

When in the `FREE_LENGTH_CHECK` state, a `FREE_OR_USED` status also
indicates that the current word is free and the variable keeping track
of the current length is incremented. If the length is as long as the
requested number of words the search is done. 

The rest of the state transitions are easier. If `START` is found
state goes to `SKIP` and an `END` transitions to state `INIT` for
example. Really there should be some checks here for unexpected states
when discovering any given status to be a bit more robust against
a corrupt bitmap. 

```
uint32_t *memory_allocate(uint32_t num_words) {

  uint32_t start_ix = 0;
  uint32_t end_ix = 0;
  uint32_t free_length = 0;
  unsigned int state = INIT;

  for (unsigned int i = 0; i < (bitmap_size << 4); i ++) {
    if (state == ALLOC_DONE) break;

    switch(status(i)) {
    case FREE_OR_USED:
      switch (state) {
	  case INIT:
	    start_ix = i;
        if (num_words == 1) {
          end_ix = i;
          state = ALLOC_DONE;
        } else {
          state = FREE_LENGTH_CHECK;
          free_length = 1;
        }
        break;
      case FREE_LENGTH_CHECK:
        free_length ++;
        if (free_length == num_words) {
          end_ix = i;
          state = ALLOC_DONE;
        } else {
          state = FREE_LENGTH_CHECK;
        }
        break;
      case SKIP:
        break;
	  }
      break;
    case END:
      state = INIT;
      break;
    case START:
      state = SKIP;
      break;
    case START_END:
      state = INIT;
      break;
    default:
      return NULL;
      break;
    }
  }

  if (state == ALLOC_DONE) {
    if (start_ix == end_ix) {
      set_status(start_ix, START_END);
    } else {
      set_status(start_ix, START);
      set_status(end_ix, END);
    }
    return bitmap_ix_to_address(start_ix);
  }
  return NULL;
}
``` 

To free an allocated chunk of memory, the address of the chunk is
turned into a bitmap index. The status at this index should be either
`START` or `START_END` depending on if it is a multi-word chunk or
just a single word. The `START_END` case is the easier one, in this
case the `START_END` status is just overwritten with
`FREE_OR_USED`. The case for the status `START` needs to scan forwards
in the bitmap until the next status of `END` is found. Then both the
`START` and the `END` is overwritten with `FREE_OR_USED`.

``` 
int memory_free(uint32_t *ptr) {
  unsigned int ix = address_to_bitmap_ix(ptr);
  switch(status(ix)) {
  case START:
    set_status(ix, FREE_OR_USED);
    for (unsigned int i = ix; i < (bitmap_size << 4); i ++) {
      if (status(i) == END) {
    set_status(i, FREE_OR_USED);
    return 1;
      }
    }
    return 0;
  case START_END:
    set_status(ix, FREE_OR_USED);
    return 1;
  }

  return 0;
}
```
## Future work

Currently, figuring out memory usage is what is on the table. Future
work involves trying to place the symbol table and all continuation
stacks on a memory area managed by `memory_allocate` (or something
similar if this turns out to be a dead-end). While doing the rewrite
of the symbol table code that is needed for this, it would also be
nice to try to place constant symbols on flash memory rather than ram.

Having a granularity of 4 bytes as the smallest allocatable memory
area size means that there will be some waste when space for strings
are allocated. For now this waste is fine by me ;) 

This needs more testing. It also needs more checking for errors or
corrupted state of the bitmap. These corrupt states would be
discoverable as for example a `START` status encountered while looking
for an `END` status in the `memory_free` function. 

Thanks for reading! If you have constructive tips for improvements I
would love to hear them. 

___

[HOME](https://svenssonjoel.github.io)
