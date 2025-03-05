---
title: Running Zig binaries on a KaiOS device
date: 2021-01-27T10:00:00+02:00

---

The goal here is to:

* Cross-compile a Zig program
* Root a KaiOS device, and then install and run the cross-compiled program on the device

We're not making a GUI app or anything here, just a small "Hello world" program that we'll cross-compile and run on the device. All sorts of interesting things can be done from this point on.

I'm using Android's `adb` to push files to the device and to get a shell, so you'll have to install that first if you wanna follow along.

## Make a Hello World program in Zig

```zig
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    try stdout.print("Hello from Zig!\n", .{});
}
```

Save as `main.zig`

## Cross-compile 
Zig makes it trivial to cross-compile for the ARM Cortex A7:

```bash
zig build-exe -target arm-linux-musleabihf -femit-bin=zigapp main.zig
```

## Root the device
The recipe here is going to be different for each device type. I'm using a Nokia 2720 Flip for testing, which is easy to root with `wallace-light`:

*NOTE: I used OmniSD to perform a priviledged reset long before making this post. I'm not sure if it's necessary or not for Wallace Light to work. If the recipe below doesn't work, try installing OmniSD and do a reset; it's easy enough [2].*

1. Put your KaiOS device in debug mode, typically using `*#*#33284#*#*` (which spells debug - a bug icon should appear in the status line if it works)

2. Deploy Wallace Light [1]

```bash
adb forward tcp:6000 localfilesystem:/data/local/debugger-socket
npm i -g @filipe_x3/kdeploy
kdeploy -i ~/Downloads/wallace-lite-0.1/application

```

Now start Wallace Light on the device and click ROOT ME.

Next, I'm pushing the cross-compiled Zig binary to the device's sdcard:

```bash
adb push zigapp /sdcard

```

Now we need to jump into the device through a shell:

```bash
adb shell
```

We're now in a terminal session on the device, but most likely on a read-only file system. Let's fix that:

```bash
mount -o rw,remount /system
```

Next, move the binary from `/sdcard` to `/system/bin` and make it executable:


```bash
cp /sdcard/zigapp /system/bin
chmod +x /system/bin/zigapp
```

Now we can run it!

```bash
/system/bin/zigapp
```

Result as expected:

```bash
Hello from Zig!
```


[1] I used Wallace Light from https://sites.google.com/view/bananahackers/root/temporary-root#h.sb1y1p9fj906

Unzip it, then unzip the contained application.zip, which is what we're deploying above.

[2] https://sites.google.com/view/bananahackers/install-omnisd
