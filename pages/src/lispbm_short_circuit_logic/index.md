

# Adding Short Circuit Capable Boolean Operators

I recently added the boolean operators `and`, `or` and `not` to
[lispBM](../lispbm_current_status/index.html). When
evaluating `(and a b)` it is desirable that if `a` evaluates to
*false* (`nil` in lispBM), `b` is not evaluated at all.  Likewise for
`(or a b)` we don't want `b` to be evaluated if `a` turns out being
*true* (`t` in lispBM).

Function application in lispBM has the property that all of the
arguments are evaluated beforehand and then the function is
applied. So this new wish when it comes to these boolean operations
does not fit that well into the patterns already established.

Since many other functions can take an arbitrary number of arguments,
I thought it would be nice if also `and` and `or` works when given zero
or more arguments. So that is another feature on the wish list.

Below is an example of applying `and` to zero or more arguments.

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
is to add new symbols that can be used to identify them. 

The symbols for the boolean operators are given IDs within the range
of *special* symbols. The [Another Lisp for
Microcontrollers](../lispbm_current_status/index.html) test holds a
bit more information on this range of symbol IDs. Below the changes to
`symrepr.h` are listed.

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
`nil`. Actually all of these boolean operations are implemented to
treat `nil` as false and anything else as true. So `(not 5)` evaluates
to `nil` and `(and 2 3)` evaluates to `t`.

The `and` and `or` operators will need some special attention to be able to
get the short-circuiting behavior. 

## Adding Short Circuit Boolean Operators to the Evaluator

So to add short-circuiting boolean operators, what came to mind for me
was to add new and and specific *continuations* for evaluating the
arguments in a way suitable for short-circuiting. The new argument
evaluating continuations come in two shapes, that either continues to
evaluate arguments as long as they come out true (`and`) and one shape
that evaluates while they come out false (`or`).  So two new
continuation IDs are added to the list for, `AND` and `OR`.

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

In the entry point function for evaluation, the `run_eval`function, the
case that deals with function application is left unchanged. Currently
this case looks as follows:

```
    FATAL_ON_FAIL(done,
		    push_u32_4(&ctx->K,
			       ctx->curr_env,
			       enc_u(0),
			       cdr(ctx->curr_exp),
			       enc_u(APPLICATION_ARGS)));

      ctx->curr_exp = head; // evaluate the function
      continue;
```

It sets the `curr_exp` up to evaluate the function. For example, in
the case of a `lambda` this means that the next thing to do for the
evaluator is to turn the `lambda`into a closure. But before that a
continuation is pushed that describes what to do with the arguments,
this is the `APPLICATION_ARGS` continuation.

I chose to leave this code unchanged and instead differentiate between
short-circuitable operators and non, inside of the `APPLICATION_ARGS`
function.


Inside of the  `apply_continuation` function and the `APPLICATION_ARGS` case there,
we check if the function (with is now evaluated or at least looked up), is one of
the short-circuitable function `and` or `or`. If that is a case a new continuation
is created to deal with the arguments, in the case of an `and` operator the `AND` continuation
is pushed onto the stack and the first argument is set up to be evaluated next.
Likewise in the `or` case, the `OR` continuation is pushed and the next thing to
evaluate is the first element.

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

The above code means that if we are processing the arguments to a
short-circuitable operator we will jump over and continue along the
`AND` or `OR` continuations instead.

In the case of `AND`, arguments should be evaluated as long as they
result in something considered *true*.

```
  case AND: {
    VALUE env;
    VALUE rest;
    pop_u32_2(&ctx->K, &rest, &env);
    if (type_of(arg) == VAL_TYPE_SYMBOL &&
       dec_sym(arg) == symrepr_nil()) {
      *app_cont = true;
      return enc_sym(symrepr_nil());
    }
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
```

For the `OR`case, we evaluate arguments as long as they result in false.
The first argument to result in *true* terminates the continued evaluation of
arguments and we can treat the result as true. If we run out of arguments
without encountering true, the result is false. 


```
  case OR: {
    VALUE env;
    VALUE rest;
    pop_u32_2(&ctx->K, &rest, &env);
    if (type_of(arg) != VAL_TYPE_SYMBOL ||
       dec_sym(arg) != symrepr_nil()) {
      *app_cont = true;
      return enc_sym(symrepr_true());
    }
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

And that's it, the next section holds an example that would be awkward
to write without access to `or`.

## Sumtree Example

It is nice to finally be able to write some slightly more substantial
lispBM programs such as `sumtree` below. I have noticed that as soon
as wiring a slightly more *involved* program, new bug surface. Which
is a lot of fun.  One of the main reason for tormenting oneself with
trying to implement this is that it is an never ending source problems
to solve and that they are all right on the boundary of what I can
handle when it comes to difficulty and complexity.

The sumtree function sums up the values located at the leaves of a
tree. A leaf should hold some valid lispBM number type. The sum is
computed by recursing on the `car` and `cdr, reaching a number
terminates the recursion and returns the number upwards in the call
hierarchy to added to the running sum.

Oh, I think that perhaps I should really call `is-number`, `numberp`
or `number-p`.  We'll see, it should be part of the default library
(the prelude) later.

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
