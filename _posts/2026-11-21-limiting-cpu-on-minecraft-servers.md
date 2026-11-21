---
layout: post
title: "Keeping One Minecraft Server from Hogging the CPU"
date: 2026-11-21
tags: [minecraft, python, linux, cpulimit]
---

I used to run several Minecraft servers on the same machine. Each one had
its own memory limit, but they all shared the CPU. When one server got busy
enough, the others slowed down too.

I already had a number for how big I expected each server to be: the
`-Xmx` value in its Java startup command. So I wrote a little Python
script that used that number to decide how much CPU each server could use.

## Turning the memory limit into a CPU limit

The script looks for Java processes every five seconds and reads their
startup arguments from `/proc/<pid>/cmdline`. If it finds an `-Xmx` value
in megabytes or gigabytes, it converts it to megabytes and does this:

```python
cpu_limit = (mem_value // 512) * 50

if cpu_limit < 10:
    cpu_limit = 10

subprocess.Popen(["cpulimit", "-p", pid, "-l", str(cpu_limit), "-b"])
```

Every full 512 MB adds 50 percentage points to the limit. A server started
with `-Xmx512M` gets 50%, `-Xmx1G` gets 100%, and `-Xmx2G` gets 200%.
Anything below 512 MB gets the 10% minimum.

Here, 100% means roughly one logical CPU's worth of time, so 200% makes
sense on a multi-core machine. The Python script works out the number and
remembers which PIDs it has already handled. The actual limiting is done by
[`cpulimit`](https://github.com/opsengine/cpulimit), which briefly pauses
and resumes the process to keep its average CPU usage down.

## Why it was handy

I could change a server's heap limit and the next time it started, the
script would pick up the new value and set its CPU limit. There was no
second list of per-server limits to keep in sync.

The rule was just a shortcut for my setup. `-Xmx` tells you the maximum
Java heap, and two servers with the same heap limit can do very different
amounts of work. The script also picks up any Java process with a matching
`-Xmx` flag, so on a machine running other Java apps they'd get limited
too. For a box running my Minecraft servers, the shortcut was enough to
keep one busy instance from taking over.
