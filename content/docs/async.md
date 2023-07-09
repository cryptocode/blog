---
title: Async CPU bound workers in Zig
date: 2021-05-02T10:00:00+02:00
---

Zig has [async/await support](https://ziglang.org/documentation/0.7.1/#Async-and-Await), which is typically used for IO bound operations.

In this article, however, we'll use async/await to simplify writing a simple concurrent worker.

**Goal: use all the cores on the machine to find a randomly selected 64-bit number whose lower N bits are all cleared.**

### How it works
Instead of manually spinning up threads, we're just going to use async/await, along with `pub const io_mode = .evented;` which informs the standard library to use a non-blocking event loop. Workers ensure the event loop yields to a worker thread by calling `startCpuBoundOperation`

First we query the number of logical CPU cores. Then we allocate an array with enough space for an *async frame* for each core.

Then we simply call `async worker(...)` for each CPU and stuff the resulting frame into the array.

Next, we loop through the frames and await the result, which we print. While this is already simple, [#5263](https://github.com/ziglang/zig/issues/5263) may make it even simpler.

Now we face a small challenge: when a worker has found the solution, how do we tell the other workers to stop working on the problem?

Easy enough - we pass a completion token when doing the async call (just a pointer to bool) which the winning worker atomically sets. All workers then *check* this flag. 

If the completion token is set, we simply break out of the worker loop.

I think the end result is quite clean concurrency code (requires a recent master build):

```zig
const std = @import("std");

pub const io_mode = .evented;

pub fn main() !void {
    var cpu: u64 = try std.Thread.getCpuCount();

    // Allocate room for an async frame for every 
    // logical cpu core
    var promises = try std.heap.page_allocator.alloc(@Frame(worker), cpu);
    defer std.heap.page_allocator.free(promises);

    // Start a worker on every cpu
    var completion_token: bool = false;
    while (cpu > 0) : (cpu -= 1) {
        promises[cpu - 1] = async worker(cpu, &completion_token);
    }

    std.debug.print("Working...\n", .{});

    // Wait for a worker to find the solution
    for (promises) |*future| {
        var result = await future;
        if (result != 0) {
            std.debug.print("The answer is {x}\n", .{result});
        }
    }
}

fn worker(seed: u64, completion_token: *bool) u64 {
    // Inform the event loop we're cpu bound. 
    // This effectively puts a worker on every logical core.
    std.event.Loop.startCpuBoundOperation();

    // Seed the random number generator so each worker
    // look at different numbers
    var prng = std.rand.DefaultPrng.init(seed);
    const random = prng.random();

    while (true) {
        var attempt = random.int(u64);

        // We're looking for a number whose N lower bits 
        // are zero. Feel free to change the constant to 
        // make this take a longer or shorter amount of time.
        if (attempt & 0xffffff == 0) {
            // Tell other workers we're done
            @atomicStore(bool, completion_token, true, std.builtin.AtomicOrder.Release);
            std.debug.print("I found the answer!\n", .{});
            return attempt;
        }

        // Check if another worker has solved it, in which
        // case we stop working on the problem.
        if (@atomicLoad(bool, completion_token, std.builtin.AtomicOrder.Acquire)) {
            std.debug.print("Another worker won\n", .{});
            break;
        }
    }

    return 0;
}
```

Run the program and observe all cpu cores get to work using `htop` or similar (you may have to adjust 0xffffff if it's too fast or too slow)

*Edit: some clarifications thanks to Protty on Discord*