

# Evaluating Expressions

This text is a more in depth look at the code that evaluates
expressions in lispBM. I hope that It can be of value to someone, just
like the [Lisperator](http://lisperator.net/pltut/) code was useful to me.

This code is work in progress and some things look the way they do
currently because there is some vague thought of a future direction it
may go in. I will try to point out these extra complexities that
currently have no direct purpose.

The goal is that the evaluator should have a certain set of properties.

1. Tail-calls and even infinite recursion (in tail-position) should be
possible without ever-growing memory consumption.

2. At any point where heap space is allocated it must be possible to
pause the computation, perform *garbage collection* (gc), and then
pick up execution again where the computation was paused.

3. Evaluation should be correct in relation to environments (such that the
body of a let is evaluated in an environment extended with the
bindings). There are also other examples of where similar properties
should hold, such as when `lambda`s are evaluated into `closure`s. 

4. Evaluating an expression should give the expected result. 

Writing the code that does all 4 of the above is (at least to me) a
bit tricky and I am not entirely sure that every path through the
evaluator really maintains these properties yet. As problems shows up,
they are addressed.

After writing down the [overview of
lispBM](../lispbm_current_status/index.html), some possible
simplifications was spotted. While refactoring these I also noticed a
case of violating property 1. The code that evaluates the `PROGN`
functionality was written in a way that led to ever-growing memory use
if involved in a infinite recursion. For example:

```
(define f
  (lambda (x)
    (progn (print "hello")
	   (f (+ x 1)))))
````

But the problem was possible to fix. I will point out where I fell in
the trap later.

Property 1, is tackled by implementing the evaluator in continuation
passing style and as a while loop with explicitly managed state. For
example, the explicit continuation stack keeps track of arguments to
the continuation and which continuation to apply. Care has to be taken
that the interactions with the stack happen in the correct order so
that we do not end up with ever-growing stack usage. One way to put it
is that nothing should be pushed onto the stack before a tail-call and
then popped when finally the tail-call returns.

Property 2, is handled at each place where cons-cells are
allocated. The exact details differ a little from one place to
another, so I will point this out in the code further down. But in
essence what it means is that the stack and the evaluation context is
set to a previously known state and then the evaluation loop is
re-entered with a *perform gc* flag set. The evaluator will then
perform GC in the next iteration. When GC is done the next iteration
of the evaluator will continue work with the evaluation context.

Property 3, is dealt with in the cases involving `define`, `let` and
`lambda`. Either by making changes to the globally defined environment
(in the case of `define`) or by updating the local environment that
the evaluation context maintains.

Property 4, is loosely checked by a growing set of tests located under
the `tests` directory.

There are also a set of flags, `perform_gc`, `apply_continuation` and
`done`. These are set in the evaluator loop to signal to itself what
it should be doing in the next iteration. `done` is set when there is
an error that it is not possible to recover from. `apply_continuation`
is set if the next iteration should pop information off the stack and
process that. This happens for example when evaluation results in base
type (such as a number). That number may be a part of an addition as
in (+ 1 2). So in the next iteration of the loop the continuation will
contain information about what to do with that number. The
`perform_gc` flag is set (as hinted above), when heap is full and we
cannot proceed unless gc is executed. 

## Often Used Dependencies

There a couple of often used functions throughout the `eval_cps` code
that comes from other parts of lispBM. Many of these are called
`enc_X` where `X` can be something like `sym`, `i`, `u` and so
on. These functions encode values from *the C world* into lispBM
`VALUE`s, for example `enc_i` turns a C integer into a lispBM `VALUE`
representing a 28Bit integer. Similarly there are functions called
`dec_X` that transform values in the other direction.

## Global Data and Some Smaller Functions

Now to the code! Evaluation makes use of almost everything else. So
most of the lispBM header files are included.

```
#include "symrepr.h"
#include "heap.h"
#include "env.h"
#include "eval_cps.h"
#include "stack.h"
#include "fundamental.h"
#include "extensions.h"
#ifdef VISUALIZE_HEAP
#include "heap_vis.h"
#endif
```

Below are the new set of continuation functions (after changes applied
since [the earlier text on
lispBM](../lispbm_current_status/index.html)). There are now only 7
continuation functions instead of 9. The difference is that before
there was the three continuations `FUNCTION`, `FUNCTION_APP` and
`ARG_LIST` that are now replaced by just `APPLICATION` and
`APPLICATION_ARGS`. There was also one continuation function that was
no longer used at all and could be dropped.

``` 
#define DONE              1
#define SET_GLOBAL_ENV    2
#define BIND_TO_KEY_REST  3
#define IF                4
#define PROGN_REST        5
#define APPLICATION       6
#define APPLICATION_ARGS  7
```

There is a little bit of global state maintained by the evaluator. It
consists of the global environment (where things `define`d go) and the
evaluation context. There are also some `VALUE`s, `NIL` and `NONSENSE`
these symbols are just given names that are easier to write and stand
out a bit as they are used quite frequently. 


```
static VALUE eval_cps_global_env;
static VALUE NIL;
static VALUE NONSENSE;

eval_context_t *eval_context = NULL;
```

Then there are a bunch of functions that handle evaluation
contexts. These are not very important now. They are written in this
way because of a vague plan to let the evaluator work on more than one
context where each context represents a task. Since it is possible to
pause the evaluation of a context it should be possible to also
implement task switching by pausing one context and then activating
another.

```
eval_context_t *eval_cps_get_current_context(void) {
  return eval_context;
}

eval_context_t *eval_cps_new_context_inherit_env(VALUE program, VALUE curr_exp) {
  eval_context_t *ctx = malloc(sizeof(eval_context_t));
  ctx->program = program;
  ctx->curr_exp = curr_exp;
  ctx->curr_env = eval_context->curr_env; /* TODO: Copy environment */
  ctx->K = stack_init(100, true);
  ctx->next = eval_context;
  eval_context = ctx;
  return ctx;
}

void eval_cps_drop_top_context(void) {
  eval_context_t *ctx = eval_context;
  eval_context = eval_context->next;
  stack_del(ctx->K);
  free(ctx);
}
```

The `eval_cps_get_env` is meant to be called by the REPL
implementation in case it is interested in knowing the contents of the
environment.

``` 
VALUE eval_cps_get_env(void) {
  return eval_cps_global_env;
}
``` 

The `eval_cps_bi_eval` function connects to the `fundamental.c` module
and provides the fundamental function called `eval`.

```
VALUE eval_cps_bi_eval(VALUE exp) {
  eval_context_t *ctx = eval_cps_get_current_context();

  ctx->curr_exp = exp;
  return run_eval(ctx);
}
``` 

## Starting Up and Shutting Down

The `eval_cps_init` functions initializes the evaluation state. It
takes an argument that decides whether or not the continuation stack
is allowed to grow or if it will be of fixed size. The global
environment is initialized and a single mapping is added that map
`nil` to `nil`. The global evaluation context is allocated. 

```
int eval_cps_init(bool grow_continuation_stack) {
  int res = 1;
  NIL = enc_sym(symrepr_nil());
  NONSENSE = enc_sym(symrepr_nonsense());

  eval_cps_global_env = NIL;

  eval_context = (eval_context_t*)malloc(sizeof(eval_context_t));

  eval_context->K = stack_init(100, grow_continuation_stack);

  VALUE nil_entry = cons(NIL, NIL);
  eval_cps_global_env = cons(nil_entry, eval_cps_global_env);

  if (type_of(nil_entry) == VAL_TYPE_SYMBOL ||
      type_of(eval_cps_global_env) == VAL_TYPE_SYMBOL) res = 0;

  return res;
}
```

The following function frees up the allocated state used by the evaluator.

``` 
void eval_cps_del(void) {
  stack_del(eval_context->K);
  free(eval_context);
}
```

## Entry-point Function

The function that is called from the REPL is called `eval_cps_program`
and takes a list of expressions as argument. This function currently
modifies the global evaluation context and then loops over the list of
expressions and evaluates them from first to last using the `run_eval`
function. the result of the last expression is returned to the caller.

```
VALUE eval_cps_program(VALUE lisp) {

  eval_context_t *ctx = eval_cps_get_current_context();

  ctx->program  = lisp;
  VALUE res = NIL;
  VALUE curr = lisp;

  if (symrepr_is_error(dec_sym(lisp))) return lisp;

  while (type_of(curr) == PTR_TYPE_CONS) {
    if (ctx->K->sp > 0) {
      stack_clear(ctx->K); // clear stack if garbage left from failed previous evaluation
    }
    ctx->curr_exp = car(curr);
    ctx->curr_env = NIL;
    res =  run_eval(ctx);
    curr = cdr(curr);
  }
  return res;
}
``` 

So now all the surrounding functions are covered, somewhat, and we can
go in depth with the evaluation function, `run_eval`.

## Evaluation Loop

The run eval function starts out with pushing the `DONE` continuation
onto the stack and defines some state needed throughout the
evaluation:

```
VALUE run_eval(eval_context_t *ctx){

  push_u32(ctx->K, enc_u(DONE));

  VALUE r = NIL;
  bool done = false;
  bool perform_gc = false;
  bool app_cont = false;

  uint32_t non_gc = 0;
```

1. The `r` value is what will hold the result of evaluating the
expression. `r` also holds intermediate results throughout evaluation.

2. The `done` flag signals that evaluation is finished, either
successfully or ended in an error.

3. The `perform_gc` flag, if set, indicates that this iteration of the
evaluation loop will include a round of garbage collection.

4. The `app_cont` flag signals that a continuation should be applied
at this point.

5. There is a counter `non_gc` that counts how many time the loop
iterated without the garbage collector running. So if `perform_gc` is
set and the counter is 0, we are really entirely out of memory. I
think the counter should be replaced with a boolean flag that
indicates if GC was run in the iteration before. 

After defining up these bookkeeping variables, the loop that runs
`while (!done)` is started.

```
  while (!done) {
    
#ifdef VISUALIZE_HEAP
    heap_vis_gen_image();
#endif
```

`heap_vis` is a module of lispBM that can be used for debugging
purposes. Every time `heap_vis_gen_image()` is called an image is
generated that has one pixel per cons-cell on the heap. Different
colors in this image represent things like *free*, *marked* and *in
use*.

Next there is a check if this iteration will include a round of
garbage collection.

``` 
    if (perform_gc) {
      if (non_gc == 0) {
        done = true;
        r = enc_sym(symrepr_merror());
        continue;
      }
      non_gc = 0;
      heap_perform_gc_aux(eval_cps_global_env,
                          ctx->curr_env,
                          ctx->curr_exp,
                          ctx->program,
                          r,
                          ctx->K->data,
                          ctx->K->sp);
      perform_gc = false;
    } else {
      non_gc ++;
    }
```

If `perform_gc` is set, and we are not in the problematic case when gc
was run last iteration, a call to `heap_perform_gc_aux` will
happen. The arguments to this functions are the things that the mark
phase should mark as being in use. This involves the continuation
stack! The continuation stack can hold pointers to heap structures
that are needed in the future and that are not pointed to by any
environment. 


If the `app_cont` flag is set, the execution path goes into a function
called `apply_continuation` that takes all the status flags as well as
the current intermediate result `r` as input. The `apply_continuation`
function will pop the top off from the stack and use the value stored
there to decide what continuation (from the list of defined
continuations) to apply. The `r` value is given to this continuation
but it may also pop additional arguments from the stack.

```
    if (app_cont) {
      r = apply_continuation(ctx, r, &done, &perform_gc, &app_cont);
      continue;
    }
```     

Now it is time for the big `switch` statement that evaluates each of
the different language constructs. The value `head` is used later when
the current expression is a list, depending on what the head of that
list is evaluation will take different paths. The `value` variable is
a temporary used throughout. 

```
    VALUE head;
    VALUE value = enc_sym(symrepr_eerror());

    switch (type_of(ctx->curr_exp)) {
```

The first case deals with symbols. When a symbol is evaluated it is
looked up in the environment, local and global. A symbol can, however,
also refer to a fundamental function (such as `+`) or to an extension
(user provided c function). If the symbol is not found in any
environment and does not represent a fundamental or extension an
evaluation error is returned and the `done` flag is set. 

```
    case VAL_TYPE_SYMBOL:

      value = env_lookup(ctx->curr_exp, ctx->curr_env);
      if (type_of(value) == VAL_TYPE_SYMBOL &&
          dec_sym(value) == symrepr_not_found()) {
	  
        value = env_lookup(ctx->curr_exp, eval_cps_global_env); 

        if (type_of(value) == VAL_TYPE_SYMBOL &&
            dec_sym(value) == symrepr_not_found()) {

          if (is_fundamental(ctx->curr_exp)) {
            value = ctx->curr_exp;
          } else if (extensions_lookup(dec_sym(ctx->curr_exp)) == NULL) {
            r = enc_sym(symrepr_eerror());
            done = true;
            continue;
          } else {
            value = ctx->curr_exp; // symbol representing extension
                                   // evaluates to itself at this stage.
          }
        }
      }
      app_cont = true;
      r = value;
      break;
```

Next the switch statement deals with the other basic types, numbers
and arrays, that don't need further evaluation but rather just that
they will be passed as argument to some continuation. This is done by
setting the `app_cont` flag and setting `r` to the what we want passed
to the continuation as argument.  


```
    case PTR_TYPE_BOXED_F:
    case PTR_TYPE_BOXED_U:
    case PTR_TYPE_BOXED_I:
    case VAL_TYPE_I:
    case VAL_TYPE_U:
    case VAL_TYPE_CHAR:
    case PTR_TYPE_ARRAY:
      app_cont = true;
      r = ctx->curr_exp;
      break;
```

The next two cases `ref` and `stream` are not implemented and are
currently just vague ideas.


```
    case PTR_TYPE_REF:
    case PTR_TYPE_STREAM:
      r = enc_sym(symrepr_eerror());
      done = true;
      break;
```

So, if the expression is not a symbol or other basic type, it should
be a list. The next case checks if the expression is a list and then
depending on what the head of that list is the evaluation takes
different paths.

```
    case PTR_TYPE_CONS:
      head = car(ctx->curr_exp);
```

The head of the list is stored in the `head` variable for easy access.


```
      if (type_of(head) == VAL_TYPE_SYMBOL) {

        // Special form: QUOTE
        if (dec_sym(head) == symrepr_quote()) {
          r = car(cdr(ctx->curr_exp));
          app_cont = true;
          continue;
        }
``` 

The `QUOTE` case is easy. It corresponds to an expression such as for
example `'(+ 1 2)`, so `r` is set to `(+ 1 2)` and then the
continuation is applied.


The `define` form takes two arguments, a symbol and an expression. The
expression is evaluated and then the result of that evaluation is
stored in a mapping from the symbol to the value in the
environment. This will be the first case where a continuation is
created. This function also performs a check to see that the
programmer is not rebinding nil. 

```
        // Special form: DEFINE
        if (dec_sym(head) == symrepr_define()) {
          VALUE key = car(cdr(ctx->curr_exp));
          VALUE val_exp = car(cdr(cdr(ctx->curr_exp)));

          if (type_of(key) != VAL_TYPE_SYMBOL ||
              key == NIL) {
            done = true;
            r =  enc_sym(symrepr_eerror());
            continue;
          }

          push_u32_2(ctx->K, key, enc_u(SET_GLOBAL_ENV));
          ctx->curr_exp = val_exp;
          continue;
        }
```

The `key` and `val_exp` variables are set to the arguments to
`define`. Then at the end, the `key` and the continuation
`SET_GLOBAL_ENV` are pushed to the continuation stack. The current
expression is set to `val_exp`. This means that in the next iteration
the evaluation loop will evaluate the `val_exp`. When evaluating
`val_exp` is done, the continuation will be called with the result of
that evaluation and an additional argument, the `key`, that is already
on the stack. So when entering into the `SET_GLOBAL_ENV` continuation
all the data is available to create the global binding.


The `progn` takes a sequence of expressions as arguments that are all
supposed to be evaluated in turn, from the first to the last. The
result of evaluating the last expression in the sequence is the result
of the whole thing. There is a check for an empty sequence of
expressions which results in `nil` otherwise a continuation is created
in this case as well.

```
        // Special form: PROGN
        if (dec_sym(head) == symrepr_progn()) {
          VALUE exps = cdr(ctx->curr_exp);

          if (type_of(exps) == VAL_TYPE_SYMBOL && exps == NIL) {
            r = enc_sym(symrepr_nil());
            app_cont = true;
            continue;
          }

          if (symrepr_is_error(exps)) {
            r = exps;
            done = true;
            continue;
          }
          push_u32_2(ctx->K, cdr(exps), enc_u(PROGN_REST));
          ctx->curr_exp = car(exps);
          continue;
        }
```

The `progn` case sets up for the continuation by setting the current
expression to the head of the sequence of expressions. the *pointer
to* the rest of the list is pushed onto the stack as well as the
`PROGN_REST` continuation. The `PROGN_REST` continuation thus has two
arguments to work with once it gets called. The result of the
previously evaluated expression (head) and the rest of the
sequence. Later in this text we will see exactly what the continuation
functions does in all of these cases.


The `lambda` case does not create a continuation but it is interesting
for another reason.  This is the first example of a case that
allocates heap memory and thus must be pausable and restartable in the
case that there is no free heap space. 

```
        // Special form: LAMBDA
        if (dec_sym(head) == symrepr_lambda()) {

          VALUE env_cpy = env_copy_shallow(ctx->curr_env);
	  
          if (type_of(env_cpy) == VAL_TYPE_SYMBOL &&
              dec_sym(env_cpy) == symrepr_merror()) {
             perform_gc = true;
             app_cont = false;
             continue; // perform gc and resume evaluation at same expression
          }

          VALUE env_end;
          VALUE body;
          VALUE params;
          VALUE closure;
          env_end = cons(env_cpy,NIL);
          body    = cons(car(cdr(cdr(ctx->curr_exp))), env_end);
          params  = cons(car(cdr(ctx->curr_exp)), body);
          closure = cons(enc_sym(symrepr_closure()), params);

          if (type_of(env_end) == VAL_TYPE_SYMBOL ||
              type_of(body)    == VAL_TYPE_SYMBOL ||
              type_of(params)  == VAL_TYPE_SYMBOL ||
              type_of(closure) == VAL_TYPE_SYMBOL) {
            perform_gc = true;
            app_cont = false;
            continue; // perform gc and resume evaluation at same expression
          }

          app_cont = true;
          r = closure;
          continue;
        }
```

The `lambda` starts out by doing a shallow copy of the local
environment, this means that skeleton of the environment is traversed
and copied but the copy will point to the same mappings as the
original. So only enough new cons-cells are used to for the
skeleton-structure of the environment, none for the data itself. This
consing can fail with an *memory error*, `symrepr_merror()`, symbol as
result in which case we are out of heap. In this case the `perform_gc`
flag is set and `app_cont` is cleared. An important detail is that it
does not alter the current expression which means that once the
evaluation starts up again the current expression will still be the
same lambda case. Thus evaluation will resume. There are more
complicated cases of this where changes to the stack have been made
and we have to restore this to proper state before continuing, this
will appear later.

After copying the environment, a `closure` is created from the
environment copy and the contents of the `lambda`. This again uses
cons and allocates heap space and is followed by a similar check for a
memory error. If it all works out though, `app_cont` is set and `r` is
set to the `closure` and evaluation continues. 


Now the `if` case: 
```
        // Special form: IF
        if (dec_sym(head) == symrepr_if()) {

          push_u32_3(ctx->K,
                     car(cdr(cdr(cdr(ctx->curr_exp)))), // Else branch
                     car(cdr(cdr(ctx->curr_exp))),      // Then branch
                     enc_u(IF));
          ctx->curr_exp = car(cdr(ctx->curr_exp));
          continue;
        }
```

The `if` case sets up for a continuation that takes the then and else
branch expressions as arguments. Then the current expression is set to
be the boolean expression that is used to select then or else. So once
the boolean has been evaluated we will enter into the continuation
that can then set up the system for either evaluating the then or the
else branch.


The `let` takes two arguments, a list of bindings to extend the local
environment with and an expression to evaluate in the extended
environment. So, to begin with these pieces are bound to variables
`orig_env`, `binds` and `exp` for easy access.

```
        // Special form: LET
        if (dec_sym(head) == symrepr_let()) {
          VALUE orig_env = ctx->curr_env;
          VALUE binds    = car(cdr(ctx->curr_exp)); // key value pairs.
          VALUE exp      = car(cdr(cdr(ctx->curr_exp))); // exp to evaluate in the new env.

          VALUE curr = binds;
          VALUE new_env = orig_env;

          if (type_of(binds) != PTR_TYPE_CONS) {
            // binds better be nil or there is a programmer error.
            ctx->curr_exp = exp;
            continue;
          }

          // Implements letrec by "preallocating" the key parts
          while (type_of(curr) == PTR_TYPE_CONS) {
            VALUE key = car(car(curr));
            VALUE val = NIL;
            VALUE binding;
            binding = cons(key, val);
            new_env = cons(binding, new_env);

            if (type_of(binding) == VAL_TYPE_SYMBOL ||
                type_of(new_env) == VAL_TYPE_SYMBOL) {
              perform_gc = true;
              app_cont = false;
              continue;
            }
            curr = cdr(curr);
          }

          VALUE key0 = car(car(binds));
          VALUE val0_exp = car(cdr(car(binds)));

          push_u32_5(ctx->K, exp, cdr(binds), new_env,
                     key0, enc_u(BIND_TO_KEY_REST));
          ctx->curr_exp = val0_exp;
          ctx->curr_env = new_env;
          continue;
        }
```

The kind of `let` implemented here allows bindings defined early in
the list of bindings to be used in the definition of the later
ones. This is accomplished by a two phase approach where first the
environment containing the bindings is created using just the keys
(and mapping each of the keys to nil). Then the expressions whose
results are to be bound to the keys are evaluated one after the other,
in order, in this new environment. Once an expression is evaluated the
mapping for that key (that currently is `nil`) is updated. Of course
it is not possible to use a later binding in a currently evaluated
expression as that binding would return 'nil` rather than what it is
supposed to.

So after setting up the environment with `key` - `nil` bindings, a
continuation is created and the current expression is set to evaluate
the first value to be bound.


Now we have dealt with all special forms (but one) the application
form. Once getting to this point we just assume that if the current
expression is a list, then it is a function application. This may turn
out to be wrong (as it may also be a programmer mistake) in which case
evaluation results in an error.

```
      } // If head is symbol
      push_u32_4(ctx->K,
                 ctx->curr_env,
                 enc_u(0),
                 cdr(ctx->curr_exp),
                 enc_u(APPLICATION_ARGS));

      ctx->curr_exp = head; // evaluate the function
      continue;
```

This case creates a continuation that needs to have access to the
current environment, a counter `enc_u(0)` (to count arguments), the
list of arguments `cdr(ctx->curr_exp)` and the continuation itself is
called `APPLICATION_ARGS`. The current expression is set to the head
of the list. This will be evaluated next and will result in either a
*function object* of some kind or an error. A function object could
for example be a closure or a symbol pointing out a fundamental
operation or an extension. 


This concludes the evaluation function. The rest just checks for some
serious errors and if the loop is exited `r` is returned as the result
of evaluation.

```
    default:
      // BUG No applicable case!
      done = true;
      r = enc_sym(symrepr_eerror());
      break;
    }
  } // while (!done)
  return r;
}
``` 

## Continuation points and apply continuation

So far there have been a number of examples of the creation of
continuations but what happens when these are applied is still untold.
The `apply_continuation` function, that takes care of this
continuation application, is about the same size as `run_eval` and really
they interact at a quite deep level. `run_eval` calls
`apply_continuation` and `apply_continuation` changes the evaluation
context meaning it influences what the next iteration of the evaluation
loop does.  


``` 
VALUE apply_continuation(eval_context_t *ctx, VALUE arg, bool *done, bool *perform_gc, bool *app_cont){

  VALUE k;
  pop_u32(ctx->K, &k);

  VALUE res;

  *app_cont = false;
```

`apply_continuation` starts out by popping of the top of the stack
that should contain one of the continuation function identifiers
defined close to the top of this document. The rest of the function
is, just like the evaluation loop, a large switch statement. 


```
  switch(dec_u(k)) {
  case DONE:
    *done = true;
    return arg;
```

In case we have reached the `DONE` continuation, pushed onto the stack
when entering `run_eval`, we are done, the `done` flag is set and the
argument to the continuation `arg` is returned. `arg` corresponds to
the variable `r` in the evaluator.

```
  case SET_GLOBAL_ENV:
    res = cont_set_global_env(ctx, arg, done, perform_gc);
    if (!(*done)) 
      *app_cont = true;
    return res;
```

The `SET_GLOBAL_ENV` continuation is broken out into a separate
function. I cannot really decide if I should break all of these cases
out into functions or not. The `cont_set_global_env` function is shown
further down.

Now, the `PROGN` continuation is where a property one breaking mistake
was made. Towards the end of the function there is a comment `// allow
for tail recursion` right before a conditional that checks if the rest
of the sequence of expressions is nil. If it is nil, we are currently
in tail-call position and should not push anything to the stack that
we have to potentially wait a long time before it is freed. Forgetting
about this special case means a nil is pushed to the stack, only to be
dealt with once the tail-call is handled. 

```
  case PROGN_REST: {
    VALUE rest;
    pop_u32(ctx->K, &rest);
    if (type_of(rest) == VAL_TYPE_SYMBOL && rest == NIL) {
      res = arg;
      *app_cont = true;
      return res;
    }

    if (symrepr_is_error(rest)) {
      res = rest;
      *done = true;
      return res;
    }
    // allow for tail recursion
    if (type_of(cdr(rest)) == VAL_TYPE_SYMBOL &&
	cdr(rest) == NIL) {
      ctx->curr_exp = car(rest);
      return NONSENSE;
    }
    // Else create a continuation 
    push_u32_2(ctx->K, cdr(rest), enc_u(PROGN_REST));
    ctx->curr_exp = car(rest);
    return NONSENSE;
  }
```

The `PROGN` case sets up the head of the sequence of expressions to
evaluated in the next iteration of the evaluation loop and pushes the
rest of the sequence along with the same `PROGN_REST` continuation
onto the stack.



The `APPLICATION` case is a bit long and messy but really tricky
compared to the other cases. It just gets a bit messy because there
are a number of things that can be applied. There are `closure`s,
`fundamental` operations and `extension`s and each of these get a
special case in the `APPLICATION` continuation.

When entering into the `APPLICATION` case, all the arguments, a
counter and the function are present of the stack. in the case of
`fundamental` or `extension` operations this knowledge is utilized and
a pointer to the first argument on the stack is passed to the
function.  When it comes to the closures however, the arguments are
extracted from the stack and added to an augmented environment for the
closures body to evaluate in.

It is important that after running a `fundamental` or an `extension`
the stack is cleared, or we will fall into the property 1 issue again.

```
  case APPLICATION: { 
    VALUE count;
    pop_u32(ctx->K, &count);

    UINT *fun_args = stack_ptr(ctx->K, dec_u(count)+1);

    VALUE fun = fun_args[0];

    if (type_of(fun) == PTR_TYPE_CONS) { // a closure (it better be)
      VALUE args = NIL;
      for (UINT i = dec_u(count); i > 0; i --) {
	args = cons(fun_args[i], args);
	if (type_of(args) == VAL_TYPE_SYMBOL) {
	  push_u32_2(ctx->K, count, enc_u(APPLICATION));
	  *perform_gc = true;
	  *app_cont = true;
	  return fun;
	}
      }
      VALUE params  = car(cdr(fun));
      VALUE exp     = car(cdr(cdr(fun)));
      VALUE clo_env = car(cdr(cdr(cdr(fun))));
     
      if (length(params) != length(args)) { // programmer error
	*done = true;
	return enc_sym(symrepr_eerror());
      }

      VALUE local_env = env_build_params_args(params, args, clo_env);
      if (type_of(local_env) == VAL_TYPE_SYMBOL) { 
	if (dec_sym(local_env) == symrepr_merror() ) {
	  push_u32_2(ctx->K, count, enc_u(APPLICATION));
	  *perform_gc = true;
	  *app_cont = true;
	  return fun;
	}
	
	if (dec_sym(local_env) == symrepr_fatal_error()) {
	  return local_env;
	}
      }
      stack_drop(ctx->K, dec_u(count)+1);
      ctx->curr_exp = exp;
      ctx->curr_env = local_env;
      return NONSENSE;
    } else if (type_of(fun) == VAL_TYPE_SYMBOL) {
      
      VALUE res;
      
      if (is_fundamental(fun)) {
	res = fundamental_exec(&fun_args[1], dec_u(count), fun);
	if (type_of(res) == VAL_TYPE_SYMBOL &&
	    dec_sym(res) == symrepr_eerror()) {
	  
	  *done = true;
	  return  res;
	} else if (type_of(res) == VAL_TYPE_SYMBOL &&
		   dec_sym(res) == symrepr_merror()) {
	  push_u32_2(ctx->K, count, enc_u(APPLICATION));
	  *perform_gc = true;
	  *app_cont = true;
	  return fun;
	} 
	stack_drop(ctx->K, dec_u(count) + 1);
	*app_cont = true;
	return res;
      }
    }
    
    // It may be an extension
    
    extension_fptr f = extensions_lookup(dec_sym(fun));
    if (f == NULL) {
      *done = true;
      return enc_sym(symrepr_eerror());
    }

    VALUE ext_res = f(&fun_args[1] , dec_u(count));

    if (type_of(ext_res) == VAL_TYPE_SYMBOL &&
	(dec_sym(ext_res) == symrepr_merror())) {
      push_u32_2(ctx->K, count, enc_u(APPLICATION));
      *perform_gc = true;
      *app_cont = true;
      return fun;
    }
    
    stack_drop(ctx->K, dec_u(count) + 1);

    *app_cont = true;
    return ext_res;
  }
```

The `APPLICATION_ARGS' continuation handles evaluation of the list of arguments to a function. 
``` 
  case APPLICATION_ARGS: {
    VALUE count;
    VALUE env;
    VALUE rest;

    pop_u32_3(ctx->K, &rest, &count, &env);
    push_u32(ctx->K, arg);
    
    if (type_of(rest) == VAL_TYPE_SYMBOL &&
	rest == NIL) {
      // no more arguments
      push_u32_2(ctx->K, count, enc_u(APPLICATION));
      *app_cont = true;
      return NONSENSE;
    }
    push_u32_4(ctx->K, env, enc_u(dec_u(count) + 1), cdr(rest), enc_u(APPLICATION_ARGS));
    ctx->curr_exp = car(rest);
    ctx->curr_env = env;
    return NONSENSE; 
  }
```

Here the current expression is set up to evaluate the head of the list
of the rest of the arguments. A continuation is created for evaluating
the rest of the arguments. When creating this continuation the
number-of-arguments counter is incremented.

Each time `APPLICATION_ARGS` is entered the previously evaluated
argument (or the function in case of the first call to
`APPLICATION_ARGS`) is pushed onto the stack. This means that the
stack is now also the vehicle for transport of arguments to the
functions. When the list of arguments have been fully evaluated, the
stack will contain the function, all of the arguments and the counter
value and a `APPLICATION` continuation is created. 


The `BIND_TO_KEY_REST` continuation, that is created in the `let` case
of the evaluation loop, takes care of evaluating all the expressions
to be bound to variables in the local environment. 

```
  case BIND_TO_KEY_REST:{
    VALUE key;
    VALUE env;
    VALUE rest;

    pop_u32_3(ctx->K, &key, &env, &rest);

    env_modify_binding(env, key, arg);

    if ( type_of(rest) == PTR_TYPE_CONS ){
      VALUE keyn = car(car(rest));
      VALUE valn_exp = car(cdr(car(rest)));

      push_u32_4(ctx->K, cdr(rest), env, keyn, enc_u(BIND_TO_KEY_REST));

      ctx->curr_exp = valn_exp;
      ctx->curr_env = env;
      return NONSENSE;
    }

    // Otherwise evaluate the expression in the populated env
    VALUE exp;
    pop_u32(ctx->K, &exp);
    ctx->curr_exp = exp;
    ctx->curr_env = env;
    return NONSENSE;
  }
```

Each time `BIND_TO_KEY_REST` is called the environment is modified
with the recently evaluated value. Once all bindings have been dealt
with, the let body is set up to be evaluated within the newly formed
local environment.


The `IF` continuation gets the result of evaluating the boolean
condition as an argument. It then sets up for evaluation of either the
then or the else branch in the evaluation loop.

```
  case IF: {
    VALUE then_branch;
    VALUE else_branch;

    pop_u32_2(ctx->K, &then_branch, &else_branch);

    if (type_of(arg) == VAL_TYPE_SYMBOL && dec_sym(arg) == symrepr_true()) {
      ctx->curr_exp = then_branch;
    } else {
      ctx->curr_exp = else_branch;
    }
    return NONSENSE;
  }
```


Now, if we do reach the end of this switch, something is very wrong
and an error is return and evaluation is aborted.

```
  } // end switch
  *done = true;
  return enc_sym(symrepr_eerror());
}

```

Back to that broken out function that sets the global environment. It
does the work needed for the `SET_GLOBAL_ENV` continuation. The `key`
to use is on the stack and the value is passed as argument to the
continuation. Not we set up a new environment which is the `key` -
`val` pair consed onto the old global environment. This consing (that
is done inside of the `env_set` function) can fail as usual and we
need to be able to be able to pause the computation and perform
garbage collection. However, if all works out fine, the global
variable `eval_cps_global_env` is updated with the value of the newly
created one.

```
VALUE cont_set_global_env(eval_context_t *ctx, VALUE val, bool *done, bool *perform_gc){

  VALUE key;
  pop_u32(ctx->K, &key);
  VALUE new_env = env_set(eval_cps_global_env,key,val);

  if (type_of(new_env) == VAL_TYPE_SYMBOL) {
    if (dec_sym(new_env) == symrepr_merror()) {
      push_u32_2(ctx->K, key, enc_u(SET_GLOBAL_ENV));
      *perform_gc = true;
      return val;
    }
    if (dec_sym(new_env) == symrepr_fatal_error()) {
      *done = true;
      return new_env; 
    }
  }
  eval_cps_global_env = new_env;
  return enc_sym(symrepr_true());
}
```

## When Property 1 is Not Honored

In order to see what happens when we fail to honor property 1, the
`push_u32` function that is used in all functions that push onto the
stack is augmented with a print statement. This print statement prints
the stack pointer `sp` and the value that is being pushed onto the
stack. This experiment is run with a stack that does not allow
growing, this is checked internally in the function `stack_grow` and
it will not reallocate the stack if this flag is set.

```
int push_u32(stack *s, UINT val) {
  int res = 1;
  s->data[s->sp] = val;
  /* Added printf */
  printf("Stack sp %d : ",s->sp); simple_print(val); printf("\n");
  s->sp++;
  if ( s->sp >= s->size) {
    res = stack_grow(s);
  }
  return res;
}
```

The `PROGN_REST` continuation is a good target to introduce this error
in.  Below is what it looks like with the code that makes it `//allow
for tail recursion`.

```
  case PROGN_REST: {
    VALUE rest;
    pop_u32(ctx->K, &rest);
    if (type_of(rest) == VAL_TYPE_SYMBOL && rest == NIL) {
      res = arg;
      *app_cont = true;
      return res;
    }

    if (symrepr_is_error(rest)) {
      res = rest;
      *done = true;
      return res;
    }
    // allow for tail recursion
    if (type_of(cdr(rest)) == VAL_TYPE_SYMBOL &&
	cdr(rest) == NIL) {
      ctx->curr_exp = car(rest);
      return NONSENSE;
    }
    // Else create a continuation 
    push_u32_2(ctx->K, cdr(rest), enc_u(PROGN_REST));
    ctx->curr_exp = car(rest);
    return NONSENSE;
  }
```
So let's modify the code by removing that little block. Now it looks as follows. 

```
  case PROGN_REST: {
    VALUE rest;
    pop_u32(ctx->K, &rest);
    if (type_of(rest) == VAL_TYPE_SYMBOL && rest == NIL) {
      res = arg;
      *app_cont = true;
      return res;
    }

    if (symrepr_is_error(rest)) {
      res = rest;
      *done = true;
      return res;
    }
    // Else create a continuation 
    push_u32_2(ctx->K, cdr(rest), enc_u(PROGN_REST));
    ctx->curr_exp = car(rest);
    return NONSENSE;
  }
```

Now if we build lispBM and the REPL with this faulty code and enter the following program which is a tail-recursive run-forever program.

```
(define f
  (lambda (x)
    (progn (print "hello")
	   (f (+ x 1)))))
```

The following is the result:

```
Lisp REPL started!
Type :quit to exit.
     :info for statistics.
# (define f (lambda (x) (progn (print "hello") (f (+ x 1)))))
Stack sp 0 : 1
Stack sp 1 : f
Stack sp 2 : 2
> t
# (f 0)
Stack sp 0 : 1
Stack sp 1 : nil
Stack sp 2 : 0
Stack sp 3 : (0 nil)
Stack sp 4 : 8
Stack sp 1 : (closure ((x nil) ((progn ((print ("hello" nil)) ((f ((+ (x (1 nil))) nil)) nil))) (nil nil))))
Stack sp 2 : nil
Stack sp 3 : 1
Stack sp 4 : nil
Stack sp 5 : 8
Stack sp 2 : 0
Stack sp 3 : 1
Stack sp 4 : 7
Stack sp 1 : ((f ((+ (x (1 nil))) nil)) nil)
Stack sp 2 : 6
Stack sp 3 : ((x 0) nil)
Stack sp 4 : 0
... 
Stack sp 98 : 48
Stack sp 99 : 1
Stack sp 100 : 7
Stack sp 97 : ((f ((+ (x (1 nil))) nil)) nil)
Stack sp 98 : 6
Stack sp 99 : ((x 48) nil)
Stack sp 100 : 0
Stack sp 101 : ("hello" nil)
Stack sp 102 : 8
Segmentation fault

```

The stack just keeps growing until it hits about 100 elements (the
size of the stack) and a segfault is triggered. We can also see that `x`
has reached the value 48, so 48 iterations of the infinite recursion
have been executed and I guess we can conclude that the stack is
growing with approximately 2 element per iteration.

When the REPL runs with the fixed code and the same program it keeps running:

```
Stack sp 1 : (closure ((x nil) ((progn ((print ("hello" nil)) ((f ((+ (x (1 nil))) nil)) nil))) (nil nil))))
Stack sp 2 : ((x 3159) nil)
Stack sp 3 : 1
Stack sp 4 : nil
Stack sp 5 : 7
Stack sp 6 : ((x 3159) nil)
Stack sp 7 : 0
Stack sp 8 : (x (1 nil))
Stack sp 9 : 7
Stack sp 6 : +
Stack sp 7 : ((x 3159) nil)
Stack sp 8 : 1
Stack sp 9 : (1 nil)
Stack sp 10 : 7
Stack sp 7 : 3159
Stack sp 8 : ((x 3159) nil)
Stack sp 9 : 2
Stack sp 10 : nil
Stack sp 11 : 7
Stack sp 8 : 1
Stack sp 9 : 2
Stack sp 10 : 6
Stack sp 2 : 3160
Stack sp 3 : 1
Stack sp 4 : 6
Stack sp 1 : ((f ((+ (x (1 nil))) nil)) nil)
Stack sp 2 : 5
Stack sp 3 : ((x 3160) nil)
Stack sp 4 : 0
Stack sp 5 : ("hello" nil)
Stack sp 6 : 7
Stack sp 3 : print

```

Here execution was stopped after a short while with ctrl+c and we
can see that the stack pointer is at most 11 and also that we have
incremented `x` to 3160 by now. Looks a lot better! 

___

[HOME](https://svenssonjoel.github.io)
