

# Another Lisp for Microcontrollers 

For a while now, not too long, couple of years or so, I have been
working in the vicinity of microcontrollers (MCUs). I don't have a
whole lot of experience in low-level programming of devices like the
[STM32](https://www.st.com/en/evaluation-tools/stm32-discovery-kits.html)
, the
[NRF52](https://www.nordicsemi.com/Products/Low-power-short-range-wireless/nRF52833/GetStarted)
or the ARM Cortex A9 of the Xilinx Zynq chip on the [Trenz
ZynqBerry](https://shop.trenz-electronic.de/en/TE0726-03M-ZynqBerry-Module-with-Xilinx-Zynq-7010-in-Raspberry-Pi-Form-Faktor). But
I do enjoy learning about them, slowly. The code in C, cross-compile,
flash loop (CCFL?) is a bit heavy though. Wouldn't it be nice with a
REPL (Read, Evaluate and Print Loop)?

The [MIT 6.001 Structure and Interpretation of Computer
Programs](https://www.youtube.com/watch?v=-J_xL4IGhJA&list=PLE18841CABEA24090)
series of lecturs is a lot of fun and I recommend everyone to watch
it. I watched this lecture series several times while entertaining the
idea of some day implement some kind of a Lisp. However, I didn't want to
implement a Lisp interpreter in Lisp, or even in Haskell, it would
feel a bit like cheating. 

The [lispBM](https://github.com/svenssonjoel/lispBM) project is my
(ongoing) attempt to learn some about lisp while at the same time
learn more about MCUs while making them more accessible, once you get
used to having a REPL it is hard to go back.

I don't have very much experience or long background in programming
numerous Lipses, so my attempt of making one is most likely very
naive. A small amount of dabbling with [Emacs
Lisp](https://www.gnu.org/software/emacs/manual/html_node/elisp/) and
I did like skimming [Land Of Lisp](https://nostarch.com/lisp.htm) but
navigating the jungle of Lisp dialects is not where I am at. This
means that lispBM is not going to adhere to any standard or be a real
*Scheme* or *CL*, to me that does not matter!

Now I know that there are many other MCU-lisps. I have tried not to
peek at the insides of these at all. But I looked at
[Lisperator](http://lisperator.net/pltut/) for some tips with the
evaluator, I would definitely recommend this tutorial. I have probably
stared at the tail-call optimizing evaulator at Lisperator PL-tutorial
for hours trying to understand what is going on there. I am not going
to pretend that I fully understand it yet. Early on I used the
[MPC](https://github.com/orangeduck/mpc) Micro Parser Combinators for
parsing expressions. This is the same parser generator code that is
used in [BuidYourOwnLisp](http://www.buildyourownlisp.com/). However,
when going towards smaller MCUs (with less than 256k or RAM) the
MPC library showed to be too hungry on memory and I had to hack
something up.

Here is a list of other Lisp-on-odd-hardware projects:
 1. [Let's Run Lisp on Microcontrollers](https://dmitryfrank.com/articles/lisp_on_mcu)
 2. [uLisp](http://www.ulisp.com/)
 3. [XS](http://www.yuasa.kuis.kyoto-u.ac.jp/~yuasa/xs/)

I'm sure there are many more examples! please let me know what you are working on.

If you want to see lispBM in action, check out the video [LISPBM on
NRF52](https://youtu.be/OeQ161G_Kgs) or the video that goes over what went
into porting it to NRF52 [LISPBM Porting to
NRF52](https://youtu.be/cXSavxC3th0).

LispBM is written in C, compiles with -std=c11 flag, for 32bit
platforms. So far it has been tried out on x86 (with -m32 flag and
depending on something like multilib if you are on a 64bit platform),
ARM Cortex M4 (the STM32F4 MCU), ARM Cortex M4 (the NRF52 MCU) and ARM
Cortex A9 (The Xilinx Zynq 7000). When deploying lispBM on an MCU it
helps a lot to have access to a *Hardware Abstraction Layer*, HAL. So
far lispBM has been compiled into code based on
[ChibiOs](http://chibios.org/dokuwiki/doku.php) and
[ZephyrOs](https://www.zephyrproject.org/) both providing a lot of HAL
functionality.


## A Few Thoughts on Lisp

The series of MIT video lectures showed a simple elegance and really
made me want to try making some kind of a Lisp. If you just get over
the strange "prefix" notation where `(+ 1 2)` means `(1 + 2)` and
instead focus on the really cool part that `(+ 1 2)` is actually
exactly what the expression looks like loaded into memory (the Lisp
heap) and that it extends to *N* arguments `(+ 1 2 3 4)`. When I say
that `(+ 1 2)` is exactly what it looks like in memory, what I mean is
that it is stored as a linked list where the first element is the
*symbol* `+` and the second and third elements of the list are the
*values* `1` and `2`. The elements and pointers to the next cell are
stored in what is called *cons-cells*. a cons-cell consists of enough
bytes of memory to hold two pointers or two values or some two element
permutation of pointer and value (all the details of this as it is
implemented in lispBM can be found later in this text). In more detail
then, the expression `(+ 1 2)` will in memory be made up out of a
first cons-cell containing `+` in its first position (or CAR) and a
pointed to the next cell in its second (CDR) position. Likewise, the
next cell contains the `1` in the CAR position and a pointer to the
next cell in the CDR. The last cell in the linked up structure of
cons-cells contains a `2` in CAR and a special symbol called `nil` in
the CDR position to terminate the list.

Now, if we give `(+ 1 2)` to a Lisp interpreter, it will evaluate it
and arrive at the answer `3`. The first stage in this, though, is to
read the string `(+ 1 2)` into the heap (generating the linked list of
symbols and values), this is called *Reading*. There after the lisp
interpreter will start to consume the linked list to reduce it to an
answer, called *evaluating*. Finally the result is *printed*. This is
what a *REPL* does, it reads, evaluates and prints and then it does it
all again.

If giving the lisp interpreter a list (such as `(+ 1 2)`) it will be
assumed to mean "add 1 and 2", so how does one actually create a list
*of data*? Giving the list `(1 2 3)` to the interpreter will not
work. The interpreter always treats a list given to it in this way as
an application of the *function* represented by the first element to
the rest of the elements. There is an operator that tells the
interpreter not to evaluate, this operator is written `'` as in for
example `'(1 2 3)`. This results in an identical cons-cell based
representation in memory as `(1 2 3)` does, with the only difference
that it is not evaluated. So when you give the expression `'(1 2 3)`
to the REPL, it will reply with `(1 2 3)`. We have created a list.  It
is also possible to give the expression `'(+ 1 2)` to the REPL and
this will result in a linked list in memory consisting of elements
`+`, `1` and `2`. The REPL will now give `(+ 1 2)` as the output
result of your computation. There is an operation with the reversed
meaning as well called `eval` that means "do evaluate this". As an
example the expression `(eval '(+ 1 2))` given to the REPL again
results in it printing out the answer `3`. Together `'` and `eval` are
very powerful, it means we can construct code on the fly in memory and
then have the interpreter compute the result.

In the paragraphs above there are a couple of words that I emphasized
without much explanation. These are words that (as I understand it)
are part of the Lisp vocabulary. The heap consists of cons-cells, only
large enough to hold two pointers or values (or mix there-of) so a
symbol (such as `+`) also has to be represented by a number no larger
than a pointer in number of bits. This means that values (characters,
signed or unsigned integers), symbols, pointers all can appear on
either CAR or CDR part of a cell. To tell them apart a couple of bits
are sacrificed. I will go into the details about this in the section
about the heap below.


## Heap Consisting of Cons-cells



In lispBM a cons-cell is represented by the struct:
```
typedef struct {
  VALUE car;
  VALUE cdr;
} cons_t;
```
Where *VALUE* is defined as:
```
typedef uint32_t VALUE;
```



## Symbols

## Parsing

## Printing

## Compressed Source Code

## Built in Functions

## Add Custom Functionality Using Extensions 

## Evaluating Expressions

## A Prelude of Convenient Lisp Functions

___

[HOME](https://svenssonjoel.github.io)
