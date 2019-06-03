+++
categories = ["Linux"]
date = "2019-05-28T14:30:20+01:00"
title = "Negative numbers in binary"
tags = ["binary", "signed numbers"]
type = "post"
description = "Negative numbers in binary"

+++
This blog post is primarily based on Wikipedia article about [Signed number representations](https://en.wikipedia.org/wiki/Signed_number_representations "Signed number representations").

We'll discuss three methods of extending the binary numeral system to represent signed numbers:

- [Signed magnitude representation (SMR)](#signed-magnitude-representation-smr "Signed magnitude representation")
- [Ones' complement (1C)](#ones-complement-1c "Ones' complement")
- [Two's complement (2C)](#two-s-complement-2c "Two's complement")

The leftmost bit of a **signed** integer (known as the **sign bit**) is 0 if the number is positive or zero, 1 if it's negative.

**Note:** We'll use 4-bits to simplify our examples, 1-bit for *sign bit* and the remaining 3-bits in the number indicate the *magnitude* (absolute value). This will give us a range from 000 binary (0 decimal) to 111 binary (7 decimal).

#### Signed magnitude representation (SMR)

> In this approach, a number's sign is represented with a sign bit: setting that bit (often the most significant bit) to 0 for a positive number or positive zero, and setting it to 1 for a negative number or negative zero. The remaining bits in the number indicate the magnitude (or absolute value).

So using our 4-bit system here are the possible values:

Value|Signed magnitude|Value|Signed magnitude
:---:|:--------------:|:---:|:--------------:
0    |(0)000          |-0   |(1)000
1    |(0)001          |-1   |(1)001
2    |(0)010          |-2   |(1)010
3    |(0)011          |-3   |(1)011
4    |(0)100          |-4   |(1)100
4    |(0)101          |-5   |(1)101
6    |(0)110          |-6   |(1)110
7    |(0)111          |-7   |(1)111

We can see that we have multiple representations of zero (+0, -0) but the main drawback is that a simple addition won't work:

A quick recap how we do a conventional binary addition:

```
0 + 0 = 0
0 + 1 = 1
1 + 0 = 1
1 + 1 = 0 (1) ← carry 1
```

Now, let's consider the following example showing the case of the addition of +5 ((0)101)) to -3 ((1)011):

```
   binary    decimal
    ¹ ¹¹ 
   (0)101    +5
 + (1)011    -3
 ────────    ──
 ¹ (0)000     0  ← Not the correct answer
```

#### Ones' complement (1C)

> The ones' complement form of a negative binary number is the bitwise NOT applied to it, i.e. the "complement" of its positive counterpart. Like sign-and-magnitude representation, ones' complement has two representations of 0.

So basically we flip the bits for negative numbers (bitwise NOT) but we still have multiple representations of zero:

Value|Ones' complement|Value|Ones' complement
:---:|:--------------:|:---:|:--------------:
0    |(0)000          |-0   |(1)111
1    |(0)001          |-1   |(1)110
2    |(0)010          |-2   |(1)101
3    |(0)011          |-3   |(1)100
4    |(0)100          |-4   |(1)011
5    |(0)101          |-5   |(1)010
6    |(0)110          |-6   |(1)001
7    |(0)111          |-7   |(1)000

Addition works now but it is then necessary to do an end-around carry. That is, add any resulting carry back into the resulting sum:

```
   binary    decimal
    ¹ ¹¹ 
   (0)101    +5
 + (1)100    -3
 ────────    ──
       ¹ 
 ¹ (0)001     1  ← Not the correct answer
        1    +1  ← Add carry
 ────────    ──
   (0)010     2  ← Correct answer
```

#### Two's complement (2C)

> The problems of multiple representations of 0 and the need for the end-around carry are circumvented by a system called two's complement. In two's complement, negative numbers are represented by the bit pattern which is one greater (in an unsigned sense) than the ones' complement of the positive value.
>
> In two's-complement, there is only one zero, represented as 000. Negating a number (whether negative or positive) is done by inverting all the bits and then adding one to that result.

So in this method we have to negate all bits and add one, let's see this using zero:

```
   binary    decimal
   (0)000    +0
   (1)111    -0  ← We negate all bits first (1C)
 +      1        ← Add one
 ────────    ──
    ¹ ¹¹ 
 ¹ (0)000     0  ← We simply ignore carry here
```

We have 14 values now (from 1 to 7 and -1 to -7), both zeros have the 15th value. The 16th value is the first negative number that is outside of our range (-8):

Value|Two's complement|Value  |Two's complement
:---:|:--------------:|:-----:|:--------------:
0    |(0)000          |**-8** |**(1)000**
1    |(0)001          |  -1   |  (1)111
2    |(0)010          |  -2   |  (1)110
3    |(0)011          |  -3   |  (1)101
4    |(0)100          |  -4   |  (1)100
5    |(0)101          |  -5   |  (1)011
6    |(0)110          |  -6   |  (1)010
7    |(0)111          |  -7   |  (1)001

A few more examples with different `int` size (bits):

```
8-bit int
-128 .. +127

16-bit int
-32768 .. +32767

32-bit int
-2147483648 .. +2147483647
```

Let's also ensure that addition still works:

```
   binary    decimal
    ¹  ¹ 
   (0)101    +5
 + (1)101    -3
 ────────    ──
 ¹ (0)010     2  ← Correct answer
```

#### Integer overflow

We should now have a better understanding of *integer overflow*, a condition that occurs when an integer calculation produces a result that is outside of the available range, for instance `7 + 3 = 10` - both operands are within our range but the result is not, let's test it:

```
   binary    decimal
    ¹ ¹¹
   (0)111    +7
 + (0)011    +3
 ────────    ──
 º (1)010    -6  ← Not the correct answer (C2 notation)
```

First of all we have a special case for binary addition here, if both operands are 1 (one) and we also have to add carry, we set the result to 1 (one) and also add carry as well.

Secondly, we can see that the result is not correct and we have an integer overflow.
We can use the following rule to check for this condition - if carry value going into the sign bit is different from the one going out, like in our example, we have an overflow.

#### Example:

```
$ cat example.c 
#include <stdio.h>

int main()
{
    unsigned short int x = 65535;
    printf("sizeof: %zu, %%hu: %hu, %%hi: %hi\n", sizeof(x), x, x);

    return 0;
}
$ gcc -o example example.c
$ ./example 
sizeof: 2, %hu: 65535, %hi: -1
```

In the example above we declare a variable of type *unsigned short int*.  The size for *unsigned short* integer is usually 2 bytes (16 bits) and it can store only positive values (0 to 65535). We then print this variable using two format specifiers:

- `%hu` - Unsigned integer (short)
- `%hi` - Signed integer (short)

The first one says the 16 bits are to be interpreted as an unsigned integer so we get `65535` however the second one interprets the value as a signed integer so the most significant bit is used to hold the sign bit, thus `(1)111 1111 1111 1111` represents `-1` in [two's complement](#two-s-complement-2c "two's complement") notation:

```
$ python3 -c 'print(65535 - (1 << 16))'
-1
```
