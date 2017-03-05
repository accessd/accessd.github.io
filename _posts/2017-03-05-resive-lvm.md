---
layout: post
title: "How to resize a VirtualBox Virtual Machine hard disk with LVM"
description: "One of my production virtual machine had a lack of space and increase the disk space wasn't a simple task"
category: blog
blog: true
tags: ["virtualbox", "lvm", "disk space"]
lang: en
---

One of my production virtual machine had a lack of space and I found out that increase the disk space wasn't a simple task.
I wrote this post for future me first of all.

Here is a steps you should do:

## Backup your .vdi image

Or entire VM. Seriously.

## Resize vdi

Run command:

    vboxmanage modifyhd --resize 200000 ./my_vm.vdi

200000 - it's a new size of your disk in megabytes, so it's a 200Gb

## Resize the partition

Download [Gparted](http://gparted.org/download.php) .iso,
then in your Virtualbox VM settings add a optical drive that point to Gparted.iso, start the VM.
Then in Gparted resize first the extended partition to take all the available space, and then same for the LVM partition.
Confirm changes, and reboot.

If partitions will be locked, then press right click on the LVM partition and choose *Deactivate*

![ScreenShot](/assets/blog/gparted.png)

## Resize the LVM

Eject Gparted image and load VM normally.
You’ll need to make sure you have both the **lvm2** package and **resize2fs** tools installed.

1. Resize the PV (Physical Volume):

```bash
root@server:/root# pvs
  PV         VG          Fmt  Attr PSize   PFree
  /dev/sda5  server-vg lvm2 a--  283.20g    0

root@server:/root# pvresize /dev/sda5
  Physical volume "/dev/sda5" changed
  1 physical volume(s) resized / 0 physical volume(s) not resized
```

2. Now let’s extend the LV (Logical Volume) to the full size of the PV:

```bash
root@server:/root# lvdisplay
  --- Logical volume ---
  LV Path                /dev/server-vg/root
  LV Name                root
  VG Name                server-vg
  LV UUID                mDvKSl-bb9f-1u3T-iAnf-hNQL-juRH-gMs84V
  LV Write Access        read/write
  LV Creation host, time server, 2016-03-03 17:49:12 +0300
  LV Status              available
  # open                 1
  LV Size                281.20 GiB
  Current LE             71987
  Segments               2
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           252:0

  --- Logical volume ---
  LV Path                /dev/server-vg/swap_1
  LV Name                swap_1
  VG Name                server-vg
  LV UUID                aAZ99k-Rgm2-mfs4-pzFl-WbrD-Av9u-8rLmVM
  LV Write Access        read/write
  LV Creation host, time server, 2016-03-03 17:49:12 +0300
  LV Status              available
  # open                 2
  LV Size                2.00 GiB
  Current LE             512
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           252:1
```


3. And then extend it to full available size:

```bash
root@server:/root# lvextend -l+100%FREE /dev/server-vg/root
  Extending logical volume root to 281.20 GiB
  Logical volume root successfully resized
```

4. The last step is to extend the filesystem on the whole LV:

```bash
root@server:# resize2fs /dev/server-vg/root
```

That’s it!
