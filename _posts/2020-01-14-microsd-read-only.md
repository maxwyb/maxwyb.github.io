---
layout: post
title:  "Micro SD card automatically mounted as read-only on Linux"
date:   2020-01-14 08:30:00 -0700
categories: Linux, disk, partition
---

# A Problem

It was a just normal night after school until I checked my Arch Linux Chromebook used as desktop/home server and found that no writes can be performed to the `/home` partition. Kernel `dmesg` reports a flood of errors as well. 

```
$ dmesg
[    5.038902] mmcblk1: retrying write for general error
[    5.039475] mmcblk1: card_busy_detect: error sending status cmd, status 0x80900
[    5.061400] VFS: Dirty inode writeback failed for block device mmcblk1p1 (err=-5)
# repeat lots of times
```

The home directories are stored on a 32GB SanDisk Class 10 Micro SD card due to the low 16GB eMMC internal storage on this Samsung Chromebook 3. The Micro SD card has a single ext4 partition that has been set to mount on `/home` read+write during system boot. It also contains a 2GB swapfile that gets used occasionally complement to the 4GB RAM. 

```
# File /etc/fstab
# /dev/mmcblk1p1
UUID=(disk_uuid)    /home   ext4    rw,relatime,data=ordered    0 2
```

Somehow, this partition is now mounted as read-only automatically by the OS. Re-mounting the partition as read+write fails too.

```bash
$ sudo mount -v
# /dev/mmcblk1p1 on /home type read-only

$ sudo mount -o remount,rw /dev/mmcblk1p1 /home
# failed because disk is write-protected
```

This tiny machine usually sits there doing nothing besides contributing ADS-B data with a RTL-SDR dongle to a crowd-sourced flight tracking platform. Let's get to the result first: **seems like the Micro SD card is blocking all write requests on the hardware level without any explicit signal - it probably has reached the end of its lifespan.** To anyone who happened to get to this post: if you're sure no system/package update is performed, you have not interacted with the machine for a while and suddenly get this problem, the hardware failure may well likely be the cause.

# Exploring along

## Fix the Filesystem

First things first, I resorted to the "technology panacea" - rebooting the system, only to find the kernel floods similar errors during boot and the system boots into an "emergency mode" in maintenance runlevel that prompts for root password. Naturally, one would think the file system meta-data may have been corrupted. So we took out the SD card, plugged it into another Ubuntu workstation and tried a fairly standard filesystem check with `fsck`. As its name suggests, the check is performed on *a file system* so a virtual/partition device should be given to the tool (something like `/dev/sda1`) but not a raw device (something like `/dev/sda`). Otherwise, `fsck` would always fail with not being able to find a superblock, which reasonably does not exist. The error is similar to those thrown by other tools that expect a filesystem - for example, if we try to mount the raw device.

```
$ sudo fsck /dev/sdb
fsck from util-linux 2.31.1
e2fsck 1.44.1 (24-Mar-2018)
ext2fs_open2: Bad magic number in super-block
fsck.ext2: Superblock invalid, trying backup blocks...
fsck.ext2: Bad magic number in super-block while trying to open /dev/sdb

The superblock could not be read or does not describe a valid ext2/ext3/ext4
filesystem.  If the device is valid and it really contains an ext2/ext3/ext4
filesystem (and not swap or ufs or something else), then the superblock
is corrupt, and you might try running e2fsck with an alternate superblock:
    e2fsck -b 8193 <device>
 or
    e2fsck -b 32768 <device>

Found a gpt partition table in /dev/sdb

$ sudo mount /dev/sdb /mnt/usb_drive/
mount: /mnt/usb_drive: wrong fs type, bad option, bad superblock on /dev/sdb, missing codepage or helper program, or other error.
```

