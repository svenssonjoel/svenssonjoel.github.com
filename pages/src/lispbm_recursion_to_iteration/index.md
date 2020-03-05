

# Turn Recursion into Iteration using an Explicit Stack

The STM32f4 MCUs I am using have 192KB of ram, which is probably quite
a lot by MCU standards, and the ESP32-WROOM-32 has 520KB (wow). While
this may seem like a lot of memory when it comes to MCUs, it still should
not be wasted.

One goal with lispBM is that it should run on a wide range of 32bit
MCUs and coexist with other firmware executing on that platform. This
means it is a bit important to keep memory usage of the lispBM runtime
system down (it is quite hungry!). It is also important to know that
the runtime system can run within an allotted amount of memory and not
over time, or periodically, use up all free memory. Currently this is
not the case, I have no idea exactly how much memory lispBM uses over
the different phases of execution.

This text is taking one step in the direction of getting more of a
grip on lispBM memory usage. There are a number of functions that run
recursively over lisp-heap data structures. Such recursion can be very
space inefficient! For example if recursively traversing a heap
structure that makes up a linked list, call stack proportional to the
length of that list would be needed. For long lists and small MCUs
this can quickly become a problem.

Part of the problem here is that some functions need to traverse
arbitrary heap structures. One example is `gc_mark_phase` which is
part of the [garbage
collector](../lispbm_garbage_collector/index.html). For each cons-cell
that mark encounters, it has to process both the `car` and the `cdr`
subtrees. Up until now, `gc_mark_phase` did this by recursing on the
`car` and the `cdr` which is ok for small heap structures but really
wasteful when stumbling into a long linked-list like structure.

It is quite easy to create long linked-list structures on the
heap. For example `(iota 1000)` generates the list containing 0 to
1000. When the marking routine encounters such a list it will nest
1000 recursive calls. If the mark function uses, say, 8 bytes of local
variables and we ignore any other data included in a stack frame the
recursion over the list would use up just about 8KB of memory. Surely
there are more things than just the local variables that go into the
stack frame so 8KB is probably a very low estimate but this depends on
call convention. 

I think that it will be hard for the compiler to optimize the kind of
recursion over an arbitrary heap structure such as `gc_mark_list`. It
may be the case that if a function only recurses over the `cdr` (or
only the `car`) and does this in a tail-call kind of way, that GCC
would apply tail-call optimizations. 

It seems to me that it should be possible to iterate over
sub-trees that are linked-lists with the links in the `cdr` position
in constant space by making the stack explicit. In the general case
the stack usage will still be proportional to the depth of the tree
that is traversed but now with the benefit that we are more in control
of (and know) the size of the stack. I also imagine that linked-lists
with the next pointer in the `cdr` are a very commonly occurring thing
in lisp-like programming languages.


## Iterative Marking of Heap Structures

Rewriting `gc_mark_phase` to use an explicit stack was not that hard
(if I got it right). So, will use that function as the introductory example. 

Below is the same recursive `gc_mark_phase` function that was also
shown in the text about the [garbage
collector](../lispbm_garbage_collector/index.html). It is repeated
here to act as a starting point. I also noticed that at this point in
time the function returns an `int` that will always be `1`. I Changed
it's type `void`.

The `gc_mark_phase` function takes a `VALUE` as argument. A `VALUE`
can be either a pointer to a cons-cell or some more basic value such a
number.  If the `VALUE` passed to `gc_mark_phase` is not a pointer,
then the function immediately returns. If that call `gc_mark_phase` was
part of a recursion over a heap-structure, that is a recursion
terminating case.

If the `VALUE` is a pointer, what it points to could either be already
marked or need marking. If it is already marked that is another
recursion terminating case.

But, if we still haven't returned it is time to mark and recurse!
`gc_mark_phase` is called on both the `car` and the `cdr` field of the
cons-cell pointed at.

```
void gc_mark_phase(VALUE env) {

  if (!is_ptr(env)) {
      return; // Nothing to mark here
  }

  if (get_gc_mark(ref_cell(env))) {
    return; // Circular object on heap, or visited..
  }

  // There is at least a pointer to one cell here. Mark it and recurse over  car and cdr 
  heap_state.gc_marked ++;

  set_gc_mark(ref_cell(env));

  VALUE t_ptr = type_of(env);

  if (t_ptr == PTR_TYPE_BOXED_I ||
      t_ptr == PTR_TYPE_BOXED_U ||
      t_ptr == PTR_TYPE_BOXED_F ||
      t_ptr == PTR_TYPE_ARRAY) {
    return;
  } 

  gc_mark_phase(car(env));
  gc_mark_phase(cdr(env));
}
```

