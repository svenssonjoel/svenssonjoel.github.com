

# Adding Short Circuit Capable Boolean Operators

I recently added the boolean operators `and`, `or` and `not` to
lispBM. When evaluating `(and a b)` it is desireable that if `a`
evaluates to *false* (`nil` in lispBM), `b` is not evaluated at all.
Likewise for `(or a b)` we don't want `b` to be evaluated if `a` turns
out being *true* (`t` in lispBM).

Function application in lispBM has the property that all of the
arguments are evaluated beforehand and then the function is
applied. So this new wish when it comes to these boolean operations
does not fit that well into the patterns already establsihed.

Since many other functions can take an arbitrary number of arguments,
I thought it would be nice if also `and` and `or` works when given zero
or more arguments. So that is another feature on the wishlist.

Below is an example of applying and to zero or more arguments.

```
# (and) 
t
# (and 't)
t
# (and 't 't)
t
# (and 't 't 'nil)
nil
```

And the property of not evaluating further arguments if not needed can
be illustrated like this.

```
# (and 't 'nil (print "hello"))
nil
```
Notice that `hello` is not printed. 

## Adding Symbols for the Boolean Operators

The first step in adding more built in fundamental operations to lispBM
is to add new symbols that can be used to identify the. 

The symbols for the boolean operators are given IDs within the range
of *special* symbols. Below the changes to `symrepr.h` are listed.

```
#define SYM_AND                 0x110FFFF
#define SYM_OR                  0x111FFFF
#define SYM_NOT                 0x112FFFF

static inline UINT symrepr_and(void)         { return SYM_AND; }
static inline UINT symrepr_or(void)          { return SYM_OR; }
```

And the following is added to the `add_default_symbols` function in
`symrepr.c` to associate a textual name with each of the symbols.

```
  res = res && symrepr_addspecial("and", SYM_AND);
  res = res && symrepr_addspecial("or", SYM_OR);
  res = res && symrepr_addspecial("not", SYM_NOT);

```

The not operator is implemented just as all other fundamental
operations as a case in `fundamental.c`. It takes one argument and if
that argument is `nil` it returns `t` otherwise it returns
`nil`. Actuall all of these boolean operations are implemented to
treat `nil` as false and anything else as true. So `(not 5)` evaluates
to `nil` and `(and 2 3)` evaluates to `t`.

The `and` and `or` operators will need some special attention to be able to
get the short-circuiting behavior. 

## Adding Short Circuit Boolean Operators to the Evaluator

So to add short-circuiting boolean operators, what came to mind for me
is to add new and specific *continuations* to deal with evaluating the
arguments.  The new argument evaluating continuations come in two
shapes, that either continues to evaluate arguments as long as they
come out true (`and`) and one shape that evaluates while they come out
false (`or`).  So two new continuation IDs are added to the list for,
`AND` and `OR`.

```
#define DONE              1
#define SET_GLOBAL_ENV    2
#define BIND_TO_KEY_REST  3
#define IF                4
#define PROGN_REST        5
#define APPLICATION       6
#define APPLICATION_ARGS  7
#define AND               8
#define OR                9
```


*case APPLICATION_ARGS*
```
/* Deal with short-circuiting operators */
    if (type_of(arg) == VAL_TYPE_SYMBOL &&
	dec_sym(arg) == symrepr_and()) {
      if (type_of(rest) == VAL_TYPE_SYMBOL &&
	  rest == NIL) {
	*app_cont = true;
	return enc_sym(symrepr_true());
      } else {
	FATAL_ON_FAIL(*done, push_u32_3(&ctx->K, env, cdr(rest), enc_u(AND)));
	ctx->curr_exp = car(rest);
	ctx->curr_env = env;
	return NONSENSE;
      }
    }

    if (type_of(arg) == VAL_TYPE_SYMBOL &&
	dec_sym(arg) == symrepr_or()) {
      if (type_of(rest) == VAL_TYPE_SYMBOL &&
	  rest == NIL) {
	*app_cont = true;
	return enc_sym(symrepr_nil());
      } else {
	FATAL_ON_FAIL(*done, push_u32_3(&ctx->K, env, cdr(rest), enc_u(OR)));
	ctx->curr_exp = car(rest);
	ctx->curr_env = env;
	return NONSENSE;
      }
    }
```

*case APPLICATION*
```
+  case AND: {
+    VALUE env;
+    VALUE rest;
+    pop_u32_2(&ctx->K, &rest, &env);
+    if (type_of(arg) == VAL_TYPE_SYMBOL &&
+       dec_sym(arg) == symrepr_nil()) {
+      *app_cont = true;
+      return enc_sym(symrepr_nil());
+    }
+    if (type_of(rest) == VAL_TYPE_SYMBOL &&
+       rest == NIL) {
+      *app_cont = true;
+      return enc_sym(symrepr_true());
+    } else {
+      FATAL_ON_FAIL(*done, push_u32_3(&ctx->K, env, cdr(rest), enc_u(AND)));
+      ctx->curr_exp = car(rest);
+      ctx->curr_env = env;
+      return NONSENSE;
+    }
+  }
+  case OR: {
+    VALUE env;
+    VALUE rest;
+    pop_u32_2(&ctx->K, &rest, &env);
+    if (type_of(arg) != VAL_TYPE_SYMBOL ||
+       dec_sym(arg) != symrepr_nil()) {
+      *app_cont = true;
+      return enc_sym(symrepr_true());
+    }
+    if (type_of(rest) == VAL_TYPE_SYMBOL &&
+       rest == NIL) {
+      *app_cont = true;
+      return enc_sym(symrepr_nil());
+    } else {
+      FATAL_ON_FAIL(*done, push_u32_3(&ctx->K, env, cdr(rest), enc_u(OR)));
+      ctx->curr_exp = car(rest);
+      ctx->curr_env = env;
+      return NONSENSE;
+    }
+  }
```



## Sumtree Example

```
(define is-number
  (lambda (x)
    (or (= (type-of x) type-i28)
        (= (type-of x) type-u28)
        (= (type-of x) type-float)
        (= (type-of x) type-i32)
        (= (type-of x) type-u32))
    ))
	    
      
(define sumtree
  (lambda (x)
    (if (is-number x)
        x
      (if (= x 'nil)
          0
        (let ((a (sumtree (car x)))
              (b (sumtree (cdr x))))
          (+ a b)
          )))))

```


___

[HOME](https://svenssonjoel.github.io)
