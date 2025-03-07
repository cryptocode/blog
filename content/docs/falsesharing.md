---
toc: true
title: "Cache Me If You Can"
date: 2023-07-09T10:00:00+02:00
categories: ["multithreading", "zig"]
published: true
---

{{< table_of_contents >}}

How come making a struct in Zig _less_ densely packed can give a 56% performance increase, with far less variability? This post takes a look at false sharing and how it can be caused by packed data layouts and unintended field reorderings.
<!--more-->
While packing data closely together can be very beneficial due to cache locality, false sharing must be taken into account when designing optimal data layouts for multithreaded programs.

## Cache lines
False sharing occurs when the data layout is at odds with the memory access pattern, namely threads updating thread-specific data that happens to fall on the same _cache line_.

Imagine we're writing a multi-threaded queue. To avoid false sharing, we must to go from this situation:

![Cachelines](/blog/images/false-sharing-bad.png)

To something like this:

![Cachelines](/blog/images/false-sharing-good.png)

Of course, `padding` can be replaced by other useful data as long as it doesn't cause false sharing.

**Thread 1 only cares about `head_index`, while thread 2 only cares about `tail_index`. By padding the struct, we ensure that each thread has its own cache line and can thus update its field without invalidating the other thread's cache line.**

A cache line is the smallest unit of transfer between each core's local Lx caches and shared main memory, and thus also the unit of synchronization of coherence protocols such as MESI. Updating a cache line on one core will invalidate the corresponding line on all other cores' local caches.

Cache line invalidation is very expensive, and may additionally cause issues for applications that are sensitive to performance variability. The actual impact depends on a thread's affinity to core socket and hyperthreads, scheduling, and the number of threads. Memory access not involving false sharing may also be affected due to intra-socket coherence traffic saturation.

A cache line is typically 64 bytes, but this varies between CPU architectures. Tools such as `lstopo` are useful to determine the cache hierachy and line size on your CPU.
## A little Zig surprise

Take a look at this struct:

```zig
const Queue = struct  {
    head_index: u64 = 0,
    padding: [64]u8 = undefined,
    tail_index: u64 = 0,
};
```

Now imagine you have two threads updating `head_index` and `tail_index` respectively.

Changing the struct to the following makes the benchmark run 56% faster on my Intel Core i9, and with an order of magnitude lower standard deviation:

```zig
const Queue = extern struct  {
    head_index: u64 = 0,
    padding: [64]u8 = undefined,
    tail_index: u64 = 0,
};
```

The difference? The extern keyword. This tells the compiler to adhere to the C ABI, which means the compiler will _not_ reorder the fields.

_Note that we can not use a packed struct in this example as the padding type is not supported. In Zig, packed structs are just fancy integers_

For regular structs, Zig is free to reorder the fields to _minimize padding_, which is a good thing in general.

However, by enforcing the specified order, the `padding` field fulfills its intended purpose: It places `head_index` and `tail_index` on separate cache lines.
## The code

The benchmark is designed to demonstrate the issue at hand, and the results are not necessarily representative of false sharing slowdowns in a real-world application. That said, if false sharing occurs in a hot path, the impact can be significant.

```zig
const std = @import("std");

// Remove "extern" to observe false sharing
const Data = extern struct  {
    head_index: u64 align(64) = 0,
    padding: [128]u8 = undefined,
    tail_index: u64 = 0,
};

fn updateHeadIndex(data: anytype) anyerror!void {
    for (0..2000000000) |i| {
        data.head_index += 1;
        std.math.doNotOptimizeAway(i);
    }
}

fn updateTailIndex(data: anytype) anyerror!void {
    for (0..2000000000) |i| {
        data.tail_index += 1;
        std.math.doNotOptimizeAway(i);
    }
}

pub fn main() anyerror!void {
    var data: Data = .{};
    var head_thread = try std.Thread.spawn(.{}, updateHeadIndex, .{&data});
    var tail_thread = try std.Thread.spawn(.{}, updateTailIndex, .{&data});
    head_thread.join();
    tail_thread.join();
}
```

