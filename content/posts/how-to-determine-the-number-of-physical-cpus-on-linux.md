+++
categories = ["Quick Tips"]
date = "2012-07-17T15:51:00+01:00"
description = "How to determine the number of physical CPUs on Linux"
tags = ["/proc/cpuinfo"]
# type = "post"
title = "How to determine the number of physical CPUs on Linux"

+++

The **/proc/cpuinfo** file contains information about the CPUs installed on your computer however it’s quite confusing when you have to deal with multi-core processors.

#### - list number of physical CPUs:

```
$ cat /proc/cpuinfo | grep "physical id" | sort | uniq | wc -l
1
```

#### - number of cores:

```
$ cat /proc/cpuinfo | grep "cpu cores" | uniq
cpu cores    : 2
```

#### - how many virtual processors:

```
$ cat /proc/cpuinfo | grep "^processor"
processor    : 0
processor    : 1
```

If the number of virtual processors is greater than the number of physical processors, the CPUs are using hyper-threading. Hyper-threading will only work with the SMP kernel.
