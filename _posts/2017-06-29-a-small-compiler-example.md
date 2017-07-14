    Title: A small compiler example
    Date: 2017-06-29T13:39:44
    Tags: DRAFT
    Authors: Carl

_Objective: Implement a compiler that takes as input simple S-expressions and outputs x86 assembly_

A compiler takes a description of a computation and translates it typically to a lower level
of computation.  Compilers enable us to express programs in a form more suitable to
an application. Compilers transform programs into forms that are more compatible with
a particular machine.

For example given a trivial S-expression to add two numbers:

```
(+ 1 2)
```

The objective is to generate code in assembly something like:

```
push DWORD 2
push DWORD 1
mov ebx, [esp+4]
mov eax, [esp]
add esp, 8
add eax, ebx
push eax
```

The above two programs are trying to do the same thing, add the numbers 1 and 2,
yet they look very different.

One of the things that the upper program is not saying is how anything is to be
stored in the machine.  The assembly program explicitly pushes two numbers
onto a stack, and then moves numbers from the stack into registers.

The approach I'm familiar with is to evaluate the S-expression
recursively, emitting assembly for each sub expression.

An initial naive attempt at the compile function:

```
(define (compile exp)
  (cond
    [(fixnum? exp)   (emit "push ~a~%" exp)]
    [(eq? '+ exp)    (emit "add eax, bax~%")]))
```

The obvious thing wrong with the above is that `+` needs to generate code to
handle its arguments. This means calling `compile` for them.

Because on x86 the `add` instruction works with registers and we push
arguments on to the stack, we must pop the arguments from the stack and then
load them into `eax` and `ebx` in preparation for `add` to do its work.

```
(define (compile exp)
  (cond
    [(fixnum? exp)   (emit "push ~a~%" exp)]
    [(eq? '+ (car exp))    (begin
                              (compile (cadr exp))
                              (compile (caddr exp))
                              (emit "pop eax~%")
                              (emit "pop ebx~%")
                              (emit "add eax, ebx~%")]))
```

That is getting close. Lets implement `emit`, for now as basically an alias to `printf`,

```
(define emit printf)
```

Lets try `compile` on an expression, for example

```
(compile '(+ 1 2))
```

Now to see if that produces something Nasm can work with by redirecting the output into a file, assembling with Nasm, and then using `hexdump` on the resulting assembled code:

```
$ chez --script c.scm > c.s
$ cat c.s
push 1
push 2
add eax, ebx
$ nasm c.s
$ hexdump c
0000000 6a 01 6a 02 66 01 d8                           
0000007
```

It looks like Nasm has accepted the assembled and produced a binary!

This is good news, however we've glossed over a detail: Nasm has not generated binary code ready to run on the operating system.

We need to tell Nasm to generate code that the linker can work with.
The linker is a program that knows how to prepare binary code so it can be loaded into memory as managed by the OS.

_In this example we tell Nasm the format is Mach-O which is the format used by OSX. Mach is the name of the kernel on OSX (and BSD)_

```
$ nasm -f macho c.s
$ file c.o
c.o: Mach-O object i386
```

Now to try linking:

```
$ ld c.o
Undefined symbols for architecture i386:
  "start", referenced from:
     implicit entry/start for main executable
ld: symbol(s) not found for inferred architecture i386
```

The linker looks for a symbol called `start` (a bit like `main` in a C program).
This is where the OS starts running the program from.

Here we need to implement a symbol `start` in our object code.
This only needs to happen once, so we'll add a `compile-top` function that
emits the `start` symbol:

```
(define (compile-top exp)
  (emit "global start~%")
  (emit "start:~%")
  (compile exp))

(compile-top '(+ 1 2))
```

```
$ chez --script c.scm
global start
start:
push 1
push 2
add eax, ebx
```

Now the assembly declares a globally accessible symbol `start`,
which is used to locate a position in the code, by following it with a
semicolon.

Running Nasm on this file again produces no errors. We can also use `nm`
to display the symbols in the object file:

```
$ nm c.o
00000000 T start
```

And the linker accepts this code without an error this time, and generates a file
`a.out`:

```
$ ld c.o
$ file a.out
a.out: Mach-O executable i386
```

We have compiled an executable!

```
$ ./a.out
zsh: segmentation fault  ./a.out
```

But when we run it, there's a fault.

The problem is the OS runs our code, which doesn't exit. Beyond our code is memory
which doesn't belong to our process and so the OS generates a fault when
it looks for the next instruction to run.

The fix is to generate code that exits cleanly.

On our OS the operating system call for a process to exit is represented by number 1.
If we call an interrupt with 1 in the eax register,
the operating system handles the interrupt and
ends the process normally. Lets add that after our code is generated in `compile-top`:

```
(define (compile-top exp)
  (emit "global start~%")
  (emit "start:~%")
  (compile exp)
  (emit "mov eax, 0x1~%")   ; 1 is the system call for EXIT
  (emit "int 0x80~%"))      ; interrupt the OS
```

Now assembling, linking, and running all happens with no errors, and check `$?` for the exit code of our process:

```
$ chez --script c.scm > c.s
$ nasm -f macho c.s
$ ld c.o
$ ./a.out
$ echo $?
1
```

The number 1 is not much use to us as proof of our efforts.
Lets try emit the result of the addition.

The way to do this is to push the result of the addition on to the stack.

On Mach-O, we also have to grow the stack. Because the stack grows downwards
in memory this means decreasing the value of the stack pointer.
Mach-O pushes an extra word on the stack for book keeping, and when
doing assembly programming we must observe such conventions that the
Operating System uses.

Here's the `compile` function with the additional push of the eax register:

```
(define (compile exp)
  (cond
    [(fixnum? exp)   (emit "push ~a~%" exp)]
    [(eq? '+ (car exp))    (begin
                              (compile (cadr exp))
                              (compile (caddr exp))
                              (emit "pop eax~%")
                              (emit "pop ebx~%")
                              (emit "add eax, ebx~%")
                              (emit "push eax~%")
                              (emit "sub esp, 4~%") ; Mach-O special
                              )]))
```

Let's try it with a new pair of numbers:

```
(compile-top '(+ 42 128))
```

And go through compilation, assembly, linking, and execution:

```
$ chez --script c.scm > c.s
$ nasm -f macho c.s
$ ld c.o
$ ./a.out
$ echo $?
170
```

I'm sure you'll agree that the exit code of 170 was no accident!

Extending this compiler now is a matter of refactoring the `compile` function
to support more operators, and potentially also implementing storage for local variables
in the input expressions (hint: we can use the stack for this and remember the position/index on the stack).

There are a couple of points to take out of this:

1. S-expressions, and languages like Scheme that manipulate them, are great
for experimenting with compilers because they can be used to represent tree
structures. Much effort in writer a compiler involves parsing the syntax
of an input language into a tree structure known as an Abstract Syntax Tree.
In the example above parsing is not necessary because S-expressions and Scheme
is used.

2. When writing compilers we can gain a good understanding of the
computational model used by the input and target language.
In the case of assembly, some of the key knowledge is that there
is a stack and registers and the instructions
work on them.
In scheme key knowledge is that expressions evaluate recursively.
The input program program must be translated so that the
functions described in the input are equivalently expressed as functions
(perhaps composed of many sub functions) in the target language.

The big idea for me in a compiler is that they are not so much about syntax,
but about translating between computational models.

_I wonder if there are computational models out there in nature that we may not
recognize as computation due to our own limits of perception._
