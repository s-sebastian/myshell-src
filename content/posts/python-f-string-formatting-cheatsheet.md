+++
categories = ["Linux"]
date = "2018-11-05T11:05:30Z"
title = "Python: f-string formatting cheatsheet"
tags = ["python", "f-string"]
description = "Python: f-string formatting cheatsheet"
+++

**Note:** All examples below were executed in Python interactive shell (REPL - Read, Eval, Print and Loop).

#### Format specification:
> [[fill]align][sign][#][0][width][grouping_option][.precision][type]

\- [width]

```python
>>> f'{"text":10}'
'text      '
```

\- [[fill]align]

```python
>>> f'{"test":#>10}'
'######test'
>>> f'{"test":#<10}'
'test######'
>>> f'{"test":#^10}'
'###test###'
```

\- [[fill]align] with numbers

```python
>>> f'{12345:0>10}'
'0000012345'
```
negative numbers

```python
>>> f'{-12345:0=10}'
'-000012345'
```

\- [0] shortcut (no align)

```python
>>> f'{12345:010}'
'0000012345'
>>> f'{-12345:010}'
'-000012345'
```

\- [.precision]

```python
>>> import math
>>> math.pi
3.141592653589793
>>> f'{math.pi:.2f}'
'3.14'
```

\- [grouping_option]

```python
>>> f'{1000000:,.2f}'
'1,000,000.00'
>>> f'{1000000:_.2f}'
'1_000_000.00'
```

\- [sign] \(+/-)

```python
>>> f'{12345:+}'
'+12345'
>>> f'{-12345:+}'
'-12345'
>>> f'{-12345:+10}'
'    -12345'
>>> f'{-12345:+010}'
'-000012345'
```

\- [type]

binary

```python
>>> f'{10:b}'
'1010'
```

octal

```python
>>> f'{10:o}'
'12'
```

hexadecimal

```python
>>> f'{200:x}'
'c8'
>>> f'{200:X}'
'C8'
```

scientific notation

```python
>>> f'{345600000000:e}'
'3.456000e+11'
```

character type

```python
>>> f'{65:c}'
'A'
```

\- [type] with notation (base)

```python
>>> f'{10:#b}'
'0b1010'
>>> f'{10:#o}'
'0o12'
>>> f'{10:#x}'
'0xa'
```

\- percentage (multiply by 100)

```python
>>> f'{0.25:0%}'
'25.000000%'
>>> f'{0.25:.0%}'
'25%'
```
