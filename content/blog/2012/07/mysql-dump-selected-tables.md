+++
categories = ["Database"]
date = "2012-07-17T17:15:00+01:00"
description = ""
tags = ["mysql", "mysqldump"]
type = "post"
title = "MySQL: dump selected tables"

+++

Today I had to dump a dozen of tables from one MySQL database. I could list them all on the command line, something like this:

    mysqldump -u <user> -p <database> <table1> <table2> <table3> > dump.sql

\- however this can be quite difficult if you have lots of tables to dump.

There is a much easier solution for this:

    mysql databasename -u <root> -p -e "SHOW TABLES LIKE 'wp_wpsqt%'" | grep -v Tables | xargs mysqldump <databasename> -u <root> -p > dump.sql

\- if you need dump each table into a separate file for some reason use **for loop**:

    for tables in $(mysql databasename -u <root> -p -e "SHOW TABLES LIKE 'wp_wpsqt%'" | grep -v Table); do mysqldump <databasename> -u <root> -p $tables > $tables.sql; done
