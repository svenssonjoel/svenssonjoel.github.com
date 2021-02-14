

# Experimentation with Lava

Lava is an embedded domain specific language for hardware description
implemented in Haskell. Playing with an FPGA is a lot of fun and I have
been doing that for a while now using VHDL. So, the plan here is to experiment
a bit with Lava and try to (in time) get to the point where we can run some
Lava-generated hardware on an FPGA.

The FPGA candidates will the the Artix-7 on the Nexys A7 board as well as
the Zynq 7000 platform. 

Now! There are lots of Lavas. I found 4 of them on [Hackage](https://hackage.haskell.org/). 

1. [York Lava](https://hackage.haskell.org/package/york-lava-0.2/docs/Lava.html)
2. [Chalmers Lava 2000](https://hackage.haskell.org/package/chalmers-lava2000)
3. [Xilinx Lava](https://hackage.haskell.org/package/xilinx-lava)
4. [Kansas Lava](https://hackage.haskell.org/package/kansas-lava)

I want to peek at all of these over the course of this experimentation
but the one that is closest to my heart is the Chalmers Lava. This
Lava I have used briefly before when taking a hardware description
course at Chalmers (and used again later when being teaching assistant in
that same course).

My feeling is that Chalmers-Lava-2000 is not immediately useable as a
VHDL generator for the current state of tools for the FPGAs I have
access to (Vivado and such). Also my feeling is that it will be
important to try to, somehow, use components not described in Lava
such as a block ram on the FPGA or an AXI interface. So some
modifications will be needed and that is what this text is about. Well
about some first steps in that general direction at least.

I expect modifying the existing Lava-2000 will take quite a while and
that seeing to fruits of this labor (something running on an FPGA) is
quite distant. My Haskell is also quite rusty as I have not been using
that language very much for quite a while! So (as usual with my posts)
I would love hints, tips and feedback if someone with good Haskell
happens to read.

Chalmers-Lava-2000 comes with all this very cool interfacing to
various systems for proving properties of circuits. It would be nice
to leave that functionality as intact as possible. Not sure how that
will work out though. I don't feel like the right person to integrate new
functionality with these provers.

My Lava experimentation can be found at [Github](https://github.com/svenssonjoel/lava-experimentation).


## The heart of Chalmers-Lava-2000

Very central to the Lava EDSL is a tree type called `S` with a parameter `s`.

```
data S s
  = Bool      Bool
  | Inv       s
  | And       [s]
  | Or        [s]
  | Xor       [s]
  | VarBool   String
  | DelayBool s s
  
  | Int      Int
  | Neg      s
  | Div      s s
  | Mod      s s
  | Plus     [s]
  | Times    [s]
  | Gte      s s
  | Equal    [s]
  | If       s s s
  | VarInt   String
  | DelayInt s s
``` 

The Lava implementation uses the `unsafePerfomIO` and `Ref` approach for discovery
of sharing in the tree. This is why this `s` parameter is needed and the data type declaration
becomes rather obscure. Anyway, I am not going to try to change that at all.

The symbol type introduces the reference. A symbol is a reference to an object of type `S Symbol`. 

```
newtype Symbol
  = Symbol (Ref (S Symbol))
```
So that is how the tree is described in a bit of a roundabout way.

The type the Lava programmer very often sees is called `Signal` and it represents
either a boolean or a integer "wire" in the circuit.

```
newtype Signal a
  = Signal Symbol
```

The `a` parameter to `Signal` is a so-called phantom type (a phantom
as there is no value with type `a` associated with the `signal` type). It is expected
that a True/False signal will be a `Signal Bool` but it is entirely up to
the library implementation to enforce that property. 

## Attempting to modify the `S` type some

From the `S` type we can see that all Lava hardware will be
constructed using boolean constants, `Inv` (inversion of a signal),
`And`, `Or`, `Xor`, variables and Delay elements (and similarly for
integer signals). But there is no "foreign operation" constructor that
we could use to build a BRAM or whatever we may want that would be
inefficient to describe in logic primitives. This may be because the
FPGA provides special built in functional units to perform that
operation we are interested in or that it is just to complicated and
we want to use some existing library, macro or code generating tool to
build those for us.

So, something we want to add is a "component" constructor that
represents `N`-inputs `M`-outputs blocks whose implementation is
specified elsewhere.

It feels like when one adds components like those mentioned, the
signal count may also run quite large! To deal with that we want some
kind of vector-of-signals abstraction.  I guess one could really use
lists of lists of signals to represent the interfaces to components
but the problem with using Haskell lists for this is that as the
embedded program is evaluated we loose that structural information and
I think that we would want to carry some kind of grouping of signals
information with us all the way to the VHDL generation step.

A side effect of adding a signal-grouping primitive is that having
`Int` signals become a bit redundant. So the constructors in `S` that
are related to integer signals will be removed. 

The modified `S` datatype starts out similarly to the old one.

```
data S s
  = Bool      Bool
  | Inv       s
  | And       [s]
  | Or        [s]
  | Xor       [s]
  | VarBool   String
  | DelayBool s s

  | If       s s s
```

But then we add an attempt at a component abstraction.

```
  | Component
    String           -- name
    [s]              -- inputs
``` 

Currently the component is just a `String` that represents some kind
of an implementation of the desired functionality. This implementation
will be looked up later while doing VHDL generation.  There is also a
list of inputs. The list of inputs may need to be expended upon a bit,
I'll get back to that.

Now, a component is just one `S` node, traditionally meaning it is a
one output thing.  Some way around this is needed as we want a
component to be able to have multiple outputs. I am a bit unsure if
the attempt of that that is implemented below will work out.

```
  | ComponentOutput
    CompOut          -- A specific output bit
    s                -- Component producing output
``` 

The idea is that the constructor above represents a specific output of a component.
`CompOut` is a structure that identifies a specific output bit. I will get back to this
as well after introducing Vectors.

A vector is a grouping of some signals and represented in the tree as.

```
  | Vec -- A vector of signals
    Int -- Number of elements
    [s] -- Values
```

With this constructor you can grab any set of signals you have access to and
bunch them together in one node of the tree. To access a specific signal from
that vector there is another constructor.

```
  | VecIndex Int s 
``` 

With these two vector related constructors we can implement functions
for packing and unpacking signals into a vector.

```
packVec :: Elt a => [a] -> Vector a
unpackVec :: Elt a => Vector a -> [a]
``` 

`packVec` creates a `Vec` node and `unpackVec` creates a list of `VecIndex` nodes. 

The `ComponentOutput` node works in a similar way but I imagine that
a component output could have more structure to it than just a vector and this is
why it is not enough with just an index to identify the output bit.

```
data CompOut
  = BitOutput  
  | VecOutput   Int Int CompOut
  | TupleOutput Int Int [CompOut]
```

With the `CompOut` structure we can have components that output Tuples for example.
There is a discrepancy here between the structures possible in the output and
the input to components. I am not going to worry to much about that for now, there
are so many loose ends to tie together anyway ;). Later if need be, we can add more
structure to the input as well.

## Some more about Vectors

There is a type called `Vector` that should be visible to the Lava-programmer.
It is implemented like this:

```
data Vector a = Vector Symbol
```

Which is exactly the same way a `signal` is implemented. 

A vector is created using `mkVec`:

```
mkVec :: Elt a => [a] -> Vector a
mkVec sigs = genericLift0 (Vec n (map unwrapElt sigs))
 where n = length sigs
```

Which makes use of something called `genericLift0` and `unwrapElt` and
it creates a `Vec` node.  `genericLift0` is the following:

```
genericLift0 :: Elt a => S Symbol -> a
genericLift0 oper = wrapElt (symbol oper) 
```

which uses `wrapElt`. So unwrap and wrap of `Elt`s is what it comes down to.
These operations are part of a class:

```
class Elt a where
  unwrapElt :: a -> Symbol
  wrapElt   :: Symbol -> a
```

and there must be an instance of this class for everything that can be
put into vectors.  Currently there are two instances.

```
instance Elt (Signal a) where
  unwrapElt (Signal s) = s
  wrapElt s = Signal s
  
instance Elt (Vector a) where
  unwrapElt (Vector s) = s
  wrapElt s = Vector s
``` 

So, `packVec` and `unpackVec` currently looks like this.

```
packVec :: Elt a => [a] -> Vector a
packVec sigs = mkVec sigs

unpackVec :: Elt a => Vector a -> [a]
unpackVec (Vector s) = case unsymbol s of
                         Vec n sigs -> [genericLift0
                                         (VecIndex i s) 
                                       | i <- [0..n-1]]
                         _ -> wrong Lava.Error.NotAVec

```

The main point of these vectors that I can think of now is to use
as input to and output from components. But they could also be of a
little benefit when generating VHDL where a single variable and
assignment could be used for assigning a large number of signals for example.

## Some more about Components

I did try to implement some simple components for bitwise `neg`, `and`
and a full adder.  These are currently not very pretty but I imagine
that this can be cleaned up quite a bit later with some appropriate
type classes and functions.

``` 
bitwiseNeg :: Vector (Signal Bool) -> Vector (Signal Bool)
bitwiseNeg vec = packVec outs  
  where
    n = lengthVec vec
    output i = VecOutput n i BitOutput
  
    outs = [(wrapElt (symbol (ComponentOutput (output i) c))) :: Signal Bool| i <- [0..n-1]]
    
    -- Want just one unique symbol per component instantiation.
    -- Each output node will refer to the same component symbol.
    c = symbol (Component "bitwiseNeg" [unVec vec])
```

`bitwiseNeg` takes a vector of boolean signals and outputs a vector of
boolean signals. The `c` represents the component named "bitwiseNeg" that
takes the signals `unVec vec` as input. `unVec` does not "unpack" the vector, it rather just
unWraps the "symbol". So no structure is lost there.  

```
unVec :: Elt a => Vector a -> Symbol
unVec (Vector s) = s

```

The output is a vector created using `packVec` on the Haskell list called `outs`. Each
element of the `outs` list is a `ComponentOutput`. In this case these Component node, each
of them, hold a representation of the output shape of the component.  


The `fullAdder` is just an attempt to build a `ComponentOutput` that is a tuple

```
fullAdder :: Signal Bool -> Signal Bool -> Signal Bool -> (Signal Bool, Signal Bool)
fullAdder a b cin = (o, sum) 
  where
    o = lift0 (ComponentOutput (output 0) c)
    sum = lift0 (ComponentOutput (output 1) c)
    output i = TupleOutput 2 i [BitOutput, BitOutput]
    c = symbol (Component "fullAdder" (map unsignal [a,b,cin]))
``` 


## TODOs and Conclusion

I really feel like I am on thin ice here! I've made a lot of strange
changes and I am not really sure they are all that sound. Will proceed
and try to see what happens as a Lava program is reified into a graph
(via the sharing detection) and then see if what pops out at the other
end is something suitable for VHDL generation.

Please let me know if you think this is fun or if you have feedback or ideas.
Maybe you want to jump in and help out? ;)

Thanks and have a great day!

___

[HOME](https://svenssonjoel.github.io)
