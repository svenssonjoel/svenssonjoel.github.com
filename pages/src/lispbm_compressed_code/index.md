

# Compressed Source Code for LispBM 

The ability to parse compressed source code in lispBM is mostly just
an experiment (for fun). I don't have any current example that
requires this feature but I imagine that it can be useful if programs
are to be stored on resource constrained platforms or transfered over
very limited bandwidth channels. Also, if some lispBM source should be
stored within the binary of the firmware for an embedded platform, it
makes sense that it is small and can be parsed without having to use
potentially large buffers for decompressing and parsing. 

Once the source code has been parsed into the heap, the source itself
could potentially be deleted but one can imagine a setup where the
runtime system can be restarted during operation and then need to
re-parse the source from memory. This could for example be done to
recover from some error. In this case it makes sense that the source
is stored and takes as small amount of memory as possible.

The compression used in lispBM is based on [Huffman
coding](https://en.wikipedia.org/wiki/Huffman_coding). The Huffman
coding results in a set of so-called prefix-free code
words. Prefix-free meaning that no code is the prefix of any other
code. Normally these codes are created from an *alphabet* with
associated frequencies. So that for each letter of the alphabet the
frequency states how common that letter is in the data that is to be
compressed. This means that letters with higher frequencies are given
shorter codes.

I use a Python library called Huffman to generate a mapping from
*keys* to *codes*.  Here the keys are lispBM syntactical entities and
the codes are a sequence of bits. If the codes are based on the
frequencies of letters in the currently compressed file, the key-code
mapping must somehow be encoded into the resulting compressed output
so that it can extracted and used at the decompression side. For this
reason, I have chosed to use a fixed key-code mapping that will be the
same for all compressed lispBM sources (unless I change the mapping
when adding features, this would mean that programs compressed in the
past become unusable). The Python script assigns frequencies to groups
of syntactical entities in way so that all entities in the group have
the same frequency. It then searches for a set of frequencies that
give the smallest number of bits in total for all the codes. I dont
know if this is a very good metric but it is how it is done
currently. Another approach would have been to get the frequencies
from a set representative lispBM programs and then just go with that
for all programs.

The alphabet that is used to generate the code contains single
character syntactic entities such as `"0"` and `"a"`, but also
composite ones like `"))))"` and `"define"`. This is based on the
assumption that multiple closing brackets is pretty common and that
being able to code something like `"define"` into just 6 bits is a
pretty good gain (6 bytes to 6 bits).

The Python script for generating the codes is located in the `utils`
directory and can be run as follows using python 3:

```
exec(open('gen_codes.py').read())
print(make_c())
```

This relies on the Huffman library that can be installed as follows:

```
pip3 install huffman
```



## The Key-Code Mapping

The key-code mapping generated using the python script is held in a 2
dimensional array that is `NUM_CODES` long and 2 elements wide, called
`codes`. So the keys and codes are accessible as `codes[ix][KEY]` and
`codes[ix][CODE]`, where `KEY` is defined as 0 and `CODE` as 1.

```
#define  KEY  0
#define  CODE 1
```

The code below is all generated using the `gen_codes.py` script and
defines the number of codes, maximum code lenght in bits and maximum
key length in characters. 

```
#define NUM_CODES 66
#define MAX_KEY_LENGTH 6
#define MAX_CODE_LENGTH 7
char *codes[NUM_CODES][2] = {
    { "9", "101101" },
    { "8", "101000" },
    { "7", "101001" },
    { "6", "100111" },
    { "5", "101100" },
    { "4", "101010" },
    { "3", "110000" },
    { "2", "101011" },
    { "1", "101111" },
    { "0", "101110" },
    { " ", "110001" },
    { "'", "111011" },
    { "\\", "110010" },
    { "\"", "110011" },
    { "#", "111100" },
    { ".", "110110" },
    { ">", "110111" },
    { "<", "111000" },
    { "=", "111101" },
    { "/", "111001" },
    { "*", "110100" },
    { "-", "110101" },
    { "+", "111010" },
    { "nil", "000001" },
    { "))))", "001010" },
    { ")))", "001011" },
    { "))", "000010" },
    { ")", "000011" },
    { "((", "1111110" },
    { "(", "1111111" },
    { "cdr", "001000" },
    { "car", "001001" },
    { "cons", "000100" },
    { "let", "000101" },
    { "define", "001100" },
    { "progn", "1111100" },
    { "quote", "1111101" },
    { "list", "000110" },
    { "if", "000111" },
    { "lambda", "000000" },
    { "z", "001110" },
    { "y", "001111" },
    { "x", "011111" },
    { "w", "010100" },
    { "v", "100000" },
    { "u", "100001" },
    { "t", "011100" },
    { "s", "100100" },
    { "r", "011001" },
    { "q", "100010" },
    { "p", "100110" },
    { "o", "001101" },
    { "n", "010110" },
    { "m", "010000" },
    { "l", "011110" },
    { "k", "010101" },
    { "j", "010011" },
    { "i", "011000" },
    { "h", "100011" },
    { "g", "011011" },
    { "f", "010001" },
    { "e", "010010" },
    { "d", "100101" },
    { "c", "011010" },
    { "b", "011101" },
    { "a", "010111" }
    };
```

There are no upper case letters among the keys, this is because lispBM
ignores case. Symbols expressed as `apa`, `Apa`, `APA` or any other
upper/lower case combo all refer to the same symbol:

```
# (define apa 1) 
> t
# apa
> 1
# APA
> 1
# aPa
> 1
# 
```
-----
*Edit 2020-02-15*

After writing this down I am now thinking that having very similar
lengths of the code words is probably not a very good idea at all. It
may be better to allow for the longer multibyte keys (such as
`define`) to have a higher number of code bits if it means that some
shorter keys can have really short codes.

So I tweaked the python script that generates the codes a bit and had
it generate this sequence of codes:

```
#define NUM_CODES 64
#define MAX_KEY_LENGTH 6
#define MAX_CODE_LENGTH 10
char *codes[NUM_CODES][2] = {
    { "nil", "010011100" },
    { "cdr", "0100111010" },
    { "car", "0100111011" },
    { "cons", "0100111100" },
    { "let", "0100111101" },
    { "define", "0100111110" },
    { "progn", "0100111111" },
    { "quote", "010011000" },
    { "list", "010011001" },
    { "if", "010011010" },
    { "lambda", "010011011" },
    { "((", "1110" },
    { "))", "001" },
    { ")", "1111" },
    { "(", "000" },
    { "z", "110110" },
    { "y", "011101" },
    { "x", "110100" },
    { "w", "101110" },
    { "v", "100110" },
    { "u", "011100" },
    { "t", "100111" },
    { "s", "101100" },
    { "r", "101101" },
    { "q", "101111" },
    { "p", "011110" },
    { "o", "011111" },
    { "n", "100011" },
    { "m", "110111" },
    { "l", "110000" },
    { "k", "100000" },
    { "j", "01000" },
    { "i", "101001" },
    { "h", "101010" },
    { "g", "011011" },
    { "f", "100100" },
    { "e", "101000" },
    { "d", "110010" },
    { "c", "100010" },
    { "b", "100101" },
    { "a", "110001" },
    { "9", "1101011" },
    { "8", "1000010" },
    { "7", "1000011" },
    { "6", "1010110" },
    { "5", "1010111" },
    { "4", "1100110" },
    { "3", "1100111" },
    { "2", "010010" },
    { "1", "1101010" },
    { "0", "0110101" },
    { " ", "0110011" },
    { "'", "0101011" },
    { "\\", "0101100" },
    { "\"", "0101110" },
    { "#", "0101111" },
    { ".", "0110001" },
    { ">", "0110010" },
    { "<", "0101000" },
    { "=", "0101101" },
    { "/", "0110100" },
    { "*", "0101010" },
    { "-", "0110000" },
    { "+", "0101001" }
    };
```

Now, There wont actually be an optimal sequence of codes when using
this approach at all since the codes are not based on statistics from
the currently compressed program. But given that all the codes here
contain fewer bits than the corresponding characters would (as
8bit chars), this approach will make most programs somewhat smaller.
since string literals are not compressed and the compressed data
contains the size in bits of the data, stored in the first 4 bytes, it
is possible to come up with obscure programs that are larger after
compressing them. For example the program consisting only of the
expression `"hello world"`, would be larger compressed than not.

-----


## Compression 

Since the keys can be one or more characters long, the longest
matching key must be found. The function below takes a string and
looks at the beginning of it and compares to all the keys in the
key-code mapping array. The index of the longest match is returned.
This function is later used in the compression routine to find the code
that matches the longest matching key.

```
int match_longest_key(char *string) {

  int longest_match_ix = -1;
  int longest_match_length = 0;
  int n = strlen(string);

  for (int i = 0; i < NUM_CODES; i ++) {
    int s_len = strlen(codes[i][KEY]);
    if (s_len <= n) {
      if (strncmp(codes[i][KEY], string, s_len) == 0) {
        if (s_len > longest_match_length) {
          longest_match_ix = i;
          longest_match_length = s_len;
        }
      }
    }
  }
  return longest_match_ix;
}
```

The `set_bit` function is used when creating the compressed output. It
can be used to set a bit to 1 or 0 at any position within a
byte. This function is used as a helper within the functions that emit
codes onto the output buffer of compressed data. 

```
void set_bit(char *c, char bit_pos, bool set) {
  char bval = 1 << bit_pos;
  if (set) {
    *c = *c | bval;
  } else {
    *c = *c & ~bval;
  }
}
```

Strings that occur in source code is a bit of a special case. Since
there are many ASCII characters missing from the set of keys, we
cannot compress arbitrary strings. So strings are stored uncompressed
within the otherwise compressed result. But since we are compressing
everything around the string, the position within the output buffer
where the uncompressed character will end up is not necessarily byte
aligned! A function is needed that can insert the bits corresponding
to a character at an arbitrary bit-addressed position within a buffer.

```
void emit_string_char_code(char *compressed, char c, int *bit_pos) {

  for (int i = 0; i < 8; i ++) {
    int byte_ix = (*bit_pos) / 8;
    int bit_ix  = (*bit_pos) % 8;
    bool s = (c & (1 << i));
    set_bit(&compressed[byte_ix], bit_ix, s);
    *bit_pos = *bit_pos + 1;
  }
}
```

The `emit_string_char_code` function takes a pointer to the output
buffer (called `compressed`), a character to insert into the buffer
and a bit-position that indicates where to put the character bits.
The function loops over the bits in the character, computes the byte
index and bit index in the output buffer and copies bits from the
character to be buffer one at a time.

Emitting a compressed code is quite similar to the function above that
emits an arbitrary character.  But instead of looping for 8 iteration
it performs as many iterations as there are bits in the code. It each
iteration it checks the code to see if a 1 or a 0 should be added to
the compressed buffer.

```
void emit_code(char *compressed, char *code, int *bit_pos) {
  int n = strlen(code);

  for (int i = 0; i < n; i ++) {
    int byte_ix = (*bit_pos) / 8;
    int bit_ix  = (*bit_pos) % 8;
    bool s = (code[i] == '1');
    set_bit(&compressed[byte_ix], bit_ix, s);
    *bit_pos = *bit_pos + 1;
  }
}
```

The next function calculates how many bits the result of compressing a
given string would need. This function mimics the actual compression
function but it does not output any compressed result. Instead it just
counts the number of bits in each code that the actual compressor
would have added for each key found on the input string. This function
exists so that we can allocate an array large enough without having to
overshoot and waste space or undershoot and be forced to `realloc`. 

I will not go into details of this function, rather save that for the
following function that performs the actual compression. 

```
int compressed_length(char *string) {
  unsigned int i = 0;

  unsigned int n = strlen(string);
  int comp_len = 0; // in bits

  bool string_mode = false;
  bool gobbling_whitespace = false;

  while (i < n) {
    if (string_mode) {
      if (string[i] == '\"'  &&
          !(string[i-1] == '\\')) {
        string_mode = false;
        comp_len += 8;
        i++;
      } else {
        comp_len += 8;
        i++;
      }

    } else {

      // Gobble up any comments
      if (string[i] == ';' ) {
        while (string[i] && string[i] != '\n') {
          i++;
        }
        continue;
      }

      if ( string[i] == '\n' ||
           string[i] == ' '  ||
           string[i] == '\t' ||
           string[i] == '\r') {
        gobbling_whitespace = true;
        i ++;
        continue;
      } else if (gobbling_whitespace) {
        gobbling_whitespace = false;
        i--;
      }

      if (string[i] == '\"') string_mode = true;

      int ix;
      if (isspace(string[i])) {
        ix = match_longest_key(" ");
      } else {
        ix = match_longest_key(string + i);
      }

      if (ix == -1)return -1;
      int code_len = strlen(codes[ix][1]);
      comp_len += code_len;
      i += strlen(codes[ix][0]);
    }
  }
  return comp_len;
}
```

Now it is time to compress a string of source code. The compression
function takes a string and a pointer to a `unsigned int` that will
hold the size of the compressed output in bytes.

First the compression function runs the function that calculates the
compressed size of the string. This size is used to allocate an
appropriately sized output buffer but the size value is also added the
output and occupies the first 4 bytes of the result buffer. When
uncompressing this information is used to know how many bits should be
used in the decompression. This is because the output buffer will very
likely not be exactly the 1/8 of the number of bits.

There is a boolean called `string_mode` that, if set, indicates that a
lispBM string is being copied uncompressed into the output buffer.
The `string_mode` is entered if a `"` character is encountered in the
input data and is left again on the next occurrence of an unescaped
`"` character.

The compressor also ignores comments and gobbles up multiple
whitespace and replacing them by a single whitespace. 

```
char *compression_compress(char *string, unsigned int *res_size) {

  uint32_t c_size_bits = compressed_length(string);

  uint32_t c_size_bytes = 4 + (c_size_bits/8);
  if (c_size_bits % 8 > 0) {
    c_size_bytes += 1;
  }
  
  uint32_t header_value = c_size_bits;

  if (header_value == 0) return NULL;

  char *compressed = malloc(c_size_bytes);
  if (!compressed) return NULL;
  memset(compressed, 0, c_size_bytes);
  *res_size = c_size_bytes;
  int bit_pos = 0;

  compressed[0] = (unsigned char)header_value;
  compressed[1] = (unsigned char)(header_value >> 8);
  compressed[2] = (unsigned char)(header_value >> 16);
  compressed[3] = (unsigned char)(header_value >> 24);
  bit_pos = 32;

  bool string_mode = false;
  bool gobbling_whitespace = false;
  unsigned int n = strlen(string);
  unsigned int i = 0;

  while (i < n) {
    if (string_mode) {

      if (string[i] == '\"' &&
          !(string[i-1] == '\\')) {
        emit_string_char_code(compressed, '\"', &bit_pos);
        i ++;
        string_mode = false;
        continue;
      } else {
        emit_string_char_code(compressed, string[i], &bit_pos);
        i++;
      }

    } else {

      // Gobble up any comments
      if (string[i] == ';' ) {
        while (string[i] && string[i] != '\n') {
          i++;
        }
        continue;
      }

      // gobble up whitespaces
      if ( string[i] == '\n' ||
           string[i] == ' '  ||
           string[i] == '\t' ||
           string[i] == '\r') {
        gobbling_whitespace = true;
        *(string + i) = ' ';
        i ++;
        continue;
      } else if (gobbling_whitespace) {
        gobbling_whitespace = false;
        i--;
      }

      /* Compress string-starting " character */
      if (string[i] == '\"') {
        string_mode = true;
      }
      int ix = match_longest_key(&string[i]);

      if (ix == -1) return NULL;

      emit_code(compressed, codes[ix][CODE], &bit_pos);

      i += strlen(codes[ix][0]);
    }
  }

  return compressed;
}
```

After the special case of `string_mode` handling and comment-eating,
the actual piece of code that does compression is performed. Here the
index of the longest matching key is found and the corresponding code
is emited.

## Decompression

One goal of the decompression algorithm is that it should be possible
to extract data from the compressed buffer in an incremental way using
only a small buffer of storage. The idea is that the decompression
should slot into the stream abstraction used in the tokenizer and
parser in `tokpar.c`. The `peek` operation used in the tokenizer
however can lead to a situation where the compressed data has to be
decompressed more than once as the small buffer is filled up. This is
a trade-off between memory usage and compute resources.  

Decompression is performed by reading bits out from a buffer of
compressed data while trying to match with the longest code word in
the `codes` array. The `match_longest_code` function performs this
matching and returns the index within the `codes` array of the longest
match starting from position `start_bit` while not overshooting
`total_bits` (which is the total length in bits of the compressed
data). 

```
int match_longest_code(char *string, uint32_t start_bit, unsigned int total_bits) {

  unsigned int bits_left = total_bits - start_bit;
  int longest_match_ix = -1;
  int longest_match_length = 0;

  for (int i = 0; i < NUM_CODES; i++) {
    int s_len = strlen(codes[i][CODE]);
    if ((unsigned int)s_len <= bits_left) {
      bool match = true;
      for (uint32_t b = 0; b < (unsigned int)s_len; b ++) {
        unsigned int byte_ix = (start_bit + b) / 8;
        unsigned int bit_ix  = (start_bit + b) % 8;

        char *code_str = codes[i][CODE];

        if (((string[byte_ix] & (1 << bit_ix)) ? '1' : '0') !=
             code_str[b]) {
          match = false;
        }
      }
      if (match && (s_len > longest_match_length)) {
        longest_match_length = s_len;
        longest_match_ix = i;
      }
    }
  }
  return longest_match_ix;
}
```

Just like in the compression case, where there was an `emit_code`
function, here there is an `emit_key`. The `emit_key` function is a
bit simper though as it is outputting bytes and not bits. So, given a
destination buffer, a key entry from the `codes` array and a character
position in the output, `nk` characters are copied to the destination.


```
void emit_key(char *dest, char *key, int nk, unsigned int *char_pos) {

  for (int i = 0; i < nk; i ++) {
    dest[*char_pos] = key[i];
    *char_pos = *char_pos + 1;
  }
}
```

The `read_character` function below is a "helper" for the case when
the compressed data contains an lispBM string that is stored
uncompressed. In this case 8 bits will be read, starting from an
arbitrary bit-address in the compressed data and returned as a char.

```
char read_character(char *src, unsigned int *bit_pos) {

  char c = 0;

  for (int i = 0; i < 8; i ++) {
    int byte_ix = (*bit_pos)/8;
    int bit_ix  = (*bit_pos)%8;
    bool s = src[byte_ix] & (1 << bit_ix);
    set_bit(&c, i, s);
    *bit_pos = *bit_pos + 1;
  }
  return c;
}
```

To enable incremental or piece-wise decompression of the compressed
data, some state must be managed by the code that utilizes the
decompressor. This state keeps track of the current bit-address within
the compressed data where the next code will be read from as well as
the total length in bits of the compressed data.  It also remembers if
`string_mode` has been entered, which means that uncompressed
characters are read from the data. If in string mode, the last read
character is remembered to be able to figure out if a `"` character
encountered was escaped or not. It also holds a pointer to the source data. 

```
typedef struct {
  uint32_t compressed_bits;
  uint32_t i;
  bool string_mode;
  char last_string_char;
  char *src;
} decomp_state; 
```

The following function is called to initialize a decompression state
on a given compressed data buffer. The total number of bits are
extracted from the first 4 bytes of the data and the current bit index
is set to 32 (start decompressing after the length info). Initially
`string_mode` is false as a string starting character must be found to
enter that mode.

```
void compression_init_state(decomp_state *s, char *src) {
  memcpy(&s->compressed_bits, src, 4);
  s->i = 32;
  s->string_mode = false;
  s->last_string_char = 0;
  s->src = src;
}
```

The incremental decompressor extracts one key from the compressed data
on each call. It takes a `decomp_state`, a destination buffer of size
`dest_n`.  The caller of this function is responsible for ensuring
that there is enough space in the destination buffer for the longest
possible key.

```
int compression_decompress_incremental(decomp_state *s, char *dest_buff, unsigned int dest_n) {

  memset(dest_buff, 0, dest_n);
  uint32_t char_pos = 0;

  if (s->i < s->compressed_bits + 32) {
     if (s->string_mode) {
      char c = read_character(s->src, &s->i);
      if (c == '\"') {
        if (s->last_string_char != '\\') {
          s->string_mode = false;
          s->last_string_char = 0;
        }
      }
      s->last_string_char = c;
      dest_buff[0] = c;
      return 1;
    }

    int ix = match_longest_code(s->src, s->i, (s->compressed_bits + 32));
    if (ix == -1) {
      return -1;
    }

    if( strlen(codes[ix][KEY]) == 1 &&
        strncmp(codes[ix][KEY], "\"", 1) == 0) {
      s->string_mode = true;
      s->last_string_char = 0;
    }

    int n_bits_decoded = strlen(codes[ix][CODE]);
    emit_key(dest_buff, codes[ix][KEY], strlen(codes[ix][KEY]), &char_pos);
    s->i+=n_bits_decoded;
    return char_pos;

  } else {
    return 0;
  }
}
```

If the end of the compressed data is reached this function returns
0. Otherwise it checks if the state indicates that we are in
`string_mode`, in which case it reads a character from the data and
checks if that character is the string terminating `"`. This is where
the `last_string_char` comes in and we can check that it is not an
escaped `"` that is part of the string. If `string_mode` is exited,
the `last_string_char` is reset to 0 so that there wont be a potential
escape character left there when entering `string_mode` the next time.

If not in `string_mode`, the index of the longest matching code is
found.  This could potentially be the `"` char that signals entry to
`string_mode` but if it is not a key is emitted to the destination
buffer. 


Incremental decompression is used to hook the reading of compressed
source up to the tokenizer. It could of course be useful to have a
decompress function that decompresses the entire code on the fly. This
is the function shown below.  It repeatedly calls the incremental
decompressor until all data has been decompressed.

```
bool compression_decompress(char *dest, uint32_t dest_n, char *src) {

  unsigned int char_pos = 0;

  char dest_buff[32];
  int num_chars = 0;
  decomp_state s;

  memset(dest, 0, dest_n);

  compression_init_state(&s, src);

  while (true) {
    
    num_chars = compression_decompress_incremental(&s, dest_buff, 32);
    if (num_chars == 0) break;
    if (num_chars == -1) return false; 
    
    for (int i = 0; i < num_chars; i ++) {
      dest[char_pos++] = dest_buff[i];
    }
  }
  return true;
}
```
## Parsing Compressed Code 

To implement the parser on compressed code one needs to provide
implementations of the `more`, `get`, `peek` and `drop` functions in
the `tokenizer_char_stream` interface. A tokenizer state with
additional information is also needed. For example, this tokenizer
state contains a `decomp_state`, a decompression buffer, number of
bytes in the decompression buffer and the position in the buffer that
the next `get` operation should return. 

```
#define DECOMP_BUFF_SIZE 32
typedef struct {
  decomp_state ds;
  char decomp_buff[DECOMP_BUFF_SIZE];
  int  decomp_bytes;
  int  buff_pos;
} tokenizer_compressed_state;
```

Checking if there is more to read from the character stream is done by
checking if there is something left in the decompression buffer or it
there is still compressed data to decompress in the compressed array.

```
bool more_compressed(tokenizer_char_stream str) {
  tokenizer_compressed_state *s = (tokenizer_compressed_state*)str.state;
  bool more =
    (s->ds.i < s->ds.compressed_bits + 32) ||
    (s->buff_pos < s->decomp_bytes);
  return more;
}
```

Getting an item from the compressed stream starts out by checking if
there is anything left to get. If there is, we also check if the
decompression buffer is empty, if so it is refilled using the
incremental decompressor. Then a character is returned from the
decompression buffer. 

```
char get_compressed(tokenizer_char_stream str) {
  tokenizer_compressed_state *s = (tokenizer_compressed_state*)str.state;

  if (s->ds.i >= s->ds.compressed_bits + 32 &&
      (s->buff_pos >= s->decomp_bytes)) {
    return 0;
  }

  if (s->buff_pos >= s->decomp_bytes) {
    int n = compression_decompress_incremental(&s->ds, s->decomp_buff,DECOMP_BUFF_SIZE);
    if (n == 0) {
      return 0;
    }
    s->decomp_bytes = n;
    s->buff_pos = 0;
  }
  char c = s->decomp_buff[s->buff_pos];
  s->buff_pos += 1;
  return c;
}
```

Peeking into a compressed character stream is a bit interesting. We
have to be able to peek arbitrarily far into the stream (there could
for example be a very long string that needs its length measured). But
being able to peek arbitrarily far into the streams means the
decompression buffer will not be enough. To solve this, before doing
any peeking the current tokenizer state is saved to a backup, then we
can read off an arbitrary number of characters from the stream using
`get` (which now ruins the state), after getting the peek-depth of
characters the state can be restored from the backup and it is as if
the get operations never happened. 

```
char peek_compressed(tokenizer_char_stream str, unsigned int n) {
  tokenizer_compressed_state *s = (tokenizer_compressed_state*)str.state;

  tokenizer_compressed_state old;

  memcpy(&old, s, sizeof(tokenizer_compressed_state));

  char c = get_compressed(str);;
  for (unsigned int i = 1; i <= n; i ++) {
    c = get_compressed(str);
  }

  memcpy(str.state, &old, sizeof(tokenizer_compressed_state));
  return c;
}
```

Dropping is just getting but disregarding the output. 

```
void drop_compressed(tokenizer_char_stream str, unsigned int n) {
  for (unsigned int i = 0; i < n; i ++) {
    get_compressed(str);
  }
}
```

With these functions in place a parser for compressed character
streams can be implemented by setting up a
`tokenizer_compressed_state` and a `tokenizer_char_stream` that is
based on the functions defined above. Then run `parse_program` on the
compressed character stream.

```
VALUE tokpar_parse_compressed(char *bytes) {

  tokenizer_compressed_state ts;

  ts.decomp_bytes = 0;
  memset(ts.decomp_buff, 0, 32);
  ts.buff_pos = 0;

  compression_init_state(&ts.ds, bytes);

  tokenizer_char_stream str;
  str.state = &ts;
  str.more = more_compressed;
  str.get = get_compressed;
  str.peek = peek_compressed;
  str.drop = drop_compressed;

  return parse_program(str);
}
```
___

[HOME](https://svenssonjoel.github.io)
