---
title: "Cache Me If You Can"
date: 2023-07-08T18:45:37+02:00
---

Why would adding a single keyword to a struct in Zig give a 30% performance increase, with far less variability? As it turns out, tightly packed data structures and field reorderings can cause surprising performance issues in multithreaded programs due to _false sharing_. Let's look at the numbers.
<!--more-->
False sharing occurs when the data layout is at odds with the memory access pattern, namely multiple threads managing unrelated data that happens to fall on the same _cache line_.

A cache line is the smallest unit of transfer between Lx caches and main memory, and thus also the unit of synchronization of coherence protocols such as MESI. When a thread on a core updates a memory location, the affected cache line is marked as dirty and other cores (logical or not) must invalidate their copy of the cache line.

Cache line invalidation is very expensive, and may additionally cause issues for applications that are sensitive to performance variability. When false sharing does happen, the actual impact depends on a thread's affinity to core socket/hyperthreads, scheduling, and the number of threads.

A cache line is typically 64 bytes, but this can vary depending on the CPU architecture. Tools such as `lstopo` is useful to determine the cache line size and the cache hierachy on your system.
### A little Zig surprise

Take a look at this struct:

```zig
const Queue = struct  {
    head_index: u64 = 0,
    padding: [128]u8 = undefined,
    tail_index: u64 = 0,
};
```

Now imagine you have two threads updating `head_index` and `tail_index`.

Changing the struct to the following makes the program run 30% faster on my Intel Core i9, and with an _order of magnitude_ lower standard deviation:

```zig
const Queue = extern struct  {
    head_index: u64 = 0,
    padding: [128]u8 = undefined,
    tail_index: u64 = 0,
};
```

The difference? The `extern` keyword. This tells the compiler to adhere to the C ABI, which means the compiler will _not_ reorder the fields.

_Note that we can not use a_ `packed struct` _in this example as the padding type is not supported (In Zig, packed structs are just fancy integers)_

For regular structs, Zig may reorder the fields to _minimize padding_, which is a good thing in general.

However, by enforcing the ordering, the indended reason for padding causes `head_index` and `tail_index` to fall on different cache lines, and thus false sharing is avoided: The two threads can update their respective fields without invalidating each other's cache lines.

To reduce the chance of false sharing, we must force a suitable field ordering, add padding fields or set alignment on specific fields.

### The code

Here's a simple benchmark that demonstrates the issue:

```zig
const std = @import("std");
blah blah blah
```

The `doNotOptimizeAway(i)` boils down to `asm volatile ("" :: [i] "r" (i),);` on my machine, which is a compiler barrier that prevents the compiler from simply optimizing away the loop (which wouldn't make for a useful benchmark)
### The numbers

The following tables shows the benchmark summary and 10 timings for the case where false sharing does not occur, and 10 timings for the case where false sharing does occur. 

While a larger number of samples were taken to verify the effect, the numbers below are representative.

Not only is the average time for the false sharing case worse, the standard deviation is also an order of magnitude higher.

| | No false sharing | False sharing |
| --- | --- | --- |
mean| 3.0299017825599996 | 4.738531733004999 |
stddev| 0.011379971724662449 | 0.13044659161376454 |
median| 3.02725268766 | 4.7289631684049995 |
min| 3.01537040366 | 4.582206681904999 |
max| 3.0460710066599996 | 5.081537872905 |

The actual runs:

| No false sharing | False sharing |
| --- | --- |
| 3.04404566866 | 4.582206681904999 |
| 3.0200773766599998 | 4.732641409905 |
| 3.0460710066599996 | 5.081537872905 |
| 3.02459747266 | 4.680716246905 |
| 3.01537040366 | 4.750897377905 |
| 3.03074159466 | 4.733568547905 |
| 3.02830799766 | 4.678189750905 |
| 3.0449196536599996 | 4.725284926904999 |
| 3.02619737766 | 4.742324924905 |
| 3.0186892736599997 | 4.677949589904999 |




















### Hyperthreading adds a twist

The timings above are with hyperthreading disabled. With hyperthreading enabled, the timings for the false sharing case are much more variable, but still consistently worse than the no false sharing case.