So, if for example the argument to `gc_mark_phase` happens to be a
very long linked-list, deeply nested recursion is the outcome. It may
be possible that the last call to `gc_mark_phase` will have tail-call
optimization applied to it, which should mean that in the case of a
linked list, this function also operates in constant space. But with
this code that is up to the compiler and optimization settings and it
still does not make us able to control the maximum amount of memory
usage ourselves.


Below is an implementation of `gc_mark_phase` that uses an explicit
stack.  When introducing a fixed maximum stack size the function can
now fail and this is, correctly this time, reflected in its type (`int`).
This is of course another benefit over the previous implementation
that either would succeed, or crash in some spectacular way when the
call-stack starts overwriting other data and code. 


This new implementation of `gc_mark_phase` starts out by creating the
explicit stack. Currently this explicit stack is a block of VALUES on the
call stack, but an important difference here is that this `gc_mark_phase`
won't be recursive and those 1024 VALUES is what we have to work within.

The argument to `gc_mark_phase` is pushed onto the stack before
entering a loop that will run as long as there are `VALUE`s on the
stack. Inside of the loop a `VALUE` is popped from the stack and is
processed in a similar way to earlier (in the recursive
`gc_mark_phase`). When reaching the point where earlier we recursed
over `car` and `cdr`, these two values are instead pushed onto the
stack. Pushing to the stack can fail and in that case an error is
returned.


```
int gc_mark_phase(VALUE env) {

  VALUE stack_storage[1024];
  stack s;
  stack_create(&s, stack_storage, 1024);
  
  push_u32(&s, env);

  while (!stack_is_empty(&s)) {
    VALUE curr;
    int res = 1;
    pop_u32(&s, &curr);

    if (!is_ptr(curr)) {
      continue;
    }
    
    // Circular object on heap, or visited..
    if (get_gc_mark(ref_cell(curr))) {
      continue;
    }

    // There is at least a pointer to one cell here. Mark it and add children to stack
    heap_state.gc_marked ++;
    
    set_gc_mark(ref_cell(curr));
    
    VALUE t_ptr = type_of(curr);
    
    if (t_ptr == PTR_TYPE_BOXED_I ||
        t_ptr == PTR_TYPE_BOXED_U ||
        t_ptr == PTR_TYPE_BOXED_F ||
        t_ptr == PTR_TYPE_ARRAY) {
      continue;
    }  
    res &= push_u32(&s, cdr(curr));
    res &= push_u32(&s, car(curr));

    if (!res) return 0;
  }

  return 1;
}
``` 

After pushing `car` and `cdr` the loop body executes again and
processes the new top of the stack. Notice that here it makes a
difference in what order the `car` and `cdr` are pushed onto the
stack.  Pushing `cdr` before `car` gives the constant memory use for
linked-lists with the pointer to the next cell in the cdr (which I
think will be a common case).  If instead `car` was pushed before
`cdr`, linked-lists with the pointer in the `car` field would be
favored but these don't sound so common to me.  Also, as pointed out
earlier, in the general case the stack will grow proportionally to the
depth of the tree that is traversed.


## Iterative Printing of Heap Structures

Printing of `VALUE`s is another function that was implemented using
recursion before. Doing some changes to the functions that print
lispBM heap-structures (`VALUE`s) have been on the todo-list for a
long time, so while doing this transformation some other improvements
were applied as well.

For example, `(iota 3)` now prints like this: 
```
# (iota 3)
(0 1 2 3)
```
earlier it would output
```
# (iota 3)
(0 (1 (2 (3 nil))))
```
which is also a hint at how that structure was printed recursively.


The new `print_value` function that replaces `simple_print` turned out
almost being a small interpreter for a tiny language for printing
heap-structures. The explicit stack used does not just keep track of
what is left to print but it also holds instructions that guide the
way they are printed. The following set of definitions list the different
instructions that can occur on the stack. 
 

```
#define PRINT_STACK_SIZE 256 /* 1 KB */

#define PRINT          1
#define PRINT_SPACE    2
#define START_LIST     3
#define CONTINUE_LIST  4
#define END_LIST       5
```
What these instructions do will be shown in the code walk-through below. 

if zooming out enough and squinting appropriately the print function
works the same way as the `gc_mark_phase`. It starts out by pushing
the `VALUE` argument to the stack, then depending on the form of this
value the parts to print next are pushed.  New is that, instead of
marking cons-cells, the values held in the structure are of more
interest as they are what is supposed to be printed. The structure of
cons-cells also hold information guiding the printing but that
information is more about where spaces and parentheses should go.

