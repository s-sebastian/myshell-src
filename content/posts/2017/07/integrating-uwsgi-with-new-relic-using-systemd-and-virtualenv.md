+++
categories = ["Linux"]
date = "2017-07-05T10:13:31+01:00"
title = "Integrating uWSGI with New Relic using systemd and virtualenv"
tags = ["python", "django", "virtualenv", "uwsgi", "new relic"]
description = "Integrating uWSGI with New Relic using systemd and virtualenv"
+++

#### Introduction:

This article explains how to integrate uWSGI with New Relic for Python. We'll use the [admin script](https://docs.newrelic.com/docs/agents/python-agent/hosting-mechanisms/python-agent-uwsgi#wrapper-script "admin script"). This method is recommended because it is easy and doesn't require making changes to your app code.

#### Variables:

```
CentOS 7.3.1611
Python 3.4.5
uwsgi 2.0.15
```

Install Python 3 and uWSGI from [EPEL](https://fedoraproject.org/wiki/EPEL "EPEL") (Extra Packages for Enterprise Linux) repository:

```sh-session
# yum install -y python34 python34-pip uwsgi uwsgi-plugin-python3
```

Create Python virtual environment:

```sh-session
# mkdir -p /projects/myapp
# virtualenv --python=/usr/bin/python3 /projects/myapp/venv34
```

Install New Relic agent in your virtualenv and generate the agent configuration file:

```sh-session
# source /projects/myapp/venv34/bin/activate
(venv34) # pip install newrelic
(venv34) # newrelic-admin generate-config YOUR_LICENSE_KEY newrelic.ini
```

We need to modify the existing systemd service unit for uWSGI to prefix the `ExecStart` command with `newrelic-admin` script:

```sh-session
# cp /usr/lib/systemd/system/uwsgi.service /etc/systemd/system/
```

```diff
# diff -u /usr/lib/systemd/system/uwsgi.service /etc/systemd/system/uwsgi.service
--- /usr/lib/systemd/system/uwsgi.service	2017-05-19 15:31:54.000000000 +0100
+++ /etc/systemd/system/uwsgi.service	2017-07-05 09:38:04.413117252 +0100
@@ -4,9 +4,10 @@
 
 [Service]
 EnvironmentFile=-/etc/sysconfig/uwsgi
+Environment="NEW_RELIC_CONFIG_FILE=/projects/myapp/newrelic.ini"
 ExecStartPre=/bin/mkdir -p /run/uwsgi
 ExecStartPre=/bin/chown uwsgi:uwsgi /run/uwsgi
-ExecStart=/usr/sbin/uwsgi --ini /etc/uwsgi.ini
+ExecStart=/projects/myapp/venv34/bin/python /projects/myapp/venv34/bin/newrelic-admin run-program /usr/sbin/uwsgi --ini /etc/uwsgi.ini
 ExecReload=/bin/kill -HUP $MAINPID
 KillSignal=SIGINT
 Restart=always
```

This is my uWSGI config for Django but the above setup should work with any other web framework like Flask etc.:

```ini
# cat /etc/uwsgi.d/myapp.ini
[uwsgi]

# Variables
base = /projects/myapp

# Generic Config
auto-procname = true
procname-prefix-spaced = [%i]
plugins = python3
virtualenv = %(base)/venv34
pythonpath = %(base)/myapp
module = myapp.wsgi
logto = /var/log/uwsgi/%n.log
socket = /run/uwsgi/%n.sock
chmod-socket = 777
harakiri = 30
vacuum = true
max-requests = 1000
enable-threads = true
processes = 50
cheaper-algo = spare
cheaper = 2
cheaper-step = 1

```

Reload the configuration and start uWSGI service:

```sh-session
# systemctl daemon-reload
# systemctl enable uwsgi.service
# systemctl start uwsgi.service
```
