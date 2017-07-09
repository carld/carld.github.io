# Scheme Macros

While looking into ways of implementing compilers I came across
[The COMFY 6502 Compiler by Henry Baker](http://home.pipeline.com/~hbaker1/sigplannotices/sigcol04.pdf).

COMFY uses macros to implement some things that aren't primitive
operations on the 6502.

_There's a compiler used by Larceny called Two Bit, and another called Sassy that I also looked into for inspiration._

I imagined a DSL that could represent x86 assembly in Scheme:

```
(section .text
  (mov eax 4)
  (mov ebx 7)
  (add eax ebx))
```

I thought that should be trivial to implement using Scheme macros, but for one reason or another I had trouble coming up with a macro that could emit correct input for Nasm.

The core difficulty for me was that the above DSL required assembly operation macros nested inside the scope of a `section` macro.

Ultimately the key that unlocked a solution was the knowledge that macros expand head first and
carry on expanding recursively. This means a macro definition can expand to an expression
containing macros, which are then expanded, and so forth.

For example, given expansion transitions for macros `A` and `B`:

```
A -> B
B -> c d
```

The net result of macro expansion of expression `A` is `c d`.

Try it out:

```
(define-syntax A                                                                     
  (syntax-rules ()                                                                   
                [(_)   (B)]))                                                        

(define-syntax B                                                                     
  (syntax-rules ()                                                                   
                [(_)   '(c d)]))                                                     

(printf "~a~%" (A))
```

_Note: Most Schemes have a `expand` (Chez) or similarly named `macro-expand` function that can help with debugging macros in the REPL._

Running this on Chez:

```
Chez Scheme Version 9.4.1
Copyright 1984-2016 Cisco Systems, Inc.

(c d)
>
```

For the curious, macros can recurse, and changing the above macro `B` to transition back `B -> A` is possible, but don't expect your program to finish expanding those macros in a hurry.

Another essential feature of macros is the ellipsis `...` which indicates that the preceding
pattern, be it a single token or list, can repeat zero or more times.
And therefore the expansion template occurs for every ellipsis match.

For example, a macro transition like `(M a ...) -> ((1 a) ...)`, for an expression `(M a b c d)` results in: `((1 a) (1 b) (1 c) (1 d))`.

Try it out:

```
(define-syntax R
  (syntax-rules ()
                [(_ token ...)  '((1 token) ...)]))

(printf "~a~%" (R a b c d))
```

```
Chez Scheme Version 9.4.1
Copyright 1984-2016 Cisco Systems, Inc.

((1 a) (1 b) (1 c) (1 d))
>
```

With the above information, the section transition can be defined with `S`, and the assembly operator transition with `O`:

```
S t (i ...) ... -> section type
                   (O i ...) ...
O i -> i
O i a -> i a
O i a b -> i a b
```

In Scheme the section macro can be defined, with the expansion template containing a macro `asm-op`:

```                         
(define-syntax section                                                                  
  (syntax-rules ()                                                                   
     [(_ type (instruction ...) ...)                                                 
      (begin                                                                         
        (printf "~a ~a~%" 'section 'type)                                            
        (asm-op instruction ...) ... )]))                                              

(define-syntax asm-op                                                                 
  (syntax-rules ()                                                                   
     [(_ i)  (printf "~a~%" 'i )]                                                    
     [(_ i op1)  (printf "~a ~a~%" 'i 'op1)]                                         
     [(_ i op1 op2)  (printf "~a ~a, ~a~%" 'i 'op1 'op2)]))
```

Now writing a Scheme program that uses the `section` macro for example:

```
(section .text
         (hello)
         (world)
         (push 1))
```

We get something closer to a Nasm compatible source file:

```
section .text
hello
world
push 1
```

A variation is to scope the asm-op macro using `let-syntax`, restricting the expansion of that to only within the section macro:

```
(define-syntax section
  (syntax-rules ()
    [(_ type (instruction ...) ... )

     (let-syntax ([asm-op (syntax-rules ()
                       [(_ i)  (printf "~s~%" 'i )]
                       [(_ i op1)  (printf "~s ~s~%" 'i 'op1)]
                       [(_ i op1 op2)  (printf "~s ~s, ~s~%" 'i 'op1 'op2)])])
       (begin
         (printf "~s ~s~%" 'section 'type)
         (asm-op instruction ...) ...
          ))]))
```
