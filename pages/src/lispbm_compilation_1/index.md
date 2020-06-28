

# Towards a bytecode compiler for lispBM

## Table of contents
- [Introduction](#introduction)
- [Compiler main function](#compiler-main-function)
- [Library of helper functions](#library-of-helper-functions)
  - [Helpful predicates](#helpful-predicates)
  - [Labels](#labels)
  - [Instructions and registers](#instructions-and-registers) 
  - [Sequences of instructions](#sequences-of-instructions)
  - [Preserving registers](#preserving-registers)
  - [Connecting instruction sequences](#connecting-instruction-sequences)
- [Compilers](#compilers)
  - [Compilation of self-evaluating expressions](#compilation-of-self-evaluating-expressions)
  - [Compilation of symbols](#compilation-of-symbols)
  - [Compilation of quoted expressions](#compilation-of-quoted-expressions)
  - [Compilation of definitions](#compilation-of-definitions)
  - [Compilation of lambda expressions](#compilation-of-lambda-expressions)
  - [Compilation of the progn form](#compilation-of-the-progn-form)
  - [Compilation of let expressions](#compilation-of-let-expressions)
  - [Compilation of function application expressions](#compilation-of-function-application-expressions)
- [Examples of compilation output](#examples-of-compilation-output)
- [Conclusion and the problem with symbols](#conclusion-and-the-problem-with-symbols)  
 
## Introduction

There has always been a plan to,at some point, attempt to compile
lispBM programs to some kind of bytecode. I started on an attempt a
while ago, then written in C and it was annoying. It feels like it
should be much more pleasant to write a compiler in a language that
does automatic garbage collection and where it is very easy and
natural to build structures in memory such as linked lists,
association lists and so on. The [SICP
book](https://mitpress.mit.edu/sites/default/files/sicp/full-text/book/book.html)
shows how to compile a lisp into a list of instructions, this is in
the final chapter (5.5) of the book.

So for this attempt at writing a bytecode compiler for lispBM, lispBM
is used as the implementation language as well. The compiler is very
strongly influenced by the presentation in Chapter 5.5 of SICP, but
there are differences.

This is work in progress and while the end goal is to compile to an
array of bytes, as a fist step the compiler just creates a list of
instructions in a way similar to how SICP does it. The instructions
are at a lower level compared to the description in SICP, as I see it,
and more directly amenable to bytecode translation. To translate the
list of instructions into an array of bytes is future work as is
implementing some machine to evaluate the bytecode. I will keep
writing about this as the pieces are finished. 

This lispBM compiler is the largest program written in lispBM so
far. It has been really helpful to try and write a larger program, so
many bugs have been found along the way!

As usual I much appreciate all constructive feedback and thanks a lot
for reading!


## Compiler main function

This part of the code is very similar to [SICP chapter
5.5.1](https://mitpress.mit.edu/sites/default/files/sicp/full-text/book/book-Z-H-35.html#%_sec_5.5.1)
except that lispBM does not have the `cond` form. So here the main
branching structure of the compiler is implemented as nested `if`
conditionals. Compared to SICP, there is a `let` form branch here
that, I think, is not present in SICP as they are converted into
lambdas. There are also differences when it comes to sequences of
expressions. In lispBM only the `progn` form allows sequencing of
expressions. The real differences compared to SICP are inside of the
individual compiler cases that will be shown further below. 


The compiler creates a list containing three elements. The first
element is a list of registers read within the code, the second is a
list of registers written to and the third element is a list of
instructions. Each instruction is a list of op-code and arguments. 


Below is an example that shows what the output is when compiling the
simple program `(+ 1 2)`. 

```
# (compile-instr-list '(+ 1 2) 'val 'next)

> (nil (proc argl val)
    ((movimm proc +)
     (movimm argl nil)
     (movimm val 2)
     (cons argl val)
     (movimm val 1)
     (cons argl val)
     (callf)))
```

It does not depend on values in any registers and it updates the
`proc`, `argl` and `val` registers. A list of all registers and their intended
usage will be shown later in the text. 

The compiler takes an expression as input together with a target
register and a specified linkage. The linkage is most importance when
compiler is called recursively to generate different sub-parts of the
instruction list. The linkage describes how the compiled instruction
list should end. Returning from a function application will be special
for example, but there will be more info about this in the following
sections.

The compiler looks at the passed in expression and dispatches to an
appropriate function that generates an instruction list for that case. Each of
these case specific functions may call `compile-instr-list` on subexpressions. 

```                                         
(define compile-instr-list
  (lambda (exp target linkage)
    (if (is-self-evaluating exp)
        (compile-self-evaluating exp target linkage)
      (if (is-quoted exp)
          (compile-quoted exp target linkage)
        (if (is-symbol exp)
            (compile-symbol exp target linkage)
          (if (is-def exp)
              (compile-def exp target linkage)
            (if (is-lambda exp)
                (compile-lambda exp target linkage)
              (if (is-progn exp)
                  (compile-progn (cdr exp) target linkage)
                (if (is-let exp)
                    (compile-let exp target linkage)
                  (if (is-list exp)
                      (compile-application exp target linkage)
                    (print "Not recognized")))))))))))
```


## Library of helper functions

### Predicates

The `compile-instr-list` function makes use of a number of
predicates to decide which branch to take. The implementation of these
are mostly very similar to each other. They take an expression as argument
and check if it is of some particular shape, such as it being a symbol or that it is
a list and the first element is a `lambda`. 

```
(define is-symbol
  (lambda (exp) (= type-symbol (type-of exp))))

(define is-label
  (lambda (exp) (= (car exp) 'label)))

(define is-def
  (lambda (exp) (= (car exp) 'define)))

(define is-nil
  (lambda (exp) (= exp 'nil)))

(define is-quoted
  (lambda (exp) (= (car exp) 'quote)))

(define is-lambda
  (lambda (exp) (= (car exp) 'lambda)))

(define is-let
  (lambda (exp) (= (car exp) 'let)))

(define is-progn
  (lambda (exp) (= (car exp) 'progn)))

(define is-list
  (lambda (exp) (= type-list (type-of exp))))

(define is-last-element
  (lambda (exp) (and (is-list exp) (is-nil (cdr exp)))))
``` 

Some expressions evaluate to them self, such as a number or an array. These
will be compiler into immediate values in the resulting instruction list.
The following functions are used to identify `self-evaluating` expressions.

```
(define is-number
  (lambda (exp)
    (let ((typ (type-of exp)))
      (or (= typ type-i28)
          (= typ type-u28)
          (= typ type-i32)
          (= typ type-u32)
          (= typ type-float)))))

(define is-array
  (lambda (exp)
    (= (type-of exp) type-array)))

(define is-self-evaluating
  (lambda (exp)
    (or (is-number exp)
        (is-array  exp))))
```
### Labels

The generated code will contain jumps and when at some later producing
the actual byte code, these will be jumps to absolute addresses. When
generating the instruction list, however, there will be labels with a
unique number at the place for each jump target. A later pass will
calculate what the "address" into the bytecode each label is at and
replace jumps to label with jumps to address.


A label counter keeps track of the next label identifier to generate
and a function called `mk-label` is called to generate a fresh label.

```
(define label-counter 0)

(define incr-label
  (lambda ()
    (progn
      ;; Define used to redefine label-counter
      (define label-counter (+ 1 label-counter))
      label-counter)))

(define mk-label
  (lambda (name)
    (list 'label name (incr-label))))
```

A label is a list containing the symbol `label` as first element, the
next element is a string (mostly for easier understanding of a
generated instruction list) and the third element is the label value.


### Instructions and registers 

The compiler generates code for a machine with 5 special purpose registers.

```
(define all-regs '(env
                   proc
                   val
                   argl
                   cont))
```

- **env** holds a pointer to the currently in use local environment.
- **proc** holds a procedure description. Used in function applications.
- **val** results go here.
- **argl** argument list. The actual arguments are stored on the heap, this register just holds a pointer.
- **cont** holds jump address to use when returning from a function application. 


Below is a list of instructions and their sizes in bytes (with their arguments).

```
(define instr-size
  '((jmpcnt       1)
    (jmpimm       5) 
    (jmpval       1)
    (movimm       6)
    (mov          3)
    (lookup       6) 
    (setglbval    5)
    (push         2)
    (pop          2)
    (bpf          5) 
    (exenvargl    5)
    (exenvval     5)
    (cons         3)
    (consimm      6)
    (cdr          3)
    (cadr         3)
    (caddr        3)
    (callf        1)
    (label        0)))
```

This is something that will most likely change a bit but for now and
for simplicity this is what I am working with. Using a full byte for
an op-code is a waste as there are only roughly 17 of
them. Instructions that right now take a register as argument may
later be replicated for each possible register that is can use,
turning fewer general instructions into more and specialized but
smaller in size.

A machine that can execute these instructions will also have a program
counter, `pc`, register.

The `jmpcnt` instruction can then be thought of as performing the operation: 
```
pc <- cont
```

As a more involved example, `exenvargl <symbol>` performs the following operations:
```
env <- (cons (cons <symbol> (car argl)) env)
argl <- cdr argl
``` 


### Sequences of instructions

the output of the compiler is a triple containing the list of
registers needed, the list of registers modified and the list of
instructions. This section shows a set of functions to create and
modify such triples. 

The first of these function is `mk-instr-seq` that takes tree input
and puts those three in a list. This is used to create a snippet of output code
while specifying also the registers needed and used. 

```
(define mk-instr-seq
  (lambda (needs mods stms)
    (list needs mods stms)))
```

A set of accessor functions can be used to extract information from
an instruction sequence.

```
(define regs-needed
  (lambda (s)
    (if (is-label s)
        '()
      (car s))))

(define regs-modified
  (lambda (s)
    (if (is-label s)
        '()
      (car (cdr s)))))

(define statements
  (lambda (s)
    (if (is-label s)
        (list s)
      (car (cdr (cdr s))))))
``` 

The following two functions check if a specific register is used or needed by
a instruction sequence.

```
(define needs-reg
  (lambda (s r)
    (mem r (regs-needed s))))

(define modifies-reg
  (lambda (s r)
    (mem r (regs-modified s))))
```

`mem` is the function that checks if an element is present in a list.

```
(define mem
  (lambda (x xs)
    (if (is-nil xs)
        'nil
      (if (= x (car xs))
          't
        (mem x (cdr xs))))))
```


The `instr-seq-bytes` function calculates how many bytes a compact
representation (bytecode) of an instruction sequence will require.

```
(define instr-seq-bytes
  (lambda (s)
    (let ((sum-bytes
           (lambda (x acc)
             (if (is-nil x) acc
               (sum-bytes (cdr x) (+ (lookup (car (car x)) instr-size) acc))))))
      (sum-bytes (car (cdr (cdr s))) 0))))
```

Combining two instruction sequences is a bit tricky as the resulting
combined instruction sequence must also represent registers needed and
modified correctly. Most of the code, with few exceptions, in this
section is close to identical to the presentation in SICP only
translated to the lispBM-dialect ;). 


The following functions implements appending of two instructions
sequences, appending of an arbitrary number of sequences passed in as
a list, combination of totally independent sequences of instructions
(called `parallel-instr-seqs`) and a tack-on function used to insert a
function body into the instruction sequence. 

```
(define append-two-instr-seqs
  (lambda (s1 s2)
           (mk-instr-seq
            (list-union (regs-needed s1)
                        (list-diff (regs-needed s2)
                                   (regs-modified s1)))
            (list-union (regs-modified s1)
                        (regs-modified s2))
            (append (statements s1) (statements s2))))))

(define append-instr-seqs
  (lambda (seqs)
    (if (is-nil seqs)
        empty-instr-seq
      (append-two-instr-seqs (car seqs)
                  (append-instr-seqs (cdr seqs))))))

(define tack-on-instr-seq
  (lambda (s b)
    (mk-instr-seq (regs-needed s)
                  (regs-modified s)
                  (append (statements s) (statements b)))))
  
(define parallel-instr-seqs
  (lambda (s1 s2)
    (mk-instr-seq
     (list-union (regs-needed s1)
                 (regs-needed s2))
     (list-union (regs-modified s1)
                 (regs-modified s2))
     (append (statements s1) (statements s2)))))
```

The helper functions `list-union` and `list-diff` are implemented as
follows in lispBM. Again this is just a translation from the SICP lisp
dialect to the lispBM-dialect. 

```
(define list-union
   (lambda (s1 s2)
     (if (is-nil s1) s2
       (if (mem (car s1) s2)
           (list-union (cdr s1) s2)
         (cons (car s1) (list-union (cdr s1) s2))))))

(define list-diff
  (lambda (s1 s2)
    (if (is-nil s1) '()
      (if (mem (car s1) s2) (list-diff (cdr s1) s2)
        (cons (car s1) (list-diff (cdr s1) s2))))))
```

### Preserving registers

When generating code for things like nested function applications (for
example `(+ 1 (+ 2 3) 4)`, there is a need for preserving registers
that are used on both levels. The `preserving` function combines two
instruction sequences while automatically inserting `push` and `pop` of
the registers specified around the first of the sequences. 

```
(define preserving
  (lambda (regs s1 s2)
    (if (is-nil regs)
        (append-instr-seqs (list s1 s2))
      (let ((first-reg (car regs)))
        (if (and (needs-reg s2 first-reg)
                 (modifies-reg s1 first-reg))
            (preserving (cdr regs)
                        (mk-instr-seq
                         (list-union (list first-reg)
                                     (regs-needed s1))
                         (list-diff (regs-modified s1)
                                    (list first-reg))
                         (append `((push ,first-reg))
                                 (append (statements s1) `((pop ,first-reg)))))
                        s2)

          (preserving (cdr regs) s1 s2))))))
```

### Connecting instruction sequences

There are two different kinds of ways that a sequence of instructions
can end.  an instruction sequence that represents a function body ends
with a jump to an address stored in the `cont` registers. Other
instruction sequences just flow naturally into each other.

The `end-with-linkage` function inserts the appropriate code depending
on an argument called `linkage` that is passed as an argument. If this
`linkage` is `'return` code that returns from a function call is
inserted and if the `linkage` is `next` nothing is added at all.

The linkage can also be a label. In this case code that jumps to that
label is inserted.


```
(define empty-instr-seq (mk-instr-seq '() '() '()))

(define compile-linkage
  (lambda (linkage)
    (if (= linkage 'return)
        ;; jmpcnt implies usage of cont register
        (mk-instr-seq '(cont) '() '((jmpcnt)))
      (if (= linkage 'next)
          empty-instr-seq
        (mk-instr-seq '() '() `((jmpimm ,linkage)))))))

(define end-with-linkage
  (lambda (linkage seq)
    (preserving '(cont)
                seq
                (compile-linkage linkage))))

```


## Compilers 

Ok! All up to this point is just plumbing. Now we cget to write
compilers for the different kinds of language constructs. This part
differs more from the SICP implementation than previous sections but
still is quite similar. The differences come from lispBM having a bit
different set of special forms compared to the SICP dialect. Another
difference is the "abstraction level" of the generated instructions,
here the aim is at slightly lower level.

The different `compile-X` functions take three arguments, the
expression to compile, the register to store the result in and a
linkage. The output is an instruction sequence. 


### Compilation of self-evaluating expressions

An expression is considered self evaluating if it is a number or an
array.  In this case the expression is stored into the target
registers using a move immediate instruction. 

```
(define compile-self-evaluating
  (lambda (exp target linkage)
    (end-with-linkage linkage
                      (mk-instr-seq '()
                                    (list target)
                                    `((movimm ,target ,exp))))))
```

The idea is that this can be turned into bytecode consisting of one
byte for instruction, one byte for target register and 4 bytes for the
value.  Since we have very few registers and instructions using a full
byte for each is a huge waste. But it is ok to be working in this way
for now. Later a more compact representation should be used. I don't
want to do this "optimization" too early, before knowing exactly what
other kinds of instructions or registers that may be added later.


### Compilation of symbols

Compilation of a symbol generates code that performs a lookup in the environment
and stores the result of that lookup in a target register. 

``` 
(define compile-symbol
  (lambda (exp target linkage)
    (end-with-linkage linkage
                      ;; Implied that the lookup looks in env register
                      (mk-instr-seq '(env)
                                    (list target)
                                    `((lookup ,target ,exp))))))
```				      

### Compilation of quoted expressions

Now it gets a bit complicated and also very different from how SICP
does it.  In SICP, a quoted expression doesn't seem to be compiled at
all (as I understand it).  Rather the expression is just "pointed to"
from the compiled instructions sequence.  I do not see how this would
allow that the compiled result is stored and loaded into a fresh
runtime system. Being able to load compiled source into a freshly
started RTS is one of the goals with the compiler implemented
here. There are Other problems with generating code that can be loaded
into a fresh RTS, I will try to explain these in the closing section. 

So, to be able to load in the compiled code that contains quoted
expressions, my feeling is that the code mush reconstruct the quoted
expression on the heap. This is my first attempt at this compilation
and it has not been extensively tested, so there surely are buggy
corner cases.

`compile-quoted` passes the expression following the `quote` symbol to
a function called `compile-data` that traverses the heap structure
while generating code that recreates that same structure.

`compile-data` calls either `compile-data-list` or `compile-data-prim`
depending on the type of subexpressions (if they are lists or
not). `compile-data-list` calls `compile-data-nest` that then forms a
mutually recursive loop. `compile-data-nest takes care of the
preservation of registers needed when dealing with nested
structures. The `preserving` function is not used, but it is likely
that the code could be reimplemented using it. 

```
(define compile-quoted
  (lambda (exp target linkage)
    (end-with-linkage linkage
                      (compile-data (car (cdr exp)) target))))

(define compile-data
  (lambda (exp target)
    (if (is-list exp)
        (append-two-instr-seqs
         (mk-instr-seq '() (list target)
                       `((movimm ,target nil)))
         (compile-data-list (reverse exp) target))
      (compile-data-prim exp target))))

(define compile-data-nest
  (lambda (exp target)
    (if (is-list exp)
        (append-two-instr-seqs
         (mk-instr-seq (list target) '()
                       `((push ,target)
                         (movimm ,target nil)))
         (append-two-instr-seqs
          (compile-data-list (reverse exp) target)
          (mk-instr-seq (list target) (list target 'argl)
                        `((mov argl ,target)
                          (pop ,target)
                          (cons ,target argl)))))
      (append-two-instr-seqs
       (compile-data-prim exp 'argl)
       (mk-instr-seq '(argl) (list target)
                     `((cons ,target argl)))))))

(define compile-data-prim
  (lambda (exp target)
    (mk-instr-seq '() (list target)
                  `((movimm ,target ,exp)))))

(define compile-data-list
  (lambda (exp target)
    (if (is-nil exp)
        empty-instr-seq
      (append-instr-seqs
       (map (lambda (e)
              (compile-data-nest e target))
            exp)))))
```


### Compilation of definitions

Definitions are compiled into code that updates the global environment. 

```
(define compile-def
  (lambda (exp target linkage)
     (let ((var (car (cdr exp)))
           (get-value-code
            (compile-instr-list (car (cdr (cdr exp))) 'val 'next)))
       (end-with-linkage linkage
                         (append-two-instr-seqs get-value-code
                                                (mk-instr-seq '(val) (list target)
                                                              `((setglbval ,var)
                                                                (movimm ,target ,var))))))))
```

The generated code uses `setglbval` to update the environment. It can
be thought of as performing the following operations.

```
global-env <- (cons (cons (var val)) global-env)
```

The target register will hold the bound variable after executing the
generated sequence.

### Compilation of lambda expressions

Lambda expressions are also slightly different in lispBM. The function is just a single
expression and cannot be a list of expressions as in the SICP dialect.

Lambdas are compiled into code that creates a procedure "object" and a
function body.  The procedure object is the result of executing the
lambda. This procedure can then later be called, which should result
in a jump to the function body.

The code that generates the procedure object and the function body are
stored consecutively in the instruction stream.

A procedure object is a list with first element `'proc`, second
element is a jump label and third element is an environment. This is
very similar to how in the evaluator a lambda is evaluated into a closure that
can be called later.

```
(define compile-lambda
  (lambda (exp target linkage)
    (let ((proc-entry    (mk-label "entry"))
          (after-lambda  (mk-label "after-lambda"))
          (lambda-linkage (if (= linkage 'next) after-lambda linkage)))
      (append-two-instr-seqs
       (tack-on-instr-seq
        (end-with-linkage lambda-linkage
                          (mk-instr-seq '(env) (list target)
                                        `((movimm ,target nil)
                                          (cons ,target env)
                                          (consimm ,target ,proc-entry)
                                          (consimm ,target proc)))) ;; put symbol proc first in list
        (compile-lambda-body exp proc-entry))
       after-lambda))))

(define compile-lambda-body
  (lambda (exp proc-entry)   
    (let ((formals (car (cdr exp))))
      (append-two-instr-seqs
       (append-two-instr-seqs 
        (mk-instr-seq '(env proc argl) '(env)
                      `(,proc-entry
                        (caddr env proc)))
        (append-instr-seqs
         (map (lambda (p)
                (mk-instr-seq '(argl) '(env)
                              `((exenvargl ,p))))
              formals)))
       (compile-instr-list (car (cdr (cdr exp))) 'val 'return)))))

```

### Compilation of the progn form

The only way to write a program that sequentially executes a number of
expressions in lispBM is the `progn` form. Oh, actually a top level
program is also a list of sequential expressions. But in a function, you need to use `progn`.

The code that is generated for the sequence of expressions insert
register preserving pushes and pulls. To allow `progn` to be used in a
function body of a recursive function and getting tail-recursion, the
last expression in the list must be surrounded by these pushes and
pulls. This is why there is a special case for the last element. 

```
(define compile-progn
  (lambda (exp target linkage)
    (if (is-last-element exp)
        (compile-instr-list (car exp) target linkage)
      (preserving '(env continue)
                  (compile-instr-list (car exp) target 'next)
                  (compile-progn (cdr exp) target linkage)))))
```


### Compilation of let expressions

This is a first go at compiling let expressions. I think it will not allow
the definition of mutual recursion as it stands now. Must fix this.

The generated code for a let expression will contain an `evenvval`
instruction per variable to bind. Between these instructions there will
be some sequence generated by `compile-instr-list` that each output
their result into the `val` register. 

```
(define compile-let
  (lambda (exp target linkage)
    (append-two-instr-seqs
     (append-instr-seqs (map compile-binding (car (cdr exp))))
     (compile-instr-list (car (cdr (cdr exp))) target linkage))))

(define compile-binding
  (lambda (keyval)
    (let ((get-value-code
           (compile-instr-list (car (cdr keyval)) 'val 'next))
          (var (car keyval)))
      (append-two-instr-seqs get-value-code                          
                             (mk-instr-seq '(val) (list 'env)
                                           `((exenvval ,var)))))))
```

### Compilation of function application expressions

Function application is compiled by first compiling all of the
arguments.  The arguments are all appended to a list created in the
`argl` register (the actual arguments are stored as a list on the
heap, a pointer to the list is in argl).

```
(define compile-application
  (lambda (exp target linkage)
    (let ((proc-code
	   (if (is-fundamental (car exp))
	       (mk-instr-seq '() '(proc)
			     `((movimm proc ,(car exp))))
	     (compile-instr-list (car exp) 'proc 'next)))
	  (operand-codes (map (lambda (o) (compile-instr-list o 'val 'next)) (cdr exp))))
      (preserving '(env cont)
      		  proc-code
      		  (preserving '(proc cont)
      			      (construct-arglist operand-codes)
			      (if (is-fundamental (car exp))
				  (compile-fundamental-proc-call target linkage)
				(if (is-lambda (car exp))
				    (compile-lambda-proc-call target linkage)
				  (compile-proc-call target linkage))))))))

``` 

```
(define add-fst-argument
  (lambda (arg-code)
    (append-two-instr-seqs
     arg-code
     (mk-instr-seq '(val) '(argl)
		   '((movimm argl ())
		     (cons argl val))))))
	  

(define construct-arglist
  (lambda (codes)
    (let ((operand-codes (reverse codes)))
      (if (is-nil operand-codes)
	  (mk-instr-seq '() '(argl)
			'((movimm argl ())))
	(if (is-nil (cdr operand-codes))
	    (add-fst-argument (car operand-codes))
	  (preserving '(env)
		      (add-fst-argument (car operand-codes))
		      (get-rest-args (cdr operand-codes))))))))

(define get-rest-args
  (lambda (operand-codes)
    (let ((code-for-next-arg
	   (preserving '(argl)
		       (car operand-codes)
		       (mk-instr-seq '(val argl) '(argl)
				     '((cons argl val))))))
      (if (is-nil (cdr operand-codes))
	  code-for-next-arg
	(preserving '(env)
		    code-for-next-arg
		    (get-rest-args (cdr operand-codes)))))))
```


As an optimization, to generate small code when it is known that
the procedure is a fundamental or the result of lambda, the
specific `compile-fundamental-proc-call` or `compile-lambda-proc-call`
functions can be used. When it is unknown what type of procedure
is at hand, the general ´compile-proc-call` function is used.

The general function inserts code that checks what kind of procedure
is going to be called and then branches either to a fundamental or a
lambda case.

```
(define compile-fundamental-proc-call
  (lambda (target linkage)
      (end-with-linkage linkage
			(mk-instr-seq '(proc argl)
				      (list target)
				      '((callf))))))

(define compile-lambda-proc-call
  (lambda (target linkage)
    (let ((after-call       (mk-label "after-call"))
	  (compiled-linkage (if (= linkage 'next)
				after-call
			      linkage)))
      (append-two-instr-seqs
       (compile-proc-appl target compiled-linkage)
       after-call))))


(define compile-proc-call
  (lambda (target linkage)
    (let ((fund-branch      (mk-label "fund-branch"))
	  (compiled-branch  (mk-label "comp-branch"))
	  (after-call       (mk-label "after-call"))
	  (compiled-linkage (if (= linkage 'next)
				after-call
			      linkage)))
      (append-instr-seqs
       (list (mk-instr-seq '(proc) '()
			     `((bpf ,fund-branch)))
	     (parallel-instr-seqs
	      (append-two-instr-seqs
	       compiled-branch
	       (compile-proc-appl target compiled-linkage))
	      (append-two-instr-seqs
	       fund-branch
	       (end-with-linkage linkage
				 (mk-instr-seq '(proc argl)
					       (list target)
					       '((callf))))))
	     after-call)))))
```

Jumping to and how to return from a procedure call is implemented in
function below. It is split up into cases depending on target register
and linkage type.

If linkage is a label and target is the val register, then code
is inserted that sets the cont register to the label from the linkage,
the function body label is extracted from the procedure object and then a jump
instruction transfers execution to that label.

The other cases are slight modifications to this flow. 

```
(define compile-proc-appl
  (lambda (target linkage)
    (if (and (= target 'val) (not (= linkage 'return)))
	(mk-instr-seq '(proc) all-regs
		      `((movimm cont ,linkage)
			(cadr val proc)
			(jmpval)))
      (if (and (not (= target 'val))
	       (not (= linkage 'return)))
	  (let ((proc-return (mk-label "proc-return")))
	    (mk-instr-seq '(proc) all-regs
			  `((movimm cont ,proc-return)
			    (cadr val proc)
			    (jmpval)
			    ,proc-return
			    (mov ,target val)
			    (jmpimm ,linkage))))
	(if (and (= target 'val)
		 (= linkage 'return))
	    (mk-instr-seq '(proc continue) all-regs
			  '((cadr val proc)
			    (jmpval)))
	  'compile-error)))))
```

## Examples of compilation output


First example is an application of a fundamental operation. This
generates code that sets up the argument list in `argl` and then
issues `callf` that will execute the fundamental operation stored in
the `proc` register.


```
# (compile-instr-list '(+ 1 2) 'val 'next)

> (nil (proc argl val)
       ((movimm proc +)
        (movimm val 2)
        (movimm argl nil)
        (cons argl val)
        (movimm val 1)
        (cons argl val)
        (callf)))
```

The next example is an application of a lambda, tis results in code
that creates a procedure object in the `proc` register and a lambda
function body.  After setting up the procedure object, control is
transfered to the label that is directly after the function
body. Here, the arguments are added to the `argl` list, `cont` is set
to a label to return to after the function application and a jump to
the function body is performed.

```
# (compile-instr-list '((lambda (x y) y) 10 20) 'val 'next)

> ((env) (env proc val argl cont)
         ((movimm proc nil)
          (cons proc env)
          (consimm proc (label "entry" 1))
          (consimm proc proc)
          (jmpimm (label "after-lambda" 2))
          (label "entry" 1)
          (caddr env proc)
          (exenvargl x)
          (exenvargl y)
          (lookup val y)
          (jmpcnt)
          (label "after-lambda" 2)
          (movimm val 20)
          (movimm argl nil)
          (cons argl val)
          (movimm val 10)
          (cons argl val)
          (movimm cont (label "after-call" 3))
          (cadr val proc)
          (jmpval)
          (label "after-call" 3)))
```


The last example is the result of compiling a quoted expression.  This
results in code that recreates the quoted expression on the heap and
points to this structure from the `val` register.  Here `argl` is used
as a temporary register while building the data structure.

```
# (compile-instr-list ''(+ 1 2) 'val 'next)

> (nil (argl val)
       ((movimm val nil)
        (movimm argl 2)
        (cons val argl)
        (movimm argl 1)
        (cons val argl)
        (movimm argl +)
        (cons val argl)))
```

## Conclusion and the problem with symbols

Since the goal is to be able to generate code that can be saved to disk and then
later loaded into a fresh RTS, a problem with symbols presents itself.

Say I load the lisp source containing the code `(define apa 10)` into
lispBM, then a mapping is created between the text string "apa" and a
symbol identifier in the symbol table. The actual heap representation
does not contain these symbol-strings, rather just a value that via
the symbol table is mapped to a name. So the internal representation
of `(define apa 10) could be something like ([431431] [342432] 10)
where the [] illustrates that these are numbers of a different type to
the number 10.

The code generated from the compiler, as it is currently implemented,
will only contain the numerical symbol values. This means that if
compiled code is loaded into a fresh RTS, no binding between the
numerical symbol identifier and the string "apa" will exist.  This
will be a problem if the compiled code is supposed to return a symbol
and that symbol should be presented to the user. There would be no way
to do that!

So some way to store the symbol mapping into the generated code must
be designed. A first thought here is to store a symbol renumbering
mapping in the code. This mapping would contain string names of
symbols and the value it has in the code. Then when loading the code
the symbols will be registered with the RTS and the values provided
from the RTS for the symbol should be substituted into the code. This
has a problem though. If the there are two symbols in the code "apa"
and "bepa" with ID 42 and 43, lets say when loading this into the RTS
the renumbering for apa is 43 so we replace all 42 with 43 in the
code. Then "bepa" is given renumbering 768, so we replace all 43s with
768s in the code. Now there are no apas in the code, only bepas...

I currently plan to give symbols in the code IDs pulled from a completely
different type compared to symbol ids. I call this type symbol-indirection

So the compiler should be augmented with functions that find symbols
used (that are not special always present symbols at unique ids, such
as `+`, `define` and so on) and renames it using a symbol-indirection
value. A mapping of "name" to symbol-indirection-value will be created
and stored into the code. This should rule out that problem with
renaming where a name ends up overwritten since the replacement will
be into values of a completely different type. 

This is by far the largest program written in lispBM so far and of
course many bugs have been found while writing it. I'm sure there are
more bugs lurking. 

Thanks for reading! As usual I would be very thankful for helpful
feedback.

**Todos** 
- The `mk-instr-seq` could be rewritten to derive the sets of used and modified registers
  from the instruction sequence. 

___

[HOME](https://svenssonjoel.github.io)
