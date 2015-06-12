---
layout : post
categories: [android]
tags : []
keywords : 
excerpt: 
---
{% include JB/setup %}

## 普通调试

1. 用ida读入libcocos2dcpp.so
2. 将ida目录下的`dbgsrv/android_server`拷贝到android手机
3. `adb shell`进入, 并以root运行`android_server`
4. 手机运行app, ida使用`Remote ARM Linux/Android debugger`, 并attach对应的进程
5. 下断点调试即可

## app在attach之前不运行

一般先运行app, 这个时候比如lib就会被加载并运行, 想断点某些入口点怎么办?

1. 在adb的shell中使用`am start -D -n 包名/Activity类名`运行app, 这个时候app会显示`waiting debugger to attach`
2. ida慢慢attach;在想要断的地方下好断子 (这个attach不是上面的attach)
3. 使用 `jdb -connect com.sun.jdi.SocketAttach:hostname=127.0.0.1,port=8700` 真正attach, 此时游戏运行

## attach之后只看到一个进程: 

`android_server`没有用root运行, 或者shell并不是root, 比如魅族mx3, 虽然有root权限, 但是shell不是真root

## 魅族MX3获得shell root

mx3的rom默认可以开启`系统权限`, 但是使用shell还是无法切换到root用户; 解决办法是安装`SuperSU`软件

## 放在/sdcard 里不能运行

sdcard 默认会格式化为FAT32, 所以在这个里面的所有文件都没有-x 权限, 解决办法就是放到`/data/local/tmp`目录下

## mac怎么调试
在Paralle 跑win8, win8 跑 ida, ida是通过ip去attach的, 那就得让win8 连到`外界`能`看到`的网络, 虚拟网卡使用`桥接`, 接到比如公司wifi上

如果用真机, 真机也连到wifi上就行了

如果是Genymotion, VitrualBox里的虚拟网卡同样设置成`桥接`, 连到wifi, 那win8和Genymotion 就在同一个网络了

然后在路由器把ip固定住


