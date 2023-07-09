---
title: A surprisingly capable RPN calculator in about 100 lines of Zig code
date: 2021-03-01T10:00:00+02:00
---

Summary:

1. [Reverse polish notation](https://en.wikipedia.org/wiki/Reverse_Polish_notation) is awesome
2. [Zig](https://ziglang.org) is awesome

Before solving quadratic equations and diving into code, let's see if 2 + 2 is still 4:

```bash
2 2 +
M0 = 4
```
Phew! Notice how every answer is placed in the next available memory slot. This will be useful in our next example where we calculate the two solutions to a quadratic equation:

First we input the known values so these get memory slots we can reference, rather than repeating the constants. In the quadratic formula, we thus use M0, M1 and M2 instead of a, b and c.

```bash
$zig run rpn.zig

# You can place comments using the hash sign.
# Let's solve the quadratic equation where a=5, b=6, c=1 
5
M0 = 5
6
M1 = 6
1
M2 = 1

# First solution
M1 1 ~ * M1 2 ^ 4 M0 * M2 * - √ + 2 M0 * /
M3 = -0.2

# Second solution
M1 1 ~ * M1 2 ^ 4 M0 * M2 * - √ - 2 M0 * /
M4 = -1
```

There we have it, the solutions are `x = -0.2` and `x = -1`

*Why RPN anyway? There's less to type (because you don't need parentheses), and some fans claim it leads to fewer mistakes. For this demo, though, the main reason is that implementing an RPN calculator is really simple. You don't need any parsing beyond trivial tokenizing/converting to numbers, and operator precedence is a non-issue.*

Let's do one more example, calculating the volume of a sphere. We first put the radius in a memory slot, then we evaluate the formula:

```bash
4.8
M0 = 4.8

4 3 / 𝜋 * M0 3 ^ *
M1 = 463.2466863277365
```

[Here's a nice little tool](https://www.mathblog.dk/tools/infix-postfix-converter/) if you need help converting from infix to RPN.

# Supported operators and functions

This is what's implemented so far. Adding support for more functions and operators is easy - just update the switch statement in the tokenizer loop.

|Op | Description |
|--- | --- |
|* / + - % | Basic arithmetic/modulus |
|^ | Exponentiation |
|~ | Negation |
|M... | Memory reference, such as M0 |
|sin, cos, tan | Trigonometric functions |
|sqrt or √ | Square root |
|pi or 𝜋 | Push 𝜋 to the stack |
|e | Push Euler's number to the stack |
|avg | Pop all numbers from the stack, push the average |

In addition, you can type `mc` to clear the memory slots and `exit` to terminate the calculator.

There are 256 slots available, which wraps around if you evaluate more expressions in a session.

# How does it work?
The program sets up a read-eval-print loop (REPL) where it reads expressions, line by line, from standard input.

Each line is then split on space. Each resulting token is then checked in a switch statement:

* If it's a number, push it to the evaluation stack. 
* If it's a unary operator/function, then the top of the stack is replaced by the value after applying the operator. 
* If it's a binary operator like multiplication, then two numbers are popped off the stack and replaced by the result.
* If it's a memory reference, the corresponding value is pushed to the stack
* If it's an empty line or a comment, read the next line.

The final result is then stored in the next available memory slot and printed.

# The code

```zig
const std = @import("std");

pub fn main() !void {
    const stdin = std.io.getStdIn().reader();
    const stdout = std.io.getStdOut().writer();
    try stdout.print("Zig RPN Calculator\nType mc to clear memory slots, exit to quit.\n", .{});

    var repl: [1024]u8 = undefined;
    var memslot = [_]f64{0} ** 256;
    var memslot_index: usize = 0;

    repl_loop: while (true) {
        const line = try stdin.readUntilDelimiterOrEof(&repl, '\n');
        var input = std.mem.trimRight(u8, line.?, "\r\n");
        if (std.mem.eql(u8, input, "")) {
            continue;
        }
        else if (std.mem.eql(u8, input, "exit")) {
            return;
        }
        else if (std.mem.eql(u8, input, "mc")) {
            memslot_index = 0;
            continue;
        }

        var it = std.mem.split(input, " ");

        var stack = [_]f64{0} ** 256;
        var sp: usize = 0;
        while (it.next()) |tok| {
            if (tok.len == 0) {
                continue :repl_loop;
            }
            switch (tok[0]) {
                '*' => {stack[sp-2] = stack[sp-2] * stack[sp-1]; sp -= 1;},
                '/' => {stack[sp-2] = stack[sp-2] / stack[sp-1]; sp -= 1;},
                '+' => {stack[sp-2] = stack[sp-2] + stack[sp-1]; sp -= 1;},
                '-' => {stack[sp-2] = stack[sp-2] - stack[sp-1]; sp -= 1;},
                '^' => {stack[sp-2] = std.math.pow(f64, stack[sp-2], stack[sp-1]); sp -= 1;},
                '%' => {stack[sp-2] = try std.math.mod(f64, stack[sp-2], stack[sp-1]); sp -= 1;},
                '~' => {stack[sp-1] = -stack[sp-1];},
                '#' => continue :repl_loop,
                'M' => {
                    const index = std.fmt.parseInt(usize, tok[1..], 10) catch |err| { 
                        std.debug.print("Invalid memory memory slot index in {s}\n", .{tok}); continue :repl_loop;
                    };
                    stack[sp] = memslot[index % 256];
                    sp += 1;
                },
                '0'...'9', '.' => {
                    const num = std.fmt.parseFloat(f64, tok) catch |err| {
                        std.debug.print("Invalid character in floating point number {s}\n", .{tok}); continue :repl_loop;
                    };
                    stack[sp] = num;
                    sp += 1;
                },
                else => {
                    if (std.mem.eql(u8, tok, "sin")) {
                        stack[sp-1] = std.math.sin(stack[sp-1]);
                    }
                    else if (std.mem.eql(u8, tok, "cos")) {
                        stack[sp-1] = std.math.cos(stack[sp-1]);
                    }
                    else if (std.mem.eql(u8, tok, "tan")) {
                        stack[sp-1] = std.math.tan(stack[sp-1]);
                    }
                    else if (std.mem.eql(u8, tok, "sqrt") or std.mem.eql(u8, tok, "√")) {
                        stack[sp-1] = std.math.sqrt(stack[sp-1]);                                      
                    }
                    else if (std.mem.eql(u8, tok, "pi") or std.mem.eql(u8, tok, "𝜋")) {
                        stack[sp] = std.math.pi;
                        sp += 1;
                    }
                    else if (std.mem.eql(u8, tok, "e")) {
                        stack[sp] = std.math.e;
                        sp += 1;
                    }                                                    
                    else if (std.mem.eql(u8, tok, "avg")) {
                        const count = sp;
                        var sum: f64 = 0;
                        for (stack) |val| {
                            sum += val;
                        }
                        sp-=count;
                        stack[sp] = sum / @intToFloat(f64, count);
                        sp+=1;
                    }
                    else {
                        try stdout.print("Invalid operation {s}\n", .{tok});                        
                        continue :repl_loop;
                    }
                },
            }
        }
        memslot[memslot_index % 256] = stack[sp-1];
        try stdout.print("M{d} = {d}\n", .{memslot_index % 256, stack[sp-1]});
        memslot_index += 1;
    }
}
```
