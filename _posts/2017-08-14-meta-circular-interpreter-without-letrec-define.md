_Objective: implement a functional Lisp/Scheme interpreter without using define, letrec, set! etc_

After watching a youtube video about [lambda calculus](https://www.youtube.com/watch?v=eis11j_iGMs)
and being impressed by the simplicity and power of this model of computation
I got interested in trying to implement a language based on the lambda calculus myself.
The excellent [blog article by Matt Might](http://matt.might.net/articles/implementing-a-programming-language/)
provided a complete example and inspiration.

One of the goals I had in mind was for the lambda calculus interpreter to be able to interpret itself.
It was most likely this goal that led to the path of discovery ending at
the Y Combinator - also known as the
[Fixed point combinator](https://en.wikipedia.org/wiki/Fixed-point_combinator#Fixed_point_combinators_in_lambda_calculus)

I started out with an interpreter that used `define`, which binds
a name to a value in the environment.
The interpreter looked kind of like this in outline:

```
(define (eval exp env)
 ... )

(define (apply fn args)
 ... )
```

Using `define` seemed like a big leap from the simplicity of lambda calculus.
It meant I had to implement `define` in my interpreter for it to be capable
of interpreting itself. This meant having more to implement and a resulting bigger language,
as well as introducing mutable environments.

I started thinking about replacing top level define expressions with a `letrec`.
A `letrec` implementation might look something like this in outline:

```
(letrec ((eval (lambda (exp env) ... ))
         (apply (lambda (fn args) ...)))
  ...
)
```

Then while browsing through the book, [The Little Schemer](https://www.amazon.com/Little-Schemer-Daniel-P-Friedman/dp/0262560992/ref=sr_1_1?ie=UTF8&qid=1502698392&sr=8-1),
I happened across some curious notes:

> But what about (define ...) ?
>
> It isn't needed because recursion can be obtained
> from the Y combinator.

> Is (define ...) really not needed?
>
> Yes, but see The Seasoned Schemer.

> Does that mean we can run the interpreter
> on the interpreter if we do the transformation
> with the Y combinator?
>
> Yes, but don't bother.

Unfortunately I didn't own a copy of The Seasoned Schemer.
So I revisited a few
other great sources of information on implementing Lisp
(which I have used copies of purchased online):
[The Anatomy of Lisp](https://www.amazon.com/Anatomy-Lisp-McGraw-Hill-computer-science/dp/007001115X/ref=sr_1_1?s=books&ie=UTF8&qid=1502698667&sr=1-1)
by John Allen,
[Functional Programming: Application and Implementation](https://www.amazon.com/Functional-Programming-Application-Implementation-Henderson/dp/0133315797/ref=sr_1_4?s=books&ie=UTF8&qid=1502698741&sr=1-4)
by Peter Henderson,

In the end it was [Structure and Interpretation of Computer Programs](https://www.amazon.com/Structure-Interpretation-Computer-Programs-Engineering/dp/0262510871/ref=sr_1_1?s=books&ie=UTF8&qid=1502698871&sr=1-1)
by Harold Abelson and Gerald Jay Sussman,
that revealed an answer.

> It is indeed possible to specify recursive procedures without using letrec (or even define)

See [Exercise 4.21](https://mitpress.mit.edu/sicp/full-text/book/book-Z-H-26.html#%_thm_4.21),
for the complete example.

The basis of recursion using this approach is for
the function that will recurse to take itself as one of its arguments.
It can then call itself passing itself as an argument.
E.g.

```
(lambda (fn)
  (fn fn))
```

If `fn` above evaluates to `(lambda (fn) (fn fn))` the above will recurse indefinitely.

However to call this function to start off with, another function is necessary:

```
(lambda (fn)
  (fn fn))
```

The above is a function that takes a function, which
then calls with itself as its only argument.
In this case it looks the same as the former recursive function.

To make it happen, the later function is applied to the former,
and in this case they are both the same function:

```
((lambda (fn) (fn fn))  (lambda (fn) (fn fn)))
```

By the way, this curiosity is a program called Omega.

Taking this approach to the interpreter,
the interpreter code could look like this in outline:

```
(lambda (exp env)
  (
   ; calls eval, passing the eval and apply functions as arguments
   ; so that they may call themselves,
   ; also pass the expression to evaluate and the environment
   (lambda (eval apply)
     (eval eval apply exp env))

   ; the definition of the eval function.
   ; the ^ (caret) suffix is to distinguish the recursive arguments
   (lambda (eval^ apply^ exp env)
     ... )

   ; the definition of the apply function
   (lambda (apply^ eval^ fn args)
     ... )
  )
)
```

Because both `eval` and `apply` can call `eval` recursively
they both take themselves as arguments.

The next consideration is what supporting functions are necessary
in the implementations of `eval` and `apply`.
For the interpreter to be able to interpret itself it must
be able to access implementations of the supporting functions.
The supporting functions are sometimes referred to as "built-in" or "primitive".

The interpreter uses an association list for its environment.
Using quasi-quoting, the environment of built-in / primitive functions
can bind to the same functions provided by the host language Racket:

```
`((cons    ,cons)
  (car     ,car)
  (cdr     ,cdr)
  (list    ,list)
  (eq?     ,eq?)
  (symbol? ,symbol?)
  (null?   ,null?)
  (pair?   ,pair?)
  (procedure? ,procedure?)
  (assq    ,assq)
  (apply   ,apply)))
```

In this case it was a prudent design decision to use Racket as the host
language as it comes with the necessary functions and its
calling conventions are the same as for the interpreter.
In the future we could implement a compiler for the
interpreted language which would mean providing equivalent functions in
a target language.

Now to see if the interpreter can really interpret itself.
To do this a test function is applied to the interpreter and
its support environment.
The interpreter will be evaluated in
an outer context, where it is evaluated by the host language, Racket,
and an inner context, where it is evaluated by the outer evaluator, itself.

The inner interpreter will evaluate `(+ 1 2)`, which means providing
the function `+` in its environment. `+` is bound to the `+` function
in the host language, Racket.

```
((lambda (interpreter environment)

  ; Evaluate the interpreter in the Racket host language
  ; and then call it with an expression and environment.
  ; The expression to this outer interpreter passed is the interpreter itself
  ; being called to evaluate the expression (+ 1 2).
  ;
  ; Because the interpreter evaluates all arguments, `quote` is necessary
  ; to ensure the inner interpreter evaluates (+ 1 2).
  ; Without `quote` the outer interpreter would evaluate (+ 1 2) resulting in
  ; the inner interpreter evaluating 3 - which we don't want.

  ((eval interpreter)     ; host eval interpreter: the outer interpreter.
    `(,interpreter        ; inner interpreter eval of (+ 1 2).
       (quote (+ 1 2))
       (quote ((+ ,+))))
     environment))

  ; the interpreter function
  `(lambda (e env)
    ... )

  ; the environment that supports the interpreter
  `((cons   ,cons)
    ... )
))
```

The above should evaluate to 3 when run in Racket.

Note that the interpreter does not support imperative style multiple expression
statements like `begin` in Scheme or `progn` in Lisp.

The complete source code:

```
#lang racket
;
; A self-evaluating Lisp interpreter implemented without define, letrec, let
;
; Copyright (C) 2017  A. Carl Douglas
;

; without current-namespace, this racket error occurred:
;   ?: function application is not allowed;
;   no #%app syntax transformer is bound in:
(current-namespace (make-base-namespace))

((lambda (interpreter environment)
   ;
   ; Note that the evaluator evaluates arguments, which is why the quote is necessary:
   ; The outer interpreter is evaluating the inner interpreter
   ; as well as the arguments to the inner interpreter
   ; so to ensure the inner interpreter gets it's arguments unevaluated we must quote.
   ;
   ; Also note that the evaluator does not support multi statement lambdas (like begin)
   ; which means adding debugging like printf causes incorrect behavior
   ;
   ((eval interpreter) `(,interpreter (quote (+ 1 2)) (quote ((+ ,+)))) environment))

 `(lambda (e env)
    ((lambda (eval apply)
       (eval eval apply e env)) ; call eval
     (lambda (eval^ apply^ e env) ; define eval
       (if (symbol? e)
           (car (cdr (assq e env)))
           (if (pair? e)
               (if (eq? (car e) 'lambda)
                   (list 'closure e env)
                   (if (eq? (car e) 'if)
                       (if (eval^ eval^ apply^ (car (cdr e)) env)
                           (eval^ eval^ apply^ (car (cdr (cdr e))) env)
                           (eval^ eval^ apply^ (car (cdr (cdr (cdr e)))) env))
                       (if (eq? (car e) 'quote)
                           (car (cdr e))
                           (apply^ apply^ eval^ (eval^ eval^ apply^ (car e) env)
                                   ; inline evlist
                                   ((lambda (e1 env1)
                                      ((lambda (evlist)
                                         (evlist evlist e1 env1))  ; call evlist
                                       (lambda (evlist^ e1 env1)  ; define evlist
                                         (if (null? e1)
                                             '()
                                             (cons (eval^ eval^ apply^ (car e1) env1) (evlist^ evlist^ (cdr e1) env1))))))
                                    (cdr e) env)))))
               e)))
     (lambda (apply^ eval^ f x) ; define apply
       (if (procedure? f)
           (apply f x)
           (eval^ eval^ apply^ (car (cdr (cdr (car (cdr f)))))
                  ; inline newenv - extends the environment
                  ((lambda (names values env)
                     ((lambda (newenv)
                        (newenv newenv names values env)) ; call newenv
                      (lambda (newenv^ names values env) ; define newenv
                        (if (null? names)
                            env
                            (cons (list (car names) (car values))
                                  (newenv^ newenv^ (cdr names) (cdr values) env))))))
                   (car (cdr (car (cdr f)))) x (car (cdr (cdr f)))))))))

 ; environment containing built-in primitives required by the eval implementation
 `((cons    ,cons)
   (car     ,car)
   (cdr     ,cdr)
   (list    ,list)
   (eq?     ,eq?)
   (symbol? ,symbol?)
   (null?   ,null?)
   (pair?   ,pair?)
   (procedure? ,procedure?)
   (assq    ,assq)
   (printf  ,printf)
   (read    ,read)
   (print   ,display)
   (apply   ,apply)))
```

The source code is also available here
[https://gist.github.com/carld/885448d6f8630ca2c9e8ad7a6e2c1b61](https://gist.github.com/carld/885448d6f8630ca2c9e8ad7a6e2c1b61)
