---
toc: true
title: "Embedding machine code in Zig"
date: 2023-07-29T12:01:51+02:00
categories: ["asm", "zig"]
published: true
---

{{< table_of_contents >}}

Using assembly code with Zig is usually done via inline assembly, or by linking to object files produced by assemblers.

This post explores a third way: write the assembly in a separate source file, run an assembler, and then embed the resulting machine code using `@embedFile`

## Linux and macOS Rosetta notes
The code is tested on macOS, but also works with minor modifications on Linux (primarily the syscall number)

Since the code here is x86_64, you can do the following on aarch64 macs to use Rosetta translation:

```bash
arch -x86_64 /bin/zsh --login

zig build-exe -target x86_64-macos asm.zig
./asm
```

## Assembling

We'll be using NASM to assemble the code, which allows us to emit raw machine code without any additional metadata.

Below is a simple snippet that adds two 64-bit numbers and returns the result:

```as
[BITS 64]
mov rax, rdi
add rax, rsi
ret
```

There's no function prologue and epilogue - we're not using the stack, so there's no need for the overhead. There isn't even a label naming the function.

However, as far as the System V AMD64 ABI is concerned, this _is_ a function, and we'll be able to call it from Zig. Given the ABI calling convention, we know that the first argument is passed in `rdi` and the second in `rsi`, and we know that we should put the result in `rax`. The `BITS 64` directive tells NASM to produce 64-bit machine code.

Let's assemble it:

```bash
nasm -f bin -o asm-add.bin asm-add.s
```

Thanks to `-f bin`, this will produce a file called `asm-add.bin` containing the raw machine code, with no metadata like headers and sections. Just raw machine code for each instruction in the assembly file.

Let's disassemble it to verify that claim:

```bash
ndisasm -b 64 asm-add.bin
```

This will output the same instructions as in the assembly file:

```plain
hexdump -C asm-add.bin 
00000000  48 89 f8 48 01 f0 c3
```

That's 7 bytes of machine code to add two numbers: 

```plain
; 0x48 is the REX prefix to indicate 64-bit operands
; 0x89 is the MOV instruction
; 0xf8 is the MOD R/M byte to specify the source and destination registers
48 89 f8: mov rax, rdi

; 0x48 is the REX prefix
; 0x01 is the ADD instruction
; 0xf0 is the MOD R/M byte to specify operand registers
48 01 f0: add rax, rsi

; 0xc3 is simply the RET instruction. It pops the return address from the stack and jumps to it 
; The return address is placed there by the CALL instruction that Zig generates.
c3: ret
```

## Calling from Zig

Next, let's call the _add_ function from Zig.

The process is roughly as follows:

* Allocate a page of memory
* Write the machine code to the page. The machine code is loaded at compile time with `@embedFile`
* Mark the page as readable and executable using `mprotect`
* Cast the address of the buffer to a function pointer
* Call the function

`asm.zig`

```zig
const std = @import("std");

pub fn main() !void {
    const code = try std.heap.page_allocator.alignedAlloc(u8, @intCast(std.heap.pageSize()), std.heap.pageSize());
    defer {
        // In order to deallocate, we have to make the page writable again
        _ = std.c.mprotect(@ptrCast(code), std.heap.pageSize(), std.c.PROT.WRITE);
        std.heap.page_allocator.free(code);
    }

    // Wrap the code page in a buffer stream and write the machine code to it
    var buf = std.io.fixedBufferStream(code);
    _ = try buf.write(@embedFile("asm-add.bin"));
    _ = std.c.mprotect(@ptrCast(code), std.heap.pageSize(), std.c.PROT.READ | std.c.PROT.EXEC);

    // Make a Zig function pointer for adding two u64s and returning the result
    const add: *const fn (a: u64, b: u64) callconv(.C) u64 = @ptrCast(code);

    // Call the machine code through the function pointer
    // This will put the arguments into rdi and rsi, and return the result in rax
    const res = add(1, 2);
    std.debug.print("Res = {d}\n", .{res});
}
```

Let's run it:

```zig
zig run asm.zig
Res = 3
 ```

The structure is pretty nice: all the assembly code is in a separate file, and we can call it from Zig with no overhead beyond using registers for arguments, according to a specific calling convention.

If multiple CPU architectures are supported, the correct machine code file can be selected at compile time, typically by switching on `builtin.cpu.arch`

Let's take a quick look at what Zig generates for the `add(1,2)` call:

```as
mov     rax, qword ptr [rbp - 96]
mov     edi, 1
mov     esi, 2
call    rax
```

The first line puts the pointer to the _add_ function into `rax`. The next two lines put the arguments into `edi` and `esi`, and the last line calls the function that was loaded into `rax`. Note that edi/esi are the 32-bit lower halves of rdi/rsi - the upper halves are zeroed out, and thus reading rdi/rsi will work as expected in the _add_ implementation.

## A larger example with fast memcpy and syscalls

Here's an expanded version of the example, with two more functions: an AVX based memcpy, and an example of using syscalls to print a string to stdout.

`asm.zig`

