

# How lispBM Parses Source Code

I really didn't want to have to write a tokenizer and parser *by hand*
for lispBM. Writing something like this is not what I feel entirely
comfortable with. But, it is important to step out of the comfort zone
now and then. 

For a long time the lispBM parser was implemented using the
[MPC](https://github.com/orangeduck/mpc) library. This worked very
well and all I had to do was come up with a set of regular expressions
that expressed what kind of syntactical objects the parser should be
able to recognize and specify up how these were to be related to
eachother in a tree structure (the parse tree). With that little bit
of code in place MPC provided the parser that read source code
returned a tree data structure. Then a little bit of code is needed to
traverse the tree and generate the heap structure. On the
microcontrollers that lispBM target, memory is a valuable resource and
MPC was a bit hungry on that resource. So that is why this code came
to be. 

The code for parsing lispBM source into heap representations is
located in the files `tokpar.c` and `tokpar.h`.  The parser can handle
parsing both plain text source code and compressed source data. This
text will however just show the plain text case, saving compression
and parsing of compressed code for later.


The parsing is done by reading *tokens* from the source code. The
tokens represent, in a way, complete syntactical *objects*. In the
lispBM tokenizer that means that for example `(` is one token and
`3.14159` is another. That is, a token can be one or more consecutive
characters. The parser then uses these tokens read out from the text
to produce corresponding heap representation.

If you are interested in more of an overview of what lispBM is, read
[here](../lispbm_current_status/index.html).

## Plumbing 

The file `tokpar.c` starts off by defining a few names for the
different tokens that make up valid lispBM programs. A value to
represent error is also defined as well as a value that signals that
the end of the source code has been reached.


```
#define TOKOPENPAR      0
#define TOKCLOSEPAR     1
#define TOKQUOTE        2
#define TOKSYMBOL       3
#define TOKINT          4
#define TOKUINT         5
#define TOKBOXEDINT     6
#define TOKBOXEDUINT    7
#define TOKBOXEDFLOAT   8
#define TOKSTRING       9
#define TOKCHAR         10
#define TOKENIZER_ERROR 1024
#define TOKENIZER_END   2048
```

There is a function called `next_token` that returns the next token in
the source string. The return value is of the following type:

``` 
typedef struct {

  unsigned int type;

  unsigned int text_len;
  union {
    char  c;
    char  *text;
    INT   i;
    UINT  u;
    FLOAT f;
  }data;
} token;
```

the `token` type can represent all the valid tokens using the `union`
field in the `struct`. There is a `type` field to differentiate between
types this will be set to one of the defined values above. If there are
no more tokens the `type` field is set to `TOKENIZER_END` and if there
is an error it will be `TOKENIZER_ERROR`.


We also need to keep track of where in the source text we are
currently reading tokens from. This state is managed by a `struct`
called `tokenizer_state`, holding a pointer to the string and an index
(`pos`).

```
typedef struct {
  char *str;
  unsigned int pos;
} tokenizer_state;
``` 

The next `struct`, called `tokenizer_char_stream` exists with the
purpose of making code reuse between the tokenizer for plain strings
and compressed strings better. It is an abstracted representation of a
stream of characters that we can `peek` into, `drop` from and so on.

```
typedef struct tcs{
  void *state;
  bool (*more)(struct tcs);
  char (*get)(struct tcs);
  char (*peek)(struct tcs, int);
  void (*drop)(struct tcs, int);
} tokenizer_char_stream;
```

The `void *state` can be instantiated with either the
`tokenizer_state` seen above or another variant of state for
compressed source that also contains for example a decompression
buffer. 


Then a set of functions that work on any `tokenizer_char_stream` are
defined to make the code more readable.  

```
bool more(tokenizer_char_stream str) {
  return str.more(str);
}

char get(tokenizer_char_stream str) {
  return str.get(str);
}

char peek(tokenizer_char_stream str,int n) {
  return str.peek(str,n);
}

void drop(tokenizer_char_stream str,int n) {
  str.drop(str,n);
}
``` 

And the implementation of the different stream functions for the plain
string case are defined as follows. When creating a
`tokenizer_char_stream` for use on plain strings the function pointers
in the struct has to be pointed to these (or similar) functions. 

```
bool more_string(tokenizer_char_stream str) {
  tokenizer_state *s = (tokenizer_state*)str.state;
  return s->str[s->pos] != 0;
}

char get_string(tokenizer_char_stream str) {
  tokenizer_state *s = (tokenizer_state*)str.state;
  char c = s->str[s->pos];
  s->pos = s->pos + 1;
  return c;
}

char peek_string(tokenizer_char_stream str, int n) {
  tokenizer_state *s = (tokenizer_state*)str.state;
  // TODO error checking ?? how ?
  char c = s->str[s->pos + n];
  return c;
}

void drop_string(tokenizer_char_stream str, int n) {
  tokenizer_state *s = (tokenizer_state*)str.state;
  s->pos = s->pos + n;
}
``` 

## Tokenizer

That is what is needed when it comes to plumbing (and *helper
functions*) and it is time to implement the tokenizer. First off there
is one function per token. These functions can be thought of as a *try to tokenize
as* functions. These all return an integer that is `0` in case it
could not read its dedicated kind of token from the head of the
stream. If the token can be read from the stream, the number of
consumed characters is returned.

In the case of the `(`, `)` and `'` tokens, these functions return `1`
if the sought token is there. They produce no other data. 

```
int tok_openpar(tokenizer_char_stream str) {
  if (peek(str,0) == '(') {
    drop(str,1);
    return 1;
  }
  return 0;
}

int tok_closepar(tokenizer_char_stream str) {
  if (peek(str,0) == ')') {
    drop(str,1);
    return 1;
  }
  return 0;
}

int tok_quote(tokenizer_char_stream str) {
  if (peek(str,0) == '\'') {
    drop(str,1);
    return 1;
  }
  return 0;
}
```

Symbols are made up of a set of allowed characters. The characters
that are allowed in the first position of a symbol is different from
those allowed in any of the other positions. The following two boolean
functions return `true` for valid symbol characters and `false`
otherwise, of course.

``` 
bool symchar0(char c) {
  const char *allowed = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ+-*/=<>";

  int i = 0;
  while (allowed[i] != 0) {
    if (c == allowed[i++]) return true;
  }
  return false;
}

bool symchar(char c) {
  const char *allowed = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789+-*/=<>";

  int i = 0;
  while (allowed[i] != 0) {
    if (c == allowed[i++]) return true;
  }
  return false;
}
```

In addition to returning the number of characters consumed, the rest
of the *try to tokenize* functions also provide a value in the case
of success. 

``` 
int tok_symbol(tokenizer_char_stream str, char** res) {

  if (!symchar0(peek(str,0)))  return 0;

  int i = 0;
  int len = 1;
  int n = 0;

  while (symchar((peek(str,len)))) {
    len++;
  }

  *res = malloc(len+1);
  memset(*res,0,len+1);

  for (i = 0; i < len; i ++) {
    (*res)[i] = tolower(get(str));
    n++;
  }
  return n;
}
``` 

The `tok_symbol` function checks if the next character in the stream
is a valid beginning character for a symbol. If it is the function
loops through the string as long as each character it looks at is a
valid *inner* symbol character. As soon as a non symbol character is
found the loop exits and the valid characters are taken out from the
stream (using `get`) and added to the result `res` pointer
provided. From this code you can also see that `tolower` is used on
the symbols. This means that the strings `APA` and `apa` refer to the
same symbol in lispBM. 

```
int tok_string(tokenizer_char_stream str, char **res) {

  int i = 0;
  int n = 0;
  int len = 0;
  if (!(peek(str,0) == '\"')) return 0;

  get(str); // remove the " char
  n++;

  // compute length of string
  while (peek(str,len) != 0 &&
	 peek(str,len) != '\"') {
    len++;
  }

  // str ends before tokenized string is closed.
  if ((peek(str,len)) != '\"') {
    return 0;
  }

  // allocate memory for result string
  *res = malloc(len+1);
  memset(*res, 0, len+1);

  for (i = 0; i < len; i ++) {
    (*res)[i] = get(str);
    n++;
  }

  get(str);  // throw away the "
  return (n+1);
}
``` 

The `tok_string` function is very similar to `tok_symbol`. If there is
an `"` character at the head of the stream, everything onwards (until
there is another `"` is read into the result. The `tok_string`
function needs to be improved a bit to recognize some kind of escaped
characters (like newline), which it currently doesn't. If you enter
the string `"hello \n"` or even `(print "hello \n")`, for example,
into the REPL it will reply with `hello \n` and not `hello` followed
by a line break. As we will see below character literals are given to
the REPL as `\#a`, for the character `a` and the newline character is
expressed as `\#newline`, maybe it would make sense to also escape the
newline character in the same way in a string? I've noticed that
emacs-lisp uses the syntax `?a` for the character a. Maybe going over
to the same representation here make sense.
 

So, currently, the `tok_char` function below looks for the character
syntax, that is `\#`, followed by a character.

``` 
int tok_char(tokenizer_char_stream str, char *res) {

  int count = 0;
  if (peek(str,0) == '\\' &&
      peek(str,1) == '#' &&
      peek(str,2) == 'n' &&
      peek(str,3) == 'e' &&
      peek(str,4) == 'w' &&
      peek(str,5) == 'l' &&
      peek(str,6) == 'i' &&
      peek(str,7) == 'n' &&
      peek(str,8) == 'e') {
    *res = '\n';
    drop(str,9);
    count = 9;
  } else if (peek(str,0) == '\\' &&
	     peek(str,1) == '#' &&
	     isgraph(peek(str,2))) {
    *res = peek(str,2);
    drop(str,3);
    count = 3;
  }
  return count;
}
```

The tokenizers for 28 and 32 bit signed and unsigned integers are all
very similar.  Some of them, 28bit unsigned, 32bit signed and 32bit
unsigned, require that the number is followed by a type qualifier such
as `u28`, `u32` or `i32` so that the appropriate type can be set for,
and the suitable cons cell structure can be allocated.

``` 
int tok_i(tokenizer_char_stream str, INT *res) {

  INT acc = 0;
  int n = 0;

  while ( peek(str,n) >= '0' && peek(str,n) <= '9' ){
    acc = (acc*10) + (peek(str,n) - '0');
    n++;
  }

  // Not needed if strict adherence to ordering of calls to tokenizers.
  if (peek(str,n) == 'U' ||
      peek(str,n) == 'u' ||
      peek(str,n) == '.' ||
      peek(str,n) == 'I') return 0;

  drop(str,n);
  *res = acc;
  return n;
}

int tok_I(tokenizer_char_stream str, INT *res) {
  INT acc = 0;
  int n = 0;

  while ( peek(str,n) >= '0' && peek(str,n) <= '9' ){
    acc = (acc*10) + (peek(str,n) - '0');
    n++;
  }

  if (peek(str,n) == 'i' &&
      peek(str,n+1) == '3' &&
      peek(str,n+2) == '2') {
    *res = acc;
    drop(str,n+3);
    return n+3;
  }
  return 0;
}

int tok_u(tokenizer_char_stream str, UINT *res) {
  UINT acc = 0;
  int n = 0;

  while ( peek(str,n) >= '0' && peek(str,n) <= '9' ){
    acc = (acc*10) + (peek(str,n) - '0');
    n++;
  }

  if (peek(str,n) == 'u' &&
      peek(str,n+1) == '2' &&
      peek(str,n+2) == '8' ) {
    *res = acc;
    drop(str,n+3);
    return n+3;
  }
  return 0;
}
``` 

32bit values can also be entered using hexadecimal notation. The
tokenizer for this case is a little bit larger to also handle reading
this hexidecimal notation.

```
int tok_U(tokenizer_char_stream str, UINT *res) {
  UINT acc = 0;
  int n = 0;

  // Check if hex notation is used
  if (peek(str,0) == '0' &&
      (peek(str,1) == 'x' || peek(str,1) == 'X')) {
    n+= 2;
    while ( (peek(str,n) >= '0' && peek(str,n) <= '9') ||
	    (peek(str,n) >= 'a' && peek(str,n) <= 'f') ||
	    (peek(str,n) >= 'A' && peek(str,n) <= 'F')){
      UINT val;
      if (peek(str,n) >= 'a' && peek(str,n) <= 'f') {
	val = 10 + (peek(str,n) - 'a');
      } else if (peek(str,n) >= 'A' && peek(str,n) <= 'F') {
	val = 10 + (peek(str,n) - 'A');
      } else {
	val = peek(str,n) - '0';
      }
      acc = (acc * 0x10) + val;
      n++;
    }
    *res = acc;
    drop(str,n);
    return n;
  }

  // check if nonhex
  while ( peek(str,n) >= '0' && peek(str,n) <= '9' ){
    acc = (acc*10) + (peek(str,n) - '0');
    n++;
  }

  if (peek(str,n) == 'u' &&
      peek(str,n+1) == '3' &&
      peek(str,n+2) == '2') {
    *res = acc;
    drop(str,n+3);
    return n+3;
  }
  return 0;
}
```

Floating point numbers are identified by there being a decimal point
somewhere within the number. So the tokenizer reads numerals for as
long as possible, then at some point it should find a `.` and some
following digits. Once the string has been identified as a float
value, the characters involved are extracted into a buffer and
converted to a float using `strtod`. 

```
int tok_F(tokenizer_char_stream str, FLOAT *res) {

  int n = 0;
  int m = 0;
  char fbuf[256];

  while ( peek(str,n) >= '0' && peek(str,n) <= '9') n++;

  if ( peek(str,n) == '.') n++;
  else return 0;

  if ( !(peek(str,n) >= '0' && peek(str,n) <= '9')) return 0;
  while ( peek(str,n) >= '0' && peek(str,n) <= '9') n++;

  if (n > 255) m = 255;
  else m = n;

  int i;
  for (i = 0; i < m; i ++) {
    fbuf[i] = get(str);
  }

  fbuf[i] = 0;
  *res = strtod(fbuf, NULL);
  return n;
}
```

The `next_token` function extracts a token from the stream if one is
available. If the end of the stream is reached a `TOKENIZER_END` token is returned.

Between tokens, one whitespace may be required but there is never any
requirement for more than one whitespace between tokens. So, before
trying to fetch the next token all heading whitespace on the stream is
read out and discarded. Comments in the source is also treated just
like whitespace.

Then each of the tokenizer functions (the *try to tokenize* functions)
are applied one after the other in order of kind and size of token
they match. The first of these to return a value larger than 0 signals
what token was available and `next_token` returns.

The order, from larger to smaller tokens, is important in the cases
where different tokens have the same valid initial sub-string. In
these cases it is important to get the longest, most specific, match.


```
token next_token(tokenizer_char_stream str) {

  token t;

  INT i_val;
  UINT u_val;
  char c_val;
  FLOAT f_val;
  int n = 0;

  if (!more(str)) {
    t.type = TOKENIZER_END;
    return t;
  }

  // Eat whitespace and comments.
  bool clean_whitespace = true;
  while ( clean_whitespace ){
    if ( peek(str,0) == ';' ) {
      while ( more(str) && peek(str, 0) != '\n') {
	drop(str,1);
      }
    } else if ( isspace(peek(str,0))) {
      drop(str,1);
    } else {
      clean_whitespace = false;
    }
  }

  // Check for end of string again
  if (!more(str)) {
    t.type = TOKENIZER_END;
    return t;
  }

  n = 0;

  if ((n = tok_quote(str))) {
    t.type = TOKQUOTE;
    return t;
  }

  if ((n = tok_openpar(str))) {
    t.type = TOKOPENPAR;
    return t;
  }

  if ((n = tok_closepar(str))) {
    t.type = TOKCLOSEPAR;
    return t;
  }

  if ((n = tok_symbol(str, &t.data.text))) {
    t.text_len = n;
    t.type = TOKSYMBOL;
    return t;
  }

  if ((n = tok_char(str, &c_val))) {
    t.data.c = c_val;
    t.type = TOKCHAR;
    return t;
  }

  if ((n = tok_string(str, &t.data.text))) {
    t.text_len = n - 2;
    t.type = TOKSTRING;
    return t;
  }

  if ((n = tok_F(str, &f_val))) {
    t.data.f = f_val;
    t.type = TOKBOXEDFLOAT;
    return t;
  }

  if ((n = tok_U(str, &u_val))) {
    t.data.u = u_val;
    t.type = TOKBOXEDUINT;
    return t;
  }

  if ((n = tok_u(str, &u_val))) {
    t.data.u = u_val;
    t.type = TOKUINT;
    return t;
  }

  if ((n = tok_I(str, &i_val))) {
    t.data.i = i_val;
    t.type = TOKBOXEDINT;
    return t;
  }

  // Shortest form of integer match. Move to last in chain of numerical tokens.
  if ((n = tok_i(str, &i_val))) {
    t.data.i = i_val;
    t.type = TOKINT;
    return t;
  }

  t.type = TOKENIZER_ERROR;
  return t;
}
```

## Parsing

The parser is, I believe, an example of a recursive descent parser. It
is split up into a number of mutually recursive functions. The
functions involved are: `tokpar_parse` (the entry point),
`parse_program`, `parse_sexp` and `parse_sexp_list`.  The `sexp` in
these function names comes from
[s-expression](https://en.wikipedia.org/wiki/S-expression) which is
what the kind of nested-tree-structured list-based expressions that
lisp use are called.

The `tokpar_parse` functions sets up a tokenizer state and a tokenizer
character stream and then call `parse_program` on that stream.

```
VALUE tokpar_parse(char *string) {

  tokenizer_state ts;
  ts.str = string;
  ts.pos = 0;

  tokenizer_char_stream str;
  str.state = &ts;
  str.more = more_string;
  str.peek = peek_string;
  str.drop = drop_string;
  str.get  = get_string;

  return parse_program(str);
}
```

`parse_program` parses a sequence of expressions from the stream. It
does this calling `parse_sexp` on the tokenizer character stream,
which if successful creates the heap representaion of that first
expression. It then recurses on the rest of the character stream and
allocates a cons cell on the heap to combine the result of the
recursive call and the call to `parse_sexp`. If an error occurs or the
stream ends, the function returns.

``` 
VALUE parse_program(tokenizer_char_stream str) {
  token tok = next_token(str);
  VALUE head;
  VALUE tail;

  if (tok.type == TOKENIZER_ERROR) {
    return enc_sym(symrepr_rerror());
  }

  if (tok.type == TOKENIZER_END) {
    return enc_sym(symrepr_nil());
  }

  head = parse_sexp(tok, str);
  tail = parse_program(str);

  return cons(head, tail);
}
``` 

The `parse_sexp` function gets a token and a tokenizer character
stream as input. It checks what token it got as input and then selects
a case in a `switch` statement. One interesting case is when the token
is an opening parenthesis in which case we are dealing with a lisp
list and the `parse_sexp_list` function should take over. In other
cases `parse_sexp` create heap representations for the tokens.

```
VALUE parse_sexp(token tok, tokenizer_char_stream str) {

  VALUE v;
  token t;

  switch (tok.type) {
  case TOKENIZER_END:
    return enc_sym(symrepr_rerror());
  case TOKENIZER_ERROR:
    return enc_sym(symrepr_rerror());
  case TOKOPENPAR:
    t = next_token(str);
    return parse_sexp_list(t,str);
  case TOKSYMBOL: {
    UINT symbol_id;

    if (symrepr_lookup(tok.data.text, &symbol_id)) {
      v = enc_sym(symbol_id);
    }
    else if (symrepr_addsym(tok.data.text, &symbol_id)) {
      v = enc_sym(symbol_id);
    } else {
      v = enc_sym(symrepr_rerror());
    }
    free(tok.data.text);
    return v;
  }
  case TOKSTRING: {
    heap_allocate_array(&v, tok.text_len+1, VAL_TYPE_CHAR);
    array_t *arr = (array_t*)car(v);
    memset(arr->data.c, 0, (tok.text_len+1) * sizeof(char));
    memcpy(arr->data.c, tok.data.text, tok.text_len * sizeof(char));
    free(tok.data.text);
    return v;
  }
  case TOKINT:
    return enc_i(tok.data.i);
  case TOKUINT:
    return enc_u(tok.data.u);
  case TOKCHAR:
    return enc_char(tok.data.c);
  case TOKBOXEDINT:
    return set_ptr_type(cons(tok.data.i, enc_sym(DEF_REPR_BOXED_I_TYPE)), PTR_TYPE_BOXED_I);
  case TOKBOXEDUINT:
    return set_ptr_type(cons(tok.data.u, enc_sym(DEF_REPR_BOXED_U_TYPE)), PTR_TYPE_BOXED_U);
  case TOKBOXEDFLOAT:
    return set_ptr_type(cons(tok.data.u, enc_sym(DEF_REPR_BOXED_F_TYPE)), PTR_TYPE_BOXED_F);
  case TOKQUOTE: {
    t = next_token(str);
    VALUE quoted = parse_sexp(t, str);
    if (type_of(quoted) == VAL_TYPE_SYMBOL &&
	dec_sym(quoted) == symrepr_rerror()) return quoted;
    return cons(enc_sym(symrepr_quote()), cons (quoted, enc_sym(symrepr_nil()))); 
  }
  }
  return enc_sym(symrepr_rerror());
}
```

The `parse_sexp_list` is conceptually similar to `parse_program` it also
parses the head part using `parse_sexp` and then it parses the rest
using `parse_sexp_list`. It generates cons cells on the heap to tie the
results together. Differently from `parse_program`, `parse_sexp_list`
is done when in encounters an closing parenthesis in which case it
returns a `nil` to close the list.

```
VALUE parse_sexp_list(token tok, tokenizer_char_stream str) {

  token t;
  VALUE head;
  VALUE tail;

  switch (tok.type) {
  case TOKENIZER_END:
    return enc_sym(symrepr_rerror());
  case TOKENIZER_ERROR:
    return enc_sym(symrepr_rerror());
  case TOKCLOSEPAR:
    return enc_sym(symrepr_nil());
  default:
    head = parse_sexp(tok, str);
    t = next_token(str);
    tail = parse_sexp_list(t, str);
    if ((type_of(head) == VAL_TYPE_SYMBOL &&
	 dec_sym(head) == symrepr_rerror() ) ||
	(type_of(tail) == VAL_TYPE_SYMBOL &&
	 dec_sym(tail) == symrepr_rerror() )) return enc_sym(symrepr_rerror());
    return cons(head, tail);
  }

  return enc_sym(symrepr_rerror());
}
```

That is it for parsing as it is done in lispBM. If you spot problems I am very thankful to hear about it. 

___

[HOME](https://svenssonjoel.github.io)
