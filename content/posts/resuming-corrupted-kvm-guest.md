+++
categories = ["Quick Tips"]
date = "2012-07-12T07:49:00+01:00"
description = "Restoring a KVM guest from corrupted resume state"
tags = ["kvm", "virsh"]
title = "Restoring a KVM guest from corrupted resume state"
+++

#### Problem:

The virtual machine doesn’t boot throwing the following error:

```
# virsh start vm1
error: failed to get domain 'vm1'
error: Unable to read from monitor: Connection reset by peer
```

#### Solution:

    # virsh managedsave-remove vm1
