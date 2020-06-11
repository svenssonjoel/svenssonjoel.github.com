

# Quasiquotation in lispBM

One feature that has been missing from lispBM is quasiquotation. The
quasiquotation concept is very confusing to me and seemed too tricky
to attempt. But, recently I decided to have a go at it and try to
adapt the algorithm that is shown in [Bawden's paper Quasiquotation in
Lisp](https://www.brics.dk/NS/99/1/BRICS-NS-99-1.pdf#page=6) to
lispBM.

Bawden's paper show an implementation quasiquotation for a lisp
dialect implemented in scheme code. Lisp/scheme is of course very
suitable for writing such meta programs but figuring out how they work
is a real headache to me. The implementation of quasiquotation as
shown there uses quasiquotation in the implementation language! I
stared at this code in the appendix of the paper for a very long time
before trying to translate it to C code to include in lispBM. I am far
from certain that my implementation is a correct translation of the
algorithm, but it does seem to work (for the cases I have tried). As
usual I really appreciate feedback, so if you find bugs or maybe even
just have information that would clarify my understanding of this I
would be very happy to hear it.

## Quasiquotation (as I understand it)

The lisp quasiquotation features are, as I understand them, there to
make it easier to write programs that generate programs. The syntax
related to quasiquotation consist of the `` ` `` (back-quote), `,`
(unquote) and `,@` (splice).

The `` ` `` alone, has a result similar to `'` such that:

```
# `apa
apa

# 'apa
apa

# '(+ 1 2)
(+ 1 2)

# `(+ 1 2)
(+ 1 2)
```

It is when `` ` `` is used together with `,` that it gets more
interesting.  The `,` can be used within an expression that is quoted
using `` ` `` to indicate that the expression *tagged* with the `,`
should be evaluated.

```
# `(+ 1 ,(+ 1 1))
(+ 1 2)
```

The `,@` tag, is similar to the `,` except that the expression that
follows the `,@` should evaluate to a list and the items of that list
are *spliced* into the result. The example below illustrates this use
of `,@` by splicing the result of a call to the library function
`iota` into the argument list in an application of the function
`list`.

```
# `(list 1 2 3 ,@(iota 10) 3 2 1)
(list 1 2 3 0 1 2 3 4 5 6 7 8 9 10 3 2 1)
```

The operators `` ` `` (back-quote), `,` (unquote) and `,@` (splice)
are shorthand notation for certain usages of other list creating and
manipulating functions as well as the quote `'` operator. So,
occurrences of these quasiquotation related operators can be expanded
into programs based on `list`, `cons`, `append` and `'` for
example. In other words, the addition of these operations is just for
convenience and makes writing programs that generate programs
easier. Another benefit is that a program that generates a program now
looks very similar to the program it generates, there are just some
extra ticks/tags here and there. `(+ 1 2)` is a program that adds up
`1` and `2`, `` `(+ 1 2) `` is a program that generates a program that
adds up `1` and `2`.

To see what something like `` `(+ 1 2) `` is expanded into it can be
preceded with a `'`.

```
# '`(+ 1 2)
(append (list +) (append (list 1) (append (list 2) (quote nil))))
```

## Implementation

The algorithm for expanding quasiquotations as it is expressed in
Bawden's paper operates on an s-expression where sub-expressions are
tagged with back-quote, unquote and splice tags. The Algorithm
operates in a top down fashion which means that, in the case of nested
quasiquotation, further back-quotes can be found when traversing the
s-expression. The expansion algorithm is assumed to be applied to a
back-quote tagged s-expression where the top-level tag has been
removed.

For the purpose of lispBM, I want to expand quasiquotation as part of
the parsing process and in a bottom-up way. Doing it bottom-up should
mean that no back-quotes will ever be found when performing the
expansion as the outermost back-quote is always eliminated first in
the expansion process, so the case that recursively deals with
back-quotes can be dropped from this bottom-up approach.

To add quasiquotation, really only the tokenizer and parser needs to
be extended a bit. But since I chose to use symbols to tag unquote and
splice expressions, two additional *special* symbols (called `comma`
and `commaat`) are added in `symrepr.c` and `symrepr.h`. These symbols
will also be removed from the expressions as part of the
quasiquotation expansion routine and if any occurrence of them are
still in the code once it reaches the evaluator, then that is an
indication of a programmer error (the quasiquotation operators
was likely nested in some incorrect way).

Note that there is no need for a back-quote symbol to exist at all, we
will see why when looking at how the parser deals with the case of
encountering a back-quote token.

The example below shows what happens if there are left-over unquotes
not dealt with by the expander.

```
# (+ ,1 ,2)
eval_error
# '(+ ,1 ,2)
(+ (comma 1) (comma 2))
```

### Changes to tokenizer and parser

See [parsing expressions](../lispbm_parsing_expressions/index.html)
for more details about the tokenizer and parser. 

Three new token identifiers are added. 

```
#define TOKBACKQUOTE    11
#define TOKCOMMA        12
#define TOKCOMMAAT      13
```

Then, three token-recognizing functions are added as well.  These peek
into the character stream and if the token they look for is found, the
number of characters consumed by that token is returned. 

```
int tok_backquote(tokenizer_char_stream str) {
  if (peek(str,0) == '`') {
    drop(str, 1);
    return 1;
  }
  return 0;
}

int tok_commaat(tokenizer_char_stream str) {
  if (peek(str,0) == ',' &&
      peek(str,1) == '@') {
    drop(str,2);
    return 2;
  }
  return 0;
} 

int tok_comma(tokenizer_char_stream str) {
  if (peek(str,0) == ',') {
    drop(str, 1);
    return 1;
  }
  return 0;
}
```

In the `next_token` function, checks are added for each of the tokenizer function
defined above. Note that `tok_commaat` is tried before `tok_comma`. Since
the splice syntax shares a prefix with the unquote syntax, but is longer, it is
important to try it first to not falsely return a unquote token.

```
token next_token(tokenizer_char_stream str) {

  token t;

  ...

  if ((n = tok_backquote(str))) {
    t.type = TOKBACKQUOTE;
    return t;
  }

  if ((n = tok_commaat(str))) {
    t.type= TOKCOMMAAT;
    return t;
  }
  
  if ((n = tok_comma(str))) {
    t.type = TOKCOMMA;
    return t;
  }

  ...

  t.type = TOKENIZER_ERROR;
  return t;
}

``` 

Three cases are added to the parser in order to deal with the new
possible tokens.  The cases for `TOKCOMMAAT` and `TOKCOMMA` tags the
following s-expression with the appropriate symbol.

In the back-quote case, the bottom-up part of quasiquotation expanding is
implemented.  Here the expression following the back-quote is parsed
recursively, the result of that is then passed through the function
`qq_expand`. The `qq_expand` function is shown in the next section.
 
This becomes bottom-up expansion of quasiquotation through interplay
with the recursive-descent parser. The recursive calls to `parse_sexp`
will trickle all the way down to the leaves. Then when reaching the
bottom and the return values start bubbling up the expansion be
applied.

```
VALUE parse_sexp(token tok, tokenizer_char_stream str) {

  VALUE v;
  token t;

  switch (tok.type) {
    ...
  case TOKBACKQUOTE: {
    t = next_token(str);
    VALUE quoted = parse_sexp(t, str);
    if (type_of(quoted) == VAL_TYPE_SYMBOL &&
	dec_sym(quoted) == symrepr_rerror()) return quoted;
    VALUE expanded = qq_expand(quoted);
    if (type_of(expanded) == VAL_TYPE_SYMBOL &&
	symrepr_is_error(dec_sym(expanded))) return expanded;
    return expanded;
  }
  case TOKCOMMAAT: {
    t = next_token(str);
    VALUE splice = parse_sexp(t, str);
    if (type_of(splice) == VAL_TYPE_SYMBOL &&
	dec_sym(splice) == symrepr_rerror()) return splice;
    return cons(enc_sym(symrepr_commaat()), cons (splice, enc_sym(symrepr_nil())));
  }
  case TOKCOMMA: {
    t = next_token(str);
    VALUE unquoted = parse_sexp(t, str);
    if (type_of(unquoted) == VAL_TYPE_SYMBOL &&
	dec_sym(unquoted) == symrepr_rerror()) return unquoted;
    return cons(enc_sym(symrepr_comma()), cons (unquoted, enc_sym(symrepr_nil()))); 
  }
  }
  return enc_sym(symrepr_rerror());
}

```

### Expansion of quasiquotatations

The functions `qq_expand` and `qq_expand_list` below are mostly just
translations of Bawden's lisp functions with the same names. One
difference is that I have removed the cases that deal with back-quote
as no back-quotes will appear when doing this bottom-up traversal.


In the parser, `qq_expand` is applied to an s-expression in the case
that deals with the back-quote token. So the `VALUE` passed to
`qq_expand` represents a quoted expression. 

In the case where the expression is a pair (a cons-cell)
`qq_expand_list` is called on the `car` and `qq_expand` on the
`cdr`. The results of these calls are then appended together. The
`append` function used here does not actually append lists but rather
it generates the code that appends those lists. 

```
VALUE append(VALUE front, VALUE back) {
  return cons (enc_sym(symrepr_append()),
	       cons(front,
		    cons(back, enc_sym(symrepr_nil()))));
}
```

Expressions that are tagged with unquote are just untagged and returned.
Expression that are neither made from cons-cells or tagged with
unquote gets quoted (these are for example, numbers or symbols).

```
VALUE qq_expand(VALUE qquoted) {

  VALUE res = enc_sym(symrepr_nil()); 
  VALUE car_val;
  VALUE cdr_val; 
  
  switch (type_of(qquoted)) {
  case PTR_TYPE_CONS:
    car_val = car(qquoted);
    cdr_val = cdr(qquoted);
    if (type_of(car_val) == VAL_TYPE_SYMBOL &&
        dec_sym(car_val) == symrepr_comma()) {
      res = car(cdr_val);
    } else if (type_of(car_val) == VAL_TYPE_SYMBOL &&
               dec_sym(car_val) == symrepr_commaat()) {
      res = enc_sym(symrepr_rerror()); 
    } else {
      VALUE expand_car = qq_expand_list(car_val);
      VALUE expand_cdr = qq_expand(cdr_val);
      res = append(expand_car, expand_cdr);
    }
    break;
  default:
    res = cons(enc_sym(symrepr_quote()), cons(qquoted, enc_sym(symrepr_nil())));
    break;
  }
  return res;
}
``` 

The `qq_expand_list` is called on items within a list (`car` positions
of cons-cells) and is very similar to `qq_expand` except that it
always returns a list. This function also expands splices, as these
are only allowed within lists (if I understand Bawden's algorithm
correctly). The cases (except for the one that splices) are all
identical to `qq_expand` except that their results are put in lists. In
the case for the splice tag the splice expression is untagged and
returned (these splice expressions should evaluate to lists so the
types still match). 

```
VALUE qq_expand_list(VALUE l) {
  VALUE res = enc_sym(symrepr_nil());
  VALUE car_val;
  VALUE cdr_val;

  switch (type_of(l)) {
  case PTR_TYPE_CONS:
    car_val = car(l);
    cdr_val = cdr(l);
    if (type_of(car_val) == VAL_TYPE_SYMBOL &&
        dec_sym(car_val) == symrepr_comma()) {
      res = cons(enc_sym(symrepr_list()),
                 cons(car(cdr_val), res));
    } else if (type_of(car_val) == VAL_TYPE_SYMBOL &&
               dec_sym(car_val) == symrepr_commaat()) {
      res = car(cdr_val);
    } else {
      VALUE expand_car = qq_expand_list(car_val);
      VALUE expand_cdr = qq_expand(cdr_val);
      res = cons(enc_sym(symrepr_list()),
                 cons(append(expand_car, expand_cdr), enc_sym(symrepr_nil())));
    }
    break;
  default:
    res = cons(enc_sym(symrepr_list()),
               cons(l, enc_sym(symrepr_nil())));			  
  }
  return res;
}
```
---
**Bugfix June 10 2020**

`qq_expand_list` above is flawed in the `default` case that generates the code
that generates a list. It is rather supposed to be a quoted list. 

```
VALUE qq_expand_list(VALUE l) {
  VALUE res = enc_sym(symrepr_nil());
  VALUE car_val;
  VALUE cdr_val;

  switch (type_of(l)) {
  case PTR_TYPE_CONS:
    car_val = car(l);
    cdr_val = cdr(l);
    if (type_of(car_val) == VAL_TYPE_SYMBOL &&
        dec_sym(car_val) == symrepr_comma()) {
      res = cons(enc_sym(symrepr_list()),
                 cons(car(cdr_val), res));
    } else if (type_of(car_val) == VAL_TYPE_SYMBOL &&
               dec_sym(car_val) == symrepr_commaat()) {
      res = car(cdr_val);
    } else {
      VALUE expand_car = qq_expand_list(car_val);
      VALUE expand_cdr = qq_expand(cdr_val);
      res = cons(enc_sym(symrepr_list()),
                 cons(append(expand_car, expand_cdr), enc_sym(symrepr_nil())));
    }
    break;
  default: {
    VALUE a_list = cons(l, enc_sym(symrepr_nil()));
    res =
      cons(enc_sym(symrepr_quote()), cons (a_list, enc_sym(symrepr_nil())));
  }
  }
  return res;
}
``` 

---


## Future work

Before adding quasiquotation, expression parsing didn't generate much
garbage at all (garbage in the sense of temporary heap objects that
really should be garbage collected). So, thus far I didn't think very
much about how to add some interplay between garbage collection and
parsing. I just thought that if there is not enough heap to parse the
program, you cannot run the program anyway so why bother. Now,
however, the quasiquotation expansion process generates garbage that
probably should be dealt with somehow as it could mean that parsing a
program that really does fit in memory will fail to parse because the
heap becomes full.

More testing is needed. I am not an old lisp guru and really don't
know much about how tricky you can get with quasiquotation, so it is a
bit hard for me to know if this implementation works as it should! As
time goes by and I try more things, bugs will be taken care of as they
appear (as long as they aren't total show-stoppers).  

Macros is another interesting thing to look into. I have a feeling
that with quasiquotation, the main part of that is done but what could
be added is some lambda-like form that is applied without evaluating
the arguments. Need to look into this a bit more before attempting
anything. 

___

[HOME](https://svenssonjoel.github.io)