The `print_value` is a bit long and mainly consists of a lot of just
slightly different cases. Instead of showing the entire function I will
just list some representative cases of each kind. 

``` 
int print_value(char *buf,int len, char *error, int len_error, VALUE t) {

  VALUE stack_storage[PRINT_STACK_SIZE];
  
  stack s;
  stack_create(&s, stack_storage, PRINT_STACK_SIZE);

  int n = 0;
  int offset = 0;
  char *str_ptr;
  int res;
  
  push_u32_2(&s, t, PRINT);
```

The beginning of the function sets up the stack and initializes some
state used throughout. The function operates a little bit like
`snprintf` in the sense that it prints at most `len` characters into
the output `buf`. It also has an error string argument that is filled
in with some text in case it is not possible to print the
heap-structure given. 

Before entering into a loop that runs while there are things on the
stack, two things are pushed onto the stack. First the value to print
is pushed and then above it an "Opcode" called `PRINT` is pushed. When
later popping `PRINT` we know it has an additional argument on the stack,
which represents what to print. 


``` 
  while (!stack_is_empty(&s) && offset <= len - 5) {
    
    VALUE curr;
    UINT  instr;
    pop_u32(&s, &instr);

    switch(instr) {
```

The loop starts out by popping the top of the stack, which gives an
instruction. Depending on what that instruction is a case is selected. Let's
look at the `PRINT` case first.


When entering the `PRINT` case, its argument is popped and depending
on the type of that argument we proceed. If the argument is a symbol or number
it is directly printed into the `buf`. If, however, the argument is
a pointer to a cons-cell it signals the start of a list that should be wrapped
in parentheses. This is dealt with by pushing the value back onto the stack and
then pushing the instruction `START_LIST`.

```
    case PRINT:

      pop_u32(&s, &curr);
      
      switch(type_of(curr)) {

      case PTR_TYPE_CONS:{
        res = 1;
        res &= push_u32(&s, curr);
        res &= push_u32(&s, START_LIST);
        if (!res) {
          n = snprintf(error, len_error, "Error: Out of print stack\n");
          return -1;
        }
        break;
      }

      /* ... */ 
      
      case VAL_TYPE_SYMBOL:
        str_ptr = symrepr_lookup_name(dec_sym(curr));
        if (str_ptr == NULL) {
          
          snprintf(error, len_error, "Error: Symbol not in table %"PRI_UINT"", dec_sym(t));
          return -1;
        } 
        n = snprintf(buf + offset, len - offset, "%s", str_ptr);
        offset += n;
        break; //Break VAL_TYPE_SYMBOL
        
      case VAL_TYPE_I:
        n = snprintf(buf + offset, len - offset, "%"PRI_INT"", dec_i(curr));
        offset += n;
        break;
        
      case VAL_TYPE_U:
        n = snprintf(buf + offset, len - offset, "%"PRI_UINT"", dec_u(curr));
        offset += n;
        break;
        
      case VAL_TYPE_CHAR:
        n = snprintf(buf + offset, len - offset, "\\#%c", dec_char(curr));
        offset += n;
        break;
        
      default:
        snprintf(error, len_error, "Error: print does not recognize type of value: %"PRIx32"", curr);
        return -1;
        break;
      } // Switch type of curr
      break; // case PRINT
```


When interpreting the `START_LIST` instruction, an argument is popped
from the stack and a start parenthesis is printed to the buffer.  So
when printing a list like this, the `car` value should be printed
first and thus has to be pushed last. So before pushing the
instruction to `PRINT` the car we have to deal with the `cdr`.
depending on the type of `cdr` there are some options, if it is a
pointer we should `CONTINUE_LIST`, if is the `nil` symbol we should
`END_LIST` otherwise it is something that should be printed (Actually
I think that this case is what should be the dotted pair case, if
lispBM had them). 

*EDIT 2020-03-05* bug-fix. Added `res &= push_u32(&s, END_LIST);`
to the cases in `START_LIST` and `CONTINUE_LIST` where the value in the
`cdr` position is neither a pointer to a cons-cell or a `nil`. This fix
adds a closing parentheses to things like `(cons 1 2)`. 


