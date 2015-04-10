---
layout: post
categories : [android]
tags : [boot.img, android, rom]
excerpt: rom 中 boot.img 文件格式
---
{% include JB/setup %}

* 4 * 2, magic，固定为”ANDROID!”
* 4 * 1, kernel长度，小端unsigned类型
* 4 * 1, kernel地址，应为base + 0×00008000
* 4 * 1, ramdisk长度，小端unsigned
* 4 * 1, ramdisk地址，应为base + 0×01000000
* 4 * 1, second stage长度，小端unsigned，为0
* 4 * 1, second stage地址，应为base + 0x00f00000
* 4 * 1, tags地址，应为base + 0×00000100
* 4 * 1, page大小，小端unsigned, 为2048或者4096
* 4 * 2, 未使用，固定为0×00
* 4 * 4, 板子名字，一般为空
* 4 * 128, 内核命令参数，一大串
* 4 * 8, id,不知道啥玩意，0×00