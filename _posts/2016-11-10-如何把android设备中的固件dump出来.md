---
layout : post
categories: [android]
tags : [固件, rom, dumper]
keywords :
excerpt:
---


android固件是在mtdblock中, 但是会有很多个block, 

```bash
root@android: # cat /proc/partitions

major minor  #blocks  name
  31        0       4096 mtdblock0
  31        1      16384 mtdblock1
  31        2      16384 mtdblock2
  31        3      16384 mtdblock3
  31        4     393216 mtdblock4
  31        5     131072 mtdblock5
  31        6    2097152 mtdblock6
  31        7       4096 mtdblock7
  31        8     524288 mtdblock8
  31        9    4509696 mtdblock9
 179        0    3941376 mmcblk0
 179        1    3941344 mmcblk0p1
 ```


各个block分别对应哪个分区? 

```bash
root@android: # cat /proc/mtd

dev:    size   erasesize  name
mtd0: 00400000 00004000 "misc"
mtd1: 01000000 00004000 "kernel"
mtd2: 01000000 00004000 "boot"
mtd3: 01000000 00004000 "recovery"
mtd4: 18000000 00004000 "backup"
mtd5: 08000000 00004000 "cache"
mtd6: 80000000 00004000 "userdata"
mtd7: 00400000 00004000 "kpanic"
mtd8: 20000000 00004000 "system"
mtd9: 113400000 00004000 "user"
```

这样我们就知道了, mtd1 是kernel 所在, 所以我们想dump kernel.img 就用dd即可:

```bash
root@android: # dd if=/dev/block/mtdblock1 of=/mnt/external_sd/dd.out/kernel.img
```

