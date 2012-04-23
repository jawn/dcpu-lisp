LCPU is a simple Lisp-like language that compiles to optimized DCPU-16 assembly code. It is a very restricted subset of Lisp. There is no GC, and thus no data structures or run-time closures.

This is intended for usage in Mojang's next game, [0x10c](<http://0x10c.com/>).

* Example program: https://github.com/jlongster/dcpu-lisp/blob/master/examples/print-number.l
* Generated assembly: https://github.com/jlongster/dcpu-lisp/blob/master/examples/print-number.asm
* Watch it run here: http://0x10co.de/imq63

# High-level Assembly

Think of it as a light wrapper around the assembly code, providing named variables, lexically-bound closures, and a little bit of helpful magic here and there.

Create a function like so:

    (define (foo x)
      (+ x 1))

Variable references are statically referenced and compiled out straight to registers. In one function, you are not allowed more variable definitions than there are registers (7, and the 8th is used for the return value).

Nested functions are allowed:

    (define (foo x)
      (define (bar y)
        (+ x y))
      (bar 10))

Run-time closures are not available because of the lack of GC, so the closed function cannot out-live its parent (you can't return `bar`).

Only 2 kinds of values exist: functions and 16-bit numbers.

There is a lot more we could do to enrich the language but still keep provable semantics statically.

It is written in Outlet, a Lisp that compiled to Javascript, so the compiler can run in browser. See https://github.com/jlongster/outlet. The js is packaged with the project so you can run it with Node.

# Features

* Named functions and variables
* Nested functions
* Inlinable arithmetic expressions
* Inline assembly
* Macros
* Standard library for allocation, I/O, etc.
* Probably other stuff I've forgotten about

# Does Not Have

* Garbage Collector
* Heap
* Run-time closures
* Data types/structures
* Type inference

# Usage

Use the `lcpu` script in the `bin` directory:

`./bin/lcpu program.l`

It will print the generated assembly to standard output.

    lcpu [-p] [-c1] [-c2] [-c3] [-l] [-e] <program/expression>

    * -p: print the expanded program
    * -c1: print the code after the first compilation phase
    * -c2: print the code after the second compilation phase
    * -c3: print the code after the third compilation phase
    * -l: print the code after the linearization phase
    * -e: run an expression instead of a file

If you get an error, you may have to run this: `touch compiler.ol && make`

# Standard Library

Eventually, LCPU will target a specific operating system or programming environment in the DCPU-16 machine. I'm not interested in writing and maintaining custom memory allocation, I/O, and other libraries.

Unfortunately, DCPU-16 is so new we don't really have any of those yet. However! A few awesome people have started to write them, along with libraries for things like allocation and I/O.

We've started bundling these basic libraries under a single project called [stdlib](https://github.com/0x10c-dev/stdlib). LCPU comes with this bundled in, so you can use it straight out of the box. The included procedures are:

* ``(malloc owner length)``
* ``(free ptr)``
* ``(getline)``
* ``(putc)``

NOTE: I haven't added the interfaces by default yet, but view the [console example](https://github.com/jlongster/dcpu-lisp/blob/master/examples/console.l) to see how to use them.

# Examples

You can see all the examples in the [examples](https://github.com/jlongster/dcpu-lisp/tree/master/examples>)
directory.

## Number Printing

This code defines `print-number` which prints a number to the console:

    (define (print color bg-color x y text)
      (MUL y 32)
      (ADD y x)
      (ADD y 0x8000)
      (SHL color 12)
      (SHL bg-color 8)
      (BOR text color)
      (BOR text bg-color)
      (SET [y] text))

    (define (%print-number n i)
      (define (p n)
        (print 0xf 0 (- 31 i) 0 (+ 0x30 n)))

      (if (< n 10)
          (p n)
          (begin
            (p (% n 10))
            (%print-number (/ n 10) (+ i 1)))))

    (define (print-number n)
      (%print-number n 0))

    (print-number 12345)

## Fib

Here is the fib program and the resulting assembly code (without the runtime, which just provides a few helpful functions). You can get the full assembly code in examples/fib.asm.

    (define (fib a)
      (if (<= a 1)
          1
          (+ (fib (- a 1))
             (fib (- a 2)))))
    
    ;; result will be in register J
    (fib 8)

Generated assembly:

    JSR global_dash_entry
    SET PC, __exit

    :global_dash_entry
    SET PUSH, return_dash_o1957346
    SET PUSH, 0x8
    SET PC, global_dash_entry_dash_fib
    :return_dash_o1957346
    SET PC, POP
    :global_dash_entry_dash_fib
    SET A, POP
    SET PUSH, A
    SET PUSH, return_dash_o3554470
    SET PUSH, A
    SET PUSH, 0x1
    SET PC, global_dash__lt__eq_
    :return_dash_o3554470
    SET A, POP
    IFE J, 0x0
    SET PC, alt_dash_o5960250
    SET J, 0x1
    SET PC, exit_dash_o9848488
    :alt_dash_o5960250
    SET PUSH, A
    SET PUSH, return_dash_o7693500
    SET PUSH, A
    SET PUSH, return_dash_o6022101
    SET J, A
    SUB J, 0x1
    SET PUSH, J
    SET PC, global_dash_entry_dash_fib
    :return_dash_o6022101
    SET A, POP
    SET PUSH, J
    SET PUSH, A
    SET PUSH, return_dash_o8008109
    SET J, A
    SUB J, 0x2
    SET PUSH, J
    SET PC, global_dash_entry_dash_fib
    :return_dash_o8008109
    SET A, POP
    SET PUSH, J
    SET PC, _plus_
    :return_dash_o7693500
    SET A, POP
    :exit_dash_o9848488
    SET PC, POP
    :__exit
    SET PC, __exit

You can also view the tests in the `tests` directory to see how certain expressions are compiled.

# Inline Assembly

If you want, you can code straight DCPU-16 assembly into your program. For example, here is a function that prints values to the console:

    (define (print color bg-color x y text)
      (MUL y 32)
      (ADD y x)
      (ADD y 0x8000)
      (SHL color 12)
      (SHL bg-color 8)
      (BOR text color)
      (BOR text bg-color)
      (SET [y] text))

Dereferencing is supported with the normal bracket syntax (i.e. `[y]`).

# Macros

`define-macro` is provided for defining macros:

    (define-macro (foo t x y)
      `(begin
         (define ,t (+ ,x ,y))
         (MUL ,t 50)))

    (foo z 1 2)

is converted into:

    (begin
      (define z (+ 1 2))
      (MUL z 50))

This is a powerful construct to make sure you can generate optimized assembly code.

# Iteration

The `do` construct provides iteration. There are two versions of `do`:

    ;; Runs the expression with x starting at 0 and incrementing by 1
    ;; until it hits 32
    (do (x 0 32)
        (print (* x 20)))

    ;; Or you can provide your own start, stepping and stopping
    ;; expressions.
    ;; Here x starts as 0, is incremented by 5 and continues looping while
    ;; x is less than 100
    (do (x 0 (+ x 5) (< x 100))
        (print x)
        (print (/ x 2)))


# Future work (ABI refactoring, etc)

## ABI

I made up my own ABI, since I haven't really written assembly before. Now that I know more about it, and the 0x10c community seems to be converging on a [standard](https://github.com/0x10cStandardsCommittee/0x10c-Standards/tree/master/ABI), I need to refactor how I store data and pass it around.

Luckily, you can just write code and not worry about all of that. When it changes, you'll just have to recompile.

## Debugging

It's pretty painful to debug the code right now (although, its pretty painful to debug any assembly code). This could be another *huge* benefit to using LCPU: write an interpreter which runs code and lets you debug it on the fly. You could debug it easily and then compile it down when you're ready.

## Optimizations

There are many more static optimizations we could do. I'm sure there are bugs in this too, as it is rather untested. Please report issues on github if you find any, or contact me at longster@gmail.com.

Follow me on twitter: [@jlongster](http://twitter.com/jlongster)
