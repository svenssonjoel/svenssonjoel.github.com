

# Type Level Full-Adder in Haskell

I haven't really done much Haskell programming for a couple of years.
Recent changes in employment will make it more likely that I will be
doing some Haskell programming from now on though. I have always felt
like a "katt bland hermelinerna" (not quite fitting in "skunk at the
garden party" or "fox in the hen-house" perhaps) in the Haskell
community. Haskell is hard and messy and especially when there is too
much "type-magic" I very easily get lost and confused. Don't get me
wrong, the Haskell community is very friendly, just on average way
more clever than I am. 

What I am getting at is that I have absolutely no idea what I am doing
in the code presented in this blog-post! But this is quite ok by me and
totally in line with the goals of my blogging. I want to post about my
work as unfiltered as possible to share my stupid mistakes and
horrible misunderstandings. I hope there are the occasional useful
bit of info as well of course! I also greatly appreciate any pointers
I get from anyone reading. So thanks ;)

I got a really out-of-character urge today to see if I could implement
a full-adder in Haskell types. From what I can tell it seems to work.

Two extensions are used for this experiment, `TypeFamilies` and
`UndecidableInstances`.  `UndecidableInstances` sounds really
menacing, it apparently makes it possible end up with a
non-terminating type-check. The `TypeFamilies` extension, as far as I
understand, allows a limited form of function definition on the
type-level. 

```
{-# LANGUAGE TypeFamilies #-}
{-# LANGUAGE UndecidableInstances #-}
``` 

The data types `One` and `Zero` are the values that a "bit" can hold.

```
data One
data Zero
``` 

Two type level functions `And` and `Or` implement the truth tables for
the these operations over tuples of bits. 

```
type family And a
type instance And (Zero,  One) = Zero
type instance And (One,  Zero) = Zero
type instance And (One,   One) = One
type instance And (Zero, Zero) = Zero

type family Xor a
type instance Xor (Zero,  One) = One
type instance Xor (One,  Zero) = One
type instance Xor (One,   One) = Zero
type instance Xor (Zero, Zero) = Zero
``` 

A full-adder can be constructed by combining two so-called half
adders.  So the `HalfAdder` function implements this operation. To
make sense, it should be applied to a tuple of bits (are there any of
enforcing this nicely?) and it computes the sum of these bits and also
computes a carry value. `Xor a` is the sum part and `And a` is the
carry part.

``` 
type family HalfAdder a
type instance HalfAdder a = (Xor a, And a)
``` 

So that part fell into place without to much trouble. So the next step
is to combine two half adders and a bit of extra logic to form the full
adder. Here I got very stuck. And the thing causing trouble is that
you cannot partially apply the type family based type level functions
(as I understand it) or pass them around unapplied. This [answer on
stackoverflow](https://stackoverflow.com/questions/50494228/type-family-as-argument-to-type-synonym)
provided a way through this obstacle though.

A full adder takes three bits as input, call them a, b and carry-in,
and outputs two bits called sum and carry-out.  This way many full
adders can connected together to form multi-bit adders, so called
ripple carry adders. For the full adder, two half adders and an or
gate is needed.

I started out something like this. There needs to be half adder somewhere in there to compute the sum of `a` and `b`. 
``` 
type family FullAdder a
type instance FullAdder ((a,b),c) = ... (HalfAdder (a, b)) ... 
```

But after that it gets messy. Now one of the outputs of the half adder
should be fed to another half adder together with `c`. So either the
result of `HalfAdder (a, b)` should be "untupled" in some way and then
one of the elements in that tuple should be "retupled" with `c` and
fed to half adder number 2. This doesn't work!

So instead I thought it may be possible to express some operations over the result tuples..

```
type family AppFst (f :: * -> *) p
type instance AppFst f ((a, b), c) = (f (c, a), b)
```

Then change the full adder like this:

```
type family FullAdder a
type instance FullAdder ((a,b),c) = ... AppFst HalfAdder ((HalfAdder (a,b)), c)
```

That doesn't work at all!

```
 error:
    • The type family ‘HalfAdder’ should have 1 argument, but has been given none
    • In the type instance declaration for ‘FullAdder’
```     

This is where the fix from
[stackoverflow](https://stackoverflow.com/questions/50494228/type-family-as-argument-to-type-synonym)
comes in. It seems to not be possible to have `HalfAdder` unapplied in
there like that, but it is possible to have a "tag" representing the
half adder in there together with an extra `App` (for apply) type
family that represents the application of one of these "tags".

```
type family App (token :: *) a

data TokHalfAdd
data TokOr

type instance App TokHalfAdd a = HalfAdder a
type instance App TokOr a = Or a
```

The code above implements application of tags and the tags needed for
the half adder and the or gate. The two type instances are a kind of evaluator for
the tags. 

Now the `AppFST` combinator can be restated like this:

```
type family AppFST (f :: *) p
type instance AppFST f ((a, b), c) = (App f (c,a), b)
```

And a bit of progress can be made on the full adder.

```
type family FullAdder a
type instance FullAdder ((a, b),c) = ... (AppFST TokHalfAdd ((HalfAdder (a,b)),c))
``` 

The last step is to or together the two carry bits and this means adding a `AppSND` combinator.

```
type family AppSND (f :: *) p
type instance AppSND f ((a, b),c) = (a, App f (c,b))
```

And the full adder can be completed.

``` 
type family FullAdder a
type instance FullAdder ((a, b),c) = AppSND TokOr (AppFST TokHalfAdd ((HalfAdder (a,b)),c))
```


Now, this seems to make it possible to "simulate" full adders in the Haskell type-checker.
for example 

```
a1 :: FullAdder ((Zero, Zero), Zero)
a1 = undefined
```

The result of this addition should be (Zero, Zero), so let's ask GHCI. 
```
*Main> :t a1
a1 :: (Zero, Zero)
``` 

Good, but not very interesting. Let's try some more: 

``` 
-- (One, Zero)
a2 :: FullAdder ((Zero, Zero), One)
a2 = undefined 

-- (one, Zero)
a3 :: FullAdder ((Zero, One), Zero)
a3 = undefined

-- (Zero, One)
a4 :: FullAdder ((Zero, One), One)
a4 = undefined 

-- (One, Zero)
a5 :: FullAdder ((One, Zero), Zero)
a5 = undefined 

-- (Zero, One)
a6 :: FullAdder ((One, Zero), One)
a6 = undefined 

-- (One, One)
a7 :: FullAdder ((One, One), One)
a7 = undefined 
```

And ask GHCI about these. 

``` 
*Main> :t a2
a2 :: (One, Zero)
*Main> :t a3
a3 :: (One, Zero)
*Main> :t a4
a4 :: (Zero, One)
*Main> :t a5
a5 :: (One, Zero)
*Main> :t a6
a6 :: (Zero, One)
*Main> :t a7
a7 :: (One, One)
```

## Conclusion

Help! Does any of this make any sense? 

This was lots of fun though! So I thought I'd share it ;) 


___

[HOME](https://svenssonjoel.github.io)
