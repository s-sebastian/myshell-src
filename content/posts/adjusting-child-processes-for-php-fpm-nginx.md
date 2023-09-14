+++
categories = ["Linux"]
date = "2012-07-26T15:03:00+01:00"
description = "Adjusting child processes for PHP-FPM (Nginx)"
tags = ["awk", "nginx", "php-fpm"]
# type = "post"
title = "Adjusting child processes for PHP-FPM (Nginx)"

+++

#### Problem:

The following warning message appears in the logs:

```
[26-Jul-2012 09:49:59] WARNING: [pool www] seems busy (you may need to increase pm.start_servers, or pm.min/max_spare_servers), spawning 32 children, there are 8 idle, and 58 total children
[26-Jul-2012 09:50:00] WARNING: [pool www] server reached pm.max_children setting (50), consider raising it
```

It means that there are not enough PHP-FPM processes.

#### Solution:

We need to calculate and change these values based on the amount of memory on the system:

**/etc/php-fpm.d/www.conf**

```
pm.max_children = 50
pm.start_servers = 5
pm.min_spare_servers = 5
pm.max_spare_servers = 35
```

\- the following command will help us to determine the memory used by each (PHP-FPM) child process:

    ps -ylC php-fpm --sort:rss

The RSS column shows non-swapped physical memory usage by PHP-FPM processes in kilo Bytes.

On an average each PHP-FPM process took ~75MB of RAM on my machine.

Appropriate value for **pm.max_children** can be calculated as:

pm.max_children = Total RAM dedicated to the web server / Max child process size - in my case it was 85MB

The server has 8GB of RAM, so:

pm.max_children = 6144MB / 85MB = 72

I left some memory for the system to breath. You need to take into account any other services running on the machine while calculating memory usage.

I've changed the settings as follow:

```
pm.max_children = 70
pm.start_servers = 20
pm.min_spare_servers = 20
pm.max_spare_servers = 35
pm.max_requests = 500
```

Please note that very high values does not mean necessarily anything good.

You can check an average memory usage by single PHP-FPM process with this handy command:

    ps --no-headers -o "rss,cmd" -C php-fpm | awk '{ sum+=$1 } END { printf ("%d%s\n", sum/NR/1024,"M") }'

You can use the same steps above to calculate the value for **MaxClients** for Apche web server - just substitute the **php-fpm** with **httpd**.
