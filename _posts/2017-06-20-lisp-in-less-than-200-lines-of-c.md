    Title: a brief and simple programming language implementation
    Tags: lambda calculus, Lisp, C, programming
    Authors:

_Objective: implement a lambda calculus based programming language like LisP, simply and briefly in C_

After learning some Scheme and Lisp and implementing LispKit and reading about eval/apply and how minimal the evaluator is, I decided to try implement Lisp in as little C as I could.

Since it's less than 200 lines of C code I'll just discuss the code inline:

```c
  #include <stdio.h>
  #include <stdlib.h>
  #include <string.h>
```
Included are standard headers files: `stdio.h` gives us `printf` and `puts` for printing to stdout, and `getchar` for retreving a character from stdin.
`stdlib.h` provides `calloc` for dynamically allocating memory while the program is running.
`string.h` provides `strcmp` for comparing two strings, and `strdup` for making a duplicate copy of a string.

```c
  #define debug(m,e) printf("%s:%d: %s:",__FILE__,__LINE__,m); print_obj(e,1); puts("");
```

This `debug` macro was used to help troubleshoot the program when it didn't work. I'd add a line like `debug('evaluating', exp)` and it would print out the file, line number, a message, and the Lisp expression representation in a readable form.

```c
  typedef struct List {
    struct List * next;
    void * data;
  } List;
```

The `List` structure is the fundamental data structure used to represent code and data. It is a singly linked list with two pointers: `next` points to the next item in the list, and `data` points to either a symbol or another list structure. `data` could be cast to either a `char *` or `List *`. To determine which one keep reading (spoiler: pointer tagging is used).

```c
  List *symbols = 0;
```

The global variable `symbols` represents the head of a list of symbols. When symbol is parsed, we'll look for it in the list of symbols, if it's not there we'll add it. This way we can compare two symbols by using the equals comparison operator, `==`. It saves a little bit of storage space when the same symbol is repeated many times in a LisP program, but with 8GB of RAM memory in my computer I probably won't notice the space saving.

```c
  static int look; /* look ahead character */
  static char token[32]; /* token */
```

Because a symbol can contain more than one character, we have a complete symbol when a character that doesn't belong in a symbol is encountered. Non symbol characters include whitespace (space, tab, newline etc), and syntax characters such as parenthesis, `(`, `)`. To determine whether the end of a symbol has been reached we need to look ahead by one character. The `look` variable stores the look ahead character. If this character contains a non-symbol character we'll know to stop reading the symbol.
The `token` variable is an array of characters, it stores the current symbol that has been read from the input. Note that it has a size of 32, so the maximum length of a symbol will be 31 characters, because the token is a NULL terminated string, so the token is always terminated with a `\0` character.


```c
  #define is_space(x)  (x == ' ' || x == '\n')
  #define is_parens(x) (x == '(' || x == ')')
```

The two macros above are really just a convenience for the sake of readability and possibly maintainability and extensibility of the program. `is_space` takes a single character and will return true if that character is a space or a newline. `is_parens` takes a single character and will return true if that character is a parenthesis.

```c
  static void gettoken() {
    int index = 0;
    while(is_space(look)) {
      look = getchar();
    }
    if (is_parens(look)) {
      token[index++] = look;
      look = getchar();
    } else {
      while(look != EOF && !is_space(look) && !is_parens(look)) {
        token[index++] = look;
        look = getchar();
      }
    }
    token[index] = '\0';
  }
```

The function `gettoken` is responsible for reading characters from standard input and determining whether parenthesis or a symbol has been discovered.
First it will skip over any whitespace. If the `look` variable, the look ahead character, is a parenthesis, it is stored in `token`, and the next character in the input stream read into `look`.
If the lookahead character is not a parenthesis, it's assumed to belong to a symbol. Keep looking ahead and saving the character until either `EOF` the end of the file is reached, or the look ahead character is whitespace, or the look ahead character is a parenthesis.
`index` stores the current position in the `token` array so it is incremented every time a character belonging to the symbol is stored. At the end the token is NULL terminated.

```c
  #define is_pair(x) (((long)x & 0x1) == 0x1)  /* tag pointer to pair with 0x1 (alignment dependent)*/
  #define untag(x)   ((long) x & ~0x1)
  #define tag(x)     ((long) x | 0x1)
```

