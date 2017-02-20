+++
categories = ["Networking"]
date = "2013-10-24T21:47:00+01:00"
description = "Cisco ASA – Cheat Sheet"
tags = ["asa", "cisco"]
type = "post"
title = "Cisco ASA – Cheat Sheet"

+++

Command prompts:

'>' - basic mode  
'#' - ‘exec’ mode  
'(config)#' - configuration mode  
'(config-if)#' - interface configuration mode

\- enable ‘exec’ mode (Turn on privileged commands):

    > enable

\- enter global configuration mode:

    # configure terminal

\- enter interface configuration mode:

    (config)# interface eth 0/0

\- exit mode:

    # exit

or

    # end

\- enumerate content stored on flash:

    # show flash

\- specify OS image to boot from:

    (config)# boot system flash:/filename

\- specify ASDM image to boot from:

    (config)# asdm image flash:/filename

\- show current boot variables:

    (config)# show bootvar

\- system details/info:

    # show version

\- save running configuration to flash:

    # copy running-config startup-config

\- restore to default configuration (factory defaults):

    # write erase

\- show running configuration:

    # show run

\- start interactive configuration wizard:

    (config)# setup

\- enable traffic between two or more interfaces which are configured with same security levels:

    (config)# same-security-traffic permit inter-interface

\- enable traffic between two or more hosts connected to the same interface

    (config)# same-security-traffic permit intra-interface

\- enable jumbo frames on specific interface:

    (config-if)# jumbo-frame reservation

\- configure VLAN:

```
(config)# interface eth 0/1
(config-if)# no shutdown
(config-if)# no nameif
(config-if)# interface eth0/1.10
(config-subif)# vlan 10
(config-subif)# nameif eth0/1.10
(config-subif)# security-level 50
(config-subif)# ip addr 192.168.10.1 255.255.255.0
```

\- file system management:

```
# dir - list files
# more - list the content of a file
# copy - copy files
# delete - delete files
```
