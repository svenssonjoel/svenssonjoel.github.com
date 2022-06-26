

# Defunctionalization of a Continuation-Passing Style Evaluator

LispBM evaluates expressions in a way that is strongly influenced by
the [lisperator](https://lisperator.net/pltut/cps-evaluator/)
blog. The evaluator described at lisperator.net is written in
so-called continuation passing style (CPS) and implemented in java
script (if my memory does not fail me). Java script supports higher
order functions (HOFs) which makes CPS more pleasant to work with
(it's never entirely pleasant, at least not for me, it feels like a
very backwards and roundabout way to think). LispBM is implemented in
C, so there are no nice HOFs to work with. Some workaround is needed.

Defunctionalization is a transformation one can apply to code that
uses HOFs resulting in new code with no HOFs, a first order program.
The defunctionalization transformation can be automated, for example
as part of a compiler, but in this text it will be used as a manual
transformation of a higher order evaluator into a first order
evaluator. The resulting first order evaluator can then be used
as a recipe for the implementation of the same thing in C.

For more information about defunctionalization see
[Defunctionalization at
Work](https://www.brics.dk/RS/01/23/BRICS-RS-01-23.pdf) by Danvy and
Nielsen or [Definitional interpreters for higher-order programming
languages](https://dl.acm.org/doi/10.1145/800194.805852) by John
C. Reynolds. Also see the blog post [What is
Defunctionalization](https://www.geeksforgeeks.org/what-is-defunctionalization/).

Defunctionalization transforms lambdas (functions) into data together
with a function for interpretation of that data. Let's look at those
details when we start applying the method ;) 


## Implementation of a Mini-Lisp Evaluator in LispBM 

The example used here will be a miniature lisp implemented in continuation
passing style in LispBM. The mini-lisp will have a small set of operations `+`,`-`,`*` and `=`,
it will allow definition in a global environment using `define`, sequencing
of operations using `progn`, conditionals using `if` and allow function abstraction using  `lambdas`.
Evaluation of a lambda results in a closure.

The mini-lisp evaluation function will be called `evalk` and takes 3
arguments, `env`, `exp` and `k`. The `env` argument maintains local environments
and will be used when evaluating inside of a closure application, `exp` is
the expression to evaluate and `k` is the continuation. Using `evalk` to evaluate
an expression looks as follows in the LispBM REPL:

```lisp
# (evalk nil '(+ 1 2) done)
> 3

```

The `env` argument will most likely be `nil` (empty) when starting
evaluation and the continuation will be `done`. The `done` function is
applied to the result of evaluating `(+ 1 2)` as the very last step of
evaluation and is implemented as the identity function. `evalk` calls
itself recursively and those calls may have way more interesting `env`
and `k` arguments.

Let's jump right in, start thinking upside down, backwards and inside-out
and implement the `evalk` function.

```
(defun evalk (env exp k)
  (if (is-operator exp)
      (k exp)
    (if (is-symbol exp)
        (let ((res (assoc env exp)))
          (if (eq res nil)
              (k (assoc global-env exp))
            (k res)))
      (if (is-number exp)
          (k exp)
        (match exp
               ((progn  . (? ls)) (eval-progn  env ls k))
               ((define . (? ls)) (eval-define env ls k))
               ((lambda . (? ls)) (eval-lambda env ls k))
               ((if . (? ls))     (eval-if env ls k))
               ((?cons ls)        (eval-list env ls nil
                                             (lambda
                                               (rs)
                                               (apply env rs k))))
               )))))
```

The `evalk` function has 4 main cases. If the expression we evaluate is an
operator (one or `+`, `-`, `*` and `=`) there is no more evaluation we do to
that, so apply the continuation. What it means to apply the continuation to the
operator is quite unclear for now. Most likely an operator will appear as
the first element in a list (that is how lisps work), so the continuation `k`
in this case will likely be something related to evaluation of lists or expressions
 which is the lisp way of expression application. So eventually following the
 `k` continuation will end up applying the operator to some arguments.

The next case checks if `exp` is a symbol. In this case we check if
there is a value bound to the symbol in either the global or the local
environment and then apply the continuation to the result of looking
the symbol up in the environment). For example, `(+ a 2)` should
evaluate to the sum of whatever number a is associated with in the
environment and 2.

Next is another case where there is not much to do. If `exp` is a
number, apply the continuation to that number.


The rest of the `evalk` function is more interesting. This part is expressed
using pattern matching on `exp` and `exp` is assumed to be a list .

``` 
        (match exp
               ((progn  . (? ls)) (eval-progn  env ls k))
               ((define . (? ls)) (eval-define env ls k))
               ((lambda . (? ls)) (eval-lambda env ls k))
               ((if . (? ls))     (eval-if env ls k))
               ((?cons ls)        (eval-list env ls nil
                                             (lambda
                                               (rs)
                                               (apply env rs k))))
               )))))
```

`(progn . (? ls))` is a pattern that matches lists where the symbol
progn is the first element. The rest of the list is bound to `ls`. So
if `exp` is `(progn (+ 1 2) (+ 3 4))` then `ls` is `((+ 1 2) (+ 3
4))`. If there is a match here `eval-progn` is called with arguments
`env`, `ls` and `k`.  The `define`, `lambda` and `if` cases are all
expressed the same.

The fifth case `(?cons ls)` matches any list and binds that list to
`ls`.  In this final case `eval-list` is called and now the
continuation is more interesting. The continuation passed to
`eval-list` takes one argument called `rs` and this argument will be a
list with the results of evaluating all of the expressions in
`ls`. So, when this continuation is invoked, each element of the list
will have been evaluated and as a list in lisp means application, we
can now proceed with the application of the first element of the list
to the rest of the list. This is done using the `apply` function.

There are a lot of functions called from `evalk` that are yet to be
defined. Let's take a look at `apply` first.

```lisp
(defun apply (env ls k)
  (let ((f (car ls)))
    (if (is-operator f)
        (k (eval ls))
      (if (is-closure f)
          (apply-closure env ls k)
        'error))))
```

`apply` starts out with calling the head of the input list `ls` `f` as
we are going to assume it is a function of some kind. `f` can be
either one of the built in operators or it can be a closure. If `f` is
an operator we cheat here and calls LispBM's `eval` function to
evaluate the operator application.  That is what will happen in case
the list is for example `(+ 1 2)`.  If the `f` is a closure we instead
call an `apply-closure` function to deal with the list `ls`. Closures
are created when a `lambda` is evaluated so let's return to
`apply-closure` later.

Let's look at `eval-progn`, `eval-define`, `eval-lambda`, `eval-if`
and `eval-list` now. All of these functions each take 3 arguments, an
environment, a list of arguments and the continuation, except
`eval-list` that takes an additional argument to accumulate an
evaluated list into.

```lisp
(defun eval-progn (env args k)
  (match args
         (nil (k nil))
         (((? l) . nil) (evalk env l k))
         (((? l) . (? ls))
          (evalk env l
                 (lambda (x)
                   (eval-progn env ls k)))))
  )
```

`eval-progn` is implemented using pattern matching on three cases, the
empty list `nil`, a list with exactly one element `((? l) . nil)` and
a general non-empty list case `((? l) . (? ls))`.

If the list of arguments is empty then someone wrote the program
`(progn)` which doesn't really do very much. In this case we consider
the result to be `nil` and apply the continuation to `nil`.

If There is just one element in the list the evaluator `evalk` is
called on this element.

in the general list case (that here will match for lists with at least
2 elements) calls `evalk` with the first element in this list and a
continuation that again goes back to the `eval-progn` function to
continue evaluating the sequence of expressions. `eval-progn` is
recursive in a roundabout way via the continuation passed to `evalk`.

Note that `progn-cont` ignores the `x` argument. This is related to
what `progn` means in a user program. `progn` evaluates a sequence of
expression and throws away the results (except the result of the last
expression which is the result of the entire `progn` block). The
result of the previous expression in the `progn` sequence is exactly
what you find in the `x` argument.



Next up is `eval-define` that adds a binding to a global environment.
```lisp
(define global-env 'nil)

(defun eval-define (env args k)
  (let ((key (car args))
        (val (car (cdr args))))
    (evalk env val
           (lambda (x)
             (progn
               (setvar 'global-env
                       (acons key x global-env))
               (k x))))))
``` 

The arguments list passed to `eval-define` should contain one symbol
and one expression. The expression will be evaluated and the value
will be associated with the symbol in the environment. `eval-define`
starts out with renaming these two elements from the argument list to
`key` and `val`.

The `val` expression needs to be evaluated so `evalk` is called on it
and is passed a continuation that updates the global environment with
the value of `val` bound to the key `key`. Note that the argument `x`
in continuation function will is the result of evaluating `val` at the
time this continuation gets invoked.

The continuation itself finally applies the original continuation `k`
to `x`, the evaluated version of `val`. This means that the result of
a `define` expression is considered to be `val` or concretely that the
result of `(define apa (+ 2 3))` is `5` and a side effect of the call
is that the global env contains the mapping `apa` is 5.

For example: 
```lisp
# (evalk nil '(define apa (+ 2 3)) done)
> (+ 2 3)
# global-env
> ((apa . 5))
# 
```


Functions, lambdas, are evaluated into closures in the `eval-lambda`
function.  A closure is very similar to a lambda but also contains an
environment of bindings that were present at the place the lambda was
created.

For example in `(lambda (x) (lambda (y) (+ x y)))` the `x` is a free
variable from the point of the inner most lambda. When this inner
lambda is converted into a closure, it needs an environment to tell it
what the value of `x` is. In this interpreter, we just take the entire
local environment and stick it into the closure.

```lisp
(defun eval-lambda (env args k)
  (k (append (cons 'closure args) (list env))))
```

If the expression given to `evalk` was `(lambda (x) (+ x 1))` then the
`args` list given to `eval-lambda` will be `((x) (+ x 1))`. What
`eval-lambda` does is stick the symbol `closure` first in this list
and then also adds the local environment to the end of the list
resulting in `(closure (x) (+ x 1) env)`.  The continuation is then
applied to the closure.

Now that we know how closures are made we can take a look at closure
application that was left out earlier.

The `apply-closure` function is called from `apply` in the case where
the first element of a list is a closure.  So the expression given to
apply closure is of the form `((closure parameters body env) arg1
... argn)`

```
(defun apply-closure (env ls k)
  (let ((clo  (car ls))
        (args (cdr ls))
        (ps (car (cdr clo)))
        (body (car (cdr (cdr clo))))
        (env1 (car (cdr (cdr (cdr clo)))))
        (arg-env (zip ps args))
        (new-env (add-bindings (append env1 env) arg-env)))
    (evalk new-env body k)))
```

The `apply-closure` function unpacks the input list and names the
different parts. The next step is to create a new environment
augmented with the parameters bound to the arguments `arg1` ... `argn`
together with the environment from the closure. The body is then
evaluated in this newly constructed environment.


Conditionals are of the form `(if cond-exp then-exp else-exp)`, so the
`args` argument given to `eval-if` is a list of 3 elements.

```lisp
(defun eval-if (env args k)
  (let ((cond-exp  (car args))
        (then-branch (car (cdr args)))
        (else-branch (car (cdr (cdr args)))))
    (evalk env cond-exp
           (lambda (x) (if x
                           (evalk env then-branch k)
                           (evalk env else-branch k))))))
```

The three elements of the list are given the names `cond-exp`,
`then-exp` and `else-branch` as a first step in `eval-if`. Then
`evalk` is called to evaluate the `cond-exp`.  The continuation passed
to `evalk` will, when it is invoked, either continue by evaluating the
then or the else branch.



`eval-list` is just like `eval-progn` recursive via the continuation
it creates and passes to `evalk`. Unlike `progn` `eval-list` needs to
keep track of all of the results along this recursion.

```lisp
(defun eval-list (env ls acc k)
  (if (eq ls nil)
      (k acc)
    (let (( l (car ls))
          ( r (cdr ls)))
      (evalk env l
             (lambda (x)
               (eval-list env r (append acc (list x)) k))))))
```

In `eval-list` `ls` is the list to evaluate and `acc` is where results
are accumulated.  If the `ls` list is empty we are done and can apply
the continuation to `acc`.  Otherwise, the first element of `ls`
should be evaluated by `evalk` and the continuation will continue with
another call to `eval-list` with the result of evaluating the first
element appended to the the `acc` list.


Below you find the complete listing including all little helper
functions that where not mentioned above (such as `is-symbol` and
`is-number`).

```lisp
(define global-env 'nil)

(defun is-number (e)
  (or (eq (type-of e) type-i)
      (eq (type-of e) type-u)))

(defun is-symbol (e)
  (eq (type-of e) type-symbol))

(defun is-operator (e)
  (or (eq e '+)
      (eq e '-)
      (eq e '=)
      (eq e '*)
      ))

(defun is-closure (e)
  (and (eq (type-of e) type-list)
       (eq (car e) 'closure)))

(defun add-bindings (env binds)
  (match binds
         (nil env)
         (((? b) . (? rs))
          (add-bindings (setassoc env b) rs))))

(defun done (e)
  e)

(defun eval-progn (env args k)
  (match args
         (nil (k nil))
         (((? l) . nil) (evalk env l k))
         (((? l) . (? ls))
          (evalk env l
                 (lambda (x)
                   (eval-progn env ls k)))))
  )

(defun eval-define (env args k)
  (let ((key (car args))
        (val (car (cdr args))))
    (evalk env val
           (lambda (x)
             (progn
               (setvar 'global-env
                       (acons key x global-env))
               (k val))))))

(defun eval-lambda (env args k)
  (k (append (cons 'closure args) (list env))))

(defun eval-if (env args k)
  (let ((cond-exp  (car args))
        (then-branch (car (cdr args)))
        (else-branch (car (cdr (cdr args)))))
    (evalk env cond-exp
           (lambda (x) (if x
                           (evalk env then-branch k)
                           (evalk env else-branch k))))))

(defun eval-list (env ls acc k)
  (if (eq ls nil)
      (k acc)
    (let (( l (car ls))
          ( r (cdr ls)))
      (evalk env l
             (lambda (x)
               (eval-list env r (append acc (list x)) k))))))

(defun apply-closure (env ls k)
  (let ((clo  (car ls))
        (args (cdr ls))
        (ps (car (cdr clo)))
        (body (car (cdr (cdr clo))))
        (env1 (car (cdr (cdr (cdr clo)))))
        (arg-env (zip ps args))
        (new-env (add-bindings (append env1 env) arg-env)))
    (evalk new-env body k)))

(defun apply (env ls k)
  (let ((f (car ls)))
    (if (is-operator f)
        (k (eval ls))
      (if (is-closure f)
          (apply-closure env ls k)
        'error))))

(defun evalk (env exp k)
  (if (is-operator exp)
      (k exp)
    (if (is-symbol exp)
        (let ((res (assoc env exp)))
          (if (eq res nil)
              (k (assoc global-env exp))
            (k res)))
      (if (is-number exp)
          (k exp)
        (match exp
               ((progn  . (? ls)) (eval-progn  env ls k))
               ((define . (? ls)) (eval-define env ls k))
               ((lambda . (? ls)) (eval-lambda env ls k))
               ((if . (? ls))     (eval-if env ls k))
               ((?cons ls)        (eval-list env ls nil
                                             (lambda
                                               (rs)
                                               (apply env rs k))))
               )))))
```


## Defunctionalizing the Evaluator

The defunctionalized version of the evaluator is called `evald` and
it also takes three arguments, `env`, `exp` and `k`. The `k` argument
is however not a function anymore, here `k` is a data-structure. 

`evald` and `evalk` are very similar, but note that where `evalk` does
`(k exp)`, `evald` instead does `apply-cont k exp`. Here `apply-cont
is a kind of evaluator for the continuation data-structure.

```lisp
(defun evald (env exp k)
  (if (is-operator exp)
      (apply-cont k exp)
    (if (is-symbol exp)
        (let ((res (assoc env exp)))
          (if (eq res nil)
              (apply-cont k (assoc global-env exp))
            (apply-cont k res)))
      (if (is-number exp)
          (apply-cont k exp)
        (match exp
               ((progn  . (? ls)) (eval-progn  env ls k))
               ((define . (? ls)) (eval-define env ls k))
               ((lambda . (? ls)) (eval-lambda env ls k))
               ((if . (? ls))     (eval-if env ls k))
               ((?cons ls)        (eval-list env ls nil
                                             (list 'application-cont env k)))
               )))))
```

One last difference between `evalk` and the defunctionalized `evald`
is in the `(?cons ls)` case, where eval-list is called. The
continuation argument to `eval-list` is now a list containing the
symbol `application-cont` an environment and the input continuation
`k`. The list `(application-cont env k)` is data that should later be evaluated
by the `apply-cont` function. So we need to add a first case to that function
to handle lists where the first element is `application-cont`:

```lisp
(defun apply-cont (k exp)
  (match k
         ((application-cont (? env) (? k1))
          (apply env exp k1))))
```

If the `application-cont` case matches, the apply function is applied. This
function is very similar to before but where the old one applies continuation
functions, this one must interpreter continuation data, notice that `apply` calls
`apply-cont` to interpret continuation objects.

```lisp
(defun apply (env ls)
   (let ((f (car ls)))
     (if (is-operator f)
         (apply-cont (eval ls))
         (if (is-closure f)
             (apply-closure env ls)
             'error))))
```

`eval-progn`, `eval-define`, `eval-lambda`, `eval-if` and `eval-list`
must be given new implementations to work on continuation data
now, rather than continuation functions.

Let's look at how defunctionalization works by comparing the previous
implementation of `eval-progn` with the transformed one.

***Original: eval-progn***
```lisp
(defun eval-progn (env args k)
  (match args
         (nil (k nil))
         (((? l) . nil) (evalk env l k))
         (((? l) . (? ls))
          (evalk env l
                 (lambda (x)
                   (eval-progn env ls k)))))
  )
```

To defunctionlize the continuation `(lambda (x) (eval-progn env ls
k))` we come up with a name `progn-cont` to identify it. Inside the
function `env`, `ls` and `k` are free, these will be needed when we
interpret the `progn-cont` data. So the complete data object we create
for this continuation is `(list 'progn-cont env ls k)`. A list containing
the identification of the kind of continuation and the arguments needed to
later interpret it. 

***Defunctionalized: eval-progn***
```lisp
(defun eval-progn (env args k)
  (match args
         (nil (apply-cont k nil))
         (((? l) . nil) (evald env l k))
         (((? l) . (? ls))
          (evald env l
                 (list 'progn-cont env ls k)))))
```

Now `eval-progn` creates a data-structure and passes that one on to
`evald` and at some point that data will need to be interpreted by the
`apply-cont` function. There will be a case in `apply-cont` for each
of the different continuation identities that we make up as we
progress.

`apply-cont` takes two arguments a continuation data-structure and an
expression and pattern matches on the continuation data.

```lisp
(defun apply-cont (k exp)
  (match k
         ((application-cont (? env) (? k1))
          (apply env exp k1))
         ((progn-cont (? env) (? ls) (? k1)) (eval-progn env ls k1))))

``` 

If the case for `progn-cont` is matched, `eval-progn` is called
recursively.  Note that the right hand side in the `progn-cont` case
is identical to the body of the original continuation function before
defunctionalization (Also note that this time it is a call to the new
defunctionalized eval-progn).

So the defunctionalization process can be summarized as:

1. Come up with a name for to identify the function (replace lambda with data)
2. Collect the free variables in the body of the lambda in a list together with
   the identifier
3. Add a case to `apply-cont` that matches on the data and has a right hand side
   identical to the body of the original lambda.

Let's proceed and look at `eval-define`

***Original: eval-define***

```lisp
(defun eval-define (env args k)
  (let ((key (car args))
        (val (car (cdr args))))
    (evalk env val
           (lambda (x)
             (progn
               (setvar 'global-env
                       (acons key x global-env))
               (k x))))))
```	     

We pick an identifier, `define-cont` and find the free variables `key` and `k`.
`global-env` is a global so no need to also add that to the data structure. 

***Defunctionalized: eval-define***

```lisp
(defun eval-define (env args k)
  (let ((key (car args))
        (val (car (cdr args))))
    (evald env val
           (list 'define-cont key k))))
```

And again the `apply-cont` gets a new case and we copy over the body from the
lambda to the right hand side in the match. 

```lisp
(defun apply-cont (k exp)
  (match k
         ((application-cont (? env) (? k1))
          (apply env exp k1))
         ((progn-cont (? env) (? ls) (? k1)) (eval-progn env ls k1))
         ((define-cont (? key) (? k1))
          (progn
            (setvar 'global-env (acons key exp global-env))
            (apply-cont k1 exp)))))
```


When we look at `eval-lambda` we notice that it does not create a new
continuation but rather just applies one, so no defunctionalization to do here.
Just have to remember that continuations are now data-structures that need to
be interpreted.


***Original: eval-lambda***

```lisp
(defun eval-lambda (env args k)
  (k (append (cons 'closure args) (list env))))
```

***New: eval-lambda***

```lisp
(defun eval-lambda (env args k)
  (apply-cont k (append (cons 'closure args) (list env))))
```

As there was no continuation to defunctionalize in `eval-lambda`, no
case is added to `apply-cont`.


The `eval-list` function is transformed in a now familiar way. 

***Original: eval-list***
```lisp
(defun eval-list (env ls acc k)
  (if (eq ls nil)
      (k acc)
    (let (( l (car ls))
          ( r (cdr ls)))
      (evalk env l
             (lambda (x)
               (eval-list env r (append acc (list x)) k))))))
```

We invent the name `list-cont` and find that `env`, `r`, `acc` and `k`
are free so we build the list `(list 'list-cont env r acc k)`. 

***Defunctionalized: eval-list***
```lisp
(defun eval-list (env ls acc k)
  (if (eq ls nil)
      (apply-cont k acc)
      (let (( l (car ls))
            ( r (cdr ls)))
        (evald env l
               (list 'list-cont env r acc k)))))
```

Add a pattern to `apply-cont` that matches a `(list-cont ...)` list and
copies the body of the original lambda into the right hand side of the
pattern match. 

```lisp
(defun apply-cont (k exp)
  (match k
         ((application-cont (? env) (? k1))
          (apply env exp k1))
         ((progn-cont (? env) (? ls) (? k1)) (eval-progn env ls k1))
         ((define-cont (? key) (? k1))
          (progn
            (setvar 'global-env (acons key exp global-env))
            (apply-cont k1 exp)))
         ((list-cont (? env) (? r) (? acc) (? k1))
          (eval-list env r (append acc (list exp)) k1))))
```

Almost done with defunctionalizing the entire evaluator now just
a few more steps. Conditionals! 

***Original: eval-if***
```lisp
(defun eval-if (env args k)
  (let ((cond-exp  (car args))
        (then-branch (car (cdr args)))
        (else-branch (car (cdr (cdr args)))))
    (evalk env cond-exp
           (lambda (x) (if x
                           (evalk env then-branch k)
                           (evalk env else-branch k))))))
```

Pick the name `if-cont` to identify this continuation data and
collect the free variables `env`, `then-branch`, `else-branch` and `k`.


***Defunctionalized: eval-if***
```lisp
(defun eval-if (env args k)
  (let ((cond-exp  (car args))
        (then-branch (car (cdr args)))
        (else-branch (car (cdr (cdr args)))))
    (evald env cond-exp
           (list 'if-cont env then-branch else-branch k))))
```

Add a case to the `match` expression in `apply-cont` that recognizes
 `if-cont` data and executes the appropriate code.

```lisp
(defun apply-cont (k exp)
  (match k
         ((application-cont (? env) (? k1))
          (apply env exp k1))
  	 ((progn-cont (? env) (? ls) (? k1)) (eval-progn env ls k1))
         ((define-cont (? key) (? k1))
          (progn
            (setvar 'global-env (acons key exp global-env))
            (apply-cont k1 exp)))
         ((list-cont (? env) (? r) (? acc) (? k1))
          (eval-list env r (append acc (list exp)) k1))
         ((if-cont (? env) (? then-branch) (? else-branch) (? k1))
          (if exp
              (evald env then-branch k1)
            (evald env else-branch k1)))))
```

Now we are mostly done, except we need a continuation object to represent
that there is nothing else to do and the computation is done. Let's call
this continuation `done`.


```lisp
(defun apply-cont (k exp)
  (match k
         (done exp)
         ((application-cont (? env) (? k1))
          (apply env exp k1))
  	 ((progn-cont (? env) (? ls) (? k1)) (eval-progn env ls k1))
         ((define-cont (? key) (? k1))
          (progn
            (setvar 'global-env (acons key exp global-env))
            (apply-cont k1 exp)))
         ((list-cont (? env) (? r) (? acc) (? k1))
          (eval-list env r (append acc (list exp)) k1))
         ((if-cont (? env) (? then-branch) (? else-branch) (? k1))
          (if exp
              (evald env then-branch k1)
            (evald env else-branch k1)))))
```

The last pattern added to `apply-cont` matches the symbol  `done` and
the result is the `exp` argument. Computation is done. 


Let's try the defunctionalized evaluator out a bit:

``` 
# global-env
> nil
# (evald nil '(define apa 1) 'done)
> 1
# global-env 
> ((apa . 1))
# 
```

Note that `done` is now just a symbol.

```
# (evald nil '(+ apa 100) 'done)
> 101
# 
```
Below you find the complete code for the defunctionalized continuation-passing
style evaluator! 

```lisp

(define global-env 'nil)

(defun is-number (e)
  (or (eq (type-of e) type-i)
      (eq (type-of e) type-u)))

(defun is-symbol (e)
  (eq (type-of e) type-symbol))

(defun is-operator (e)
  (or (eq e '+)
      (eq e '-)
      (eq e '=)
      (eq e '*)
      ))

(defun is-closure (e)
  (and (eq (type-of e) type-list)
       (eq (car e) 'closure)))

(defun add-bindings (env binds)
  (match binds
         (nil env)
         (((? b) . (? rs))
          (add-bindings (setassoc env b) rs))))

(defun eval-progn (env args k)
  (match args
         (nil (apply-cont k nil))
         (((? l) . nil) (evald env l k))
         (((? l) . (? ls))
          (evald env l
                 (list 'progn-cont env ls k)))))

(defun eval-define (env args k)
  (let ((key (car args))
        (val (car (cdr args))))
    (evald env val
           (list 'define-cont key k))))

(defun eval-lambda (env args k)
  (apply-cont k (append (cons 'closure args) (list env))))

(defun eval-if (env args k)
  (let ((cond-exp  (car args))
        (then-branch (car (cdr args)))
        (else-branch (car (cdr (cdr args)))))
    (evald env cond-exp
           (list 'if-cont env then-branch else-branch k))))

(defun eval-list (env ls acc k)
  (if (eq ls nil)
      (apply-cont k acc)
      (let (( l (car ls))
            ( r (cdr ls)))
        (evald env l
               (list 'list-cont env r acc k)))))

(defun apply-closure (env ls k)
  (let ((clo  (car ls))
        (args (cdr ls))
        (ps (car (cdr clo)))
        (body (car (cdr (cdr clo))))
        (env1 (car (cdr (cdr (cdr clo)))))
        (arg-env (zip ps args))
        (new-env (add-bindings (append env1 env) arg-env)))
    (evald new-env body k)))

(defun apply (env ls k)
   (let ((f (car ls)))
     (if (is-operator f)
         (apply-cont k (eval ls))
         (if (is-closure f)
             (apply-closure env ls k)
             'error))))

(defun apply-cont (k exp)
  (match k
         (done exp)
         ((progn-cont (? env) (? ls) (? k1)) (eval-progn env ls k1))
         ((define-cont (? key) (? k1))
          (progn
            (setvar 'global-env (acons key exp global-env))
            (apply-cont k1 exp)))
         ((list-cont (? env) (? r) (? acc) (? k1))
          (eval-list env r (append acc (list exp)) k1))
         ((application-cont (? env) (? k1))
          (apply env exp k1))
         ((if-cont (? env) (? then-branch) (? else-branch) (? k1))
          (if exp
              (evald env then-branch k1)
            (evald env else-branch k1)))))

(defun evald (env exp k)
  (if (is-operator exp)
      (apply-cont k exp)
    (if (is-symbol exp)
        (let ((res (assoc env exp)))
          (if (eq res nil)
              (apply-cont k (assoc global-env exp))
            (apply-cont k res)))
      (if (is-number exp)
          (apply-cont k exp)
        (match exp
               ((progn  . (? ls)) (eval-progn  env ls k))
               ((define . (? ls)) (eval-define env ls k))
               ((lambda . (? ls)) (eval-lambda env ls k))
               ((if . (? ls))     (eval-if env ls k))
               ((?cons ls)        (eval-list env ls nil
                                             (list 'application-cont env k)))
               )))))
```


We now have a first order program for evaluation of mini-lisp programs
in continuation passing style. 


## Making the Stack Explicit


If we look at the continuation objects created in the previous section:

```
(list 'application-cont env k)
(list 'progn-cont env ls k)
(list 'define-cont key k)
(list 'if-cont env then-branch else-branch k)
(list 'list-cont env r acc k)
done
``` 
All of these, except for `done` is a list, where the last element is `k`.
The `k` here is itself a continuation. So continuations in the defunctionalized
interpreter is a list of lists or in other words a list of continuation objects.
`done` takes the role of a terminating `nil` value.

If we look at the functions that create continuations we can see that each
of these functions only ever add a new continuation object to the head of this
continuation list. 

And if we then look at `apply-cont` we see that it processes the continuation
at the head of the list.

So, what we really have is a stack of continuation objects and this last
version of the interpreter makes that stack of continuations explicit. 

Let's first add a stack abstraction and then go through a few examples:

```lisp
(define stack 'nil)

(defun push (v)
  (setvar 'stack (cons v stack)))

(defun pop ()
  (let ((r (car stack)))
    (progn
      (setvar 'stack (cdr stack))
      r)))
``` 

The code above defines a stack and the operations `push`, for adding
to the top and `pop, for removing the and returning the top
element. As the stack defined here is global, there is no need to pass
the k argument around everywhere in the new evaluator called
`eval-stack`. So `eval-stack` takes just two arguments, an environment
and an expression.


***Defunctionalized: eval-progn***
```
(defun eval-progn (env args k)
  (match args
         (nil (apply-cont k nil))
         (((? l) . nil) (evald env l k))
         (((? l) . (? ls))
          (evald env l
                 (list 'progn-cont env ls k)))))
```

***Explicit Stack: eval-progn***
```lisp
(defun eval-progn (env args)
  (match args
         (nil (apply-cont nil))
         (((? l) . nil) (eval-stack env l))
         (((? l) . (? ls))
          (progn
            (push (list 'progn-cont env ls))
            (eval-stack env l)))))
```

The explicit stack version of `eval-progn`, pushes the object `(list 'progn-cont env ls)`  (note there is no `k` in there).


When it is time to interpret the continuation, `apply-cont` pops the top of the
stack and interprets it.

```lisp
(defun apply-cont (exp)
  (let (( k (pop)))
    (match k
           (done exp)
           ((progn-cont (? env) (? ls)) (eval-progn env ls))
           ((define-cont (? key))
            (progn
              (setvar 'global-env (acons key exp global-env))
              (apply-cont exp)))
           ((list-cont (? env) (? r) (? acc))
            (eval-list env r (append acc (list exp))))
           ((application-cont (? env))
            (apply env exp))
           ((if-cont (? env) (? then-branch) (? else-branch))
            (if exp
                (eval-stack env then-branch)
                (eval-stack env else-branch))))))
``` 

Below you can find the code for the explicit stack evaluator. 

```lisp
(define global-env 'nil)

(define stack 'nil)

(defun push (v)
  (setvar 'stack (cons v stack)))

(defun pop ()
  (let ((r (car stack)))
    (progn
      (setvar 'stack (cdr stack))
      r)))

(defun is-number (e)
  (or (eq (type-of e) type-i)
      (eq (type-of e) type-u)))

(defun is-symbol (e)
  (eq (type-of e) type-symbol))

(defun is-operator (e)
  (or (eq e '+)
      (eq e '-)
      (eq e '=)
      (eq e '*)
      ))

(defun is-closure (e)
  (and (eq (type-of e) type-list)
       (eq (car e) 'closure)))

(defun add-bindings (env binds)
  (match binds
         (nil env)
         (((? b) . (? rs))
          (add-bindings (setassoc env b) rs))))

(defun eval-progn (env args)
  (match args
         (nil (apply-cont nil))
         (((? l) . nil) (eval-stack env l))
         (((? l) . (? ls))
          (progn
            (push (list 'progn-cont env ls))
            (eval-stack env l)))))
              

(defun eval-define (env args)
  (let ((key (car args))
        (val (car (cdr args))))
    (progn
      (push (list 'define-cont key))
      (eval-stack env val))))
           

(defun eval-lambda (env args)
  (apply-cont (append (cons 'closure args) (list env))))

(defun eval-if (env args)
  (let ((cond-exp  (car args))
        (then-branch (car (cdr args)))
        (else-branch (car (cdr (cdr args)))))
    (progn 
      (push (list 'if-cont env then-branch else-branch))
      (eval-stack env cond-exp))))
          

(defun eval-list (env ls acc)
  (if (eq ls nil)
      (apply-cont acc)
      (let (( l (car ls))
            ( r (cdr ls)))
        (progn
          (push (list 'list-cont env r acc))
          (eval-stack env l)))))
               

(defun apply-closure (env ls)
  (let ((clo  (car ls))
        (args (cdr ls))
        (ps (car (cdr clo)))
        (body (car (cdr (cdr clo))))
        (env1 (car (cdr (cdr (cdr clo)))))
        (arg-env (zip ps args))
        (new-env (add-bindings (append env1 env) arg-env)))
    (eval-stack new-env body)))

(defun apply (env ls)
   (let ((f (car ls)))
     (if (is-operator f)
         (apply-cont (eval ls))
         (if (is-closure f)
             (apply-closure env ls)
             'error))))

(defun apply-cont (exp)
  (let (( k (pop)))
    (match k
           (done exp)
           ((progn-cont (? env) (? ls)) (eval-progn env ls))
           ((define-cont (? key))
            (progn
              (setvar 'global-env (acons key exp global-env))
              (apply-cont exp)))
           ((list-cont (? env) (? r) (? acc))
            (eval-list env r (append acc (list exp))))
           ((application-cont (? env))
            (apply env exp))
           ((if-cont (? env) (? then-branch) (? else-branch))
            (if exp
                (eval-stack env then-branch)
                (eval-stack env else-branch))))))

(defun evals (env exp)
  (progn
    (setvar 'stack nil)
    (push 'done)
    (eval-stack env exp)))
  

(defun eval-stack (env exp)
  (if (is-operator exp)
      (apply-cont exp)
    (if (is-symbol exp)
        (let ((res (assoc env exp)))
          (if (eq res nil)
              (apply-cont (assoc global-env exp))
            (apply-cont res)))
      (if (is-number exp)
          (apply-cont exp)
        (match exp
               ((progn  . (? ls)) (eval-progn  env ls))
               ((define . (? ls)) (eval-define env ls))
               ((lambda . (? ls)) (eval-lambda env ls))
               ((if . (? ls))     (eval-if env ls))
               ((?cons ls)        (progn
                                    (push (list 'application-cont env))
                                    (eval-list env ls nil)))
               )))))
```

## Relation to LispBM

The approach to evaluation in LispBM is very similar to the ***explicit
stack defunctionalized continuation-passing style evaluator*** defined in
the section above, only it is implemented in C (and actually does some
amount of error checking).

The continuation identities that exist in the LispBM implementation
are many more of course:

```c
#define ILLEGAL_CONT      ((0 << LBM_VAL_SHIFT) | LBM_TYPE_U)
#define DONE              ((1 << LBM_VAL_SHIFT) | LBM_TYPE_U)
#define SET_GLOBAL_ENV    ((2 << LBM_VAL_SHIFT) | LBM_TYPE_U)
#define BIND_TO_KEY_REST  ((3 << LBM_VAL_SHIFT) | LBM_TYPE_U)
#define IF                ((4 << LBM_VAL_SHIFT) | LBM_TYPE_U)
#define PROGN_REST        ((5 << LBM_VAL_SHIFT) | LBM_TYPE_U)
#define APPLICATION       ((6 << LBM_VAL_SHIFT) | LBM_TYPE_U)
#define APPLICATION_ARGS  ((7 << LBM_VAL_SHIFT) | LBM_TYPE_U)
#define AND               ((8 << LBM_VAL_SHIFT) | LBM_TYPE_U)
#define OR                ((9 << LBM_VAL_SHIFT) | LBM_TYPE_U)
#define WAIT              ((10 << LBM_VAL_SHIFT) | LBM_TYPE_U)
#define MATCH             ((11 << LBM_VAL_SHIFT) | LBM_TYPE_U)
#define MATCH_MANY        ((12 << LBM_VAL_SHIFT) | LBM_TYPE_U)
#define READ              ((13 << LBM_VAL_SHIFT) | LBM_TYPE_U)
#define APPLICATION_START ((14 << LBM_VAL_SHIFT) | LBM_TYPE_U)
#define EVAL_R            ((15 << LBM_VAL_SHIFT) | LBM_TYPE_U)
#define SET_VARIABLE      ((16 << LBM_VAL_SHIFT) | LBM_TYPE_U)
#define RESUME            ((17 << LBM_VAL_SHIFT) | LBM_TYPE_U)
#define EXPAND_MACRO      ((18 << LBM_VAL_SHIFT) | LBM_TYPE_U)
#define QUOTE_RESULT      ((19 << LBM_VAL_SHIFT) | LBM_TYPE_U)
#define BACKQUOTE_RESULT  ((20 << LBM_VAL_SHIFT) | LBM_TYPE_U)
#define COMMAAT_RESULT    ((21 << LBM_VAL_SHIFT) | LBM_TYPE_U)
#define COMMA_RESULT      ((22 << LBM_VAL_SHIFT) | LBM_TYPE_U)
#define DOT_TERMINATE     ((23 << LBM_VAL_SHIFT) | LBM_TYPE_U)
#define EXPECT_CLOSEPAR   ((24 << LBM_VAL_SHIFT) | LBM_TYPE_U)
#define APPEND_CONTINUE   ((25 << LBM_VAL_SHIFT) | LBM_TYPE_U)
#define READ_DONE         ((26 << LBM_VAL_SHIFT) | LBM_TYPE_U)
#define CLOSURE_ARGS      ((27 << LBM_VAL_SHIFT) | LBM_TYPE_U)
#define CLOSURE_APP       ((28 << LBM_VAL_SHIFT) | LBM_TYPE_U)
``` 

The part of LispBM that corresponds to `eval-stack` from above is implemented
as a C switch statements:

```c
  switch (lbm_type_of(ctx->curr_exp)) {

  case LBM_TYPE_SYMBOL: {
    lbm_value s;
    if (ctx->curr_exp == NIL) {
      ctx->app_cont = true;
      ctx->r = NIL;
      return;
    }

    if (eval_symbol(ctx, &s)) {
      ctx->app_cont = true;
      ctx->r = s;
      return;
    }

    if (dynamic_load_callback) {
      dynamic_load(ctx);
    }
    return;
  }
  case LBM_TYPE_FLOAT: /* fall through */
  case LBM_TYPE_DOUBLE:
  case LBM_TYPE_U32:
  case LBM_TYPE_U64:
  case LBM_TYPE_I32:
  case LBM_TYPE_I64:
  case LBM_TYPE_I:
  case LBM_TYPE_U:
  case LBM_TYPE_CHAR:
  case LBM_TYPE_ARRAY:
  case LBM_TYPE_REF:
  case LBM_TYPE_STREAM: eval_selfevaluating(ctx);  return;

  case LBM_TYPE_CONS:
    head = lbm_car(ctx->curr_exp);

    if (lbm_type_of(head) == LBM_TYPE_SYMBOL) {

      lbm_uint sym_id = lbm_dec_sym(head);

      switch(sym_id) {
      case SYM_QUOTE:   eval_quote(ctx); return;
      case SYM_DEFINE:  eval_define(ctx); return;
      case SYM_PROGN:   eval_progn(ctx); return;
      case SYM_LAMBDA:  eval_lambda(ctx); return;
      case SYM_IF:      eval_if(ctx); return;
      case SYM_LET:     eval_let(ctx); return;
      case SYM_AND:     eval_and(ctx); return;
      case SYM_OR:      eval_or(ctx); return;
      case SYM_MATCH:   eval_match(ctx); return;
      case SYM_RECEIVE: eval_receive(ctx); return;
      case SYM_CALLCC:  eval_callcc(ctx); return;

      case SYM_MACRO:   /* fall through */
      case SYM_CONT:
      case SYM_CLOSURE: eval_selfevaluating(ctx); return;

      default: break; /* May be general application form. Checked below*/
      }
    } // If head is symbol
    /*
     * At this point head can be a closure, fundamental, extension or a macro.
     * Anything else would be an error.
     */
    lbm_value *reserved = lbm_stack_reserve(&ctx->K, 3);
    if (!reserved) {
      error_ctx(lbm_enc_sym(SYM_STACK_ERROR));
      return;
    }
    reserved[0] = ctx->curr_env;
    reserved[1] = lbm_cdr(ctx->curr_exp);
    reserved[2] = APPLICATION_START;

    ctx->curr_exp = head; // evaluate the function
    break;
  default:
    // BUG No applicable case!
    error_ctx(lbm_enc_sym(SYM_EERROR));
    break;
  }
``` 

And the part of LispBM that corresponds to `apply-cont` looks as follows. 

```c
if (ctx->app_cont) {
    lbm_value k;
    lbm_pop(&ctx->K, &k);
    ctx->app_cont = false;

    switch(k) {
    case DONE:              advance_ctx(); return;
    case SET_GLOBAL_ENV:    cont_set_global_env(ctx); return;
    case PROGN_REST:        cont_progn_rest(ctx); return;
    case WAIT:              cont_wait(ctx); return;
    case APPLICATION_ARGS:  cont_application_args(ctx); return;
    case AND:               cont_and(ctx); return;
    case OR:                cont_or(ctx); return;
    case BIND_TO_KEY_REST:  cont_bind_to_key_rest(ctx); return;
    case IF:                cont_if(ctx); return;
    case MATCH:             cont_match(ctx); return;
    case MATCH_MANY:        cont_match_many(ctx); return;
    case READ:              cont_read(ctx); return;
    case APPLICATION_START: cont_application_start(ctx); return;
    case EVAL_R:            cont_eval_r(ctx); return;
    case SET_VARIABLE:      cont_set_var(ctx); return;
    case RESUME:            cont_resume(ctx); return;
    case EXPAND_MACRO:      cont_expand_macro(ctx); return;
    case CLOSURE_ARGS:      cont_closure_application_args(ctx); return;
    default:
      error_ctx(lbm_enc_sym(SYM_EERROR));
      return;
    }
  }
``` 

The LispBM evaluator was not created by first writing it all in some
higher order language and then defunctionalizing it. It was all come
up with quite ad-hoc and that was actually quite messy.

This experiment, of implementing a CPS evaluator in lisp and then
defunctionalizing it and making the stack explicit was fun though. It
also feels like I am a bit more confident about the LispBM evaluator
implementation after performing this experiment.

Constructive feedback, discovered bugs, deeper insights or general
comments are most welcome! Thank you. 

___

[HOME](https://svenssonjoel.github.io)