The Linux Kernel documentations give a detailed [disk layout specification](https://www.kernel.org/doc/html/latest/filesystems/ext4/overview.html#blocks) of the ext4 filesystem. Just as a memory refresh, the [Superblock](https://ext4.wiki.kernel.org/index.php/Ext4_Disk_Layout#The_Super_Block) lies in the partition's beginning and contains metadata of the filesystem's metadata (Ex. total & free inode count/block count, block size, error handling strategy, etc.) It contains a magic signature `0xEF53` at byte offset `0x38` so the OS can identify it. It's so crucial for the partition to be mounted that multiple Alternate Superblocks exist as backups at various offsets in the partition besides this Primary Superblock.  

| Group 0 Padding | ext4 Super Block | Group Descriptors | Reserved GDT Blocks | Data Block Bitmap | inode Bitmap | inode Table | Data Blocks      |
| --------------- | ---------------- | ----------------- | ------------------- | ----------------- | ------------ | ----------- | ---------------- |
| 1024 bytes      | 1 block          | many blocks       | many blocks         | 1 block           | 1 block      | many blocks | many more blocks |


To know where these alternate superblocks might be stored, we can dry-run "create" a ext4 filesystem on the target partition and look at its parameters (the `-n` flag specifies not to actually create one but to output the details if it would).

```bash
$ mkfs.ext4 -n /dev/sdb1
mke2fs 1.44.1 (24-Mar-2018)
/dev/sdb1 contains a ext4 file system
	last mounted on /home on Sat Oct 19 13:50:48 2019
Proceed anyway? (y,N) y
Creating filesystem with 7791360 4k blocks and 1949696 inodes
Filesystem UUID: bc18340d-2eb1-4b36-9215-bb334aa5a4f1
Superblock backups stored on blocks:
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
	4096000

$ mke2fs -n /dev/sdb1
# similar results
```

In case the primary superblock is corrupted, `fsck` can be directed to use an alternative superblock found above with `e2fsck -b` or `fsck -b`.

Is there any chance that someone gets really unlucky and all alternate superblocks have been corrupted as well? Very unlikely, but it may have happened to this person who also documented [his/her data recovery process](https://unix.stackexchange.com/questions/33284/recovering-ext4-superblocks) carefully. Kyle Jone's comment offers an interesting heads-up in case you believe to have a similar case:

> I can't make myself believe that you were unlucky enough to have all copies of the superblock wiped. So there must be something wrong with the partition table, which in turn is throwing off the logical block offsets in the filesystem causing fsck to not be able to find the alternate superblocks.

`fdisk` can be used to re-create a partition table - people having reinstalled \*nix OSs may be familiar. Loading the partition into the tool first, deleting the partition and creating it again with the same starting sector should do the job. We haven't tried it because the GPT partition table is reported to be normal, but here is a [discussion thread](https://askubuntu.com/questions/48717/how-to-manually-fix-a-partition-table) about this process. 

```bash
$ sudo fdisk -l /dev/sdb
Disk /dev/sdb: 29.7 GiB, 31914983424 bytes, 62333952 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 847DB5F9-878D-46CF-A272-23481BC9D3AA

Device     Start      End  Sectors  Size Type
/dev/sdb1   2048 62332927 62330880 29.7G Linux filesystem
```

In case all superblocks are truly broken and you're absolutely sure about the partition's block size, we can force to re-create the superblock by `mke2fs -S` without touching block and inode bitmaps. This is only the last resort and further damage can be done to the disk if the block size is gotten wrong. 

## Data Recovery

[The Data Recovery guide](https://help.ubuntu.com/community/DataRecovery) by Ubuntu offers a good starting point. We never want to write to the failed device which may further corrupt the data, so it's best to image the entire device onto another drive first and perform operations there. The GNU `ddrescue` suits the purpose: functionally it works like an extended version of `dd` just for device recovery. It supports re-trying failed reads multiple times, pause-and-resume with a mapfile/logfile, or even recover the entire data using multiple damaged devices containing the same data (because it's very unlikely that these devices are all physically damaged at the same data block).

Then, `mmls` from `sleuthkit` helps discover the partition table on the device image, and then we can mount the target partition by giving the device image and the partition's byte offset. Fortunately, imaging our Micro SD card only took one single pass with 100% successful reads, and mounting the partition is successful with all data still intact.

```bash
$ ddrescue -r3 /dev/sdb1 ~/arch-linux-sd-card ~/ddrescue-log
$ mmls ~/arch-linux-sd-card  # target partition starts at sector 2048, units are in 512-byte sectors
$ sudo mount -o loop,offset=1048576 ~/arch-linux-sd-card /mnt/data_recovery  # 2048 * 512 = 1048576
```

## Discovering the Problem

The good news gave us confidence and we proceeded to test mounting the physical Micro SD card directly as read+write, which succeeded as well. Then we unmounted the ext4 partition, and a `fsck` on it found inconsistencies from a non-empty journal but failed to fix the problem: 

```bash
$ sudo fsck /dev/sdb1
fsck from util-linux 2.31.1
e2fsck 1.44.1 (24-Mar-2018)
/dev/sdb1: recovering journal
Superblock needs_recovery flag is clear, but journal has data.
Run journal anyway<y>? no
Clear journal<y>? yes
fsck.ext4: unable to set superblock flags on /dev/sdb1


/dev/sdb1: ********** WARNING: Filesystem still has errors **********
```

It looks like we can write new files or add changes to existing files in the drive without error when we mounted the drive earlier. But now all file changes were gone when the partition was mounted again. This interesting find inspired us of a potential quick dive by a raw disk read & write:

```bash
dd if=/dev/zero of=/dev/sdb1 bs=1k count=1  # write 1024 zero to the partition's beginning ("seek" or "skip" parameters can be used)
sync

dd if=/dev/sdb1 of=/tmp/sd-card-read bs=1k count=1  # read out the partition's data, are they all 0s?
```

We haven't tried this as we still want to preserve the partition's metadata on the SD card, but this test is highly likely to fail potentientially accompanied with errors in the kernel ring. There's no physical write-protect switch on the Micro SD card itself, so it appears that the card has entered a fail-safe read-only mode on the physical level as discussed above. Plenty of people are having the same problem online so we'll stop here with this explanation for now, while preserving this SD card for future scrutiny when in the mood again.

# Lessons Learned

1. It's surprising to know that the lifespan of Micro SD cards may not be as long as we think before. The total system up time of this Chromebook is about a full year or so with the Micro SD card in it. Also, using a flash device as swap may contribute its quick worn out. Well, it's not too surprising as the high quality flashes are usually left for the SSDs.

2. Backup regularly, as people always talk about. My friend and I were just talking about a tech review article days ago, claiming it's practically impossible to wear out a particular well-known brand's flash storage device. But you never know when it fails in what unexpected way.

## References

[fsck.ext4: unable to set superblock flags, even after mounting ok](https://bugzilla.kernel.org/show_bug.cgi?id=196677)

[What is a Superblock, Inode, Dentry and a File?](https://unix.stackexchange.com/questions/4402/what-is-a-superblock-inode-dentry-and-a-file)

[Linux: Recover Corrupted Partition From A Bad Superblock](https://www.cyberciti.biz/faq/recover-bad-superblock-from-corrupted-partition/): `dumpe2fs`

Instances of `fsck` setting superblock flag failure: [1](https://unix.stackexchange.com/questions/327863/fsck-wont-fsck-unable-to-set-superblock-flags), [2](https://www.linuxquestions.org/questions/linux-hardware-18/a-fix-e2fsck-unable-to-set-superblock-flags-on-2tbinternal-4175656091/), [3](https://bugzilla.kernel.org/show_bug.cgi?id=196677)
