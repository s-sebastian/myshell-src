+++
type = "post"
description = "LAMP: Troubleshooting slow website with strace and lsof"
categories = ["Linux"]
date = "2017-02-20T10:29:28Z"
title = "LAMP: Troubleshooting slow website with strace and lsof"
tags = ["apache", "lamp", "lsof", "strace"]

+++

#### Problem:

I was recently debugging a slow website. When tested in the browser, it was not showing any content and just continuously loading.
It was hosted on a big web farm behind a load balancer, with no external access to individual web nodes.

#### Solution:

In order to strace a single Apache process I had to access this site from a web server itself:

I changed the `User-Agent` header to easily grep through the logs.

```
# curl -v -s -o /dev/null -A "DEBUG" -H "Host: exmaple.com"
http://10.10.0.10:80/
* About to connect() to 10.10.0.10 port 80 (#0)
*
Trying 10.10.0.10... connected
* Connected to 10.10.0.10 (10.10.0.10) port 80 (#0)
> GET / HTTP/1.1
> User-Agent: DEBUG
> Accept: */*
> Host: example.com
>
```

This was taking to long and after a few minutes we finally got a reply:

```
< HTTP/1.1 200 OK
< Date: Fri, 17 Feb 2017 11:10:44 GMT
< Server: Apache
< X-Frame-Options: DENY
< Cache-Control: no-cache, no-store
< Expires: -1
< Pragma: no-cache
< Set-Cookie:
cp_session=Xp5xCpbtRVRrZev22rcXkxMk4rIjtc8S6o8ndE8yXlIIVlMzD-nE1HxPmY9KVDBawpzgo%2CH-9o8wVy3%2CLPT5A1; path=/; httponly
< Content-Length: 68053
< Vary: User-Agent,Accept-Encoding
< Content-Type: text/html; charset=UTF-8
<
{ [data not shown]
* Connection #0 to host 10.70.0.64 left intact
* Closing connection #0
```

While curl is still running, we'll check its source port, it will help us to identify PID for Apache:

```
# lsof -i 4 -a -c curl
COMMAND   PID  USER   FD   TYPE     DEVICE SIZE/OFF NODE NAME
curl    18421  root   3u  IPv4 1556243712      0t0  TCP web01.local:59054->web01.local:http (ESTABLISHED)
```

Now, we can find Apache process that is communicating with curl on port from the command above:

```
# lsof -a -c httpd -i TCP:59054
COMMAND  PID   USER   FD   TYPE     DEVICE SIZE/OFF NODE NAME
httpd   4955 apache    8u  IPv4 1556230695      0t0  TCP web.local:http->web01.local:59054 (ESTABLISHED)
```

Time to see what is taking so long:

```
# strace -ttT -s 512 -f -p 4955
Process 4955 attached
04:58:29.245680 epoll_wait(10,
```

OK, so it's waiting on some event using file descriptor number 10, let's check that:

```
# lsof -a -d 10 -p 4955
COMMAND  PID   USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
httpd   4955 apache   10u  0000    0,9        0 5380 anon_inode
```

That's not helpful, let's dig deeper and check all threads:

```
# pstree -Ap 4955
httpd(4955)---php(18427)
```
**NOTE:** another way to check the threads, is to use "ps" command with the "-L" switch and look for "LWP" column: `ps -efL | grep 4955`

We'll check what this bad boy is doing and also exclude some system calls that make to much noise:

```
# strace -ttT -s 512 -f -e 'trace=!clock_gettime' -p 18427
Process 18427 attached
[ Process PID=18427 runs in 32 bit mode. ]
05:00:06.797195 restart_syscall(<... resuming interrupted call ...>) = 0 <0.825651>
05:00:07.622969 poll([{fd=13, events=POLLIN|POLLPRI}], 1, 0) = 0 (Timeout) <0.000019>
05:00:07.623196 poll([{fd=13, events=POLLIN|POLLPRI}], 1, 1000) = 0 (Timeout) <1.000856>
05:00:08.624109 poll([{fd=13, events=POLLIN|POLLPRI}], 1, 0) = 0 (Timeout) <0.000010>
05:00:08.624287 poll([{fd=13, events=POLLIN|POLLPRI}], 1, 1000) = 0 (Timeout) <1.001037>
05:00:09.625374 poll([{fd=13, events=POLLIN|POLLPRI}], 1, 0) = 0 (Timeout) <0.000011>
05:00:09.625544 poll([{fd=13, events=POLLIN|POLLPRI}], 1, 1000^CProcess 18427 detached
 <detached ...>
```

Gotcha Ya, it's trying to read from FD 13, an external IP that is timing out:

```
# lsof -a -d 13 -p 18427
COMMAND   PID    USER   FD   TYPE     DEVICE SIZE/OFF NODE NAME
php     18427 nonpriv   13u  IPv4 1556386925      0t0  TCP web01.local:54981->xxx.xxx.xxx.xxx:https (ESTABLISHED)
```

In my case that was a PHP framework, pulling content from external sources.