Above contains a curiosity that can be found in many language implementations. Remember from the `List` structure that the `data` pointer can be either a `char *` a symbol, or `List *` another List.
The way we are indicating the type of pointer is by setting the lowest bit on the pointer on.
For example, given a pointer to the address `0x100200230`, if it's a pair we'll modify that pointer with a bitwise or with 1 so the address becomes `0x100200231`.
The questionable thing about modifying a pointer in this way is how can we tell a pointer tagged with 1, from a regular untagged address. Well, partly as a performance optimization, many computers and their Operating Systems, allocate memory on set boundaries. It's referred to as memory alignment, and if for example the alignment is to an 8-bit boundary, it means that when memory is allocated it's address will be a multiple of 8. For example the next 8 bit boundary for the address `0x100200230` is `0x100200238`. Memory could be aligned to 16-bits, 32-bits as well. Typically it will be aligned on machine word, which means 32-bits if you have a 32-bit CPU and bus. A more thorough discussion is on wikipedia [https://en.wikipedia.org/wiki/Data_structure_alignment](https://en.wikipedia.org/wiki/Data_structure_alignment).
Effectively for us it means that whenever we call `calloc` we'll always get back an address where the lowest bit is off (0), so we can set it on if we want.
The macro `is_pair` returns non-zero if the address is a pair (which means we'll need to unset the lowest bit to get the address). It uses a bitwise and with 1 to determine this. The `untag` macro switches the lowest bit off, with a bitwise and of the ones complement of 1. The `tag` macro switches the lowest bit on with a bitwise or of 1.
```c
  #define car(x)     (((List*)untag(x))->data)
  #define cdr(x)     (((List*)untag(x))->next)
```
There's two fundamental primitive operations in a typical Lisp/Scheme, `car` which returns the head of a list, and `cdr` which returns the tail of the list. They are named after operations on an IBM computer, some information on the history is on Wikipedia [https://en.wikipedia.org/wiki/CAR_and_CDR](https://en.wikipedia.org/wiki/CAR_and_CDR).
We could as easily call them head and tail, but since they are so ingrained in Lisp and Scheme conventions they are perpetuated here.
```c
  #define e_true     cons( intern("quote"), cons( intern("t"), 0))
  #define e_false    0
```
The `e_true` and `e_false` macros are a convenience for defining a what true and false are in this implementation. Basically so long as true is non-zero everything should be ok. It will help if the values they have can be readily printed in human readable form.

```c
  List * cons(void *_car, void *_cdr) {
    List *_pair = calloc( 1, sizeof (List) );
    _pair->data = _car;
    _pair->next = _cdr;
    return (List*) tag(_pair);
  }
```
Another fundamental Lisp/Scheme operation is `cons`. It constructs a pair, which means a pair of pointers, in this implementation the `List` structure that holds the `data` pointer and the `next` pointer. [https://en.wikipedia.org/wiki/Cons](https://en.wikipedia.org/wiki/Cons)
Because pointers to a `List` (a pair) must be tagged using the lowest bit, we rely on `calloc` to provide memory large enough to hold the `List` data structure and that the memory is aligned to an address that does not involve the lowest bit.
The `cons` function here takes two arguments, the first is an address that will be stored in the `data` field, and the second an address that will be stored in the `next` field.
Finally the address where the `List` structure is stored is returned, after being tagged as a special kind of pointer.

```c
  void *intern(char *sym) {
    List *_pair = symbols;
    for ( ; _pair ; _pair = cdr(_pair)) {
      if (strncmp(sym, (char*) car(_pair), 32)==0) {
        return car(_pair);
      }
    }
    symbols = cons(strdup(sym), symbols);
    return car(symbols);
  }
```
Here's where a symbol is retreived from the global list of symbols, or added if it is not found. It takes a single string argument. It uses `strncmp` to determine if anyone of the symbols are equivalent to the string passed in.
If we get to the end of the list of symbols and didnt find a match. The symbol is duplicated with `strdup` and added to the head of the list. This is the effect of `cons` when given an existing list as the second parameter: a new symbol is pushed onto the list, and a new list head is constructed.
The reason `strdup` is used, and the string is duplicated, is because we want a more permanent copy of the string. When the program runs, the `sym` parameter could be a pointer to the `token` global variable which will be modified as symbols are read from the input stream. The function is called `intern` out of convention, see [https://en.wikipedia.org/wiki/String_interning](https://en.wikipedia.org/wiki/String_interning) for more background on string interning.

```c
  List * getlist();
```
Above is a forward declaration of the function `getlist` which is defined further down. A forward declaration is needed because the `getobj` function can call it, and `getlist` can call `getobj` which is a chicken and egg kind of problem. The C compiler needs to know that the full signature of this function so it can be used before it is defined.

```c
  void * getobj() {
    if (token[0] == '(') return getlist();
    return intern(token);
  }
```
All `getobj` has to do is check if the current token from the input stream was an opening parenthesis, which means a list is being defined, and `getlist` can be called to construct the list.
Otherwise, the token is treated as a symbol, and `intern` is used to either return the single copy, or create a single copy and add it to the list of symbols.

```c
  List * getlist() {
    List *tmp;
    gettoken();
    if (token[0] == ')') return 0;
    tmp = getobj();
    return cons(tmp, getlist());
  }
```

The function `getlist` reads the next token from the input. If the token is a closing parenthesis it returns 0 (a NULL pointer).
Otherwise the token is probably a symbol, so call `getobj` and intern that symbol, then use `cons` to add that symbol to the head of the list, calling `getlist` recursively to get the tail of the list.
Take note that the variable `tmp` - an abbreviation of temporary - and explicity assigned to the return value of `getobj` before the `cons`. This is to ensure that the list is constructed in the correct order from head towards tail. Before the `cons` function is called, it's arguments are evaluated, and in this case it's second argument is a function call to `getlist`. So `getlist` is called again before `cons` is called, and either the end of the list (right parens) is discovered, or the next item in the list is.
How this recursive function call works is worthwhile understanding. In C, when functions are called, the arguments to the function, and the variables in the function are pushed on top of a data structure called a stack. A stack is literally a stack of things, like a stack of plates, where the last thing on top is the first thing that will come off. The arguments and variables to the function come off the stack when the function returns, literally where you see `return` in the code.
With every call to the `getlist` function as it comes across items in the list it is processing, the stack grows with another set of variables needed by `getlist`. So 3 recursive calls to `getlist` means the stack grows by 3 times the `getlist` functions storage requirements.
The inefficiency here is the longer the list, the taller the stack. Some programming languages have a stack overflow error where the stack has out grown the available memory. Wikipedia has a page about this [https://en.wikipedia.org/wiki/Stack_overflow](https://en.wikipedia.org/wiki/Stack_overflow)
Programming languages like Scheme implement something called tail call optimization where the language can determine if the variables used by a recursive function call will be needed after it returns and if not, it does not grow the stack.  This is a pretty cool feature of a programming language and it would be great to have in this language, and maybe we can add it later on. For more on tail calls, [https://en.wikipedia.org/wiki/Tail_call](https://en.wikipedia.org/wiki/Tail_call)

```c
  void print_obj(List *ob, int head_of_list) {
    if (!is_pair(ob) ) {
      printf("%s", ob ? (char*) ob : "null" );
    } else {
      if (head_of_list) {
        printf("(");
      }
      print_obj(car(ob), 1);
      if (cdr(ob) != 0) {
        if (is_pair(cdr(ob))) {
          printf(" ");
          print_obj(cdr(ob), 0);
        }
      } else {
        printf(")");
      }
    }
  }
```
The `print_obj` function is tremendously useful in that it can print either a symbol, or an entire list, to stdout so that we can read it. If the first argument, `object` isn't the specially tagged pointer, it's just a symbol so it can be output with `printf` using the `%s` format specifier, which says that the provided pointer is a null terminated string.
Otherwise `print_obj` is being asked to print a list, so `ob` will be the address of a `List` structure, meaning it is somewhere, either the beginning, middle or end, or printing a list. the `head_of_list` argument is the giveaway here. If `head_of_list` is non-zero, it's the beginning of a new list, so print the left parenthesis. In any case it has to print the value of the current item (it could either be a symbol or a nested listed) so it calls itself with the value of the current head of the list, `car(ob)`. If the tail of the list is non-zero, this means there's more, so as long as the tail of the list is a pointer to another `List` structure, print a space, and then print the tail of the list.
Otherwise, the tail of the list is zero, which means we're at the end of the list, so print the closing parenthesis.
```c
  List *fcons(List *a)    {  return cons(car(a), car(cdr(a)));  }
  List *fcar(List *a)     {  return car(car(a));  }
  List *fcdr(List *a)     {  return cdr(car(a));  }
  List *feq(List *a)      {  return car(a) == car(cdr(a)) ? e_true : e_false;  }
  List *fpair(List *a)    {  return is_pair(car(a))       ? e_true : e_false;  }
  List *fsym(List *a)     {  return ! is_pair(car(a))     ? e_true : e_false;  }
  List *fnull(List *a)    {  return car(a) == 0           ? e_true : e_false; }
  List *freadobj(List *a) {  look = getchar(); gettoken(); return getobj();  }
  List *fwriteobj(List *a){  print_obj(car(a), 1); puts(""); return e_true;  }
```
Above are defined the basic primitive operations required by Lisp, all using the same return value and argument specification.
These functions will be referenced in the interpreters environment so they can be used from a Lisp program.
Because the Lisp language we're implementing will know nothing about C and how many arguments and what type they should be in C, the arguments are represented using the linked list structure, which has an equivalent Lisp representation using parenthesis, whitespace and symbols.
These functions are prefixed with `f` which stands for function. They are called indirectly only when a Lisp program looks one up and wants to apply it.

```c
  List * eval(List *exp, List *env);
```
This is a forward declaraction of `eval` the meta-circular evaluator.

```c
  List * evlist(List *list, List *env) {
    List *head = 0, **args = &head;
    for ( ; list ; list = cdr(list) ) {
      *args = cons( eval(car(list), env) , 0);
      args = &( (List *) untag(*args) )->next;
    }
    return head;
  }
```

Above is the `evlist` function, short for "evaluate list". It takes a list and an environment, and evaluates each item in the list, returning a corresponding list with the evaluation of each input item, maintaining the order.
There is use of a pointer to a pointer here which makes this code less immediately obvious, but it means we can walk through the list, creating a parallel list with the evaluated elements in the same order.
In "The C Programming Language" by Brian Kernighan and Dennis Ritchie, a pointer is said to be a variable that contains the address of another variable. The `*` operator dereferences a pointer, giving the object pointed to. The `&` operator gives the address of a variable.
`evlist` iterates through the `list` argument in a for loop. Two local variables, a pointer, `head`, is initialized to 0, the purpose of `head` is to store the head of the list that will be returned. `args` is a pointer to a pointer, it is initialied to the address of `head`.
On each iteration, `args` is dereferenced and the resulting pointer is assigned to a newly constructed cell. On the next line, `args` is assigned to the address of the `next` field in that constructed cell. This means that on the next iteration, `args` is a pointer to a pointer to the `next` field of the previous element. When it is dereferenced with a single `*` and assigned, we are effectively setting the `next` field to point to the newly constructed cell in the current iteration.

```c
  List * apply_primitive(void *primfn, List *args) {
    return ((List * (*) (List *)) primfn)  ( args );
  }
```

The `apply_primitive` function does nothing more than cast the `primfn` to a pointer to a function that takes a single `List *` and returns a `List *`, and then calls that function with `args`.

```c
  List * eval(List *exp, List *env) {
    if (!is_pair(exp) ) {
      for ( ; env != 0; env = cdr(env) )
        if (exp == car(car(env)))  return car(cdr(car(env)));
      return 0;
    } else {
      if (!is_pair( car (exp))) { /* special forms */
        if (car(exp) == intern("quote")) {
          return car(cdr(exp));
        } else if (car(exp) == intern("if")) {
          if (eval (car(cdr(exp)), env) != 0)
            return eval (car(cdr(cdr(exp))), env);
          else
            return eval (car(cdr(cdr(cdr(exp)))), env);
        } else if (car(exp) == intern("lambda")) {
          return exp; /* todo: create a closure and capture free vars */
        } else if (car(exp) == intern("apply")) { /* apply function to list */
          List *args = evlist (cdr(cdr(exp)), env);
          args = car(args); /* assumes one argument and that it is a list */
          return apply_primitive( eval(car(cdr(exp)), env), args);
        } else { /* function call */
          List *primop = eval (car(exp), env);
          if (is_pair(primop)) { /* user defined lambda, arg list eval happens in binding  below */
            return eval( cons(primop, cdr(exp)), env );
          } else if (primop) { /* built-in primitive */
            return apply_primitive(primop, evlist(cdr(exp), env));
          }
        }
      } else { /* should be a lambda, bind names into env and eval body */
        if (car(car(exp)) == intern("lambda")) {
          List *extenv = env, *names = car(cdr(car(exp))), *vars = cdr(exp);
          for (  ; names ; names = cdr(names), vars = cdr(vars) )
            extenv = cons (cons(car(names),  cons(eval (car(vars), env), 0)), extenv);
          return eval (car(cdr(cdr(car(exp)))), extenv);
        }
      }
    }
    puts("cannot evaluate expression");
    return 0;
  }
```
The `eval` function is the heart of LiSP. It interprets LisP expressions.
If the expression is not a pair (not a `List` structure), we look for that value it is associated with in the environment. In other implementations of eval, the equivalent test is if the expression is an `atom`.
Otherwise the expression must be a list, and then the first element of that list is checked, if that first element is not a `List` structure - it is a symbol, or more officially an atom, then the following series of if statements handle it: if the first element is a `quote` symbol, the next element is return, that is, the head of the tail of the list; if the first element is an `if` symbol, the head of the tail of the list is evaluated, if that returns non-zero, the head of the tail of the tail of the list is evaluated and returned, if it returns zero, the head of the tail of the tail of the tail is evaluated and returned.
If the first element is the symbol `lambda` the expression is simply returned (maybe this is redundant so may indicate a bug or some optimization that is missing). In a Scheme interpreter, a closure would be created and the free variables in the closure captured using the current environment.
If the first symbol is `apply` that means, in this interpreter at least, that the next element is a function and the element after that, the third element in this list is a list - the `(b c d)` in `(apply a (b c d))`. The assumption is that `apply` is being used to call one of the basic primitive operations defined above: `car`, `cdr`, `cons`, `eq?`, `pair?`, `symbol?`, `null?`, `read`, `write`.
If the first symbol did not match any of the prior if statements, we assume a the first symbol is in the environment and is either a user defined function - a lambda, or a primitive function (and apply is not being used to call it). We find out which it is by evaluating that first element, if it's a pair, it's a list, i.e. an expression in the form `(lambda (arg) (body expressions ...))`. If it's not a pair we assume it's a pointer to a function, and use `apply_primitive` to invocate that function, evaluating it's arguments before calling it.
The remaining block is the `else` which meant the first argument in the expression was a pair - eval was called with a list nested inside a list, i.e. `((x y z))`, and the only form of nested expression handled, is lambda, e.g. `((lambda (arg) (body expr ...)) value )`.
In this case the names of the arguments in the lambda definition are bound to the corresponding values, and the name value pairs are pushed onto the head of the environment, until there are no more arguments (names) left to bind. The body of the lambda is then evaluated with the  extended environment.

A newer article describing eval is called "The Roots of Lisp" by Paul Graham, and can be downloaded from [http://www.paulgraham.com/rootsoflisp.html](http://www.paulgraham.com/rootsoflisp.html)
A thorough explanation can be found in "Structure and Interpretation of Computer Programs", by Harold Ableson and Gerald Jay Sussman. This book can be found online: [https://mitpress.mit.edu/sicp/full-text/book/book-Z-H-26.html#%_sec_4.1](https://mitpress.mit.edu/sicp/full-text/book/book-Z-H-26.html#%_sec_4.1)
The earliest implementation of eval I have found is in the Lisp 1.5 Programmers Manual.

```c
  int main(int argc, char *argv[]) {
    List *env = cons (cons(intern("car"), cons((void *)fcar, 0)),
                cons (cons(intern("cdr"), cons((void *)fcdr, 0)),
                cons (cons(intern("cons"), cons((void *)fcons, 0)),
                cons (cons(intern("eq?"), cons((void *)feq, 0)),
                cons (cons(intern("pair?"), cons((void *)fpair, 0)),
                cons (cons(intern("symbol?"), cons((void *)fsym, 0)),
                cons (cons(intern("null?"), cons((void *)fnull, 0)),
                cons (cons(intern("read"), cons((void *)freadobj, 0)),
                cons (cons(intern("write"), cons((void *)fwriteobj, 0)),
                cons (cons(intern("null"), cons(0,0)), 0))))))))));
    look = getchar();
    gettoken();
    print_obj( eval(getobj(), env), 1 );
    printf("\n");
    return 0;
  }
```

`main` is the entry point for this program when it is run. It has one variable, `env` which is assigned to a list of lists, effectively just associating a symbol with a primitive function.
The remaining lines, look ahead one character, load the first token with `gettoken`, and then print with `print_obj`, the evaluated object read by `getobj`.

That is it a very small and incomplete interpreter... Noticeably there is no garbage collection, or even any explicit free of the memory allocated by `calloc`. Neither is there any error handling, so a program with missing or unmatched parenthesis, unresolved symbols, etc will likely just result in something like a segmentation fault.

Despite the limitations, this interpreter provides enough primitive functions to implement an equivalent eval on itself.

The complete source code and some tests can be found at [https://github.com/carld/micro-lisp](https://github.com/carld/micro-lisp)

An implementaion of eval that runs on the interpreter above can be found in `repl.lisp`. It implements a Read Eval Print Loop and it can be run using:

`cat repl.lisp - |./micro-lisp`

