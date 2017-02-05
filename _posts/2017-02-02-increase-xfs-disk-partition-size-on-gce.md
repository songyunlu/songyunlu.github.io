---
layout: post
title:  Increase XFS Disk Partition Size on GCE
date:   2017-02-02T16:00:00+8:00
---

It's not uncommon to face **"out of disk space"** issue in a data engineer's daily life, espcically when you are handling instances with default storage size (which is small) on the public cloud. fourtunately, we can always increase the size of a detached disk as you want. However, we still need a little bit extra work to make the chnage visible to our system.

1. First of all, create a snapshot of the disk we are dealing with to buy ourself an insurance in case we screw up.

2. And then, make sure to uncheck the the option `Delete boot disk when instance is deleted` to keep our disk preserved even if our delete the instance. 
![]({{site.baseurl}}/images/uncheck-delete-boot-disk-when-instance-is-deleted.png)

3. Delete your instance so we can attach the detached disck to another instance for manipulations.

4. Increase the disk size.
![]({{site.baseurl}}/images/increase-disk-size.png)

5. Create a new instance and attached the disk to it.
![]({{site.baseurl}}/images/attach-the-fully-filled-disk-to-an-instance.png)

6. Login to the instance and execute following commands to make the change visible.
  
  ```bash
  $ lsblk # list all block devices

  The disk sdb has 1T disk space but only has one partition with 100G, which means you can expand the size of the partition 
  # NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
  # sda      8:0    0   10G  0 disk
  # `-sda1   8:1    0   10G  0 part /
  # sdb      8:16   0    1T  0 disk
  # `-sdb1   8:17   0  100G  0 part
  
  $ sudo fdisk /dev/sdb

  follow the instructions to modify the partitions
  # The device presents a logical sector size that is smaller than
  # the physical sector size. Aligning to a physical sector (or optimal
  # I/O) size boundary is recommended, or performance may be impacted.
  # Welcome to fdisk (util-linux 2.23.2).
  # 
  # Changes will remain in memory only, until you decide to write them.
  # Be careful before using the write command.
  # 
  print the partitions 
  # Command (m for help): p
  # 
  # Disk /dev/sdb: 1099.5 GB, 1099511627776 bytes, 2147483648 sectors
  # Units = sectors of 1 * 512 = 512 bytes
  # Sector size (logical/physical): 512 bytes / 4096 bytes
  # I/O size (minimum/optimal): 4096 bytes / 4096 bytes
  # Disk label type: dos
  # Disk identifier: 0x00091e3d
  # 
  #    Device Boot      Start         End      Blocks   Id  System
  # /dev/sdb1   *        2048   209712509   104855231   83  Linux
  # 
  delete the partition
  # Command (m for help): d
  # Selected partition 1
  # Partition 1 is deleted
  # 
  create a new partition
  # Command (m for help): n
  # Partition type:
  #    p   primary (0 primary, 0 extended, 4 free)
  #    e   extended
  # Select (default p): p
  # Partition number (1-4, default 1): 1
  # First sector (2048-2147483647, default 2048):
  # Using default value 2048
  # Last sector, +sectors or +size{K,M,G} (2048-2147483647, default 2147483647):
  # Using default value 2147483647
  # Partition 1 of type Linux and of size 1024 GiB is set
  # 
  write the partition table to the disk
  # Command (m for help): w
  # The partition table has been altered!
  # 
  # Calling ioctl() to re-read partition table.
  # Syncing disks.
  
  $ mkdir /tmp/sdb1 # make a mount point
  $ sudo mount /dev/sdb1 /tmp/sdb1 # mount the device
  $ sudo xfs_growfs /dev/sdb1 # expand the partition size
  
  # meta-data=/dev/sdb1              isize=256    agcount=41, agsize=655296 blks
  #          =                       sectsz=4096  attr=2, projid32bit=1
  #          =                       crc=0        finobt=0 spinodes=0
  # data     =                       bsize=4096   blocks=26213807, imaxpct=25
  #          =                       sunit=0      swidth=0 blks
  # naming   =version 2              bsize=4096   ascii-ci=0 ftype=0
  # log      =internal               bsize=4096   blocks=2560, version=2
  #          =                       sectsz=4096  sunit=1 blks, lazy-count=1
  # realtime =none                   extsz=4096   blocks=0, rtextents=0
  # data blocks changed from 26213807 to 268435200
  ```

That's it! We successfully increased the disk size. We can now create a new instance with this disk with all our data intact.
