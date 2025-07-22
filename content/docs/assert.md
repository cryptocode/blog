---
title: "Zig asserts are not C asserts"
date: 2025-07-22T20:32:37+02:00
draft: false
---

I recently came across a piece of code in a Ziggit.dev post that gave me pause:

```zig
pub fn putOne(q: *@This(), io: Io, item: Elem) Cancelable!void {
    assert(try q.put(io, &.{item}, 1) == 1);
}
pub fn getOne(q: *@This(), io: Io) Cancelable!Elem {
    var buf: [1]Elem = undefined;
    assert(try q.get(io, &buf, 1) == 1);
    return buf[0];
}
```
...which lead me to ask the following:
> Just a quick side quest: Doesn’t the assert here risk `put` and `get` calls being optimized away? If not I think I might have misunderstood std.debug.assert’s doc comment and could use some education.
>
> Since assert is a regular function, the argument is evaluated (and is indeed in debug/safe builds), but since the expression is assumed to be true (otherwise unreachable) it seems like the whole expression is allowed to be removed
>
> Is there a difference whether the expression is fallible or not, or is deemed to have side effects?

The nice thing about [Ziggit](https://ziggit.dev) is that core team members frequently chimes in with their expertise, and that was the case here as well. So I figured I'd try to summarize my understanding of Zig assertions based on those posts.

The short answer to my question is: no, the *put* and *get* calls will definitely *not* get optimized away. We'll see why in a bit.

## std.debug.assert and unreachable

If you've ever hit an assertion in Zig, then you have also looked at the implementation of `std.debug.assert` since it appears in the panic trace:

```
thread 33583038 panic: reached unreachable code

lib/std/debug.zig:550:14: 0x10495bc93 in assert (sideeffects)

if (!ok) unreachable; // assertion failure
```

That's all assert does: `if (!ok) unreachable;`

* If *unreachable* is hit in safe modes, then... well, you've seen the panic trace. Very helpful.

* In optimizing modes, *unreachable* becomes a promise that control flow will not reach this point at all. Also very helpful: faster code!

Here's the doc comment on `assert` that helped myself and some others get confused, despite the comment being 100% verifiably correct:

> In ReleaseFast and ReleaseSmall modes, calls to this function are optimized
> away, and in fact the optimizer is able to use the assertion in its
> heuristics.

On closer inspection, this is just what the language reference entry on `unreachable` promise us. No more, no less.

This is very different from C's assert which nukes the whole thing through macros and preprocessor directives. It's a similar story in many other languages. They have special constructs for asserts. Zig does not; `std.debug.assert` is a plain old function for which no special treatment is given.

The idea that `if (!ok) unreachable;` somehow magically wires up the optimizer to always delete "the whole thing" is wrong.

## Does this mean asserts can be expensive even in ReleaseFast mode?

Yes, because while the call to assert is gone, the LLVM optimizer that's supposed to remove dead code isn't always able to do so. Bugs, limitations and side effects. An expensive check passed as an argument to assert *may* remain. Simple expressions like `data.len > 0` will almost certainly be
optimized out, but it's less clear for anything non-trivial.

I shared an example in the Ziggit thread where dead code removal does *not* happen. Here's an improved version by *TibboddiT*:

```zig
const std = @import("std");

fn check(val: []u8) bool {
    var sum: usize = 0;
    for (0..val.len * 500_000_000) |v| {
        sum += val[v % val.len];
    }

    return sum == 6874500000000;
}

pub fn main() void {
    var prng: std.Random.DefaultPrng = .init(12);
    const rand = prng.random();

    var buf: [100]u8 = undefined;
    rand.bytes(&buf);

    std.debug.assert(check(&buf));
}
```

Compile and run this under ReleaseFast on Zig 0.14.x and you'll see that the program is busy for a good while. The core team believe this is a missed optimization in LLVM.

If profiling shows that an assertion is expensive, or you're just not confident it will be fully elided, you can do something like this:

`if (std.debug.runtime_safety) std.debug.assert(check(&buf));`

...or check against build modes when that makes more sense.

### When the optimizer will definitely not remove code

Now back to the original question, which is about the opposite of trying to get rid of dead code. We want to *keep* code.

There are many reasons why code will never be removed by a correctly implemented optimizer. One of them is the presence of side effects. Another example is when writes to memory must be observable when that memory is later read. Basically, the optimizer's rule is that code removal must not lead to correctness bugs.

The *put* call in `assert(try q.put(io, &.{item}, 1) == 1);` have side effects *and* depends on memory coherence as there's a *get* call elsewhere. We're all good.

By the way, Andrew Kelley shared Zig's concrete list of side effects:

> * loading through a volatile pointer
> * storing through a volatile pointer
> * inline assembly with volatile keyword
> * atomics with volatile pointers
> * calling an extern function
> * @panic, @trap, @breakpoint
> * unreachable in safe optimization modes (equivalent to @panic)

There isn't really anything to worry about here beyond perhaps LLVM optimization bugs.

# Conclusion:

* `std.debug.assert(expr)` is nothing more than `if (!expr) unreachable` where *unreachable* yields a helpful trace in safe build modes, and provides the optimizer with useful information in unsafe builds
* The optimizer may or may not be able to optimize away `expr`
* The optimizer will never optimize away `expr` if doing so would lead to correctness issues

I'll round this off with some wise words from ifreund [on the issue](https://github.com/ziglang/zig/issues/10942) if Zig should match C's assert behavior:

> I think trying to match C's assert is exactly what we should not do. I've seen many bugs caused by putting expressions with side effects inside the assert macro.