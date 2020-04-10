

# Concurrent Tasks - Part 2

In a previous [post](../lispbm_towards_concurrency/index.html) I
started to experiment with time-sharing of the lispBM evaluator
between multiple tasks. Since then a few bugs have been found (there
are probably more to fix as time goes by). The bugs I find, and their
attempts at a fix, are added into the text alongside the original buggy
code. 

Since last time, a `spawn` and a `wait` operation has been added. The
`spawn` operation allows a program to start up sub-tasks that will run
concurrently to each other and the "spawner". `spawn` takes an
arbitrary number of arguments and each argument should be a lispBM
program (which is a list of expressions). Each of the arguments to
`spawn` is started in a fresh context. `spawn` itself, returns a list
of context IDs and does not wait for the sub-tasks to finish. If a task
should wait for a sub-task to finish it should use the operation `wait`
which takes a single context ID as argument. `wait` makes the task
sleep until the sub-task with the ID given as argument finishes
execution, `wait` then returns the result of that given sub-task.

## Small Example Programs

The first example program spawns 3 new tasks that each will run a
program that performs a little arithmetic.

``` 
(spawn ((+ 1 2 3))
       ((+ 4 5 6))
       ((+ 7 8 9)))
```

When entered into the REPL it produces the following result. 

``` 
# (spawn ((+ 1 2 3)) ((+ 4 5 6)) ((+ 7 8 9)))
started ctx: 2
# << Context 2 finished with value (5 4 3) >>
# << Context 3 finished with value 6 >>
# << Context 4 finished with value 15 >>
# << Context 5 finished with value 24 >>
# 
```

The task that performs the spawning gets context id 2 and we can see
that this task finishes with the result `(5 4 3)`, these are the context ids for the 
three sub-tasks. Then the sub-tasks finish with results `6`, `15` and `24`. 

Another fun thing to try, but hard to show that it works in text, is
to spawn tasks that go to sleep for a while.

``` 
(spawn ((yield 500000))
       ((yield 1000000))
       ((yield 2000000)))
```

This programs starts one task that sleeps for half a second, another
that sleeps for one second and a third that sleeps for 2 seconds. The result of 
running this in the REPL is as follows. 

```
# (spawn ((yield 500000)) ((yield 1000000)) ((yield 2000000)))
started ctx: 6
# << Context 6 finished with value (9 8 7) >>
# << Context 7 finished with value t >>
# << Context 8 finished with value t >>
# << Context 9 finished with value t >>
# 
``` 

Of course, the interesting aspect of this program is the timing. The
sub-tasks should finish half a second, one second and two seconds
after being started. [This video](https://youtu.be/-hYnFBsvCS0) shows a what 
running this program looks like. 


The last example then, here the parent task spawns a sub-task that
performs a bit of arithmetic. The parent waits for the sub-task to
finish.

```
(let ((cids (spawn ((+ 1 2 3)))))
  (wait (car cids)))
```

Typed into the REPL, results in this: 
``` 
# (let ((cids (spawn ((+ 1 2 3))))) (wait (car cids)))
started ctx: 6
# << Context 7 finished with value 6 >>
# << Context 6 finished with value 6 >>
# 
``` 

Contexts 6 and 7 both finish with value 6. The `wait` operation results in the 
value computed by the task with the context id that it is waiting for. 


## Implementation 

The `spawn` operation is added as a special form while the `wait`
operation is treated similarly to a fundamental operation.  Special
forms and fundamental operations are both a kind of built-in function
in lispBM but they differ a bit. A fundamental operation is treated as
a regular function application and thus its arguments will be
evaluated before the application. When adding a special form I am much
more free in how to deal with its evaluation and treatment of
arguments. So, both are "built-in" only difference is how deeply so ;)

### Spawn

The `spawn` special form is evaluated in `eval_cps_run_eval` by
creating a continuation called `SPAWN_ALL`. The `SPAWN_ALL`
continuation expect a pointer to a list of programs to spawn on the
continuation stack as well as the environment in which to launch these
tasks. 

The current context intermediate result is set to an empty list that
is then used to accumulate the context ids as the `SPAWN_ALL`
continuation is spawning tasks.


