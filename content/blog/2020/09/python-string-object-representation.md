+++
date = "2020-09-30T13:53:55+01:00"
title = "Python string object representation"
tags = ["cpython", "python", "gdb", "object", "structure", "unicode"]
type = "post"
description = "Python string object representation"
categories = ["Linux"]

+++
#### Introduction:

This post will try to explain this simple code snippet:

```
>>> import sys
>>> s = "A"
>>> len(s)
1
>>> sys.getsizeof(s)
50
```

Why the size  of a single [ASCII](https://en.wikipedia.org/wiki/ASCII) character string is 50 bytes?

The strings in Python are not simply an array of characters like C-strings also known as ASCIIZ (ASCII Zero-terminated) strings.

According to the documentation, [sys.getsizeof](https://docs.python.org/3/library/sys.html#sys.getsizeof) returns the size of an **object** in bytes. A string objects in Python are really sequences of unicode characters and we can use this simple trick to get the actual size of a single code point:

```
>>> s = "A"
>>> sys.getsizeof(s + "@") - sys.getsizeof(s)
1
>>> s = "🐙"
>>> sys.getsizeof(s + "@") - sys.getsizeof(s)
4
```

(The subtraction is required because the actual number of bytes required to store a string is greater than the size of its characters.)

So let's find out why the size of our string is not equal to the sum of individual characters (a single character "A" in our case). We know that a size of a single character in C data types is 1 byte (8 bits), let's see what are these extra 49 bytes.

**NOTE:** All examples in this post are specific to CPython (`Python 3.8.0`) implementation so there is no guarantee that these structures won't change in future releases. The results may also vary in case of a different platforms and their data models ([LP64  or LLP64](https://en.wikipedia.org/wiki/64-bit_computing#64-bit_data_models)).

The [PEP 393](https://www.python.org/dev/peps/pep-0393/) introduced *"Flexible String Representation"* to support multiple internal representations for different encodings.

We use ASCII character in our first example so it should be stored as  `PyASCIIObject` instance:

```
typedef struct {
  PyObject_HEAD
  Py_ssize_t length;
  Py_hash_t hash;
  struct {
      unsigned int interned:2;
      unsigned int kind:2;
      unsigned int compact:1;
      unsigned int ascii:1;
      unsigned int ready:1;
  } state;
  wchar_t *wstr;
} PyASCIIObject;
```

<sup>GitHub: [python/cpython/Include/cpython/unicodeobject.h](https://github.com/python/cpython/blob/master/Include/cpython/unicodeobject.h)</sup>

We'll open a `gdb` (GNU Debugger) session to check this (please refer to my previous [blog post](/blog/2020/05/python-the-old-bug-in-sqlite-module-strikes-back/) for configuration details) and set a breakpoint on the built-in `print` function (`builtin_print`) that is defined in [cpython/Python/bltinmodule.c](https://github.com/python/cpython/blob/master/Python/bltinmodule.c):

```
$ gdb -q ~/.pyenv/versions/3.8.0-debug/bin/python
Reading symbols from /home/test/.pyenv/versions/3.8.0-debug/bin/python3.8...done.
(gdb) b builtin_print
Breakpoint 1 at 0x67efb0: file Python/bltinmodule.c, line 1825.
```

We then just print our single character string:

```
(gdb) run -c 'print("A")'
Starting program: /home/test/.pyenv/versions/3.8.0-debug/bin/python -c 'print("A")'
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib64/libthread_db.so.1".

Breakpoint 1, builtin_print (self=<module at remote 0x7ffff7f84d70>, args=0xa6ecc0, nargs=1, kwnames=0x0) at Python/bltinmodule.c:1825
1825	    PyObject *sep = NULL, *end = NULL, *file = NULL, *flush = NULL;
```

The `args` parameter holds all arguments that were supplied to the `print` function. We'll cast the first one into `PyASCIIObject`:

```
(gdb) set print pretty on
(gdb) p args[0]
$3 = 'A'
(gdb) p (PyASCIIObject *) args[0]
$4 = (PyASCIIObject *) 0x7ffff7e07d10
(gdb) p *(PyASCIIObject *) args[0]
$5 = {
  ob_base = {
    ob_refcnt = 4, 
    ob_type = 0x9c1480 <PyUnicode_Type>
  }, 
  length = 1, 
  hash = 5707903384534129446, 
  state = {
    interned = 1, 
    kind = 1, 
    compact = 1, 
    ascii = 1, 
    ready = 1
  }, 
  wstr = 0x0
}
(gdb) p sizeof(*(PyASCIIObject *) args[0])
$6 = 48
```

So here we have it, our string object size is 48 bytes, the remaining 1 byte is occupied by one more character to store the `'\0'` at the end of the string.

We can also find the string itself, according to PEP 393, our string should be located after the base structure so we need to add the size of it (48 bytes) to the memory address:

> Objects for which both size and maximum character are known at creation time are called "compact" unicode objects; character data immediately follow the base structure.

```
(gdb) x/1sb (unsigned long) *args + 48
0x7ffff7e86bb0:	"A"
(gdb) x/2cb (unsigned long) *args + 48
0x7ffff7e86bb0:	65 'A'	0 '\000'
```

Let's decode and confirm some of these fields in Python:

```
>>> import ctypes
>>> import struct
>>> s = "A"
>>> s_mem = ctypes.string_at(id(s), sys.getsizeof(s))
>>> s_mem
b'\x14\x00\x00\x00\x00\x00\x00\x00\xc0&\x90\x00\x00\x00\x00\x00\x01\x00\x00\x00\x00\x00\x00\x001\xac\xc0[\t@\xf0\x8b\xe5eyboard\x00\x00\x00\x00\x00\x00\x00\x00A\x00'
>>> len(s_mem)
50
```

The last two bytes are our string and null character (`'\0'`)

```
>>> s_mem[-len(s)-1:]
b'A\x00'
>>> len(s_mem[-len(s)-1:])
2
```

The first 48 bytes are occupied by the object structure:

```
>>> len(s_mem[:-len(s)-1])
48
```

We can also decode and check the values for `length` and `hash` fields from the struct and they should be equal to `len(s)` and `hash(s)` respectively:

```
>>> obj_length, obj_hash = struct.unpack('<Qq', s_mem[16:16+8+8])
>>> len(s)
1
>>> hash(s)
-8363114099088774095
>>> obj_length
1
>>> obj_hash
-8363114099088774095
```

The index slicing is to skip the first 16 bytes (`ob_base` field) and then get the next pair of 8 bytes for the fields in question.

Now, things will change when we use a unicode character, for instance a pictogram of an 🐙 that has the following unicode number `\U0001F419`.

PEP 393 mentions that "for non-ASCII strings, the `PyCompactObject` structure is used" and each code point will occupy 4 bytes in that case (`UCS-4`).

So let's go back to the `gdb` session to see this in action:

```
(gdb) run -c 'print("🐙")'
The program being debugged has been started already.
Start it from the beginning? (y or n) y
Starting program: /home/test/.pyenv/versions/3.8.0-debug/bin/python -c 'print("🐙")'
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib64/libthread_db.so.1".

Breakpoint 1, builtin_print (self=<module at remote 0x7ffff7f84d70>, args=0xa6ecc0, nargs=1, kwnames=0x0) at Python/bltinmodule.c:1825
1825	    PyObject *sep = NULL, *end = NULL, *file = NULL, *flush = NULL;
(gdb) p *(PyCompactUnicodeObject *) args[0]
$21 = {
  _base = {
    ob_base = {
      ob_refcnt = 3, 
      ob_type = 0x9c1480 <PyUnicode_Type>
    }, 
    length = 1, 
    hash = 3301279203973567679, 
    state = {
      interned = 0, 
      kind = 4, 
      compact = 1, 
      ascii = 0, 
      ready = 1
    }, 
    wstr = 0x7ffff7e19808 L"\x1f419"
  }, 
  utf8_length = 0, 
  utf8 = 0x0, 
  wstr_length = 1
}
(gdb) p sizeof(*(PyCompactUnicodeObject *) args[0])
$22 = 72
```

The size of the object structure is 72 bytes now and we can see that each code point is using 4 bytes:

```
(gdb) x/1sw (unsigned long) *args + 72
0x7ffff7e19808:	U"\x1f419"
(gdb) x/8xb (unsigned long) *args + 72
0x7ffff7e19808:	0x19	0xf4	0x01	0x00	0x00	0x00	0x00	0x00
```

The bytes sequence is expressed in [little-endian (LE)](https://en.wikipedia.org/wiki/Endianness) so the least significant byte (LSB) (the first set of bits 0-7) is located in the first byte.

We can also confirm the same in Python interactive shell:

```
>>> s = "🐙"
>>> s_mem = ctypes.string_at(id(s), sys.getsizeof(s))
>>> s_mem
b'\x01\x00\x00\x00\x00\x00\x00\x00\xc0&\x90\x00\x00\x00\x00\x00\x01\x00\x00\x00\x00\x00\x00\x00xd\x8b\xfb|\x165\xfd\xb0bj_leng8m\xc3Y\xa7\x7f\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x01\x00\x00\x00\x00\x00\x00\x00\x19\xf4\x01\x00\x00\x00\x00\x00'
>>> len(s_mem[-4-4:])
8
>>> s_mem[-4-4:]
b'\x19\xf4\x01\x00\x00\x00\x00\x00'
>>> len(s_mem[:-4-4])
72
```

So the bytes from position 72 to 74 hold 4 bytes code point for our octopus:

```
>>> chr(int.from_bytes(s_mem[72:72+4], 'little'))
'🐙'
```

That's it!
