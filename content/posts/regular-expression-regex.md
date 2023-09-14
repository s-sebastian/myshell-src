+++
categories = ["Linux"]
date = "2012-07-27T20:49:00+01:00"
description = "Regular-Expression (RegEx)"
tags = ["regex"]
title = "Regular-Expression (RegEx)"
+++

#### Meta-characters:

Basic RegExs

1\. Alphanumeric (a-z, A-Z, 0-9, _) - literal characters

Meta-characters

1\. "*" - matches zero or more times the preceding token

i.e. [Ll]inux[0-9]* - matches 'linux', 'Linux', 'Linux1', 'Linux[n]'

2\. "?" - marks preceding token as optional - zero OR one time

i.e. favou?r - matches 'favor', 'favour'

3\. "+" - matches the preceding token one or more times

i.e. Linu+x - matches 'Linux', 'Linuux'

4\. "[ ]" - brackets -are used to define character classes. The match ONE charter and NOT a group of characters

i.e Linux[Xx0-9] - matches 'Linuxx', 'LinuxX', 'Linux0-9'

5\. "( )" - parentheses - are used to group characters and to constrain alternation

**NOTE:** Parentheses permit the grouping of several characters (words) for matches

i.e (Linux)+

6\. "^" - anchors text at the beginning, including at the beginning of a line

7\.  "$" - anchors text at the end, including at the end of a line

8\. "|" - matches alternate character

9\. "." - matches any characters except line breaks

10\. "\" - used to escape the following character

11\. "{max}" - quantifier for preceding token

12\. "{min,max}" - {3,5} - matches at least 3 times and up to 5 times

i.e Linu[x]{2,3} - matches 'Linuxx', 'Linuxxx'
