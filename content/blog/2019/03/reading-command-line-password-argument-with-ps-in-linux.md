+++
categories = ["Linux"]
date = "2019-03-04T15:21:11Z"
title = "Reading command line password argument with \"ps\" in Linux"
tags = ["linux", "ps", "top", "/proc"]
type = "post"
description = "Reading command line password argument with \"ps\" in Linux"

+++
#### Introduction:

I was recently reading an article online about some methods to hide password argument from system status programs like "ps" or "top" in Linux and decided to take a closer look.

I've created a simple [Python script](#program-py "program.py") that accepts a single argument for password, let's run it:

```
$ ./program.py -p secret123 &
[1] 18200
Sleeping for 600s. PID: 18200
```

Now, we check the output from "ps":

```
$ ps -lfp 18200
F S UID        PID  PPID  C PRI  NI ADDR SZ WCHAN  STIME TTY          TIME CMD
0 S root     18200 16045  0  80   0 - 32500 poll_s 12:26 pts/2    00:00:00 python3 ./program.py -p secret123
```

Hmm, that's not good, any user on the system can ready our password now but if we check the help menu for our program, it looks like it can accept password either from the argument or environment variable:

```
$ ./program.py --help
usage: program.py [-h] [-p PASSWORD]

My program.

optional arguments:
  -h, --help            show this help message and exit
  -p PASSWORD, --password PASSWORD
                        User password. Can be provided as an env variable.
```

Assuming we can't modify the original script, let's use [a wrapper script](#wrapper-sh "wrapper.sh") that will store the password in environmental variable:

```
$ ./wrapper.sh
Enter your secret:
Sleeping for 600s. PID: 18639
```
\
```
$ ps -lfp 18639
F S UID        PID  PPID  C PRI  NI ADDR SZ WCHAN  STIME TTY          TIME CMD
0 S root     18639 18640  0  80   0 - 32500 poll_s 12:42 pts/2    00:00:00 python3 ./program.py
```

Cool, the password argument is no longer listed however we can still read environment variables from the stack of a running process with root privileges.

Here is how, the `/proc/<pid>/maps` file contains the ranges of memory mapped to the process and what they are mapped to. Near the bottom there is a line with the range mapped to `[stack]`. Take a note of the start address (they are in hex notation) and calculate the size in bytes:

```
$ sudo cat /proc/18639/maps | grep "\[stack\]"
7fff5f1a1000-7fff5f1c2000 rw-p 00000000 00:00 0                          [stack]

$ python -c 'print(hex(int("7fff5f1c2000", 16) - int("7fff5f1a1000", 16)))'
0x21000
```

Then run the command below to read the memory content. The environment variables should be printed out together with their hex values and address location, including our `PASSWORD=secret123` variable:

```
$ sudo xxd -s 0x7fff5f1a1000 -l 0x21000 /proc/18639/mem | grep -A 1 "PASSWORD"
7fff5f1c13a00 5041 5353 574f 5244 3d73 6563 7265  :.PASSWORD=secre
7fff5f1c17431 3233 0053 5348 5f41 5554 485f 534f  t123.SSH_AUTH_SO
```

You need root privileges to read the pseudo-filesystem (`/proc`) and guess the variable name.

There are other methods, for instance the `mysql` client [replaces](https://github.com/mysql/mysql-server/blob/8.0/client/mysql.cc#L1930 "mysql.cc") the password from the command line with `x`:

###### mysql.cc:

```
while (*argument) *argument++ = 'x'; // Destroy argument
```

#### Scripts:

###### program.py

```
#!/usr/bin/env python3

import argparse
import os
import time

SLEEP = 600

parser = argparse.ArgumentParser(description='My program.')

parser.add_argument(
    '-p',
    '--password',
    action='store',
    help='User password. Can be provided as an env variable.',
)

opts = parser.parse_args()
password = opts.password

if not password:
    password = os.getenv('PASSWORD', None)

print(f'Sleeping for {SLEEP}s. PID: {os.getpid()}')
time.sleep(SLEEP)
```

###### wrapper.sh

```
#!/usr/bin/env bash

read -s -p "Enter your secret: " secret

umask 077
echo "export PASSWORD=${secret}" > tmp.$$
source tmp.$$ && ./program.py &
shred -fuz tmp.$$
```

I've also created another Python script that we'll dump memory from the stack. The output is a long bytestring so pipe it to `less`:

###### stack_mem.py

```
#!/usr/bin/env python3

import argparse
import re


def parse_args():
    parser = argparse.ArgumentParser(description='Read stack memory from a process.')
    parser.add_argument(
        '-p', '--pid', action='store', required=True, help='Process ID.'
    )
    return parser.parse_args()


def stack_mem(pid):
    with open(f'/proc/{pid}/maps', 'r') as maps, open(
        f'/proc/{pid}/mem', 'rb', 0
    ) as mem:
        regex = re.compile(r'([0-9A-Fa-f]+)-([0-9A-Fa-f]+) ([-r])(.*\[stack\]$)')
        for line in maps:
            match = re.match(regex, line)
            if match:
                start = int(match.group(1), 16)
                end = int(match.group(2), 16)
                mem.seek(start)
                print(mem.read(end - start))


def main():
    opts = parse_args()
    try:
        stack_mem(opts.pid)
    except FileNotFoundError as e:
        print(f'Process not found (PID: {opts.pid}).')
        exit(1)


if __name__ == '__main__':
    main()
```

**Note:** All Python examples are using version 3.7+.
