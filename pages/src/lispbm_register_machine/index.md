

# Evaluation of lispBM expressions using a Register Machine

There is a section in the
[SICP](https://mitpress.mit.edu/sites/default/files/sicp/full-text/book/book-Z-H-34.html)
book about an "explicit control evaluator". In my physical copy of the
book it is chapter 5.2 but in the online version it seems to be
chapter 5.4. The code that I am about to write about here is based on
(my understanding of) chapter 5.2 in the physical "dead tree" version
of the book.

The code here is strongly influenced by the code in SICP, a very
similar register machine is used but with 8 registers instead of
7. LispBM and the SICP "Lisp" are different though, so there are small
differences in how the evaluation is performed. A bigger difference,
presentation-wise , is that this implementation is expressed in C
compared to the lisp dialect used as implementation language in SICP.

LispBM is my, now pretty long-running, hobby project with the main
purpose of having fun but also for learning about lisps and
microcontrollers. My intention with lispBM is that it should be
possible to run on some somewhat resource constrained targets, for
example the STM32, NRF52 and ESP32. For more information and an
overview of lispBM look
[here](../lispbm_current_status/index.html). All source code is
available at [github](http://www.github.com/svenssonjoel/lispbm).

Let's jump right in!

## Register machine

The register machine consists of 8 registers with very special
purposes. The table below gives a short introduction to the purposes
of the registers but this will become more clear as we go deeper into the
implementation later. 


| Register | Description/Purpose |
|:---------|:--------------------:|
|cont      | Hold information on what to do after evaluation of a leaf node expression |
|env       | Local environments are held here. Evaluated bindings in a `let` expression end up here |
|unev      | Is a storage location for unevaluated expressions to evaluate "later" |
|prg       | Contains the rest of the program. A lispBM program is a sequence (list) of expressions |
|exp       | The expression under consideration right now |
|argl      | Arguments in an application are accumulated here|
|val       | Results end up in this register|
|fun       | Evaluated function expression end up in the fun register in preparation to be applied to the contents of argl|


In the implementation, the registers of the machine are represented by
the following C struct.

```
typedef struct {
  uint32_t cont;
  VALUE env;
  VALUE unev;
  VALUE prg;
  VALUE exp;
  VALUE argl;
  VALUE val;
  VALUE fun;

  stack S;
} register_machine_t;
```

The state that is manipulated as the register machine is "running" is shown below.
it consists of one set of registers and a global environment.

```
register_machine_t rm_state;
VALUE ec_eval_global_env;

```

lispBM has a global environment that can be extended using
`define`. The global environment feels very useful when working with a
REPL as it remembers definitions between the expressions entered. 


## More preliminaries 

Evaluation using the register machine has an entry point, called
`ec_eval` that we will look at later. In this entry point, depending
on the expression and a little bit of other state, a branch will be
taken that performs a "step" of evaluation. So, some way to
discriminate between different kinds of expressions is needed.

Garbage collection also has to be adapted to this slightly new way of
evaluating expressions. 

### Discriminate between different kinds of expressions 

The different kinds of expression that take different paths through `ec_eval` are defined
in the enumeration below. 

```
typedef enum {
  EXP_KIND_ERROR,
  EXP_SELF_EVALUATING,
  EXP_VARIABLE,
  EXP_QUOTED,
  EXP_DEFINE,
  EXP_LAMBDA,
  EXP_IF,
  EXP_PROGN,
  EXP_NO_ARGS,
  EXP_APPLICATION,
  EXP_LET,
  EXP_AND,
  EXP_OR
} exp_kind;
```

The `kind_of` function takes an expressions and returns the appropriate `exp_kind`
```
exp_kind kind_of(VALUE exp) {

  switch (type_of(exp)) {
  case VAL_TYPE_SYMBOL:
    if (!is_special(exp))
      return EXP_VARIABLE;
    // fall through
  case PTR_TYPE_BOXED_F:
  case PTR_TYPE_BOXED_U:
  case PTR_TYPE_BOXED_I:
  case VAL_TYPE_I:
  case VAL_TYPE_U:
  case VAL_TYPE_CHAR:
  case PTR_TYPE_ARRAY:
    return EXP_SELF_EVALUATING;
  case PTR_TYPE_CONS: {
    VALUE head = car(exp);
    if (type_of(head) == VAL_TYPE_SYMBOL) {
      UINT sym_id = dec_sym(head);

      if (sym_id == symrepr_and())
        return EXP_AND;
      if (sym_id == symrepr_or())
        return EXP_OR;
      if (sym_id == symrepr_quote())
        return EXP_QUOTED;
      if (sym_id == symrepr_define())
        return EXP_DEFINE;
      if (sym_id == symrepr_progn())
        return EXP_PROGN;
      if (sym_id == symrepr_lambda())
        return EXP_LAMBDA;
      if (sym_id == symrepr_if())
        return EXP_IF;
      if (sym_id == symrepr_let())
        return EXP_LET;
      if (type_of(cdr(exp)) == VAL_TYPE_SYMBOL &&
          dec_sym(cdr(exp)) == symrepr_nil()) {
        return EXP_NO_ARGS;
      }
    } // end if symbol
    return EXP_APPLICATION;
  } // end case PTR_TYPE_CONS:
  }
  return EXP_KIND_ERROR;
}
```

Really, all that this does is make it possible write the body of
`ec_eval` in a little bit cleaner way (fewer levels of nested conditionals).

### Garbage collection in the register machine setting

Together with the global environment and the so-called freelist, the
contents of the registers specify exactly what has to be kept alive
past a run of the garbage collector. So, the `gc` function starts a
mark phase from the contents of each register.

```
static int gc(VALUE env,
              register_machine_t *rm) {

  gc_state_inc();
  gc_mark_freelist();
  gc_mark_phase(env);

  gc_mark_phase(rm->env);
  gc_mark_phase(rm->unev);
  gc_mark_phase(rm->prg);
  gc_mark_phase(rm->exp);
  gc_mark_phase(rm->argl);
  gc_mark_phase(rm->val);
  gc_mark_phase(rm->fun);
  gc_mark_aux(rm->S.data, rm->S.sp);

  return gc_sweep_phase();
}

```

This `gc` routine is called, if necessary, from the evaluator at any
point where a cons cell is allocated. If the allocation of a cons-cell
results in the memory_error symbol. `gc` is initiated. When `gc`
completes another attempts at allocating a cons-cell is
performed. This is shown in more detail as it appears when later
describing the evaluator. 

### Predicates used throughout 

A few helpful predicates are used to make the code a bit shorter and
easier to read. 

``` 
static inline bool is_symbol(VALUE exp) {
  return (type_of(exp) == VAL_TYPE_SYMBOL);
}
```
The `is_symbol` returns `true` if `exp` is a symbol.

```
static inline bool is_symbol_nil(VALUE exp) {
  return (is_symbol(exp) && dec_sym(exp) == symrepr_nil());
}
```

The `is_symbol_nil` returns `true` only for the specific symbol that
represents `nil`. Likewise, `is_symbol_eval` and `is_symbol_merror` return
true only for the eval symbol and the memory error symbol.

```
static inline bool is_symbol_eval(VALUE exp) {
  return (is_symbol(exp) && dec_sym(exp) == symrepr_eval());
}

static inline bool is_symbol_merror(VALUE exp) {
  return (is_symbol(exp) && dec_sym(exp) == symrepr_merror());
}
```

`last_operand` checks if there is exactly one element in a list.

```
static inline bool last_operand(VALUE exp) {
  return is_symbol_nil(cdr(exp));
}
```


## The dispatcher

The function that is called from the REPL to evaluate a program
is called `ec_eval_program` and takes just a program as a single argument.
This function sets up the register machine in an initial state, allocates a stack
and then calls `ec_eval` that is doing the real work. When `ec_eval` is finished
control returns to `ec_eval_program` that frees the stack and returns the contents of
the val register to the user.

```
VALUE ec_eval_program(VALUE prg) {

  rm_state.prg = cdr(prg);
  rm_state.exp = car(prg);
  rm_state.cont = enc_u(CONT_DONE);
  rm_state.env = enc_sym(symrepr_nil());
  rm_state.argl = enc_sym(symrepr_nil());
  rm_state.val = enc_sym(symrepr_nil());
  rm_state.fun = enc_sym(symrepr_nil());
  stack_allocate(&rm_state.S, 256, false);
  ec_eval();

  stack_free(&rm_state.S);
  return rm_state.val;
}
```

The only really important detail within `ec_eval_program` is that the
cont register is set to `CONT_DONE`. When the `CONT_DONE` continuation
is "run", after the evaluation of an expression, either evaluation
steps to the next expression in the program or if there are no more
expressions, evaluation finishes.

The evaluator, `ec_eval` can be in one of three states called
`EVAL_DISPATCH`, `EVAL_CONTINUATION` and `EVAL_APPLY_DISPATCH`.  This
will become more clear later, but in the `EVAL_DISPATCH` state the
control will branch away depending on what kind of expression is
residing in the exp register. In the `EVAL_CONTINUATION` state,
control branches depending on the contents of the cont register
instead. Finally, in state `EVAL_APPLY_DISPATCH` the branching depends
on what kind of function resides in the fun register.


```
typedef enum {
  EVAL_DISPATCH,
  EVAL_CONTINUATION,
  EVAL_APPLY_DISPATCH
} eval_state;
```

There are a number of different continuations that can be indicated
within the cont register. The continuations are listed below and
their use will become clear as we look at where they are "created"
within the implementation. 

```
typedef enum {
  CONT_DONE,
  CONT_DEFINE,
  CONT_SETUP_NO_ARG_APPLY,
  CONT_EVAL_ARGS,
  CONT_ACCUMULATE_ARG,
  CONT_ACCUMULATE_LAST_ARG,
  CONT_BRANCH,
  CONT_BIND_VAR,
  CONT_SEQUENCE,
  CONT_AND,
  CONT_OR
} continuation;

```

With that in place, the `ec_eval` function is just a big switch
wrapped in a while loop. The loop will repeat until a `done` flag is
set to true. This flag is set to `true` within the `CONT_DONE`
continuation in case there are no more expressions to evaluate.

Depending on the state stored in `es`, `ec_eval` either dispatches
based on the kind of expression within register exp, or the
continuation stored within register cont, or finally depending on kind
of function application. Function application dispatch is handled by a
function called `eval_apply_dispatch` that we will look at further down.

```
void ec_eval(void) {

  eval_state es = EVAL_DISPATCH;

  bool done = false;

  while (!done) {

    switch(es) {
    case EVAL_DISPATCH:
      switch (kind_of(rm_state.exp)) {
      case EXP_SELF_EVALUATING: eval_self_evaluating(&es); break;
      case EXP_VARIABLE:        eval_variable(&es);        break;
      case EXP_QUOTED:          eval_quoted(&es);          break;
      case EXP_DEFINE:          eval_define(&es);          break;
      case EXP_NO_ARGS:         eval_no_args(&es);         break;
      case EXP_APPLICATION:     eval_application(&es);     break;
      case EXP_LAMBDA:          eval_lambda(&es);          break;
      case EXP_PROGN:           eval_progn(&es);           break;
      case EXP_IF:              eval_if(&es);              break;
      case EXP_LET:             eval_let(&es);             break;
      case EXP_AND:             eval_and(&es);             break;
      case EXP_OR:              eval_or(&es);              break;
      case EXP_KIND_ERROR:      done = true;               break;
      }
      break;
    case EVAL_CONTINUATION:
      switch (dec_u(rm_state.cont)) {
      case CONT_DONE:                cont_done(&es, &done);         break;
      case CONT_DEFINE:              cont_define(&es);              break;
      case CONT_SETUP_NO_ARG_APPLY:  cont_setup_no_arg_apply(&es);  break;
      case CONT_EVAL_ARGS:           cont_eval_args(&es);           break;
      case CONT_ACCUMULATE_ARG:      cont_accumulate_arg(&es);      break;
      case CONT_ACCUMULATE_LAST_ARG: cont_accumulate_last_arg(&es); break;
      case CONT_BRANCH:              cont_branch(&es);              break;
      case CONT_BIND_VAR:            cont_bind_var(&es);            break;
      case CONT_SEQUENCE:            cont_sequence(&es);            break;
      case CONT_AND:                 cont_and(&es);                 break;
      case CONT_OR:                  cont_or(&es);                  break;
      }
      break;
    case EVAL_APPLY_DISPATCH:  eval_apply_dispatch(&es); break;
    }
  }
}
```

So, `ec_eval` really delegates all the work to a collection of functions depending
on the state. The sections below will go through these function in groups of "eval" and "cont" functions
that depend on each other.

## Evaluation of self-evaluating expressions

Some expressions evaluate to them self. Examples of such expressions
are, a value such as 42 or a string like "hello world". When these are
encountered, the exp register is copied to the val register and the `ec_eval` state
is set to `EVAL_CONTINUATION`. 

```
static inline void eval_self_evaluating(eval_state *es) {
  rm_state.val = rm_state.exp;
  *es = EVAL_CONTINUATION;
}
```

This means that in the next iteration of the loop in `ec_eval` the
dispatch will be based on the contents of the cont register.

If for example this particular self evaluating expression was the 42
in `(+ 1 42)`, the continuation would now proceed to apply the
fundamental operation `+` to the now evaluated arguments. 

## Evaluation of variables 

Variables are evaluated by checking if they are defined within either
the global or local environment. In lispBM there are also "special"
symbols that always evaluate to them self. These "special" symbols are
used for example for names of built in "fundamental" operations. 

If no binding is found and the variable is not a special symbol, val
will contain a symbol indicating that the lookup failed.

```
static inline void eval_variable(eval_state *es) {
  if (is_special(rm_state.exp)) rm_state.val = rm_state.exp;
  else if (is_extension(rm_state.exp)) rm_state.val = rm_state.exp;
  else rm_state.val = env_lookup(rm_state.exp, rm_state.env);

  if (type_of(rm_state.val) == VAL_TYPE_SYMBOL &&
      dec_sym(rm_state.val) == symrepr_not_found()) {
    rm_state.val = env_lookup(rm_state.exp, ec_eval_global_env);
  }
  *es = EVAL_CONTINUATION;
}
```

## Evaluation of quoted expressions

Evaluation of a quoted expression such as `(quote apa)` or equivalently `'apa`
is done by putting the quoted expression directly into the val register.

Internally a quoted expression is a list with the symbol `quote` at
the head (or car) position followed by the argument expression to
return unevaluated. So this second element or the car(cdr(qouted_exp))
is result of the evaluation. 

```
static inline void eval_quoted(eval_state *es) {
  rm_state.val = car(cdr(rm_state.exp));
  *es = EVAL_CONTINUATION;
}
``` 

## Evaluation of define expressions

Evaluation of a define expression is the first of the examples that
will create a continuation. a define expression has the form `(define
variable expression)` that is a list with three elements where we are
interested in element 2 and 3. Element 3, should be evaluated and then
stored into the environment together with the key in element 2. But element 3
is not yet evaluated and this is where the continuation comes in.

The key is placed in the unev register, because we will need it later, and the
expression to evaluate and bind to the key is stored in the exp register.

The contents of unev and cont are pushed onto the stack.

The cont register is set to `CONT_DEFINE` and then the `es` state is
set to `EVAL_DISPATCH`. This means that in the next iteration of the
loop in `ec_eval` the dispatcher will dispatch depending on the
expression in the exp register and at this time that should hold the
expression we want to have evaluated to be associated with the key
within the environment!

When the evaluator is now running away and evaluates this expression,
it may use for example the unev register internally in doing so. This
is why we pushed that register to the stack before returning to the
dispatcher.

Eventually the expression will be evaluated and the continuation we
set up, `CONT_DEFINE` will be run.

```
static inline void eval_define(eval_state *es) {
  rm_state.unev = car(cdr(rm_state.exp));
  rm_state.exp  = car(cdr(cdr(rm_state.exp)));
  push_u32_2(&rm_state.S,
             rm_state.unev,
             rm_state.cont);
  rm_state.cont = enc_u(CONT_DEFINE);
  *es = EVAL_DISPATCH;
}
```

The `cont_define` function really has quite a simple task. It needs to
associate the key and evaluated result of the expression (now in the
val register) in the environment. The code however gets a little bit
complicated because of the use of `cons` that allocates a heap
cons-cell. This allocation can fail which means that the garbage
collector should be run. After running the garbage collector another
attempt to allocate the cons-cell is performed. If the second attempt
also fails to allocate a cell, the computation should be aborted since
we are clearly without any recoverable memory.

The continuation and the unev contents that we pushed are first restored.
The unev register now contains the key and the val register the value.

Then `env_set` is called (which internally uses cons) to add the new
association to the global environment.

```
static inline void cont_define(eval_state *es) {
  pop_u32_2(&rm_state.S,
            &rm_state.cont,
            &rm_state.unev);
  VALUE new_env = env_set(ec_eval_global_env,
                          rm_state.unev,
                          rm_state.val);
  if (is_symbol_nil(new_env)) {
    gc(ec_eval_global_env, &rm_state);
    new_env = env_set(ec_eval_global_env,
                          rm_state.unev,
                          rm_state.val);
  }
  if (is_symbol_nil(new_env)) {
    rm_state.cont = enc_u(CONT_DONE);
    *es = EVAL_CONTINUATION;
    return;
  }
  ec_eval_global_env = new_env;
  rm_state.val = rm_state.unev;
  *es = EVAL_CONTINUATION;
}
```

If the extension of the environment is successful, the val register is
set to the key and the `es` state is set to signal that it is time to
apply a continuation.

## Evaluation of lambda expressions 

Evaluation of lambda is another example that is really quite simple but made
more complicated because of allocation of cons-cells.

In lispBM a lambda is allocated to a closure, which is really just a
lambda extended with the environment present at the time of its
construction. A lambda has the form `(lambda (p1 p2 .. pn) expression)`
while a closure has the form `(closure (p1 p2 .. pn) expression environment)`

```
static inline void eval_lambda(eval_state *es) {
   rm_state.val = mkClosure();
  *es = EVAL_CONTINUATION;
}
``` 

So to create a closure, the list representing the lambda is broken up
and then the constituents are re-composed together with an environment
into a closure.

It then goes through the same process of checking if all the conses
were successful and possibly a retry before either succeeding or
failing.

```
static inline VALUE mkClosure(void) {

  VALUE env_end = cons(rm_state.env, enc_sym(symrepr_nil()));
  VALUE body    = cons(car(cdr(cdr(rm_state.exp))), env_end);
  VALUE params  = cons(car(cdr(rm_state.exp)),body);
  VALUE closure = cons(enc_sym(symrepr_closure()), params);

  if (is_symbol_merror(closure)) {
    gc(ec_eval_global_env, &rm_state);

    env_end = cons(rm_state.env, enc_sym(symrepr_nil()));
    body    = cons(car(cdr(cdr(rm_state.exp))), env_end);
    params  = cons(car(cdr(rm_state.exp)),body);
    closure = cons(enc_sym(symrepr_closure()), params);
  }

  if (is_symbol_merror(closure)) {
    // eval_lambda sets *es = EVAL_CONTINUATION
    // this replaces the existing continuation with "done"
    rm_state.cont = enc_u(CONT_DONE);
  }
  return closure;
}
```

In the end, the closure ends up in the val register. 


## Evaluate progn sequences of expressions

In lispBM you can evaluate a sequence of expression using the `progn`
facility.  The result of evaluating the sequence is the value that the
last expression in the sequence evaluates to. For example `(progn 1 2
3)` evaluates to 3.



```
static inline void eval_progn(eval_state *es) {
  push_u32(&rm_state.S, rm_state.cont);
  rm_state.unev = cdr(rm_state.exp);
  eval_sequence(es);
}
```

Here, the unev register is set to the expressions to evaluate and the
continuation is pushed since we will overwrite that
register. `eval_progn` then calls a function called `eval_sequence`.

`eval_sequence` is part of a loop that also involves a continuation
called `cont_sequence`. These two will take turns running for as long
as there are expressions in the sequence left to evaluate. 

`eval_sequence` sets up the exp register for evaluation of the
head of the sequence of expressions. Then, either we have exhausted the list
or we need to iterate via `cont_sequence`. 

``` 
static inline void eval_sequence(eval_state *es) {
  rm_state.exp = car(rm_state.unev);
  VALUE tmp = cdr(rm_state.unev);
  if (type_of(tmp) == VAL_TYPE_SYMBOL &&
      dec_sym(tmp) == symrepr_nil()) {
    pop_u32(&rm_state.S, &rm_state.cont);
    *es = EVAL_DISPATCH;
    return;
  }
  push_u32(&rm_state.S, rm_state.unev);
  rm_state.cont = enc_u(CONT_SEQUENCE);
  *es = EVAL_DISPATCH;
}
```

`cont_sequence` restores unev and sets up for evaluation of the next
element in the sequence. 

```
static inline void cont_sequence(eval_state *es) {
  pop_u32(&rm_state.S, &rm_state.unev);
  rm_state.unev = cdr(rm_state.unev);
  eval_sequence(es);
}
```

----
**Edited 2020-05-23 (BugFix)**

When evaluating a sequence of expressions (such as a progn construct),
the original environment also has to be stored on the heap and restored before
before evaluation of each of the expressions in the sequence.

So, the entry-point for evaluation of `progn` now pushes the
environment before going into `eval_sequence`.


```
static inline void eval_progn(eval_state *es) {
  push_u32_2(&rm_state.S, rm_state.cont, rm_state.env);
  rm_state.unev = cdr(rm_state.exp);
  eval_sequence(es);
}
```

`eval_sequence` pops the environment then takes one of two branches
depending if there is only one expression left in the sequence or
more. It is important to treat the last expression in the sequence in
a special way to allow `progn` to be used to construct a
tail-recursive function.

If there are more than one expression in the sequence, the environment
is pushed on the stack again for safekeeping. While if it is the last
expression nothing should be pushed onto the stack at all.

```
static inline void eval_sequence(eval_state *es) {

  rm_state.exp = car(rm_state.unev);
  pop_u32(&rm_state.S, &rm_state.env);
  VALUE tmp = cdr(rm_state.unev);
  if (type_of(tmp) == VAL_TYPE_SYMBOL &&
      dec_sym(tmp) == symrepr_nil()) {
    pop_u32(&rm_state.S, &rm_state.cont);
    *es = EVAL_DISPATCH;
    return;
  }
  push_u32_2(&rm_state.S, rm_state.env, rm_state.unev);
  rm_state.cont = enc_u(CONT_SEQUENCE);
  *es = EVAL_DISPATCH;
}
``` 

Storing and restoing the environment before and after each expression
in the sequence is important if the expression itself modifies the
environment. This would have an effect on the following expressions
that would no longer be evaluated in the intended environment which
may cause incorrect results or errors.


## Evaluation of conditionals

Conditionals are handled by another evaluator-continuation pair. The evaluator
sets up for evaluation of the branching condition and the continuation
then either sets up the then or the else branch for evaluation. 

```
static inline void eval_if(eval_state *es) {
  rm_state.unev = cdr(cdr(rm_state.exp));
  rm_state.exp = car(cdr(rm_state.exp));
  push_u32_2(&rm_state.S, rm_state.cont, rm_state.unev);
  rm_state.cont = enc_u(CONT_BRANCH);
  *es = EVAL_DISPATCH;
}

static inline void cont_branch(eval_state *es) {
  pop_u32_2(&rm_state.S, &rm_state.unev, &rm_state.cont);
  if (is_symbol_nil(rm_state.val)) {
    rm_state.exp = car(cdr(rm_state.unev));
  }else {
    rm_state.exp = car(rm_state.unev);
  }
  *es = EVAL_DISPATCH;
}
```

## Evaluation of let expressions

The implementation of evaluation of let expressions gets a bit
complicated as let should allow for the definition of mutually
recursive functions.  This means that a closure defined that is bound
early in the sequence of variable binding may, in its enclosed
environment, need to have a binding for a variable that is not yet set
or evaluated. To get around this a kind of ghost-bindings are created
for all of the variables to be bound in the let expression, before
evaluating any of the "right hand sides". This gets a bit further
complicated by the fact that the binding associations are created
using cons-cells and the usual checks and potential run of the garbage
collector are needed. 


```
static inline void eval_let(eval_state *es) {
  rm_state.unev = car(cdr(cdr(rm_state.exp)));
  push_u32_2(&rm_state.S, rm_state.cont, rm_state.unev);

  rm_state.unev = car(cdr(rm_state.exp));

  // Preallocate bindings
  VALUE curr = rm_state.unev;
  VALUE new_env = rm_state.env;
  while (!is_symbol_nil(curr)) {
    VALUE key = car(car(curr));
    VALUE val = enc_u(symrepr_nil());
    VALUE binding = cons(key, val);
    VALUE tmp_env = cons(binding, new_env);

    if (is_symbol_merror(binding) ||
        is_symbol_merror(new_env)) {
      gc(ec_eval_global_env, &rm_state);
      binding = cons(key, val);
      tmp_env = cons(binding, new_env);
    }
    if (is_symbol_merror(binding) ||
        is_symbol_merror(new_env)) {
      rm_state.cont = enc_u(CONT_DONE);
      *es = EVAL_CONTINUATION;
      return;
    }
    new_env = tmp_env;
    curr = cdr(curr);
  }

  rm_state.env = new_env;
  eval_let_loop(es);
}
```

If `eval_let` manages to set up the ghost-bindings it will go into a
loop similar to the case for `progn`, where a continuation and an
evaluator will take turns to run for each of the key-expression pair
to be evaluated and bound.

```
static inline void eval_let_loop(eval_state *es) {
  if (is_symbol_nil(rm_state.unev)) {
    pop_u32_2(&rm_state.S, &rm_state.exp, &rm_state.cont);
    *es = EVAL_DISPATCH;
    return;
  }
  rm_state.exp = car(cdr(car(rm_state.unev)));

  push_u32_2(&rm_state.S, rm_state.env, rm_state.unev);
  rm_state.cont = enc_u(CONT_BIND_VAR);
  *es = EVAL_DISPATCH;
}
``` 

When a "right hand side" has been evaluated it is bound to one of the already
existing ghost-bindings by a call to `env_modify_binding`.

```
static inline void cont_bind_var(eval_state *es) {
  pop_u32_2(&rm_state.S,&rm_state.unev, &rm_state.env);
  env_modify_binding(rm_state.env, car(car(rm_state.unev)), rm_state.val);
  rm_state.unev = cdr(rm_state.unev);
  eval_let_loop(es);
}
```

## Evaluation of short-circuit capable logic operators

Since we want short-circuit capable logical operators `and`, and `or`, we
cannot implement these as a regular built in fundamental
operation. The default way to apply built in fundamentals are by
evaluating all the arguments before the application and this would
make short-circuit impossible.

The `and` and `or` operators in lispBM can take an arbitrary number of
arguments.  If the number of arguments are zero, `and` should return
`t` and `or` should return `nil`.  If there are more than one
arguments, `and` should return either the last non-false value or
`nil` while `or` should return the first true value or `nil`.

Both of `and` and `or` are implemented using an evaluator and a
continuation duo. The evaluator sets up the unev register to hold the
arguments to evaluate, the exp register is set up to evaluate the
head.

```
static inline void eval_and(eval_state *es) {
  if (is_symbol_nil(cdr(rm_state.exp))) {
    rm_state.val = enc_sym(symrepr_true());
    *es = EVAL_CONTINUATION;
  }
  rm_state.unev = cdr(cdr(rm_state.exp));
  push_u32_2(&rm_state.S, rm_state.cont, rm_state.unev);
  rm_state.exp = car(cdr(rm_state.exp));
  rm_state.cont = enc_u(CONT_AND);
  *es = EVAL_DISPATCH;
}
```

Once the first argument is evaluated, the continuation will run and check
whether it is a true value or a `nil` before potentially continuing with the
next value in unev. 

``` 
static inline void cont_and(eval_state *es) {
  pop_u32(&rm_state.S, &rm_state.unev);
  if (is_symbol_nil(rm_state.val)) {
    pop_u32(&rm_state.S, &rm_state.cont);
    *es = EVAL_CONTINUATION;
    return;
  }
  if (is_symbol_nil(rm_state.unev)) {
    pop_u32(&rm_state.S, &rm_state.cont);
    *es = EVAL_CONTINUATION;
    return;
  }
  rm_state.exp = car(rm_state.unev);
  rm_state.unev = cdr(rm_state.unev);
  push_u32(&rm_state.S, rm_state.unev);
  rm_state.cont = enc_u(CONT_AND);
  *es = EVAL_DISPATCH;
}
```

`or` is similar to the `and` case, except for in how the decision to
evaluate yet another expression or not is taken. 

```
static inline void eval_or(eval_state *es) {
  if (is_symbol_nil(cdr(rm_state.exp))) {
    rm_state.val = enc_sym(symrepr_nil());
    *es = EVAL_CONTINUATION;
  }
  rm_state.unev = cdr(cdr(rm_state.exp));
  push_u32_2(&rm_state.S, rm_state.cont, rm_state.unev);
  rm_state.exp = car(cdr(rm_state.exp));
  rm_state.cont = enc_u(CONT_OR);
  *es = EVAL_DISPATCH;
}

static inline void cont_or(eval_state *es) {
  pop_u32(&rm_state.S, &rm_state.unev);
  if (!is_symbol_nil(rm_state.val)) {
    pop_u32(&rm_state.S, &rm_state.cont);
    *es = EVAL_CONTINUATION;
    return;
  }
  if (is_symbol_nil(rm_state.unev)) {
    pop_u32(&rm_state.S, &rm_state.cont);
    *es = EVAL_CONTINUATION;
    return;
  }
  rm_state.exp = car(rm_state.unev);
  rm_state.unev = cdr(rm_state.unev);
  push_u32(&rm_state.S, rm_state.unev);
  rm_state.cont = enc_u(CONT_OR);
  *es = EVAL_DISPATCH;
}
```

## The done continuation

the `done` continuation is reached whenever a "top-level" expression
has been fully evaluated. This means that either should there be more
expressions left in the program (the prg register) or computation is
done. If the computation is done, the done flag is set and we return
to the dispatcher that will not terminate the loop and finish up. 

```
static inline void cont_done(eval_state *es, bool *done) {
  if (type_of(rm_state.prg) != PTR_TYPE_CONS) {
    *done = true;
    return;
  }
  rm_state.exp = car(rm_state.prg);
  rm_state.prg = cdr(rm_state.prg);
  rm_state.cont = enc_u(CONT_DONE);
  rm_state.env = enc_sym(symrepr_nil());
  rm_state.argl = enc_sym(symrepr_nil());
  rm_state.val = enc_sym(symrepr_nil());
  rm_state.fun = enc_sym(symrepr_nil());
  stack_clear(&rm_state.S);
  *done = false;
  *es = EVAL_DISPATCH;
}
```

If there are more expressions to evaluate, the registers are set up for another round
and the cont register set to `CONT_DONE`. `EVAL_DISPATCH` then proceeds as usual. 

## Evaluation of function application

Evaluation of function applications is a bit complicated as it involves looping
over and evaluating arguments, closures, fundamental operations and extensions.

Closures are created from the user defined functions using lambda. The fundamental
operations are implemented in C and perform operations such as `+` or `=`. Extensions
are C functions added for use with a specific platform. And extension could for example
be a "print" function, that may have different implementation on different platforms
or depending on what interface one wants to print on.

Function applications in lispBM take the form `(function-expression
arg1 arg2 ... argn)`.  Some times the function-expression will be a
special symbol representing a fundamental operation (these evaluate to
them self) and other times it may be a lambda expression that has to be
evaluated to a closure. 


### Special case for zero-argument applications

In the case where there are no arguments to the function-expression the
`eval_no_args` path will be taken through the evaluator. This is to save a few
push/pops or register interactions.

`eval_no_args` sets up the exp register for evaluation of the function-expression,
pushes the cont register and sets cont to `CONT_SETUP_NO_ARG_APPLY`. This means
that after the function-expression has been evaluated to a function-"object"
the `cont_setup_no_arg_apply` function will run. 

```
static inline void eval_no_args(eval_state *es) {
  rm_state.exp = car(rm_state.exp);
  push_u32(&rm_state.S, rm_state.cont);
  rm_state.cont = enc_u(CONT_SETUP_NO_ARG_APPLY);
  *es = EVAL_DISPATCH;
}

static inline void cont_setup_no_arg_apply(eval_state *es) {
  rm_state.fun = rm_state.val;
  rm_state.argl = enc_sym(symrepr_nil());
  *es = EVAL_APPLY_DISPATCH;
}
```

`cont_setup_no_arg_apply` moves val to fun, so now fun contains the
evaluated function-object. This is what is needed in preparation for
going to dispatcher again and this time in `EVAL_APPLY_DISPATCH` mode
that depending on the contents of the fun register picks the how to
branch off next. 

### Function application the general case

In general a function application (one with arguments) will be
directed to the `eval_application` function.

```
static inline void eval_application(eval_state *es) {
  rm_state.unev = cdr(rm_state.exp);
  rm_state.exp = car(rm_state.exp);
  push_u32_3(&rm_state.S, rm_state.cont, rm_state.env, rm_state.unev);
  rm_state.cont = enc_u(CONT_EVAL_ARGS);
  *es = EVAL_DISPATCH;
}
```

`eval_application` puts the function-expression into the exp register
and the arguments into unev and sets up for running of
`CONT_EVAL_ARGS` once the function-expression has been evaluated.

When the continuation is later run, it moves val into fun and sets up
for a loop to evaluate all the arguments. argl is set to `nil`, here
all the evaluated arguments will be accumulated as the loop iterates.

```
static inline void cont_eval_args(eval_state *es) {
  pop_u32_2(&rm_state.S,&rm_state.unev, &rm_state.env);
  rm_state.fun = rm_state.val;
  push_u32(&rm_state.S,rm_state.fun);
  rm_state.argl = enc_sym(symrepr_nil());
  eval_arg_loop(es);
}
```

The `eval_arg_loop` either notices that there is only one argument
left to evaluate and it sets up for evaluation of the last
argument. Evaluation of the last argument is special because directly
after doing that the function application can be performed. So there
is a special evaluator and continuation for that case that sets up for
a run of the function application dispatcher.

```
static inline void eval_arg_loop(eval_state *es) {
  push_u32(&rm_state.S, rm_state.argl);
  rm_state.exp = car(rm_state.unev);
  if (last_operand(rm_state.unev)) {
    eval_last_arg(es);
    return;
  }
  push_u32_2(&rm_state.S, rm_state.env, rm_state.unev);
  rm_state.cont = enc_u(CONT_ACCUMULATE_ARG);
  *es = EVAL_DISPATCH;
}

static inline void eval_last_arg(eval_state *es) {
  rm_state.cont = enc_u(CONT_ACCUMULATE_LAST_ARG);
  *es = EVAL_DISPATCH;
}
```

The other option is that there are more than just one argument left and the
evaluator will set up a generic `CONT_ACCUMULATE_ARG`. 


When `cont_accumulate_arg` is launched, the value in the val register should be
consed onto the argl list. This means that it is again time for one of those checks to see if
garbage collection is needed and a possible retry.

```
static inline void cont_accumulate_arg(eval_state *es) {
  pop_u32_3(&rm_state.S, &rm_state.unev, &rm_state.env, &rm_state.argl);
  VALUE argl = cons(rm_state.val, rm_state.argl);

  if (is_symbol_merror(argl)) {
    gc(ec_eval_global_env, &rm_state);

    argl = cons(rm_state.val, rm_state.argl);
  }
  if (is_symbol_merror(argl)) {
    rm_state.cont = enc_u(CONT_DONE);
    *es = EVAL_CONTINUATION;
    return;
  }

  rm_state.argl = argl;
  rm_state.unev = cdr(rm_state.unev);
  eval_arg_loop(es);
}
```

`cont_accumulate_arg` then continues the looping behaviour.

`cont_accumulate_last_arg` is very similar, but instead of looping in the
end it sets up for the function application dispatcher to run.

Here reversal of the argument list is also performed as the arguments
have been accumulating in argl in reverse order. So, even more checks
for garbage collection in this variant of the continuation.

``` 
static inline void cont_accumulate_last_arg(eval_state *es) {
  pop_u32(&rm_state.S, &rm_state.argl);

  VALUE argl =  cons(rm_state.val, rm_state.argl);

  if (is_symbol_merror(argl)) {
    gc(ec_eval_global_env, &rm_state);
    argl = cons(rm_state.val, rm_state.argl);
  }
  if (is_symbol_merror(argl)) {
    rm_state.cont = enc_u(CONT_DONE);
    *es = EVAL_CONTINUATION;
    return;
  }
  rm_state.argl = argl;

  VALUE rev_args = reverse(rm_state.argl);

  if (is_symbol_merror(rev_args)) {
    gc(ec_eval_global_env, &rm_state);
    rev_args = reverse(rm_state.argl);
  }
  if (is_symbol_merror(rev_args)) {
    rm_state.cont = CONT_DONE;
    *es = EVAL_CONTINUATION;
  }

  rm_state.argl = rev_args;
  pop_u32(&rm_state.S, &rm_state.fun);
  *es = EVAL_APPLY_DISPATCH;
}
```

### The function application dispatcher

Below is the implementation of the application dispatcher.  It
recognises four different kinds of application, where one is a bit of
a special case.  This is the case for the eval function. This
`eval_eval` case implements the eval function that can be called to
evaluate for example quoted expressions. 

Anyway, this is just a dispatcher, it checks what kind of function is in the
fun register and branches depending. 

```
static inline void eval_apply_dispatch(eval_state *es) {
  if (is_symbol_eval(rm_state.fun)) eval_eval(es);
  else if (is_fundamental(rm_state.fun)) eval_apply_fundamental(es);
  else if (is_closure(rm_state.fun)) eval_apply_closure(es);
  else if (is_extension(rm_state.fun)) eval_apply_extension(es);
  else {
    rm_state.cont = enc_u(CONT_DONE);
    *es = EVAL_CONTINUATION;
  }
}
```

### Application of a closure

Puh! Application of functions is quite brushy but now we should have everything
set up in the fun and argl registers to apply the function.

If the function application dispatcher notices that the function in
the fun register is a closure, we will end up in the
`eval_apply_closure` function.

This case is really quite simple, but again a little bit obscured by
the checks if garbage collection is necessary. What happens is that
a local environment is set up from the environment held in closure
together with the parameter list and the contents of argl. This environment building
uses cons... and thus the garbage collection checks. 

```
static inline void eval_apply_closure(eval_state *es) {
  VALUE local_env = env_build_params_args(car(cdr(rm_state.fun)),
                                          rm_state.argl,
                                          car(cdr(cdr(cdr(rm_state.fun)))));
  if (is_symbol_merror(local_env)) {
    gc(ec_eval_global_env, &rm_state);
    local_env = env_build_params_args(car(cdr(rm_state.fun)),
                                          rm_state.argl,
                                          car(cdr(cdr(cdr(rm_state.fun)))));
  }
  if (is_symbol_merror(local_env)) {
    rm_state.cont = enc_u(CONT_DONE);
    *es = EVAL_CONTINUATION;
    return;
  }

  rm_state.env = local_env;
  rm_state.exp = car(cdr(cdr(rm_state.fun)));
  pop_u32(&rm_state.S, &rm_state.cont);
  *es = EVAL_DISPATCH;
}
```

If everything goes well with construction of the local environment,
the exp register is set to the body from the closure and then we loop
back to `EVAL_DISPATCH` that will evaluate the body in the local
environment.


### Application of a fundamental operation

If the function application dispatcher finds out that the fun register
is holding a fundamental operation we end up in
`eval_apply_fundamental`. These fundamental operations are implemented
in C and expect an array of arguments and an argument count. The array of arguments
is obtained by pushing the contents of argl onto the stack. A pointer to the first
argument on the stack is then passed to the fundamental operation. 

Now, there is a check to see if garbage collection is needed here as well. This is
because on of the fundamental operations is actually `cons`! 

```
static inline void eval_apply_fundamental(eval_state *es) {
  UINT count = 0;
  VALUE args = rm_state.argl;
  while (type_of(args) == PTR_TYPE_CONS) {
    push_u32(&rm_state.S, car(args));
    count ++;
    args = cdr(args);
  }
  UINT *fun_args = stack_ptr(&rm_state.S, count);
  VALUE val = fundamental_exec(fun_args, count, rm_state.fun);
  if (is_symbol_merror(val)) {
    gc(ec_eval_global_env, &rm_state);
    val = fundamental_exec(fun_args, count, rm_state.fun);
  }
  if (is_symbol_merror(val)) {
    rm_state.cont = enc_u(CONT_DONE);
    *es = EVAL_CONTINUATION;
    return;
  }

  rm_state.val = val;

  stack_drop(&rm_state.S, count);
  pop_u32(&rm_state.S, &rm_state.cont);
  *es = EVAL_CONTINUATION;
}

If successful, the result of the fundamental operation will be in the val register
and the state is set up for evaluation of a continuation. 

```

### Application of an extension

Extensions are very similar to fundamentals. They both expect arguments in
array form and are called via a C function pointer.  

```
static inline void eval_apply_extension(eval_state *es) {
  extension_fptr f = extensions_lookup(dec_sym(rm_state.fun));
  if (!f) {
    rm_state.cont = enc_u(CONT_DONE);
    *es = EVAL_CONTINUATION;
    return;
  }
  UINT count = 0;
  VALUE args = rm_state.argl;
  UINT *fun_args = stack_ptr(&rm_state.S, count);
  while (type_of(args) == PTR_TYPE_CONS) {
    push_u32(&rm_state.S, car(args));
    count ++;
    args = cdr(args);
  }
  rm_state.val = f(fun_args, count);
  stack_drop(&rm_state.S, count);
  pop_u32(&rm_state.S, &rm_state.cont);
  *es = EVAL_CONTINUATION;
}
``` 


### Application of the eval function

In the case of eval, argl will contain a single expression that
should be evaluated. So this case just sets up for another round of
evaluation, now with the head of argl moved into the exp register. 

``` 
static inline void eval_eval(eval_state *es) {
  rm_state.exp = car(rm_state.argl);
  pop_u32(&rm_state.S, &rm_state.cont);
  *es = EVAL_DISPATCH;
}
``` 


## Some thoughts

Now after reading about the explicit control evaluator in SICP and
having tried to implement it, I think I start to see that the older
implementation of evaluation in lispBM, called
[`eval_cps`](../lispbm_evaluation_function/index.html) is actually
quite similar. It seems to me that `eval_cps` is a much less
principled (more MacGyvered) implementation of the same ideas. There
are not as many registers in `eval_cps`, and as a result the stack is
used much more. Either way, this is a much needed cleanup of the code
that performs evaluation.

In a sense, `eval_cps` implements a four registers machine. There is a
program-, current exp-, current env- and r-register with very similar
functionality to the exp-, env-, prg-, and val-register described
here. Lists of arguments and such are allocated on the stack rather
than an argl register. 

The `eval_cps` version of the evaluator can be time-shared between
many programs. There should be no problem with doing the same with this
`ec_eval`. This is left as future work though.

Thanks for reading and, as usual, all constructive feedback is much
appreciated.

___

[HOME](https://svenssonjoel.github.io)
