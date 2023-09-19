+++
categories = ["Scripts"]
description = "Sed - UNIX Stream Editor - Cheat Sheet"
tags = ["bash", "sed"]
date = "2012-07-07T13:11:00+01:00"
title = "Sed - UNIX Stream Editor - Cheat Sheet"
+++

#### Usage:

sed [options] ‘instructions’ file.txt | PIPE | STDIN

Example file:

**file.txt**

    one
    two
    three
    one1
    two10
    three one

#### Meta-characters:

"^" \- matches character(s) at the beginning of a line:

    sed -ne '/^pattern/p'  file.txt

"$" \- matches character(s) at the end of a line:

    sed -ne '/pattern$/p'  file.txt

\- match line which contain only "pattern":

    sed -ne '/^pattern$/p'  file.txt

#### RegEx quantifiers:

"." - matches any character (typically expect new line)

"*" - 0 or more matches of the previous character

"+" - 1 or more matches of the previous character

"?" - 0 or 1 of the previous character

#### Character classes:

a. [0-9]

b. [a-z]

NOTE: Character classes match one and only one character.

\- print first line:

    sed -ne '1p' file.txt

\- prints last printable line:

    sed -ne '$p' file.txt

\- prints lines 2-4 from file:

    sed -ne '2,4p' file.txt

\- prints all lines except line 1:

    sed -ne '1!p' file.txt

\- prints all lines with at least one numeric charter:

    sed -ne '/[0-9]/p' file.txt

\- prints the line with "pattern" plus 2 extra lines:

    sed -ne '/pattern/, +2p' file.txt

\- print section of file from a line containing "pattern" to end of file:

    sed -ne '/pattern/, $p' file.txt

\- delete blank lines:

    sed -e '/^$/d' file.txt

\- deletes every 2nd line beginning with line 1 (1,3,5...):

    sed -e '1~2d' file.txt

\- replace pattern (case insensitive):

    sed -e 's/one/two/Ig' file.txt

\- edit files in place and backup:

    sed -i.bak -e 's/one/two/Ig' file.txt

\- multiply instructions - removes blank lines & substitutes patterns:

    sed -e '/^$/d' -e 's/one/two/Ig' file.txt

or

    sed -e '/^$/d; s/one/two/Ig' file.txt

\- search and replace - substitutes "one" with "two" where line contains "three":

    sed -ne '/three/s/one/two/gp' file.txt

#### SED reserves few characters based on matched pattern:

"&" - pattern matched OR the values in the pattern space

    sed -ne 's/.*/&/p' file.txt

    sed -ne 's/.*/Example: &/p' file.txt

\- returns pattern with at least 1 numeric at the end of the name:

    sed -ne 's/.*[0-9]/&/p' file.txt

\- returns pattern with only 1 numeric at the end of the name:

    sed -ne 's/[a-z][0-9]\{1\}$/&/pI' file.txt

\- returns pattern with at least 1, up to 4 numeric values:

    sed -ne 's/[a-z][0-9]\{1,4\}$/&/pI' file.txt

#### Grouping & Back-references:

\- creates two variables \1 & \2 and references \1 & \2

    sed -ne 's/\([a-z]\)\([0-9]\)/\1 \2/p' file.txt
