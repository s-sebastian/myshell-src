+++
categories = ["Linux"]
date = "2012-08-26T23:52:35+00:00"
description = ""
tags = ["lvm", "resize2fs"]
type = "post"
title = "How-to extend a root LVM partition online"

+++

This guide will explain you how to extend a root LVM partition online.

There is also a quick remedy for the emergency situation when your root partition runs out of disk space. There is a feature specific to ext3 and ext4 that can help the goal of resolving the full disk situation. Unless explicitly changed during filesystem creation, both by default reserve five percent (5%) of a volume capacity to the superuser (root).

```
# df -Th
Filesystem    Type    Size  Used Avail Use% Mounted on
/dev/mapper/vg_main-lv_root
              ext4    8.4G  8.0G  952K 100% /
tmpfs        tmpfs    499M     0  499M   0% /dev/shm
/dev/vda1     ext4    485M   33M  428M   8% /boot

# dumpe2fs /dev/vg_main/lv_root | grep 'Reserved block count'
dumpe2fs 1.41.12 (17-May-2010)
Reserved block count:     111513
```

It turned out 111513 of 4KB blocks were reserved for the superuser, which was exactly five percent of the volume capacity.

**How to enable it?**

```
# tune2fs -m 0 /dev/vg_main/lv_root 
tune2fs 1.41.12 (17-May-2010)
Setting reserved blocks percentage to 0% (0 blocks)
```

\-

```
# df -Th
Filesystem    Type    Size  Used Avail Use% Mounted on
/dev/mapper/vg_main-lv_root
              ext4    8.4G  8.0G  437M  95% /
tmpfs        tmpfs    499M     0  499M   0% /dev/shm
/dev/vda1     ext4    485M   33M  428M   8% /boot
```

Now that we have some free space on the root partition to work on we can extend the LVM partition:

Create a new partition of appropriate size using fdisk

    fdisk /dev/sdb1

This is a key sequence on the keyboard to create a new LVM type (8e) partition:

n, p, 1, enter (accept default first sector), enter (accept default last sector), t, 8e, w

Create a new Physical Volume

```
# pvcreate /dev/sdb1
  Writing physical volume data to disk "/dev/sdb1"
  Physical volume "/dev/sdb1" successfully created
```

Extend a Volume Group

```
# vgextend vg_main /dev/sdb1
  Volume group "vg_main" successfully extended
```

Extend your LVM

\- extend the size of your LVM by the amount of free space on PV

```
# lvextend /dev/vg_main/lv_root /dev/sdb1
  Extending logical volume lv_root to 18.50 GiB
  Logical volume lv_root successfully resized
```

\- or with a given size


    lvextend -L +10G /dev/vg_main/lv_root


Finally resize the file system online

```
# resize2fs /dev/vg_main/lv_root
resize2fs 1.41.12 (17-May-2010)
Filesystem at /dev/vg_main/lv_root is mounted on /; on-line resizing required
old desc_blocks = 1, new_desc_blocks = 2
Performing an on-line resize of /dev/vg_main/lv_root to 4850688 (4k) blocks.
The filesystem on /dev/vg_main/lv_root is now 4850688 blocks long.
```

Now we can set the reserved blocks back to the default percentage - 5%

    tune2fs -m 5 /dev/mapper/vg_main-lv_root

Results:

```
# df -Th
Filesystem    Type    Size  Used Avail Use% Mounted on
/dev/mapper/vg_main-lv_root
              ext4     19G  8.0G  9.4G  46% /
tmpfs        tmpfs    499M     0  499M   0% /dev/shm
/dev/vda1     ext4    485M   33M  428M   8% /boot
```
