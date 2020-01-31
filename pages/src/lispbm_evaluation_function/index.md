

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
pause the computation, perform gc, and then pick up execution again
where the computation was paused.

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
simplifications was spotted. Those simplifications (and improvements)
have been applied here. While applying those simplification I fell
into the trap of not honoring property 1. But the problem was possible
to fix. I will point out this problematic example later in this text.

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
lispBM](../lispbm_current_status/index.html)). There are now only 8
continuation functions instead of 9. The difference is that before
there was the three continuations `FUNCTION`, `FUNCTION_APP` and
`ARG_LIST` that are now replaced by just `APPLICATION` and
`APPLICATION_ARGS`. 

``` 
#define DONE              1
#define SET_GLOBAL_ENV    2
#define BIND_TO_KEY_REST  3
#define IF                4
#define EVAL              5
#define PROGN_REST        6
#define APPLICATION       7
#define APPLICATION_ARGS  8
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

## Entrypoint Function

The function that is called from the REPL is called `eval_cps_program`
and takes a list of expressions as argument. This function currently
modifies the global evaluation context and then loops over the list of
expressions and evaluates them from first to last. the result of the
last expression is returned to the caller. 

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

```
VALUE run_eval(eval_context_t *ctx){

  push_u32(ctx->K, enc_u(DONE));

  VALUE r = NIL;
  bool done = false;
  bool perform_gc = false;
  bool app_cont = false;

  uint32_t non_gc = 0;

  while (!done) {
    
#ifdef VISUALIZE_HEAP
    heap_vis_gen_image();
#endif

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

    if (app_cont) {
      r = apply_continuation(ctx, r, &done, &perform_gc, &app_cont);
      continue;
    }

    VALUE head;
    VALUE value = enc_sym(symrepr_eerror());

    switch (type_of(ctx->curr_exp)) {

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
    case PTR_TYPE_REF:
    case PTR_TYPE_STREAM:
      r = enc_sym(symrepr_eerror());
      done = true;
      break;
    case PTR_TYPE_CONS:
      head = car(ctx->curr_exp);

      if (type_of(head) == VAL_TYPE_SYMBOL) {

	// Special form: QUOTE
	if (dec_sym(head) == symrepr_quote()) {
	  r = car(cdr(ctx->curr_exp));
	  app_cont = true;
	  continue;
	}

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

	// Special form: IF
	if (dec_sym(head) == symrepr_if()) {

	  push_u32_3(ctx->K,
		     car(cdr(cdr(cdr(ctx->curr_exp)))), // Else branch
		     car(cdr(cdr(ctx->curr_exp))),      // Then branch
		     enc_u(IF));
	  ctx->curr_exp = car(cdr(ctx->curr_exp));
	  continue;
	}
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
      } // If head is symbol
      push_u32_4(ctx->K,
		 ctx->curr_env,
		 enc_u(0),
		 cdr(ctx->curr_exp),
		 enc_u(APPLICATION_ARGS));

      ctx->curr_exp = head; // evaluate the function
      continue;
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

``` 
VALUE apply_continuation(eval_context_t *ctx, VALUE arg, bool *done, bool *perform_gc, bool *app_cont){

  VALUE k;
  pop_u32(ctx->K, &k);

  VALUE res;

  *app_cont = false;

  switch(dec_u(k)) {
  case DONE:
    *done = true;
    return arg;
  case EVAL:
    ctx->curr_exp = arg;
    return NONSENSE;
  case SET_GLOBAL_ENV:
    res = cont_set_global_env(ctx, arg, done, perform_gc);
    if (!(*done)) 
      *app_cont = true;
    return res;
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
  } // end switch
  *done = true;
  return enc_sym(symrepr_eerror());
}

```

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

___

[HOME](https://svenssonjoel.github.io)