The `doNotOptimizeAway(i)` call boils down to `asm volatile ("" :: [i] "r" (i),);` on my system, which is a barrier that prevents the compiler from simply optimizing away the loop (which wouldn't make for a useful benchmark)

```bash
zig build-exe -OReleaseFast falsesharing.zig
hyperfine --export-json results-with-extern.json ./falsesharing   
```

Do the same without the `extern` keyword (rename the json output filename in the hyperfine command), and observe the difference.
## The numbers

The following tables contain the benchmark summary, 10 timings for the case where false sharing does not occur, and 10 timings for the case where false sharing does occur. 

While a larger number of samples were taken to verify the effect, the numbers below are representative for the findings.

Not only is the average time for the false sharing case significantly worse, the standard deviation is also an order of magnitude higher.

Timings are measured in seconds.

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
| 3.0440456 | 4.5822066 |
| 3.0200773 | 4.7326414 |
| 3.0460710 | 5.0815378 |
| 3.0245974 | 4.6807162 |
| 3.0153704 | 4.7508973 |
| 3.0307415 | 4.7335685 |
| 3.0283079 | 4.6781897 |
| 3.0449196 | 4.7252849 |
| 3.0261973 | 4.7423249 |
| 3.0186892 | 4.6779495 |

## Hyperthreading adds a twist

The timings above are with hyperthreading disabled. With hyperthreading enabled, the penalty for the false sharing case is sometimes less severe (depending on affinity), but average run times are still much worse than the no false sharing case. Hyperthreading also adds more variance, 0.06 vs 0.011 standard deviation.

On Linux, you might want to consider using `pthread_setaffinity_np` to pin threads to physical cores.

This is currently not supported on macOS. For the purposes of replicating this benchmark, you can disable hyperthreading by entering safe mode and entering these commands in the terminal:

```bash
nvram boot-args="cwae=2"
nvram SMTDisable=%01
```

After rebooting, System Information (click Apple icon in the menu bar while holding Alt) should report `Hyper-Threading Technology: Disabled"

Resetting the NVRAM will re-enable hyperthreading.

## Cache topology

The cache topology on my test machine (with Hyperthreading disabled) is as follows, using `lstopo` from the `hwloc` package:

```bash
lstopo - -v --no-io

  Package L#0
    NUMANode L#0 (P#0 local=33554432KB total=33554432KB)
    L3Cache L#0 (size=16384KB linesize=64)
      L2Cache L#0 (size=256KB linesize=64)
        L1dCache L#0 (size=32KB linesize=64)
          L1iCache L#0 (size=32KB linesize=64)
            Core L#0 (P#0)
              PU L#0 (P#0)
      L2Cache L#1 (size=256KB linesize=64)
        L1dCache L#1 (size=32KB linesize=64)
          L1iCache L#1 (size=32KB linesize=64)
            Core L#1 (P#1)
              PU L#1 (P#1)
      L2Cache L#2 (size=256KB linesize=64)
        L1dCache L#2 (size=32KB linesize=64)
          L1iCache L#2 (size=32KB linesize=64)
            Core L#2 (P#2)
              PU L#2 (P#2)
      L2Cache L#3 (size=256KB linesize=64)
        L1dCache L#3 (size=32KB linesize=64)
          L1iCache L#3 (size=32KB linesize=64)
            Core L#3 (P#3)
              PU L#3 (P#3)
      L2Cache L#4 (size=256KB linesize=64)
        L1dCache L#4 (size=32KB linesize=64)
          L1iCache L#4 (size=32KB linesize=64)
            Core L#4 (P#4)
              PU L#4 (P#4)
      L2Cache L#5 (size=256KB linesize=64)
        L1dCache L#5 (size=32KB linesize=64)
          L1iCache L#5 (size=32KB linesize=64)
            Core L#5 (P#5)
              PU L#5 (P#5)
      L2Cache L#6 (size=256KB linesize=64)
        L1dCache L#6 (size=32KB linesize=64)
          L1iCache L#6 (size=32KB linesize=64)
            Core L#6 (P#6)
              PU L#6 (P#6)
      L2Cache L#7 (size=256KB linesize=64)
        L1dCache L#7 (size=32KB linesize=64)
          L1iCache L#7 (size=32KB linesize=64)
            Core L#7 (P#7)
              PU L#7 (P#7)
```
