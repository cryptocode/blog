---
title: Zig scripts?!?
published: true
date: 2021-04-24T10:00:00+02:00
---

Look at this beauty:

```zig
//usr/bin/env zig run "$0" -- "$@" ; exit 
const std = @import("std");
pub fn main() !void {
    std.log.info("Awesome, it works\n", .{});
}
```

Let's make it executable and run it:

```bash
chmod +x myscript.zig

./myscript.zig
info: Awesome, it works
```

This works because `//` is a comment in Zig, and just an empty path component as far as the shell is concerned

*Thanks to squirl for making the argument passing syntax more robus*