+++
date = "2017-06-19T10:07:18+01:00"
title = "Python: get remote IP from HTTP request using the requests module"
tags = ["python", "requests"]
type = "post"
description = "Python: get remote IP from HTTP request using the requests module"
categories = ["Linux"]

+++
#### Introduction:

This is just a quick tip how to get the IP address from a http request using the requests module from Python's standard library.

#### Variables:

```
Python 3.4.5
requests 2.18.1
urllib3 1.21.1
```

In order to get the IP, we need to use "stream" parameter to keep the connection open:

```
r = requests.get('http://exmaple.com', stream=True)
```

At this point only the response headers have been downloaded and the connection remains open until we access the `Response.content`

[Here](http://docs.python-requests.org/en/master/user/advanced/#body-content-workflow "Requests") is the important part for this:

> If you set stream to True when making a request, Requests cannot release the connection back to the pool unless you consume all the data or call Response.close. This can lead to inefficiency with connections. If you find yourself partially reading request bodies (or not reading them at all) while using stream=True, you should make the request within a with statement to ensure it's always closed

We can see this in action using netstat. The Apache web server in this case, waits for the other side to shut down its half of the connection (FIN_WAIT2):

```
# netstat -entp
Active Internet connections (w/o servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       User       Inode      PID/Program name    
...
tcp6       0      0 192.168.1.10:80         192.168.1.254:38610     FIN_WAIT2   0          0          -
```

With [context manager](https://docs.python.org/3.4/library/contextlib.html "contextlib"), the socket is closed at both ends:

```
>>> with requests.get('http://exmaple.com/', stream=True) as r:
...     r.status_code
... 
404
```

\

```
# netstat -entp
Active Internet connections (w/o servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       User       Inode      PID/Program name    
...          
tcp6       0      0 192.168.1.10:80         192.168.1.254:44124     TIME_WAIT   0          0          -
```

Now, let's get back to the main point here, `requests.raw._original_response` is a `http.client.HTTPResponse` instance, `.fp` is the socketfile, which here consists of a buffer wrapping a SocketIO object with the actual socket in the _sock attribute. So the original socket is available as `requests.raw._original_response.fp.raw._sock` and we can call `.getpeername()` on that:

```
>>> with requests.get('http://exmaple.com/', stream=True) as r:
...     r.raw._original_response.fp.raw._sock.getpeername()[0]
... 
'192.168.1.10'
```
