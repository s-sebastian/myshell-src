+++
categories = "Scripts"
description = ""
tags = ["init", "lsyncd"]
type = "post"
date = "2012-07-07T11:35:00+01:00"
title = "Lsyncd “init.d” script for CentOS/RedHat"

+++

[Lsyncd](https://code.google.com/p/lsyncd/ "Lsyncd") – Live Syncing (Mirror) Daemon.

You can obtain an RPM package from EPEL repository.

Configuration file example:

**/etc/lsyncd.conf**

```
settings = {

       log = "scarce",

       logfile = "/var/log/lsyncd.log",

       statusFile = "/var/log/lsyncd-status.log",

       statusInterval = 1 
}

    sync{

       default.rsyncssh,

       source = "/sites/",

       host = "192.168.1.1",

       targetdir = "/data/",

       rsyncOps = {"-zlptgoD", "--delete"}
}
```

**/etc/init.d/lsyncd**

Don’t forget to add execute permission:

    # chmod 755 /etc/init.d/lsyncd

\

```
#!/bin/bash
#
# lsyncd: Starts the lsync Daemon
#
# chkconfig: 345 80 30
# description: Lsyncd uses rsync to synchronize local directories with a remote
# machine running rsyncd. Lsyncd watches multiple directories
# trees through inotify. The first step after adding the watches
# is to, rsync all directories with the remote host, and then sync
# single file buy collecting the inotify events.
# processname: lsyncd
# config: /etc/lsyncd.conf
# pidfile: /var/run/lsyncd.pid

# Source function library.
. /etc/init.d/functions

RETVAL=0
PIDFILE="/var/run/lsyncd.pid"
LOCKFILE="/var/lock/subsys/lsyncd"
LSYNCD="/usr/bin/lsyncd"
CONFIG="/etc/lsyncd.conf"
PROG="lsyncd"

start() {
        echo -n "Starting $PROG: "

        if [ -f $PIDFILE ]; then
                PID=`cat $PIDFILE`
                echo $PROG already running: $PID
                exit 1;
        else
                daemon --pidfile=$PIDFILE $LSYNCD -pidfile $PIDFILE $CONFIG
                RETVAL=$?
                echo
                [ $RETVAL -eq 0 ] && touch $LOCKFILE
                return $RETVAL
        fi

}

stop() {
        echo -n "Stopping $PROG: "

        killproc lsyncd
        echo
        rm -f $LOCKFILE
        return 0

}

case "$1" in
    start)
        start
        ;;
    stop)
        stop
        ;;
    status)
        status lsyncd
        ;;
    restart)
        stop
        start
        ;;
    *)
        echo "Usage:  {start|stop|status|restart}"
        exit 1
        ;;
esac
exit $?
```