```zig
const std = @import("std");

pub fn main() !void {
    const code = try std.heap.page_allocator.alignedAlloc(u8, @intCast(std.heap.pageSize()), std.heap.pageSize());
    defer {
        // In order to deallocate, we have to make the page writable again
        _ = std.c.mprotect(@ptrCast(code), std.heap.pageSize(), std.c.PROT.WRITE);
        std.heap.page_allocator.free(code);
    }

    // Wrap the code page in a buffer stream and write the machine code to it
    var buf = std.io.fixedBufferStream(code);
    _ = try buf.write(@embedFile("asm-add.bin"));
    _ = std.c.mprotect(@ptrCast(code), std.heap.pageSize(), std.c.PROT.READ | std.c.PROT.EXEC);

    // Make a Zig function pointer for adding two u64s and returning the result
    const add: *const fn (a: u64, b: u64) callconv(.C) u64 = @ptrCast(code);

    // Call the machine code through the function pointer
    const res = add(1, 2);
    std.debug.print("Res = {d}\n", .{res});

    // Fast memcpy
    _ = std.c.mprotect(@ptrCast(code), std.heap.pageSize(), std.c.PROT.WRITE);
    try buf.seekTo(0);
    _ = try buf.write(@embedFile("asm-opt-memcpy.bin"));
    _ = std.c.mprotect(@ptrCast(code), std.heap.pageSize(), std.c.PROT.READ | std.c.PROT.EXEC);

    // This memcpy requires the destination and source to be aligned to 32 bytes, and len to be a multiple of 64 bytes
    const fast_memcpy: *const fn (dst: u64, src: u64, len: u64) callconv(.C) ?[*]const u8 = @ptrCast(code);
    const dst = try std.heap.page_allocator.alignedAlloc(u8, 32, 64);
    const src = try std.heap.page_allocator.alignedAlloc(u8, 32, 64);
    const test_bytes = "0123456789012345678901234567890123456789012345678901234567891234";
    @memcpy(src, test_bytes);
    _ = fast_memcpy(@intFromPtr(dst.ptr), @intFromPtr(src.ptr), 64);
    if (std.mem.eql(u8, test_bytes, dst)) {
        std.debug.print("fast_memcpy works!\n", .{});
    } else {
        std.debug.print("fast_memcpy failed!\n", .{});
    }
    try std.testing.expectEqualStrings(test_bytes, dst);

    // Next machine code file writes a message to stdout
    _ = std.c.mprotect(@ptrCast(code), std.heap.pageSize(), std.c.PROT.WRITE);
    try buf.seekTo(0);
    _ = try buf.write(@embedFile("asm-syscall.bin"));
    _ = std.c.mprotect(@ptrCast(code), std.heap.pageSize(), std.c.PROT.READ | std.c.PROT.EXEC);

    const hello_world: *const fn (msg: ?[*:0]const u8, len: u64) callconv(.C) void = @ptrCast(code);
    hello_world("Hello, world!!!\n", 16);
}
```
Below are the two new assembly files.

`asm-opt-memcpy.s`

In this case, we do need the function prologue and epilogue, because we're using the stack to store the arguments and local variables.

This source is based on a disassembly on Compiler Explorer.

```nasm
[BITS 64]
fast_memcpy:
        push    rbp
        mov     rbp, rsp
        and     rsp, -32
        sub     rsp, 256
        mov     qword [rsp + 104], rdi
        mov     qword [rsp + 96], rsi
        mov     qword [rsp + 88], rdx
        mov     qword [rsp + 80], 64
.LBB1_1:
        cmp     qword [rsp + 88], 0
        je      .LBB1_3
        mov     rax, qword [rsp + 96]
        mov     qword [rsp + 120], rax
        mov     rax, qword [rsp + 120]
        vmovdqa ymm0, [rax]
        vmovdqa [rsp + 32], ymm0
        mov     rax, qword [rsp + 96]
        add     rax, 32
        mov     qword [rsp + 112], rax
        mov     rax, qword [rsp + 112]
        vmovdqa ymm0, [rax]
        vmovdqa [rsp], ymm0
        mov     rax, qword [rsp + 104]
        vmovdqa ymm0, [rsp + 32]
        mov     qword [rsp + 232], rax
        vmovdqa [rsp + 192], ymm0
        vmovdqa ymm0, [rsp + 192]
        mov     rax, qword [rsp + 232]
        vmovntdq        [rax], ymm0
        mov     rax, qword [rsp + 104]
        add     rax, 32
        vmovdqa ymm0, [rsp]
        mov     qword [rsp + 184], rax
        vmovdqa [rsp + 128], ymm0
        vmovdqa ymm0, [rsp + 128]
        mov     rax, qword [rsp + 184]
        vmovntdq        [rax], ymm0
        mov     rcx, qword [rsp + 80]
        mov     rax, qword [rsp + 88]
        sub     rax, rcx
        mov     qword [rsp + 88], rax
        mov     rax, qword [rsp + 80]
        add     rax, qword [rsp + 96]
        mov     qword [rsp + 96], rax
        mov     rax, qword [rsp + 80]
        add     rax, qword [rsp + 104]
        mov     qword [rsp + 104], rax
        jmp     .LBB1_1
.LBB1_3:
        mov     rsp, rbp
        pop     rbp
        vzeroupper
        ret
```

`asm-syscall.s`

```nasm
[BITS 64]
; set up syscall arguments
mov     rdx, rsi
mov     rsi, rdi

; pick the write syscall on macOS
mov     rax, 0x2000004

; stdout
mov     rdi, 1

; invoke
syscall
```

```zig
zig run asm.zig
Res = 3
fast_memcpy works!
Hello, world!!!
 ```

*Edited July 2025. Code examples now compile with Zig 0.14.1. Added notes about using Rosetta on macOS.*