```
// Special form: SPAWN
    if (sym_id == symrepr_spawn()) {
      VALUE prgs = cdr(ctx->curr_exp);
      VALUE env = ctx->curr_env;

      if (type_of(prgs) == VAL_TYPE_SYMBOL && prgs == NIL) {
        ctx->r = NIL;
        ctx->app_cont = true;
        continue;
      }
      
      VALUE cid_list = NIL;
      FOF(push_u32_3(&ctx->K, env, prgs, enc_u(SPAWN_ALL)));
      ctx->r = cid_list; 
      ctx->app_cont = true;
      continue;
    }
``` 

The `SPAWN` and `SPAWN_ALL` setup are very similar to how `PROGN` and
`PROGN_REST` are implemented, but an important difference here is that
in `PROGN` only the result of the last expression to evaluate is kept. 

In `apply_continuation` a case is added for `SPAWN_ALL`. 

```
  case SPAWN_ALL: {
    VALUE rest;
    VALUE env;
    pop_u32_2(&ctx->K, &rest, &env);
    if (type_of(rest) == VAL_TYPE_SYMBOL && rest == NIL) {
      ctx->app_cont = true;
      return;
    }

    VALUE cid_val = enc_i(next_ctx_id);
    VALUE cid_list = cons(cid_val, ctx->r);
    if (type_of(cid_list) == VAL_TYPE_SYMBOL) {
      FATAL_ON_FAIL(ctx->done, push_u32_3(&ctx->K, env, rest, enc_u(SPAWN_ALL)));
      *perform_gc = true;
      ctx->app_cont = true;
      return;
    }
    // TODO: error checking
    CID cid = create_ctx(car(rest),
             env,
             EVAL_CPS_DEFAULT_STACK_SIZE,
             EVAL_CPS_DEFAULT_STACK_GROW_POLICY);
    (void) cid;
    FATAL_ON_FAIL(ctx->done, push_u32_3(&ctx->K, env, cdr(rest), enc_u(SPAWN_ALL)));
    ctx->r = cid_list;
    ctx->app_cont = true;
    return;
  }
```

`SPAWN_ALL`, simply put, takes the head from the list of programs and
calls `create_ctx` for that program. It then pushes a new instance of
the `SPAWN_ALL` continuation onto the continuation stack. While it is
doing this it is accumulating generated context ids

Now, implementing `SPAWN_ALL` was actually quite tricky. There is a
subtle detail in the interplay of evaluation and garbage collection
that causes a bit of extra work here. And it comes from the
construction of the list of context ids. 

So the reason for appending a new context id onto the list **before**
actually creating the context is that, if you swapped the order, it could happen that: 

1. Creation of context was successful and returned cid N 
2. appending N to the result list fails because heap is full. 

In that scenario a context will have been started but there is no way
to retrieve its context id to add to the result list. So structuring
the code in this way will allow the evaluator to take a break in the
middle of `SPAWN_ALL` and run the GC, then to resume where it left of. 

There are still some corner cases to work out here. I will write about
those as they appear. 

### Wait

The `wait` operation is not a special form, so it has no case of its
own in `eval_cps_run_eval`, it is instead handled, similarly to
`yield`, within the `APPLICATION` continuation case in
`apply_continuation`.

``` 
  if (dec_sym(fun) == symrepr_wait()) {
    if (type_of(fun_args[1]) == VAL_TYPE_I) {
      CID cid = dec_i(fun_args[1]); 
      stack_drop(&ctx->K, dec_u(count)+1);
      FOF(push_u32_2(&ctx->K, enc_u(cid), enc_u(WAIT)));
      ctx->r = enc_sym(symrepr_true());
      ctx->app_cont = true;
      yield_ctx(50000);
    } else {
      ERROR
      error_ctx(enc_sym(symrepr_eerror()));
    }
    return;
  }
```

`wait` expects a single integer argument that holds a context id. It
then creates a continuation called `WAIT` that expects the context id
to wait for on the continuation stack. The result of the current
context is set to `true` and then the current context yields for 0.5
seconds (hard coded so far).  When this context eventually wakes up
again, it will be set up to apply its continuation (since `app_cont`
was set to `true`), and control will jump to the `apply_continuation`
function and case `WAIT`.


