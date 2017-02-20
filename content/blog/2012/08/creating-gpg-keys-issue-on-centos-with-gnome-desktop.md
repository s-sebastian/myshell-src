+++
categories = ["Linux"]
date = "2012-08-22T12:41:00+01:00"
description = "Creating GPG keys issue on CentOS with GNOME Desktop"
tags = ["gpg", "passphrase", "pinentry"]
type = "post"
title = "Creating GPG keys issue on CentOS with GNOME Desktop"

+++

#### Variables:

    OS: CentOS 6.3 64bit - GNOME minimal Desktop installation.

#### Problem:

When you trying to generate a GPG key the process is failing with the following errors:

You need a Passphrase to protect your secret key.

```
can't connect to `/root/.gnupg/S.gpg-agent': No such file or directory
gpg-agent[2556]: directory `/root/.gnupg/private-keys-v1.d' created
Please install pinentry-gui
gpg-agent[2556]: can't connect server: ec=4.16383
gpg-agent[2556]: can't connect to the PIN entry module: End of file
gpg-agent[2556]: command get_passphrase failed: No pinentry
gpg: problem with the agent: No pinentry
gpg: Key generation canceled.
```

#### Solution:

Install one of the "Passphrase/PIN entry dialog" that are available on the system:

yum install -y pinentry-gtk.x86_64

You can also use **Seahorse** – a GNOME application for managing encryption keys:

    yum install -y seahorse

The application can be found in *Applications > Accessories menu*.