``` 
    case START_LIST: {
      res = 1;
      pop_u32(&s, &curr);
      
      n = snprintf(buf + offset, len - offset, "(");
      offset += n;
      VALUE car_val = car(curr);
      VALUE cdr_val = cdr(curr);

      if (type_of(cdr_val) == PTR_TYPE_CONS) {
        res &= push_u32(&s, cdr_val);
        res &= push_u32(&s, CONTINUE_LIST);
      } else if (type_of(cdr_val) == VAL_TYPE_SYMBOL &&
                 dec_sym(cdr_val) == symrepr_nil()) {
        res &= push_u32(&s, END_LIST);
      } else {
      	res &= push_u32(&s, END_LIST);
        res &= push_u32(&s, cdr_val);
        res &= push_u32(&s, PRINT);
        res &= push_u32(&s, PRINT_SPACE);
      }
      res &= push_u32(&s, car_val);
      res &= push_u32(&s, PRINT);

      if (!res) {
        n = snprintf(error, len_error, "Error: Out of print stack\n");
        return -1;
      }
      
      break;
    }
``` 

The `CONTINUE_LIST` instruction is close to identical to `START_LIST` except that
it does not print an opening parenthesis, instead it prints a space to separate the elements of
the list.

``` 
    case CONTINUE_LIST: {

      res = 1;
      pop_u32(&s, &curr);

      if (type_of(curr) == VAL_TYPE_SYMBOL &&
          dec_sym(curr) == symrepr_nil()) {
        break;
      }
           
      VALUE car_val = car(curr);
      VALUE cdr_val = cdr(curr);

      n = snprintf(buf + offset, len - offset, " ");
      offset += n;

      if (type_of(cdr_val) == PTR_TYPE_CONS) {
        res &= push_u32(&s, cdr_val);
        res &= push_u32(&s, CONTINUE_LIST);
      } else if (type_of(cdr_val) == VAL_TYPE_SYMBOL &&
                  dec_sym(cdr_val) == symrepr_nil()) {
        res &= push_u32(&s, END_LIST);
      } else {
      	res &= push_u32(&s, END_LIST);
        res &= push_u32(&s, cdr_val);
        res &= push_u32(&s, PRINT);
        res &= push_u32(&s, PRINT_SPACE);
      }
      res &= push_u32(&s, car_val);
      res &= push_u32(&s, PRINT);
      if (!res) {
        n = snprintf(error, len_error, "Error: Out of print stack\n");
        return -1;
      }
      break;
    }
```

The `END_LIST` instruction takes no argument and simply adds the
closing parenthesis.

```
    case END_LIST: 
      n = snprintf(buf + offset, len - offset, ")");
      offset += n;
      break;
```

The `PRINT_SPACE` instruction is equally simple, it takes no arguments
and just adds a space.

```
    case PRINT_SPACE:
      n = snprintf(buf + offset, len - offset, " ");
      offset += n;
      break;
```      

If none of the cases above apply, the stack is corrupted.

```
    default:
      snprintf(error, len_error, "Error: Corrupt print stack!");
      return -1;
    }// Switch instruction
  }//While not empty stack


  if (!stack_is_empty(&s)) {
    snprintf(buf + (len - 5), 4, "...");
    buf[len-1] = 0;
    return len;
  }

  
  return n;
}
```

After leaving the while loop, there is a check to see if we exited
because of being done or because of running out of buffer to write
into.  If we have filled up the buffer the last few bytes of it is
replaced with the "..." to signal that there really was more stuff
here but it didn't fit.

## Future work 

As I understand it there is a technique called pointer-reversal that
can eliminate the need for an explicit stack as used in the
`gc_mark_phase` function. This technique would require the
use a few more bits within each cons-cell. These extra bits should
encode information about whether the `car` and the `cdr` have been
processed. In the case of lispBM that would mean that 2 bits would be
required for this information and that does not feel impossible to
tweak in there. Actually, the mark bit is already present in both the
`car` and the `cdr` field, but only the one in the `cdr` is used. This
leaves almost no excuse not to try it, I must learn more about this
pointer-reversal technique before attempting it though.

The techniques used here makes the maximum memory usage of two
functions (`gc_mark_phase` and `print_value`) controllable and
explicit. Later, I would like to take this further on a per module (or
subsystem) level and over time box in the total maximum memory use. It
would be nice if all the sizes of buffers and stacks, that can be
fixed at compile time, could be set in some configuration file as part
of a REPL or runtime system implementation.

It is possible to create a heap-structure that when you print it the
`print_value` function gets stuck in a loop (until it exhausts its
output buffer or stack space and exits). It is not entirely "natural"
to write code in a form so that it generates these heap-structures
with loops, but it is possible if experimenting with `lambda`, `let`
and `define` to get a closure which in it's closure-bound environment
holds itself. Printing such a structure does not crash the system (anymore)
but it also doesn't really provide any useful output. Some change is
needed to the way closures are printed to rule this out. I guess that
any circular or looping heap-structure would have this effect and
addressing it in a general fashion would be nice. 


___

[HOME](https://svenssonjoel.github.io)