```
 case WAIT: {

    VALUE cid_val; 
    pop_u32(&ctx->K, &cid_val);
    CID cid = dec_u(cid_val);

    VALUE r;

    if (eval_cps_remove_done_ctx(cid, &r)) {
      ctx->r = r;
      ctx->app_cont = true;
    } else {
      FOF(push_u32_2(&ctx->K, enc_u(cid), enc_u(WAIT)));
      ctx->r = enc_sym(symrepr_true());
      ctx->app_cont = true;
      yield_ctx(50000);
    }
    return;
  }
```

The `WAIT` continuation pops the context id to wait for from the
stack, then calls `eval_cps_remove_done_ctx` to scan through the list
of finished contexts. If this is successful the result stored in that
waiting context is returned. Otherwise a second sound of `WAIT`
continuation is set up. 

## Some Thoughts on the Current Implementation

All of this is very experimental and hacky. If it turns out to be a fun approach in the end, this code will need 
a big cleanup. 

One aspect that I am currently unsure of is if `spawn` should instead
require the arguments to be quoted? If we chose to change it so that
`spawn` expects quoted programs, then `spawn` does not need to be a
special form anymore and could be treated in a similar way to
`wait`. I think it may feel more lispy that the arguments to `spawn`
are quoted. We'll see how it ends up! 

## Future Work

in [the earlier text](../lispbm_towards_concurrency/index.html) I
outlined some future work. The current text addresses points 2 and 3
from this list, so I mark those as **(DONE)**. Now, I am quite sure
problems with the current implementation will pop up and tweaks will
be needed, so maybe it is more like "done" than **(DONE)**. 


1. More testing! The heaviest load I've started is 8 instances of the
example program that sleeps and prints hello. More and more diverse
test programs are needed. 

2. **(DONE)** Add `wait` functionality. Calling `(wait cid)` should cause the
task that called `wait` to stop until the task with context-id `cid`
finishes. The result of `(wait cid)` should be the result computed by
the task with context-id `cid`. Finished contexts are stored on a list
called `ctx_done`, so one way to implement wait is wake up a waiting
task periodically and perform a search over the `ctx_done` list. If
the `cid` is not present in the list then the waiting task can go back
to sleep for a while.

3. **(DONE)** Some way to create tasks "programmatically". Currently a new task
is created for each expression entered using the repl. I don't have
any idea of how the details of this will work out, but imagine that
there should be some kind of `spawn` or `fork` function. 

4. Some way to communicate values between running tasks would be
nice. Maybe a queue structure that those tasks who want to communicate
both know about. All tasks run in the same heap, so it is just a
matter of making both tasks aware of the existence of the queue.

5. Testing on top of  ChibiOS, ZephyrOS and FreeRTOS. 

6. There are no priorities, preemptions or periodic task switches. So
a task that is running forever and does not `yield` can starve
everything else out. This is definitely not suitable for anything
timing critical.

7. Be opportunistic and run the GC in case there is currently no
runnable context.

8. The `gc` function should perform error checking! Each call to
`gc_mark_phase` can fail and if one does, there RTS should be
reset. There is no need to try to continue executing after such a
failure as the heap will most likely be corrupt.

9. What about events from the outside, such as a button press or data
arriving over some communication channel? These things are sometimes
handled using an interrupt. Some way of dealing with these "external"
events are needed. There may be more suitable abstractions for these
kinds of interruptions to apply to this system. Streams/queues of
events? where the underlying enqueueing of events is performed by the
RTS.

Point 9 on this list has become a little bit clearer in my mind. I do
not want interrupts! Instead I will let the some underlying "driver"
code take care of such things. Such a "driver" for a button for
example would just append button-events onto an event queue. The
lispBM program would then need to periodically check this queue for
events and act upon them. 

I am also not sure about preemption and priorities. We'll see about
those. But I do think there should be periodic context switches and
some time quanta assigned to each task when it gets moved to the
running state. 

## In Closing

Thanks for reading! As usual all helpful insights are much appreciated. 



___

[HOME](https://svenssonjoel.github.io)
