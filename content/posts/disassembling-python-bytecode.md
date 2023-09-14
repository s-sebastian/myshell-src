+++
categories = ["Linux"]
date = "2019-03-12T12:31:36Z"
title = "Disassembling Python bytecode"
tags = ["python", "dis"]
# type = "post"
description = "Disassembling Python bytecode"

+++
A reference guide for disassembling Python bytecode.

#### Example:

```
>>> def test(value):
...     return len(value)
... 
>>> import dis
>>> dis.dis(test)
  2           0 LOAD_GLOBAL              0 (len)
              2 LOAD_FAST                0 (value)
              4 CALL_FUNCTION            1
              6 RETURN_VALUE
```

#### Explanation:

```
(1)|(2)|(3)|(4)|          (5)         |(6)|  (7)
---|---|---|---|----------------------|---|-------
  2|   |   |  0|LOAD_GLOBAL           |  0|(len)
   |   |   |  2|LOAD_FAST             |  0|(value)
   |   |   |  4|CALL_FUNCTION         |  1|
   |   |   |  6|RETURN_VALUE          |   |
```

The columns returned are the following:

1. The corresponding **line number** in the source code
2. Optionally indicates the **current instruction** executed (when the bytecode comes from a [frame object](https://docs.python.org/3/library/inspect.html#the-interpreter-stack  "frame object") for example)
3. A label which denotes a possible **`JUMP` from an earlier instruction** to this one
4. The **address** in the bytecode which corresponds to the byte index (those are multiples of 2 because Python 3.6 use 2 bytes for each instruction, while it could vary in previous versions):
```
>>> code = test.__code__.co_code
>>> code
b't\x00|\x00\x83\x01S\x00'
>>> l = list(code)
>>> l
[116, 0, 124, 0, 131, 1, 83, 0]
>>> l[0]
116
>>> l[2]
124
>>> l[4]
131
>>> l[6]
83
```

5. The instruction name (also called **opname**), each one is briefly explained in the [`dis`](https://docs.python.org/3/library/dis.html#python-bytecode-instructions "dis") module:
```
>>> dis.opname[116]
'LOAD_GLOBAL'
>>> dis.opname[124]
'LOAD_FAST'
>>> dis.opname[131]
'CALL_FUNCTION'
>>> dis.opname[83]
'RETURN_VALUE'
```

6. The **argument** (if any) of the instruction which is used internally by Python to fetch some constants or variables, manage the stack, jump to a specific instruction, etc.
```
>>> test.__code__.co_names[0]
'len'
>>> test.__code__.co_varnames[0]
'value'
```

7. The **human-friendly interpretation** of the instruction argument

Some examples for a bytecode operations:

```
>>> test.__code__
<code object test at 0x7f9905577e40, file "<stdin>", line 1>
>>> test.__code__.co_filename
'<stdin>'
>>> test.__code__.co_consts
(None,)
>>> test.__code__.co_varnames
('value',)
>>> test.__code__.co_names
('len',)
>>> test.__code__.co_consts
(None,)
>>> test.__code__.co_argcount
1
>>> test.__code__.co_code
b't\x00|\x00\x83\x01S\x00'
>>> list(test.__code__.co_code)
[116, 0, 124, 0, 131, 1, 83, 0]
>>> dis.opname[116]
'LOAD_GLOBAL'
>>> dis.opmap['LOAD_GLOBAL']
116
```
