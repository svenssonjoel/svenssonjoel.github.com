

# Boxed Values

LispBM can encode values of 4 different types into the `car` or `cdr`
of a cons-cell. How this works can be found in the earler [walkthrough
of lispBM](../lispbm_current_status/index.html). The 4 types that can
be encoded are, `symbol`, `character`, `integer` and `unsigned
integer`. All of these types are 28Bit.

To be able to express computations on 32Bit values, `integer`,
`unsigned integer` and `float` as well as arrays, a set of *boxed*
types were introduced.

The boxed values, except arrays, are stored in cons cells but have an
extra level of indirection in the form of a typed pointer. Since the
lisp heap is small there are unused bits on the most significant side.
The 4 most significant bits are used to encode different typed pointers.
Currently the following kinds of typed pointers are in use:

```
#define PTR_TYPE_CONS        0x10000000u
#define PTR_TYPE_BOXED_I     0x20000000u
#define PTR_TYPE_BOXED_U     0x30000000u
#define PTR_TYPE_BOXED_F     0x40000000u
#define PTR_TYPE_ARRAY       0xD0000000u
```

As an example, a float value is encoded as a cons cell where all the
32Bits of the `car` are used for the floating point value. The `cdr`
of this cell contail a special symbol `DEF_REPR_BOXED_F_TYPE`. The
special symbols used in the `cdr` position of boxed values are listed
below. 

```
#define DEF_REPR_ARRAY_TYPE     0x20FFFF
#define DEF_REPR_BOXED_I_TYPE   0x21FFFF
#define DEF_REPR_BOXED_U_TYPE   0x22FFFF
#define DEF_REPR_BOXED_F_TYPE   0x23FFFF
```

Only the `cdr` field of a cons cell hold a GC bit so there is no
problem in using all 32Bits of the `car` if only it can be identified
properly.

Arrays are a special case and not fully implemented yet. Currently the
only thing that is an array in lispBM is a text string. Most of the
plumbing is there for implementing arrays of any other lispBM value
type, it just needs to be hacked up. Arrays are encoded in a very
similar way to boxed values, just that instead of a 32Bit value in the
`car` slot there is an arbitrary pointer. The array storage itself is
allocated using `malloc`. When garbage collection frees a cons cell
with the special symbol `DEF_REPR_ARRAY_TYPE`, it also calls `free` on
the pointer in the `car` position.

___

[HOME](https://svenssonjoel.github.io)
