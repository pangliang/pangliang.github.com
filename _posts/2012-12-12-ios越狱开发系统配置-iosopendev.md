---
layout: post
categories : [ios]
tags : [iosopendev, jailbreak, ios, iphone, ipad, 越狱开发]
excerpt: 介绍jb开发的系统配置方法
---
{% include JB/setup %}

##mac端环境

    Manually download the zip and extract into the following into the respective folders:

        /opt/iOSOpenDev/                https://github.com/kokoabim/iOSOpenDev
        /opt/iOSOpenDev/templates/      https://github.com/kokoabim/iOSOpenDev-Xcode-Templates
        /opt/iOSOpenDev/frameworks/     https://github.com/kokoabim/iOSOpenDev-Framework-Header-Files

    Comment out the following lines in your /opt/iOSOpenDev/iod-setup file: (lines 530-532)

        downloadGithubTarball “https://nodeload.github.com/kokoabim/iOSOpenDev/tarball/master” “$iOSOpenDevPath” “iOSOpenDev base”
        downloadGithubTarball “https://nodeload.github.com/kokoabim/iOSOpenDev-Xcode-Templates/tarball/master” “$iOSOpenDevPath/templates” “Xcode templates”
        downloadGithubTarball “https://nodeload.github.com/kokoabim/iOSOpenDev-Framework-Header-Files/tarball/master” “$iOSOpenDevPath/frameworks” “framework header files”

    from terminal run:

        sudo /opt/iOSOpenDev-Setup/iod-setup base
        sudo /opt/iOSOpenDev-Setup/iod-setup sdk -sdk iphoneos

##手机端环境,cydia安装openssh:

* openSSH
* toggle ssh
* APT 0.6 Transitional

##mac端安装ssh证书

    iosod sshkey -h 手机当前ip
        
##手机端,使用ssh登录手机使用`apt-get`安装其他依赖

	apt-get install coreutils diskdev-cmds file-cmds \
		system-cmds com.saurik.substrate.safemode \
		mobilesubstrate preferenceloader



##获得手机syslog

    apt-get install socat
    socat – UNIX-CONNECT:/var/run/lockdown/syslog.sock
    >watch
        
##注意

如果更新了xcode或者ios版本, 或者编译打包时报异常:"but there's no such product type for the 'iphoneos' platform" 需要更新指定一下sdk :

    sudo /opt/iOSOpenDevSetup/bin/iod-setup sdk -sdk iphoneos
    

    
