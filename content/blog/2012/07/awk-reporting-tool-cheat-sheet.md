+++
categories = ["Scripts"]
date = "2012-07-14T19:02:00+01:00"
description = "Awk - Reporting Tool - Cheat Sheet"
tags = ["bash", "awk", "linux"]
type = "post"
title = "Awk - Reporting Tool - Cheat Sheet"

+++

#### Usage:

awk '/pattern/ { procedure }' | PIPE | STDIN

Example file:

**file.txt**

```
one
two
three
one1
two10
three one
four two
one two three
```

\- print entire line:

    awk '{ print }' file.txt

or

    awk '{ print $0 }' file.txt

\- print specific columns ( $1, $2 .. $n):

    awk '{ print $1}' file.txt

\- print multiply columns:

    awk '{ print $1; print $2 }' file.txt

\

    awk '{ print $1, $2 }' file.txt

\- print columns from lines containing pattern:

    awk '/pattern/ { print $1 }' file.txt

\- print columns from lines containing digits:

    awk '/[0-9]/ { print $1 }' file.txt

#### Delimiters:

Default delimiter: white-space (space, tabs):

    awk -F: '{ print $1 }' /etc/passwd

\- support for character classes in setting the default delimiter:

    awk -F "[:;.\t]"

#### Awk scripts:

Awk scripts consist of 3 parts:

1\. Before (denoted using: BEFORE)

2\. During (main Awk loop)

3\. After (denoted using: END)

    awk 'BEGIN { print "exmaple" }'

\

    awk 'BEGIN { FS = ":"; print "Beginning" } $7 ~ /nologin/ { print $1, $7 } END { print "End" }' /etc/passwd

Example:

    awk -f example.awk /etc/passwd

**example.awk**

```
# Component 1 - BEGIN
BEGIN { FS = ":"; print "Beginning" }

# Component 2 - Main Loop
$7 ~ /false/ { print $1, $7 }

# Component 3 - END
END { print "End" }
```

#### Awk variables:

Types:

1\. System - i.e. FILENAME, RS, ORS...

    awk '{ print; print "Number of fields on the line: " NF } END { print "Input file: " FILENAME }' file.txt

\

    awk 'BEGIN { OFS="\t\t\t" }; { print $1, $2 }' file.txt

2\. Scalars - i.e. a = 1

    awk 'BEGIN { test_value = 10 } { print } { print test_value }' file.txt

\- increment scalar variable “test_value” by one:

    awk 'BEGIN { test_value = 10 } { print } { print test_value; ++test_value }' file.txt

3\. Arrays - i.e. (variable_name[n]) test_value[0] = 10

    awk '{ print $1, $2; class[NR] = $2 } END { for (i=1; i <= NR; i++) print "Class" i ": "class[i] }' file.txt

#### Awk operators:

1\. Relational - ==, !=, <, >, <=, >=, ~ (RegEx matches), !~ (RegEx does NOT match)

\- print lines with two or more records:

    awk 'NF >=2 { print }' file.txt

\- print lines where second field match pattern:

    awk '$2 ~ /pattern/ { print }' file.txt

2\. Boolen - || (OR), && (AND), ! (NOT)

\-print records that have at least 2 fields and are positioned at record 6 and higher:

    awk 'NF >=2 && NR >=6 { print }' file.txt

Awk 'if' statement:

    awk '{ if ( $1 ~ /four/ ) print $2 }' file.txt

\

    awk '{ if ( $1 == "four" ) print $2; else print $1 }' file.txt

#### Awk loops:

\- **while**, **do** and **for**

\-examples:

    awk '{ for(i=1; i<=5; ++i) print $0,i }' file.txt

\

    awk 'BEGIN { for (i=1; i <= 10; ++i) print i }'

\

    awk 'BEGIN { for (i=1; i <= ARGV[1]; ++i) print i }' 10

\

    awk 'BEGIN { max=ARGV[1]; for (i=1; i <= max; ++i) print i }' 10

#### Awk Printf formatting:

Usage:

printf ("format", arguments)

Supported Printf formats:

1\. "%c" - ASCII characters

2\. "%d" - Decimals - NOT floating point values OR values to the right of the decimal point

3\. "%f" - Floating point

4\. "%s" - Strings

**NOTE:** Printf doesn’t print newline character(s)

**NOTE:** Default output is right-justified. Use ‘-‘ to indicate left-justification

General format section:

[-]width.precision[cdfs]

1\. width - influence the actual width of the column to be output

2\. precision - influence the number of places to the right of the decimal point

\- print examples:

    awk 'BEGIN { printf("test\n") }'

\

    awk 'BEGIN { printf ("Output:\n") } { printf ("%s\n", $1) }' file.txt

\

    awk 'BEGIN { printf ("Output:\n") } { printf ("%s\t%s\n", $1,$2 ) }' file.txt

\- apply precision:

    awk 'BEGIN { printf ("Output:\n") } { printf ("%.3s\n", $1 ) }' file.txt

\- apply width:

    awk 'BEGIN { printf ("Output:\n") } { printf ("%20s\t%20s\n", $1,$2 ) }' file.txt

\- apply width (output left-justified)

    awk 'BEGIN { printf ("Output:\n") } { printf ("%-20s\t%-20s\n", $1,$2 ) }' file.txt

Example file:

**file2.txt**

```
Ferrari 100000.67
Porshe 250000
Lamborgini 350000.99
```

\

    awk 'BEGIN { printf ("Price list:\n\n") } { printf ("%-10s\t£%.2f\n", $1,$2 ) }' file2.txt

\

    awk '{ cars[NR] = $1 } END { print "Total Command-line Arguments: " ARGC; for ( i=1; i <= NR; i++) printf ("%-12s %1d %-2s %-10s\n", "CARS", i, ": ", cars[i] ) }' file2.txt

\- apply upper and lower-case formatting to Printf values:

    awk '{ cars[NR] = $1 } END { for ( i=1; i <= NR; i++) printf ("%-12s %1d %-2s %-10s\n", "CARS", i, ": ", toupper(cars[i]) ) }' file2.txt

\

    awk '{ cars[NR] = $1 } END { for ( i=1; i <= NR; i++) printf ("%-12s %1d %-2s %-10s\n", "CARS", i, ": ", tolower(cars[i]) ) }' file2.txt

\- include headers:

    awk '{ cars[NR] = $1 } END { printf ("%-12s %-1s %10s\n\n", "Cars", "Count", "Make"); for ( i=1; i <= NR; i++) printf ("%-12s %1d %-9s %-15s\n", "CARS", i, ": ", cars[i] ) }' file2.txt
