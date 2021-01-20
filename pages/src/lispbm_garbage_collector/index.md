

# Garbage Collection in LispBM

In lispBM a *mark and sweep* garbage collector is used to obtain
automatic management, allocation and freeing, of heap objects. The
garbage collector (GC) also manages arrays that are allocated
off-heap. There is always a "box" on the heap that points to these
off-heap objects (see [Boxed
Values](../lispbm_boxed_values/index.html) for more details).

Garbage collection is an area I would like to learn more about. The GC
implemented in lispBM is most likely quite naive and definitely not
optimized! I wrote it before having access to the book [Garbage
Collection: Algorithms for Automatic Dynamic Memory
Management](https://www.amazon.com/Garbage-Collection-Algorithms-Automatic-Management/dp/0471941484?tag=uuid10-20).
When improvements are applied to the GC (If I get time to more than
just skim the book), I will write about that as it happens.

Even though this is not a very polished piece of code, I think it may
be good to document it here. All the earlier texts I've written on the
lispBM implementation made me see things that could be improved or
fixed! So writing it down was valuable to me, I hope it also of value
to someone else. 

The GC is tightly connected to the heap so in the lispBM
implementation the GC code is located in the files `heap.h` and
`heap.c`. For more information about the heap in lispBM
read the earlier text [Another Lisp for
Microcontrollers](../lispbm_current_status/index.html).

## Mark and Sweep Garbage Collection

In lispBM the heap is made up from cons-cells, which each is a pair of
32bit words.  Each cons-cell has a mark bit. The purpose of this mark
bit was hinted at in [Another Lisp for
Microcontrollers](../lispbm_current_status/index.html) but all the
details of GC were left out. The `free_list`, which is a linked list of
all non-used cons-cells, was also mentioned in that first text on
lispBM. In this text I will try to go into all the details.

It is up to the garbage collector to add cons-cells that are no-longer
*reachable* to the free list. This is where the mark bits come in. The
idea is that if an object on the heap cannot be reached by following a
pointer that the program *knows about* then that object can be freed
and its cons-cells can be moved to the free list. It is, however, much
easier to figure out what can be reached rather than what cannot so
the approach is to traverse the current environment and mark
everything that can be reached from there as "in-use".

The way lispBM is implemented, it is important to traverse every
data-structure that could potentially hold pointers into the heap that
are needed in the future. This means the following must be traversed:

1. The global environment,
2. The current environment (local environment).
3. The current expression.
4. The Program (which is a list of expressions that all should be evaluated).
5. The `r` intermediate result that is passed around in the evaluator and `apply_continuation`.
6. The continuation stack

The intuition behind these are that (1) the global environment should
always be available to use, so nothing in there can be removed.  The
local environment (2) is the environment we are operating within right
now, so better not remove any of that either.  The current expression
(3) is an object on the heap, it better remain there if we want to
keep working with it.  When it comes to the program, really we only
need to keep *the rest of the program* but for simplicity the entire
program will be traversed and marked. The intermediate result (5)
could potentially be a pointer to some data-structure on the heap,
better keep that!  The continuation stack must not be forgotten, it
can hold arguments to a function that could also be for example a list
on the heap. This is why anything that goes onto the continuation
stack is a lispBM `VALUE` and that is why continuation identifiers, such
as `PROGN_REST`, are encoded as lispBM 28Bit unsigned integers. If we
put arbitrary data on the continuation stack, the GC could potentially
confuse it for a heap pointer but since it is now a lispBM `VALUE` it
carries additional type information.

That is how marking of what is in use is done, conceptually. The next
step is the sweep. The sweep iterates over the entire heap and for
each cons-cell it checks the mark-bit. If the mark-bit is not set for
a cons-cell that cell is appended to the free list. The sweep also
resets all the mark-bits so that when the next marking session is run,
there wont be anything already marked (perhaps incorrectly at this
stage).

At a very high level this is how mark and sweep works, or at least how
I understand it should work. We will now go through the code in the
lispBM implementation that performs these operations.

## Mark and Sweep Garbage Collection in LispBM

There is not a whole lot of code involved in the GC in lispBM, maybe
something like 100 lines of C. These 100 lines (or so) was, however,
quite an effort to produce. It is all in the details here, be off by
one anywhere and there will be *segfaults*. Now, this code probably
still has corner cases that end up in not so good states. Its a
process, learning by doing.


There is an entry-point function called `heap_perform_gc_aux` that
takes 7 arguments. The arguments correspond to those 6 items listed
above that have to be traversed as part of the marking procedure. The
7th argument is the number of elements pushed onto the continuation
stack.

As can be seen in the code below, the free list is also traversed and
marked.  This makes more sense to me than not traversing and marking
it, since it would just end up on the free list again anyway but
potentially involving a bit more work before getting there.

So, `heap_perform_gc_aux` runs a marking algorithm on each of the
data-structures that contain things that needs to be marked. Once
everything is marked a snapshot image can be generated of the heap for
visualization purposes. This is controlled by a compile time
flag. After marking, it performs a sweep.

```
int heap_perform_gc_aux(VALUE env, VALUE env2, VALUE exp, VALUE exp2, VALUE exp3, UINT *aux_data, unsigned int aux_size) {
  heap_state.gc_num ++;
  heap_state.gc_recovered = 0;
  heap_state.gc_marked = 0;

  gc_mark_freelist();
  gc_mark_phase(exp);
  gc_mark_phase(exp2);
  gc_mark_phase(exp3);
  gc_mark_phase(env);
  gc_mark_phase(env2);
  gc_mark_aux(aux_data, aux_size);

#ifdef VISUALIZE_HEAP
  heap_vis_gen_image();
#endif

  return gc_sweep_phase();
}
```

In `heap_perform_gc_aux` we can see that there are a few different functions
that perform marking.

 1. `gc_mark_freelist`
 2. `gc_mark_phase`
 3. `gc_mark_aux`

Out of these `gc_mark_phase` is the most general. It traverses any
heap structure and marks all reachable nodes. The `gc_mark_freelist`
function assumes that the argument passed to is a proper linked-list
and can iterate over it using a while loop. `gc_mark_aux` is passed an
array as argument and can also iterate over its argument efficiently.

```
int gc_mark_phase(VALUE env) {

  if (!is_ptr(env)) {
      return 1; // Nothing to mark here
  }

  if (get_gc_mark(ref_cell(env))) {
    return 1; // Circular object on heap, or visited..
  }

  // There is at least a pointer to one cell here. Mark it and recurse over  car and cdr 
  heap_state.gc_marked ++;

  set_gc_mark(ref_cell(env));

  VALUE t_ptr = type_of(env);

  if (t_ptr == PTR_TYPE_BOXED_I ||
      t_ptr == PTR_TYPE_BOXED_U ||
      t_ptr == PTR_TYPE_BOXED_F ||
      t_ptr == PTR_TYPE_ARRAY) {
    return 1;
  } 

  int res = 1;
  res = res && gc_mark_phase(car(env));
  res = res && gc_mark_phase(cdr(env));

  return res;
}
```

The `gc_mark_phase` function takes a lispBM `VALUE` as argument. The
argument is called `env` but that is not important, it could be any
heap structure.

The first thing that is checked is if the argument, `env` is a
pointer. If it is not a pointer it is a value of some kind, for
example a symbol, this is an indication that we have reached a leaf
node and no marking is needed so traversal is aborted. The
`gc_mark_phase` function is recursive, so this leaf node case is also
a recursion terminating case.

If the argument is a pointer, then a second check is performed to see
if the cell that the pointer points to is already marked. This could
turn up if there is a circular structure stored on the heap. We do not
need to traverse down a path that is already marked, so this is also a
case that stops recursion.

If we did not already exit the `gc_mark_phase` function, it means we
have a pointer to something that is not marked. So then we mark it.
It could be that the pointer points to a special cell, a so-called
boxed value of some kind in that case we do not traverse into it, just
mark the box. The contents of the box is an arbitrary value in the car
position and a special symbol in the cdr. We cannot go into the box
here, because the arbitrary value could be mistaken for a valid heap
pointer, but we also know that we do not need to go into the box as it
wont contain any further heap pointers.

And now, if we are still in the function, two recursive calls are made
on the `car` and the `cdr`.

The free list is a very special case. We know a lot about this
list. It is a proper linked list where the `car` is always a special
symbol (called *RECOVERED*) and the `cdr` is either a pointer to
another cell or the symbol `nil`. So we can very efficiently traverse
and mark all of the free list. And if we encounter something that does
not match the rules above something is wrong and the list is surely
corrupted. 

```
int gc_mark_freelist() {

  VALUE curr;
  cons_t *t;
  VALUE fl = heap_state.freelist;

  if (!is_ptr(fl)) {
    if (val_type(fl) == VAL_TYPE_SYMBOL &&
	fl == NIL){
      return 1; // Nothing to mark here
    } else {
      return 0;
    }
  }

  curr = fl;
  while (is_ptr(curr)){
     t = ref_cell(curr);
     set_gc_mark(t);
     curr = read_cdr(t);

     heap_state.gc_marked ++;
  }

  return 1;
}


```

To mark the free list, first check if the free list empty. So, if the
free list is the symbol `nil`, that indicates that there are no free
cells and we can return immediately. If it is a symbol but not `nil`
then there is a problem.

Otherwise, just loop over the linked list as long as the current value
is pointer and mark everything along the way.

The `gc_mark_aux` function (used for marking the continuation stack)
is also very efficient but it has to run the general `gc_mark_phase`
function on each element of the stack that may be a heap structure. In
other words, all values can be skipped and ignored.

```
int gc_mark_aux(UINT *aux_data, unsigned int aux_size) {

  for (unsigned int i = 0; i < aux_size; i ++) {
    if (is_ptr(aux_data[i])) {

      TYPE pt_t = ptr_type(aux_data[i]);
      UINT pt_v = dec_ptr(aux_data[i]);

      if ( (pt_t == PTR_TYPE_CONS ||
            pt_t == PTR_TYPE_BOXED_I ||
            pt_t == PTR_TYPE_BOXED_U ||
            pt_t == PTR_TYPE_BOXED_F ||
            pt_t == PTR_TYPE_ARRAY ||
            pt_t == PTR_TYPE_REF ||
            pt_t == PTR_TYPE_STREAM) &&
           pt_v < heap_state.heap_size) {

        gc_mark_phase(aux_data[i]);
      }
    }
  }

  return 1;
}

```


That concludes marking of used heap objects and it is time for the
sweep.  The sweep function would also have been very tiny if it
weren't for boxed values that need some special attention. If we
forget about those for a moment, what the sweep does is to loop over
the entire heap and in each cell it checks the mark bit. If the bit is
not set, the cell is added to the free list.

The extra complication comes from arrays that are stored in cells
where the `car` is arbitrary pointer (not to the heap) and the `cdr`
is the symbol `DEF_REPR_ARRAY_TYPE`. If an object with this property
is encountered `free` should be called on the address in the `car` in
addition to adding the cell to the free list. 

```
int gc_sweep_phase(void) {

  unsigned int i = 0;
  cons_t *heap = (cons_t *)heap_state.heap;

  for (i = 0; i < heap_state.heap_size; i ++) {
    if ( !get_gc_mark(&heap[i])){

      // Check if this cell is a pointer to an array
      // and free it.
      if (type_of(heap[i].cdr) == VAL_TYPE_SYMBOL &&
          dec_sym(heap[i].cdr) == DEF_REPR_ARRAY_TYPE) {
        array_t *arr = (array_t*)heap[i].car;
        switch(arr->elt_type) {
        case VAL_TYPE_CHAR:
          if (arr->data.c) free(arr->data.c);
          break;
        case VAL_TYPE_I:
        case PTR_TYPE_BOXED_I:
          if (arr->data.i) free(arr->data.i);
          break;
        case VAL_TYPE_U:
        case PTR_TYPE_BOXED_U:
        case VAL_TYPE_SYMBOL:
          if (arr->data.u) free(arr->data.u);
          break;
        case PTR_TYPE_BOXED_F:
          if (arr->data.f) free(arr->data.f);
          break;
        default:
          return 0; // Error case: unrecognized element type.
        }
        free(arr);
        heap_state.gc_recovered_arrays++;
      }

      // create pointer to use as new freelist
      UINT addr = enc_cons_ptr(i);

      // Clear the "freed" cell.
      heap[i].car = RECOVERED;
      heap[i].cdr = heap_state.freelist;
      heap_state.freelist = addr;

      heap_state.num_alloc --;
      heap_state.gc_recovered ++;
    }
    clr_gc_mark(&heap[i]);
  }
  return 1;
}
```

When cells are added to the free list the `car` position is set to the
symbol `RECOVERED`, this is to help with debugging. If all cells that
have been freed contain the symbol `RECOVERED` it is easy to notice if
a bug is causing a value that is still in use to be freed as a
completely nonsense symbol would appear in the expression.

The sweep also clears the mark bit of all cells as it traverses the
heap so that things are set up properly for the next round of marking.

Wow. Looking back at this now is interesting. This code was such a
struggle to produce at one time and then I haven't really even looked
at it again for quite a while. Now it seems like there is almost
nothing to it! However, with this kind of code it is all in the
details. Getting the details right when also learning about the "big
picture" is something to not take lightly. 

If you have questions about this or tips on how I can make it more
accessible or suggestions of improvements, I would be very happy.

___

[HOME](https://svenssonjoel.github.io)
