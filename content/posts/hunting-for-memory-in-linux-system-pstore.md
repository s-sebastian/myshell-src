+++
date = "2017-08-13T14:42:24+01:00"
title = "Hunting for memory in Linux system (pstore)"
tags = ["slabinfo", "pstore"]
# type = "post"
description = "Hunting for memory in Linux system (pstore)"
categories = ["Linux"]

+++

I was troubleshooting a server with 24GB of RAM that was running "low" on memory:

```
# free -m
             total       used       free     shared    buffers     cached
Mem:         24098      21986       2112          0        581       2370
-/+ buffers/cache:      19033       5064
Swap:         2046        662       1384
```

The total memory usage for all processes was around 8GB but there was only 5GB free (including buffers and cache):

```
# ps --no-headers -eo "rss,cmd" | awk '{ sum+=$1 } END { printf ("%d%s\n", sum/1024,"M") }'
8347M
```

#### [Slabtop](https://linux.die.net/man/1/slabtop "slabtop") to the rescue.

This little handy utility displays kernel [slab cache information](http://man7.org/linux/man-pages/man5/slabinfo.5.html "slabinfo") (/proc/slabinfo) in real time:

```
# slabtop -o -s c
 Active / Total Objects (% used)    : 113308991 / 113556175 (99.8%)
 Active / Total Slabs (% used)      : 3018776 / 3018876 (100.0%)
 Active / Total Caches (% used)     : 114 / 186 (61.3%)
 Active / Total Size (% used)       : 11215879.73K / 11248446.12K (99.7%)
 Minimum / Average / Maximum Object : 0.02K / 0.10K / 4096.00K

  OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ/SLAB CACHE SIZE NAME                   
37018680 37018421  99%    0.12K 1233956       30   4935824K psinfo_cache
...
```

So, the first entry reveals that almost 5GB was used by `psinfo_cache` and after doing some research online, it turned out that [pstore](https://www.kernel.org/doc/Documentation/ABI/testing/pstore "pstore") is a persistent storage to preserve data, like console messages etc. across reboots.

There were other entries significantly contributing to memory usage but I'll skip them here.

It was enabled in kernel configuration file:

```
# grep CONFIG_PSTORE_CONSOLE /boot/config-$(uname -r)
CONFIG_PSTORE_CONSOLE=y
```

This is how it looks like after disabling and re-compiling the kernel:

```
# grep CONFIG_PSTORE_CONSOLE /boot/config-$(uname -r)
# CONFIG_PSTORE_CONSOLE is not set
```
