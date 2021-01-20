

# Fundamental Operations and Platform Specific Extensions

My plan with lispBM is that it should run on smaller and memory
constrained devices such as the STM32F4 (or smaller) and the NRF52. I
have ordered a couple of ESP32s and a 32 bit RISC-V based development
board and those are the next likely attempts to target. But when
programming microcontrollers there are often "differences" between
families and manufacturers. I do, however, expect that there will be
some kind of really basic functionality that is available on all of
them. These expected functionalities are the *fundamental operations*,
a small set of functions that I want in any lispBM setup.

So when it comes to the things that are expected to be quite different
between different platforms, these are implemented as user defined
*extensions*. An example of this could be a `print` function for
outputting text. This text could for example be sent over UART or some
USB connection, or even Bluetooth as in the NRF52. It is better to
just leave this choice open until the specifics of the target platform
are known.

Fundamental operations and extensons are quite similar in many
ways. They are both identified by a symbol and both expect that all the
arguments they operate on are pushed onto the continuation stack such
that argument 1 is furthest down in the stack.

The symbol used to identify fundamental operations are given a fixed
value in the range of *special* symbols. This means that they are
numbers of the form `0xXYZFFFF` in hexadecimal notation.  The
extension symbols however are allocated on the symbol representation
table when the extension is added to the system. 

## Fundamental Operations

The code that is involved in the execution of fundamental operations
are located in `fundamental.h` and `fundamental.c`. This subsystem
needs no initialization and provide only one external function called
`fundamental_exec`.

The `fundamental_exec` function takes a pointer to the first argument
(that is, a pointer into the stack at the position of the first
argument), the number of arguments and lastly the Symbol value that
represents which fundamental operation that should be run.

```
extern VALUE fundamental_exec(VALUE* args, UINT nargs, VALUE op);
``` 


The `fundamental_exec` function is called from deep within the
`apply_continuation` in the evaluator.  For a look at the entire
`apply_continuation` function, see [A Closer Look at LispBM's
Evaluation Function](../lispbm_evaluation_function/index.html).

```
VALUE apply_continuation(eval_context_t *ctx, VALUE arg, bool *done, bool *perform_gc, bool *app_cont){


    ...


    VALUE count;
    pop_u32(ctx->K, &count);
    UINT *fun_args = stack_ptr(ctx->K, dec_u(count)+1);
    VALUE fun = fun_args[0];


    ...
 

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


    ...

}

```

First there is a check to see if symbol `fun` represents a fundamental
operation. If the symbol represents a fundamental `fundamental_exec`
is called and provided a pointer to the arguments and number of
arguments along with the `fun` symbol.

This call could potentially fail with a memory error (out of heap).
In this case the stack is restored (the count and the `APPLICATION`
continuation is pushed back onto the stack) and the `perform_gc` and
`app_cont` flags are set. This means that after GC has been run
the next thing the evaluator does is call the continuation again
and we should arrive at the exact same case, but now with some freed
up heap.

If executing the fundamental operation is successful, the arguments
are removed from the stack and evaluation continues.


Some of the fundamental operations operate on a fixed number of
arguments, such as for example `cons` and `car`. Those functions
completely ignore any further arguments given, but they do not
complain in any way:

```
# (cons 1 2 3 4 5)
> (1 2)

# (car '(1 2 3 4) 'apa 'kurt 'bertil)
> 1

# (cdr '(1 2 3 4) 'apa 'kurt 'bertil)
> (2 (3 (4 nil)))
```

Other fundamental operation take an arbitrary number of argumens. Of
course the size of stack is a limiting factor so stay
reasonable. Examples of this kind of functions are `list` and `+`.

```
# (list)
> nil
# (list 1)
> (1 nil)
# (list 1 2)  
> (1 (2 nil))
# (list 1 2 3)   
> (1 (2 (3 nil)))

# (+) 
> 0
# (+ 1)
> 1
# (+ 1 2)
> 3
# (+ 1 2 3)
> 6
```

The `fundamental_exec` function that perform all these fundamental
operations are implemented as a huge switch statement on the function
symbol. Here I will just show a few representative cases of how these
are implemented.

`cons` takes 2 arguments. It asks the heap for a cons cell and then it
sticks the first argument in the `car` position and the second
argument in the `cdr` position of this cell.

```
  case SYM_CONS: {
    UINT a = args[0];
    UINT b = args[1];
    result = cons(a,b);
    break;
  }
```

The C functions that performs the actual "consing" is defined in
`heap.c`.  For the fundamental operations `car` and `cdr` it is the
same, they are also implemented using C function with exactly the same
name, defined in `heap.c`.

```
 case SYM_CAR: {
    result = car(args[0]);
    break;
  }
  case SYM_CDR: {
    result = cdr(args[0]);
    break;
  }
```

The `list` fundamental, that takes an arbitrary number of arguments,
loops over the arguments while it generates a list structure from cons
cells. Since the arguments are in order from first to last in the
argument array, it iterates over the arguments backwards to start off
with consing the last element to `nil`.

```
  case SYM_ADD: {
    UINT sum = args[0];
    for (UINT i = 1; i < nargs; i ++) {
      sum = add2(sum, args[i]);
      if (type_of(sum) == VAL_TYPE_SYMBOL) {
        break;
      }
    }
    result = sum;
    break;
  }
```

Likewise the `+` fundamental loops over the arguments, but here in
forwards order (doesn't matter in this case).

```
  case SYM_ADD: {
    UINT sum = args[0];
    for (UINT i = 1; i < nargs; i ++) {
      sum = add2(sum, args[i]);
      if (type_of(sum) == VAL_TYPE_SYMBOL) {
        break;
      }
    }
    result = sum;
    break;
  }
```

The case `SYM_ADD` that implements addition contains an `add2`
function. The reason it is not just using the C `+` operator is
because the arguments passed to the `+` fundamental can be anything
(well any number type) for example:

```
# (+ 1 8.4)
> {9.400000}
```

Here `1` will be parsed as a 28bit integer and `8.4` will be parsed
into a boxed 32bit floating point value. So `+` has to convert the `1`
"upwards" to a floating point value as well before adding them together.
The curly brackets surrounding `9.4` is how lispBM prints boxed values.

Another example:

```
(+ 1u32 1u28 7.5)
> {9.500000}
```

Here the boxed 32bit number `1` is added to the unboxed `1` and the
boxed float `7.5`. All of this conversion of types of numbers are
handled within the `add2` function.

## Extensions

Extensions are very similar to fundamentals but are created by the
implementor of a REPL (or any other form of interpreter) for a
specific platform. Here, a symbol is allocated and used as an identity
attached to a function pointer provided by the programmer.

The following functions are defined in `extensions.h` and `extensions.c`

```
typedef VALUE (*extension_fptr)(VALUE*,int);

extern extension_fptr extensions_lookup(UINT sym);
extern bool extensions_add(char *sym_str, extension_fptr ext);
extern void extensions_del(void);
```

When you add an extension using `extensions_add` the function pointer
and the symbol are added to a linked list. This list is then searched
when doing an `extension_lookup`.  Looking up with a symbol that does
not correspont to an extension returns `NULL`.

Just like fundamental operations all extensions take a pointer to the
first argument and the number of arguments as input. But here you are
in full control of what you want your extensions to do with these
values. For example a possible extension would be to take an ID
representing a GPIO pin as input and then toggle that pin in the
implementation. 

___

[HOME](https://svenssonjoel.github.io)
