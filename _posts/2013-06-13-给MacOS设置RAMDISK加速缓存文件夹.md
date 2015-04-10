---
layout: post
categories : [mac]
tags : [ramdisk, caches]
excerpt: 8G内存的电脑不要浪费了, 把各种缓存文件放到内存中吧...
---
{% include JB/setup %}

OS X 系统主要存放应用程序缓存的目录，也就是 ~/Library/Caches  

删除 ~/Library/Caches 这个目录

    sudo rm -rf ~/Library/Caches
    
把ramdisk链接到cache目录:
    
    ln -s /Volumes/RamDisk/ ~/Library/Caches
    
编写创建ramdisk脚本,保存为`/sbin/create_ramdisk.sh`:

    #!/bin/bash
    
    if ! test -e /Volumes/Ramdisk ; then
        diskutil erasevolume HFS+ RamDisk `hdiutil attach -nomount ram://2097152`
    fi
    
设置开机加载创建脚本:

    sudo vi /System/Library/LaunchDaemons/ramdisk.plist
    
    
    <?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
    <plist version="1.0">
    <dict>
        <key>Disabled</key>
        <false/>
        <key>Label</key>
        <string>com.liangwei.tools</string>
        <key>ProgramArguments</key>
        <string>/sbin/create_ramdisk.sh</string>
        <key>RunAtLoad</key>
        <true/>
    </dict>
    </plist>
    

