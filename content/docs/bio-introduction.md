---
title: Bio - All your parentheses are belong to us
description: A little Lisp written in Zig
date: 2021-04-25T10:00:00+02:00
---

![Header](/blog/images/bio-header.png)

[In a previous Zig post]({{< ref "prefix-calculator" >}} "previous Zig article"), we wrote a small *postfix* notation calculator.

This post takes a look at something that take *prefix* notation to the extreme: a new language in the Lisp family called [Bio](https://github.com/cryptocode/bio)

The goal of this project is to make it easy to play around with Lisp dialect ideas and to make a well-documented and readable Lisp interpreter for others to learn from. This article aims to point out some key areas people stumble on when writing a basic Lisp interpreter.

*Acknowledgments: Bio is written in [Zig](ziglang.org), a fact I think awesome designer [Joy Machs](https://artstation.com/joymachs) was able to neatly incorporate into the lambda/lisp oriented logo. I uncovered a [stdlib bug](https://github.com/ziglang/zig/issues/8454) while working on this; thanks to [jumpnbrownweasel](https://github.com/jumpnbrownweasel) for fixing it quickly.*

{{< table_of_contents >}}
## Scope

This initial iteration gives us a language with:

* Higher-order functions with lexical scoping and enough juice to implement recursion in the language itself
* The ability to create composite data types with polymorphic behavior.
* Macros
* Garbage collection and atom interning
* An error handling mechanism
* Tail call elimination
* A module system
* A language reference
* A small but extensible standard library written in Bio, with functions like filter, map, and quicksort
* A REPL and a couple of ways to load and evaluate source files

## Examples

Here's a few sample Bio expressions:

```scheme
; Define a variable, note that (define) works as well
(var x 5)

; Define a couple of functions (one using λ and π symbols) and call them
(var square (lambda (x) (* x x)))
(var circumference (λ (r) (* 2 π r)))

(print "square is:" (square x) ", circumference is:" (circumference x))

; Define a macro. Arguments are not evaluated until needed by the macro expansion
(var if-above-10 (macro (val then else)
    `(if (> ,val 10) ,then ,else)
))

; Prints aye
(if-above-10 12 (print 'aye) (print 'nay))

