---
title: Some notes on using Zig with Valgrind
published: true
date: 2021-04-25T09:00:00+02:00
---

Zig has a very nice built-in memory leak detector in its General Purpose Allocator (GPA), but sometimes you have to break out Valgrind to get to the bottom of things.

When I first did this, I ran into a couple of problems, which I'm jotting down here along with solutions.

**Thanks to mikdusan, ifreund and lemonboy for helping me  figure out various issues**

First of all, to get Valgrind going with Zig programs, you'll have to switch to the C allocator. In my case, this was as easy as flipping

```zig
pub var allocator = &gpa.allocator;
```

to

```zig
pub var allocator = std.heap.c_allocator;
```

You also need to link with libc, so add this to your build.zig file:

```zig
exe.linkLibC();
```

## Running Valgrind

I used Valgrind 3.16.1 for this; old versions of Valgrind have limited/buggy dwarf4 support which manifests itself as missing symbols in traces.

```bash
valgrind --leak-check=full --track-origins=yes \
--show-leak-kinds=all --num-callers=15 \
--log-file=leak.txt ./my-app
```

The `--num-callers=15` is important (you can pick any suitable number of course), as otherwise you'll get traces that are too short. For instance, I ran into a leak where the Reader interface was used, and the trace didn't include any of my application code, only stdlib stuff.

## Illegal instruction ??

I ran Valgrind on a Linux machine with avx512, and got "Illegal instruction" on an `@intToFloat` cast. This was solved by adding `-mcpu=native-avx512f` to the build command, with `mcpu=baseline` being the safest option if you still run into issues.

This is due to a bug in Valgrind: https://bugs.kde.org/show_bug.cgi?id=383010

Some more context here: https://github.com/ziglang/zig/issues/8615

