

# A complete noob's first attempt at Coq

It seems to me that formal methods will become extremely important
from now on. I am thinking about autonomous vehicles, industrial
automation and so on. In areas where machines interact with human
beings there should be a requirement that the machine is proven safe
and correct. This forecast is also very sad, because I know absolutely
nothing about formal methods or proving properties about hardware or
software.

I am now going to try to understand one of these proof systems and the
one chosen is the [Coq Proof Assistant](https://coq.inria.fr/). I will
share my questions and what I believe to be insights here in the hope
that it might help another beginner, who doesn't see the point, in
some way. I really do hope to catch the point of this eventually!
if any expert stumbles over this text about my first steps into
proving stuff please send me hints, tips and feedback!

As a complete noob, I have perceived a barrier to getting access to
this area.  There are lots of videos and tutorials, but they all seem
to assume that you already know a lot either about functional
programming or about logic or both. The tutorials and videos then go
directly into proving things about these functional programs, but how
do you use it? If you prove a property about some core part of an
operating system for example, what you also want to have is an
implementation of that part (expressed in some efficient language).
Ideally the implementation of this operating system component should be
generated somehow from the specification used in Coq in a trustworthy
way, right?

Below are two questions I have about all of this at a high level. At
some time in the future I hope to have some understanding of the
answers for these.

- Coq has a purely functional language that requires all functions to
  be total. This functional language has no support for file IO or any
  other kind of IO (as I understand it today), how does one write any
  real practical application in Coq?
- What does it mean when someone claims to have implemented, say, a
  compiler and proven it correct using Coq or any other proof
  assistant?

At the lower level, when it comes to actually working with Coq, there
is a lot to learn of course. So surely there will be more questions
throughout the text.

To get an introduction I looked at the first lecture of [this video
series](https://www.youtube.com/playlist?list=PLGCr8P_YncjUT7gXUVJWSoefQ40gTOz89)
and it was very good. You get a feeling for a bit of the syntax and
basics. The syntax is quite busy by the way.  My preferred way of
learning about things is "learning by doing" so after watching the
first of the lectures I switched to trying some defining and proving
of my own with the help of the [Coq reference
manual](https://coq.inria.fr/refman/index.html). 

Now, the rest of this text will be about my first attempt at some
programming and proving in Coq. I went for Booleans and a 
few operations on booleans (and, or, not) and hoped to show for example 
some De Morgan identities. 

In CoqIde I type the following to define Booleans.
``` 
Inductive boolean : Type := 
  | true : boolean
  | false : boolean.
```

You can type this directly into a `*scratch*` buffer in CoqIde.
Now pressing **CTRL + DOWN-ARROW** will highlight the definition of 
boolean in green color and the window down to the right says "boolean is defined".
The window down to the right also says 4 other things are defined but I have 
no idea what these mean so far. 

Anyway, as I understand it we have defined a type for booleans now. 
So the next step should be to write an operation on booleans. 

```
Definition or ( x y : boolean) : boolean := 
  match x with
  | true => true
  | false => y
  end.
``` 

The code above defines the `or` function that takes two arguments `x` and `y` that 
are booleans and returns a boolean. The body is implemented using pattern matching 
on `x`, the first argument.

Expressions involving `or` can now be evaluated. 

```
Eval compute in (or false true).
```
Pressing **CTRL + DOWN-ARROW** now should output the result `true` in 
the lower right message window.

The expression below should evaluate to `false`.
```
Eval compute in (or false false).
```

Proofs in Coq take the form of sequences of tactic applications and
trying to write some proofs was pretty fun. But here is where I get
really insecure. I was kind of expecting to show things like 
`or a b -> or b a` but implication is not a relation on the booleans
used. So then almost all theorems I could think of had to be phrased 
as equalities. 

``` 
Theorem boolean_or_false : forall a : boolean, or a false = a.
```

If we press **CTRL + DOWN-ARROW** so that the `Theorem` is passed to Coq, 
it will turn green and the top right window in CoqIde shows the following.

```
1 subgoal
______________________________________(1/1)
forall a : boolean, or a false = a
``` 

Saying that we have 1 thing to prove. The thing to prove is 
the statement below the line that for all booleans, `a`, `or a false = a`. 

As I recall from my courses in logic, to prove a `forall` one should
assume an arbitrary element of the set that the forall ranges over.
This is how I understand what happens in the first step of this
proof. I may be wrong though.

```
Theorem boolean_or_false : forall a : boolean, or a false = a.
Proof. 
intros a. 
```

When **CTRL + DOWN-ARROW** over the line with intros, the following 
appears in the, I don't know what to call it, proof window.

```
1 subgoal
a : boolean
______________________________________(1/1)
or a false = a
```
An `a` appeared over the line. The things above the line are 
the assumptions. 

`or a false = a` seems easy enough so I try apply the `simpl`
tactic. This doesn't do anything in this case though. `simpl` as I
understand it will try to simplify expressions and maybe the cause it
cannot do anything is the totally arbitrary `a` that is part of the
equality?

So, another thing to try is the `case` tactic that takes a variable as
argument.  The goal will then be split/duplicated into separate goals
for each possible value of that variable.

```
Theorem boolean_or_false : forall a : boolean, or a false = a.
Proof. 
intros a.
case a.
```
And stepping over that shows the following result in the proof window.

```
2 subgoals
a : boolean
______________________________________(1/2)
or true false = true
______________________________________(2/2)
or false false = false
``` 

Now there are two goals to prove, `or true false = true` and `or false
false = false` And at this stage it is fine to apply the `simpl`
tactic. 

```
Theorem boolean_or_false : forall a : boolean, or a false = a.
Proof. 
intros a.
case a.
simpl.
```

Resulting in this:

``` 
2 subgoals
a : boolean
______________________________________(1/2)
true = true
______________________________________(2/2)
or false false = false
```

The first goal is now pretty obviously a true statement and applying `reflexivity`
closes that branch (goal). 

``` 
Theorem boolean_or_false : forall a : boolean, or a false = a.
Proof. 
intros a.
case a.
simpl.
reflexivity.
```
Resulting in:

``` 
1 subgoal
a : boolean
______________________________________(1/1)
or false false = false
```

So it seems there is only one goal left and just as with the earlier
`or true false = true` this one can be closed with `simpl` followed by
`reflexivity`. Even just saying `reflexivity` is fine in both these 
cases. so the complete proof would look like this:

```
Theorem boolean_or_false : forall a : boolean, or a false = a.
Proof. 
intros a.
case a.
reflexivity.
reflexivity.
Qed.
``` 

The next theorem I attempted was that `or a true = true`.
```
Theorem boolean_or_true : forall a : boolean, or a true = true.
``` 
Will not show that proof, it is very similar to the earlier one. 

The next one is a bit interesting though as it is the first example 
where I could make use of earlier theorems in the proof of a new one. 
The theorem is that `or` is commutative. 

```
Theorem boolean_or_comm : forall a b : boolean , or a b = or b a.
Proof.
```
Starting off as usual. 

```
1 subgoal
______________________________________(1/1)
forall a0 b : boolean, or a0 b = or b a0
```

Introducing variables `a` and `b`. 

```
Theorem boolean_or_comm : forall a b : boolean , or a b = or b a.
Proof.
intros a b.
```
Adds `a` and `b` to the assumptions. 

```
1 subgoal
a, b : boolean
______________________________________(1/1)
or a b = or b a
```

Then I split into cases by the `b` variable. 

``` 
Theorem boolean_or_comm : forall a b : boolean , or a b = or b a.
Proof.
intros a b.
case b.
```

Now the proof window reads:

```
2 subgoals
a, b : boolean
______________________________________(1/2)
or a true = or true a
______________________________________(2/2)
or a false = or false a
```

Applying `simpl` at this point turns subgoal 1 into something that 
looks exactly like something we have already proved. 

```
Theorem boolean_or_comm : forall a b : boolean , or a b = or b a.
Proof.
intros a b.
case b.
simpl.
``` 
And the proof window shows:

``` 
2 subgoals
a, b : boolean
______________________________________(1/2)
or a true = true
______________________________________(2/2)
or a false = or false a
```

Subgoal (1/2) looks exactly like the `boolean_or_true` theorem. So let's apply it. 

``` 
Theorem boolean_or_comm : forall a b : boolean , or a b = or b a.
Proof.
intros a b.
case b.
simpl.
apply boolean_or_true.
```
And that closes that subgoal.

```
1 subgoal
a, b : boolean
______________________________________(1/1)
or a false = or false a
```

If we now run `simpl` the only subgoal that is left will look very much 
the `boolean_or_false` theorem. 

``` 
Theorem boolean_or_comm : forall a b : boolean , or a b = or b a.
Proof.
intros a b.
case b.
simpl.
apply boolean_or_true.
simpl.
```

The goal left to prove now is the `boolean_or_false` theorem.

``` 
1 subgoal
a, b : boolean
______________________________________(1/1)
or a false = a
```

So let's apply this theorem. 

``` 
Theorem boolean_or_comm : forall a b : boolean , or a b = or b a.
Proof.
intros a b.
case b.
simpl.
apply boolean_or_true.
simpl.
apply boolean_or_false.
Qed.
```
And now that commutativity is proven, I guess. Am I right? 


There is something going on here which feels very odd to me. 
I will try to illustrate this by showing some alternative paths 
applied in the `boolean_or_comm` theorem. Let's step all 
the way back to where we did `case b` to split into cases followed 
by `simpl`. 

```
Theorem boolean_or_comm : forall a b : boolean , or a b = or b a.
Proof.
intros a b.
case b.
simpl.
```

At that point there are two subgoals to prove and the first is `or a true = true`
``` 
2 subgoals
a, b : boolean
______________________________________(1/2)
or a true = true
______________________________________(2/2)
or a false = or false a
```

Why was the right hand side of the equality evaluated? Why not the LHS?

If we would have done `case a` followed by `simpl` instead something different 
would have happened. 

``` 
Theorem boolean_or_comm : forall a b : boolean , or a b = or b a.
Proof.
intros a b.
case a.
simpl.
```

If we look at the goals now, the LHS of the equality in subgoal 1 has been evaluated.

```
2 subgoals
a, b : boolean
______________________________________(1/2)
true = or b true
______________________________________(2/2)
or false b = or b false
```

Why is that? 

Anyway. This was a lot of fun so far and I continued by implementing 
`and` and `not`. 

```
Definition and ( x y : boolean) : boolean := 
  match x with
    | true => y
    | false => false
  end.
```

```
Definition not (x : boolean) : boolean := 
  match x with 
    | true => false
    | false => true
  end.
```

The following theorems about `and` and `not` can be showed to hold in 
ways very similar to the examples above (If even those are valid proofs?). 

``` 
Theorem boolean_and_false : forall a : boolean, and a false = false.
Theorem boolean_and_true : forall a : boolean, and a true = a.
Theorem boolean_and_comm : forall a b : boolean, and a b = and b a.
Theorem boolean_not_true : not true = false.
Theorem boolean_not_false : not false = true.
Theorem boolean_not_not : forall a : boolean, not (not a) = a.
Theorem boolean_a_and_not_a : forall a : boolean, and a (not a) = false.
Theorem boolean_a_or_not_a : forall a : boolean, or a (not a) = true.
```

In the end I did get to those De Morgan identities:

``` 
Theorem boolean_demorgan_0 : forall a b : boolean, and (not a) (not b) = not (or a b).
Proof.
intros a b. 
case a.
simpl.
reflexivity.
simpl.
case b.
reflexivity.
reflexivity.
Qed.
``` 

And also tried one of them by using the `auto` tactic. 

``` 
Theorem boolean_demorgan_1 : forall a b : boolean, or (not a) (not b) = not (and a b).
Proof.
intros a b.
case a.
auto.
auto.
Qed.
``` 

The `auto` seems a bit magical currently. Sometimes you can apply this and it 
you will be done with that subgoal. I don't really understand why the above proof 
could not be automated before the application of the `case` tactic though. 




One last thing! And this thing may be a partial answer to one of my initial 
questions about how to do real things with Coq. There is a way to extract 
Haskell code. 

```
Extraction Language Haskell.
Extraction "./a.hs" and or not.
``` 

Adding the lines above to my experiment file and running them 
generates a `a.hs` file with the following contents. 

```
module A where

import qualified Prelude

data Boolean =
   True
 | False

or :: Boolean -> Boolean -> Boolean
or x y =
  case x of {
   True -> True;
   False -> y}

and :: Boolean -> Boolean -> Boolean
and x y =
  case x of {
   True -> y;
   False -> False}

not :: Boolean -> Boolean
not x =
  case x of {
   True -> False;
   False -> True}
``` 

That does look like something that one can work with. Pretty far from something 
you would run on a microcontroller or something like that though. 


Thanks for reading this far. If you have tips, hints or answers, please send them 
to me. If all of this is complete nonsense and I have clearly missed all the points, 
please let me know that as well (preferably together with some constructive pointers). 

Have a nice day!

___

[HOME](https://svenssonjoel.github.io)
