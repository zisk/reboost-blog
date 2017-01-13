Title: Manually Removing a Disk from a Synology RAID
Date: 2017-01-07 17:00
Tags: RAID, Synology, mdadm

In rearranging my storage configuration in my homelab, I ran into a situation where I wanted to redo the layout of a fully provisioned Synology NAS. In this case, all disks were used in a single RAID 6 array, giving a two disk fault tolerance. I had already cleared off the majority of the data, with just enough left over to fit on a single disk from the array. I figured the easiest solution would be to remove a disk from the array, move the data there, then rebuild with a different configuration. Luckily, Synology uses [mdadm](https://linux.die.net/man/8/mdadm) under the hood in basic RAID types (their [SHR](https://www.synology.com/en-us/knowledgebase/DSM/tutorial/Storage/What_is_Synology_Hybrid_RAID_SHR) may be different, I haven't looked into it yet). Here's how to remove a disk.


**The following is 100% not supported in anyway by Synology. It involves intentionally compromising the integrity of your RAID array, greatly putting your data at risk. All data should be backed up before attempting any of the following. No seriously, you'll probably lose your data.**

First, [ensure ssh enable](https://www.synology.com/en-us/knowledgebase/DSM/help/DSM/AdminCenter/system_terminal) and connect as a user with administrative privileges. The following commands will need to be run as root, so utilize sudo before each.


```bash
# mdadm --detail --scan
```
This will result in a list of arrays similiar to the following:

```
ARRAY /dev/md2 metadata=1.2 name=<nashostname>:2 UUID=d5c56c64:eebd794f:1ac841a5:89891531
```
Make sure you pick the correct array from the disk and run the `detail` command again against the specific array to get extended details including what disks are included.

```
# mdadm --detail /dev/md3
/dev/md3:
        Version : 1.2
  Creation Time : Thu Jan 12 09:30:52 2017
     Raid Level : raid6


--clip--


    Number   Major   Minor   RaidDevice State
       0       8        3        0      active sync   /dev/sda3
       1       8       19        1      active sync   /dev/sdb3
       2       8       35        2      active sync   /dev/sdc3
       3       8       51        3      active sync   /dev/sdd3

```

Before a disk can be removed from an array it must be marked as failed so choose a disk and fail it out

```bash
# mdadm --fail /dev/md3 /dev/sdd3
```
At this point your NAS should start beeping like crazy to tell you that you have a failed drive. For the sake of your sanity, you probably want to go [clear the beep message](https://twitter.com/synology/status/449218680144920579?lang=en). Now, the drive can be removed from the array.

```bash
# mdadm --remove /dev/md3 /dev/sdd3
```

If you log back into DSM you should have a disk available for use. In my case I formatted it as a basic disk, copied the shares over, then converted to a RAID 1 then to RAID 5 as disks became available from the rest of the rebuild process.