; Filter out negative numbers from a sorted list => (1 2 5 40)
(filter 
    (quicksort '(5 40 1 -3 2) <) 
    (λ (x) (>= x 0)))

; Double every element in a list => (2 4 6)
(map (λ (x) (* 2 x)) '(1 2 3))

; Summation through zipping lists => (2 6 11)
(map + '(0 2 5) '(1 2 3) '(1 2 3))

; Add ten to every item and print it
; Note that "each" is just a standard library function
(var mylist '(1 2 3))
(each mylist (lambda (x) 
    (print (+ x 10) ", ")))

; Tail-call elimination makes it possible to make 
; any loop construct you want. 
; Here's the while loop from the standard library in action:
(var c 0)
(while (< c 10)
    (print "Value is now " c "\n")
    (inc! c)
)

; Read two numbers. If the second one is zero, this will
; fail with "Division by zero"
(try (math.safe-div (io.read-number) (io.read-number))
    (print "The result is: " #value)
    (print "Failed: " #!)
)

; Import a point module. By convention, module variables are Pascal cased.
(var Point (import "examples/mod-point.lisp"))

; Make a position (a composite type), where 
; new-point and new-location are just functions
(var mypos (Point new-point 61.5 42.2))
(var new-york (Point new-location 40.7554351 -73.9981619))

; Update using applicative syntax
((mypos update) 17.2 20.5)

; Update again, now using function call syntax but in a different environment
(mypos (update 27.2 20.5))

; Print the location by accessing the x and y variables of the position
(print "Moved to:" (mypos x) (mypos y) "\n")

; Print the location using the simple as-string version
(print "Located at" ((mypos as-string)) "\n")

; Print the location as longitude/latitude => "40.7554351° N  73.9981619° W"
(print "Located at" ((new-york as-string)) "\n")

; Introspection => "number, #t, #t"
(print (string (typename math.pi) ", " (symbol? 'a) ", " (callable? +)))

; Let's implement factorial in the fanciest way imaginable with lots of 
; lambda symbols and a fixpoint combinator. The standard library also 
; contains a direct-recursive version.
(var Z (λ (f) ((λ (g) (g g)) (λ (g) (f (λ (a) ((g g) a)))))))
(var ! (Z (λ (r) (λ (x) (if (< x 2) 1 (* x (r (- x 1))))))))

; Does it work?
(assert (= 120 (! 5)))
```

Source code and the language reference is available here: https://github.com/cryptocode/bio

Bio is work in progress; the [article-source](https://github.com/cryptocode/bio/releases/tag/0.1) release contains the version that existed at the time of writing this article.

## Using Bio

To build, clone the repository and run `zig build`

You can now use Bio in two ways:

1. Start the REPL with `./bio`
2. Run a program with `./bio run myprogram.lisp`

There are some examples you play around with, like this one that prints a Sierpinksi triangle. You can change the size by editing the source file.

```
./bio run examples/triangles.lisp

               ^
              ^ ^
             ^   ^
            ^ ^ ^ ^
           ^       ^
          ^ ^     ^ ^
         ^   ^   ^   ^
        ^ ^ ^ ^ ^ ^ ^ ^
       ^               ^
      ^ ^             ^ ^
     ^   ^           ^   ^
    ^ ^ ^ ^         ^ ^ ^ ^
   ^       ^       ^       ^
  ^ ^     ^ ^     ^ ^     ^ ^
 ^   ^   ^   ^   ^   ^   ^   ^
^ ^ ^ ^ ^ ^ ^ ^ ^ ^ ^ ^ ^ ^ ^ ^
```

Here's a persistent album database. It persists by saving Bio expressions to a file and then evaluating them on startup:

```scheme
./bio run examples/albums.lisp

Welcome to the Bio Album Database

1: Add    2: Find by name    3: List all    4: Exit
```

## What kind of Lisp flavor is it?
It's a single namespace Lisp inspired by Scheme. Scheme, like Lisp itself, is a family of dialects, and Bio is yet another one. Some of the differences come from ideas I want to explore, such as the ability to treat environments as expressions, which is the building block for modules and composite data types.

The best way to learn Bio is to play around with it, type (env) in the REPL for a list of functions, and consult the language reference. There are also some example files in the repository.

## The floor plan
The Bio source code is broken into its main areas of concern, thus filenames should be self explanatory: gc.zig contains the garbage collector, ast.zig the expression AST, intrinsics.zig contains the intrinsics, etc.

### Dealing with intrinsics
A decent chunk of the code base is the implementation of *intrinsic* functions and symbols, such as `stdList` to implement `(list 1 2 3)` and `#t` for the truth symbol. All these functions are prefixed "std" and have the same signature:

```zig
fn std<intrinsic>(ev: *Interpreter, env: *Env, args: []const *Expr) anyerror!*Expr
```

This signature fits the `ExprValue` union type for intrinsic functions.

### The reader and evaluator
The `Interpreter` struct implements the logic to read and evaluate expressions. When initialized, a root environment is created and all the intrinsic symbols and functions are registered with it. This way, when your Bio program calls `(list 1 2 3)`, the `list` function is readily available in the root environment.

In a Lisp interpreter, we want to move from the world of text to the world of *s-expressions* as quickly as possible.

First order of business is to split expressions like `(var x (* 2.45 scale))` into a stream of tokens like `(, var, x, (, *, 2.45, scale, ), )`

If we required the programmer to always put spaces around parentheses, we could use the `std.mem.split` function. But that wouldn't be very friendly, so we're going to extend the idea of the *Spliterator* into a token iterator for Lisp expressions. We can now easily iterate over tokens using `it.next()`

The `read()` function uses the Lisperator and produces exactly one expression: an `atom` or a `list`.

Of course, a list can contain other lists and atoms, so `read` works recursively.

An atom is either a `symbol` or a `number`. Some symbols are predefined, like `#t` and `#f` for true and false. Symbols can also be strings, like `"This message"`

The whole REPL process goes like this:

* readBalancedExpr, which reads a complete expression that may span multiple lines.
* parseAndEvalExpression is then called, which in turn calls `parse` and `eval`
    * parse is a small helper to set up the Lisperator, and call `read` with it. The `read` function recursively parses the s-expression.
    * eval recursively evaluates the expression and returns the result, which the REPL prints.


## Intrinsic functions

Rather than implementing all basic Lisp functions in Zig, we'll implement a handful of general functions from which a Bio standard library can be built. For example, all functions querying lists, such as `car` and `cdr`, are built from an intrinsic `range` function.

The standard library is located in `std.lisp`, which is loaded on startup. For example, here's how `car` and `cdr` are defined in terms of `range`:

```scheme
(var car (λ (list) (range list 0 1)))
(var cdr (λ (list) (range list 1)))
```
Note that `λ` is a synonym for `lambda` - you can use either. The λ version is less verbose when passing function literals to functions like `filter` (many editors have plugins to  automatically convert certain identifiers to symbols, like Symbol Complete for vscode)

## Logical and relational functions, or "I can do that already?"

Everyone who implements a Lisp soon reaches this magical point where not only *can* you start implementing the language in itself, you start preferring it! This is not to say the host language is bad, it's just that if you can extend your Lisp using Lisp, that makes everything easier. It enables you to experiment in the REPL, and you don't have to worry about low-level details like memory management.

As an example, here's how `or`, `<`, and `<=` are implemented in the standard library:

```lisp
(var or (lambda (x y) (if x #t (if y #t #f)) ) )
(var < (lambda (x y) (if (= (order x y) (- 1)) #t #f)))
(var <= (lambda (x y) (if (or (< x y) (= x y) ) #t #f)))
```

So first we define `or` as a function, then we use it to implement `<` using the `order` intrinsic, which finally allows us to implement `<=`.

Functional composition is great!

*Alas, time for a reality check! Implementing `or` as a function is wrong here because both arguments are always evaluated. We'll fix this as soon when we get to macros.*

## Scopes

Early Lisp dialects had *dynamic scoping*, and this is still the default in Emacs Lisp.

Like most languages these days, Bio is *lexically scoped*. What's the difference anyway?

First of all, if a variable is not **bound** to a formal argument or a local definition, it must be a **free** variable defined elsewhere. How exactly should Bio "look elsewhere"? This is where environments and their parents come in. 

Every lambda executes in its own environment, containing arguments and local variables. An environment is just a hashmap from the symbol literal to the value.

`(var x 5)` is essentially the addition of a hashmap entry in the current environment. The same is true when passing arguments.

Every environment, except the initial root environment, also has a *parent* environment.

* If the parent environment is the callers' environment, we have **dynamic scoping**
* If the parent environment is the environment that existed *when the lambda was defined*, we have **lexical scoping**

The last point tells us exactly what we need to do in the implementation:

* When we **define** a lambda, store the **current** environment in the lambda expression so we can look it up when evaluating the lambda.
* When we **evaluate** the lambda, make a brand new environment, add arguments to that environment (using the formal parameter names as key), and then make the parent environment be whatever the current environment was when the lambda was defined.

## Macros

*"One can even conjecture that Lisp owes its survival specifically to the fact that its programs are lists, which everyone, including me, has regarded as a disadvantage."
    -- John McCarthy*

I mentioned earlier that we can't implement `or` using a regular function, because all arguments are evaluated before the function is evaluated. Logical operators are supposed to use shortcut evaluation.

Implementing `or` as a macro will fix the problem:

```scheme
(var or (macro (expr1 expr2)
    `(if ,expr1
        #t
        (if ,expr2 #t #f)
    )
))
```

While the use and implementation of macros are similar to lambdas, there are some differences. These differences are key to unleashing the power of Lisp meta-programming:

* The macro arguments are not evaluated when calling the macro. It's up to the body to decide if and when to evaluate them.
* The parent environment is different. For lambdas, the parent environment is the environment that existed when it was defined. This gives us lexical scoping. But that's not what we want for macros; we kind of want to "paste" the expanded body into the very place it's being called. For this reason, the parent environment for a macro is simply the *current* one (it's not *quite* "pasting" though, since the macro has its own local environment; this helps avoid symbol clashes.)
* The last expression of the macro body (the result of evaluating the macro) is evaluated again! The first evaluation typically causes a quasiquote to be expanded into code. The second evaluation then evaluates that code.

Macros are Bio functions that write Bio code, enabling you to mutate the language to fit your needs. In Lisp, code is lists which we can manipulate with the full power of the language, where other languages usually have to degrade to something like building strings.

### Quasiquoting and unquoting

The `or` macro above uses quasiquoting to help write code. It's not strictly necessary, but it's usually more readable than producing lists and quoting manually.

Quasiquoting is like quoting, except that it allows select expressions to be unquoted. In terms of implementation, it's important to note that `unquote` and `unquote-splicing` are always executed inside a `quasiquote`.

Example:

```scheme
(var x 4)
(var mylist '(1 2 3))

`(,@mylist ,x)
(1 2 3 4)
```

If you call `(verbose)`, any quasiquote expansions will be printed.

## Garbage collection

As part of devising Lisp, John McCarthy also described a clever method for automatic memory management: garbage collection. Lisp programmers detest manual memory management, so I guess we'll have to implement it.

To prove that it works, the interpreter allocates from Zig's GeneralPurposeAllocator with leak detection turned on. 

Here are the main steps:

* When we allocate an expression, we register it with the GC
* If we don't register it with the GC, it's a *pinned* expression, such as the predefined `nil` expression; this is static data we don't need to deallocate.
* Every 100K allocation, we run the garbage collector, but it can also be run manually with `(gc)`. This will first mark all reachable expressions. Then it runs through all registered expressions. If it's not marked, we can deallocate it. By reachable, I mean that the expression is either a key or a value in some environment. If it's a key, it's always a symbol expression (denoting a variable name). We also need to sweep environments. The marking step starts at the root environment and calls `markEnvironment` on it. This will loop through all entries and mark the environment associated with the expression. For lambdas, this may be something different than the root environment, because another lambda may have returned it. We also mark any parents of reachable environments. Finally, we destroy any unmarked environments.

There's a lot of simple optimizations that can be done here, such as not using a container for marking, and only doing partial sweeps to reduce GC pauses.

## Error handling

Bio implements a simple error handling mechanism through the `try` and `error` functions.

```scheme
(try (math.safe-div (io.read-number) (io.read-number))
    (print "The doubled result is: " (* 2 #value))
    (print "Failed: " #!)
```

The `try` function is very similar to `if`: the first branch is evaluated if the function succeeds, otherwise the second (optional) branch is evaluated. The `#value` symbol contains the result of the tried expression, while `#!` contains any error expressions. Error expressions are produced with the `(error <expr>)` function.

For more information, see the Reference section on GitHub.

## Tail call elimination

Lisp programmers just love recursion. In fact, some of them love it so much that they make Lisp dialects without loop constructs! Bio macros such as `while` simply expand to a tail-recursive function, and these can fortunately be made efficient. While Bio offers a loop intrinsic, it isn't strictly necessary.

For Bio to support recursion well, we have to implement tail call optimization, or TCO. Without it, the evaluator would keep calling itself through indirection recursion, blowing up the stack in a jiffy.

The solution is conceptually simple: the body of `eval` in Zig is a loop. When `eval` evaluates, say, `if`, then the selected branch will become the next expression to evaluate. In the case of lambdas, the last expression becomes the next expression in the eval loop, and in this case we also have to change the loop's current environment to the lambda environment. The same is true for environment expressions, such as `(pos (update 3 4))`

I think the easiest way to see how this works is to fire up `zig run bio.zig` in a debugger, then step through `eval` after pasting a minimal recursive Bio function in the REPL.


## VSCode configuration
- Install a Lisp syntax plugin
- Add a run task for the currently open lisp file in `tasks.json`:

```
{
    "label": "bio run",
    "type": "shell",
    "command": "${workspaceFolder}/bio run ${file}",
    "problemMatcher": [],
    "group": {
        "kind": "build",
        "isDefault": true
    }
},
```

## Reading list
1. http://jmc.stanford.edu/articles/lisp/lisp.pdf
2. http://jmc.stanford.edu/articles/recursive/recursive.pdf
3. A different take on implementing Lisp in Zig can be found in the [Mal repository](https://github.com/kanaka/mal/tree/master/impls/zig) (not mine)*
4. A [comptime Lisp](https://github.com/igmanthony/zig_comptime_lisp/)
5. A correct [quasiquoting algorithm](https://3e8.org/pub/scheme/doc/Quasiquotation%20in%20Lisp%20(Bawden).pdf) in appendix A

## Going forward (feel free to contribute or fork the project)

* Add arbitrary precision number support, ideally a Scheme-like numerical tower system
* A more advanced macro system, including reader macros
* GC optimizations
* Expand the standard library
* Improve error handling by recording more context
* Integrate libffi or similar so std lib can do anything
* A compiler targeting a vm or maybe Zig stage2 ir
