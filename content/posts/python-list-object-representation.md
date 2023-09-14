+++
date = "2021-09-21T18:15:41+01:00"
title = "Python list object representation"
tags = ["cpython", "ctypes", "python", "object", "structure"]
description = "Python list object representation"
categories = ["Linux"]
+++

#### Introduction:

In the [previous](/blog/2020/09/python-string-object-representation/ "Python string object representation") blog post we were looking at internal representation of Python string objects in memory.

In this article we'll delve into the C-level details and read the internals of another Python data type - a [list](https://docs.python.org/3/library/stdtypes.html#list).

**NOTE:** All examples in this post are specific to CPython (`Python 3.9`) implementation so there is no guarantee that these structures won't change in future releases. The results may also vary in case of a different platforms and their data models ([LP64  or LLP64](https://en.wikipedia.org/wiki/64-bit_computing#64-bit_data_models)).

The [PyListObject](https://docs.python.org/3/c-api/list.html#c.PyListObject) is defined in [cpython/Include/cpython/listobject.h](https://github.com/python/cpython/blob/3.9/Include/cpython/listobject.h#L9-L26) file and has the following structure:

```c
typedef struct {
    PyObject_VAR_HEAD
    /* Vector of pointers to list elements.  list[0] is ob_item[0], etc. */
    PyObject **ob_item;

    /* ob_item contains space for 'allocated' elements.  The number
     * currently in use is ob_size.
     * Invariants:
     *     0 <= ob_size <= allocated
     *     len(list) == ob_size
     *     ob_item == NULL implies ob_size == allocated == 0
     * list.sort() temporarily sets allocated to -1 to detect mutations.
     *
     * Items must normally not be NULL, except during construction when
     * the list is not yet visible outside the function that builds it.
     */
    Py_ssize_t allocated;
} PyListObject;
```

The `ob_item` structure member holds pointers to the elements of the list and we'll use [ctypes](https://docs.python.org/3/library/ctypes.html "ctypes") module to re-create the structs in Python:

###### test.py

```python
#!/usr/bin/env python3

import ctypes
import sys

lst = ["red", "blue", "green"]


class PyListObject(ctypes.Structure):
    _fields_ = [
        ("ob_refcnt", ctypes.c_long),
        ("ob_type", ctypes.c_void_p),
        ("ob_size", ctypes.c_long),
        ("ob_item", ctypes.POINTER(ctypes.c_void_p)),
        ("allocated", ctypes.c_long),
    ]


class PyUnicodeObject(ctypes.Structure):
    _fields_ = [
        ("ob_refcnt", ctypes.c_long),
        ("ob_type", ctypes.c_void_p),
        ("length", ctypes.c_ssize_t),
        ("hash", ctypes.c_ssize_t),
        ("interned", ctypes.c_uint, 2),
        ("kind", ctypes.c_uint, 3),
        ("compact", ctypes.c_uint, 1),
        ("ascii", ctypes.c_uint, 1),
        ("ready", ctypes.c_uint, 1),
    ]


def main():
    pylist_obj = PyListObject.from_address(id(lst))

    for idx, s in enumerate(lst):
        addr = pylist_obj.ob_item[idx]
        pyunicode_obj = PyUnicodeObject.from_address(addr)
        s_mem = ctypes.string_at(addr, sys.getsizeof(s))

        print(
            f"{s_mem[-len(s) - 1 : -1].decode()}",
            f"length: {pyunicode_obj.length}",
            f"hash: {pyunicode_obj.hash} ({hash(s)})",
        )


if __name__ == "__main__":
    main()
```

So, in the example above we read directly from the underlying `PyObject` objects:

```
$ python test.py
red length: 3 hash: -2831683529608114332 (-2831683529608114332)
blue length: 4 hash: -1701992703184244183 (-1701992703184244183)
green length: 5 hash: -1362686309841740019 (-1362686309841740019)
```

