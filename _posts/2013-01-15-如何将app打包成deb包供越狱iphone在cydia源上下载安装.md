---
layout: post
categories : [ios]
tags : [cydia, 打包, 越狱]
excerpt: 通过编译好的app打包deb的方法
---
{% include JB/setup %}

##修改编译选项重新打包

在工程的`Build Settings -> Code Signing -> Code Signing Identity` 选项, 将 Debug 和 Release 下的 `Any iOS SDK` 都设置为 `Don't Code Sign`  
然后在重新`Archive`

##准备目录

创建一个目录用来打包,如tmp,tmp下建DEBIAN和Applications两个目录, DEBIAN下建一个文本文件control   
tmp目录结构如下:

    -DEBIAN
    ---control
    -Applications

control文件就是打包时的配置文件,它也会作为deb包的配置被打包到包中,   
文件例子:

    Package: com.sharedream.game
    Name: 游戏测试
    Version: 0.1-1
    Description: 游戏测试游戏,开发中...
    Section: 游戏
    Depends: firmware (>= 4.3)
    Priority: optional
    Architecture: iphoneos-arm
    Author: liangwei <http://weibo.com/iamliangwei>
    Homepage: http://weibo.com/iamliangwei
    Icon: file:///Applications/game.app/Icon.png
    Maintainer: liangwei <http://weibo.com/iamliangwei>

然后将xcode打包出来的.app文件整个拷贝到Applications目录下,   
结构如下:

    -DEBIAN
    ---control
    -Applications
    ---game.app

##打包

退出至tmp的上层目录

    dpkg-deb -b tmp game.deb

看到如下几行就是打包完成了.

    warning, `com.sull.sample/DEBIAN/control' contains user-defined field `Name'
    warning, `com.sull.sample/DEBIAN/control' contains user-defined field `Author'
    warning, `com.sull.sample/DEBIAN/control' contains user-defined field `Sponsor'
    dpkg-deb: ignoring 3 warnings about the control file(s)
    
拷贝到cydia源中, 重新扫描包生成Packages列表文件, 并压缩成Packages.bz2就可以啦

    dpkg-scanpackages -m debs >Packages
    bzip2 -zkf Packages

"contains ununderstood data member data.tar.xz" 的安装错误

是因为自从1.17.0版本的dpkg-deb开始, 默认使用xz格式来压缩data.tar文件
但是,cydia在ios提供的dpkg是1.14版本, 还没有支持xz这种压缩格式
所以我们需要设置"-Zgzip"参数给dpkg-deb 进行打包, 类似命令:

    dpkg-deb -Zgzip -b tmp game.deb