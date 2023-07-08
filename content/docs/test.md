---
title: "Stuff about things"
date: 2023-07-08T18:45:37+02:00
summary: "Putting some numbers on false sharing cost using Zig"
---
## Test

Stuff about things
Where we discuss stuff about things
And even more stuff about things

But also things about stuff
Moreover, more stuff about things

Who knows, maybe even things about stuff

### Code

```zig
const std = @import("std");
const warn = std.debug.warn;
pub fn main() !void {
    warn("Hello, {}!\n", .{"World"});
}
```
