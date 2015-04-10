---
layout: post
categories : ios
tags : [usbmuxd, lockdown, 与iPhone连接, libimobiledevice]
excerpt: 如何获取iphone已安装的app?如何获取iphone文件列表?iTunes使用一种叫"usbmux"的东西与iphone通信, 这个东西提供了一个USB - TCP的转换服务.usbmuxd可以实现一切...
---
{% include JB/setup %}

最开始研究与iphone通信, 都会想当然的google下usb协议, 必经iphone是通过usb线连接到电脑. 其实不然, iTunes是通过TCP协议与iPhone通信的

##usbmuxd

iTunes使用一种叫"usbmux"的东西与iphone通信, 这个东西提供了一个USB - TCP的转换服务.   
这个服务在Mac端是由`/System/Library/PrivateFrameworks/MobileDevice.framework/Resources/usbmuxd` 提供的, 当然, 开机自动启动.    
它创建了一个Unix Domain Socket 在 `/var/run/usbmuxd`. usbmuxd服务程序监控iPhone在USB口上的连接, 当它监控到iPhone以`用户模式`连接到USB,
(相对的是`recovery`模式), usbmuxd服务程序就会连接到这个`/var/run/usbmuxd`的TCP端口, 并开始成为一个USB - TCP `请求`转发器   

那么,如果想编写个第三方程序与iphone进行通信,实现类似iTunes的功能, 你的程序可以通过usbmuxd! 建立一个TCP连接到`/var/run/usbmuxd`端口,
根据协议发送对应的请求包, usbmuxd服务会将请求转发到USB的iPhone上

##lockdownd协议

    //协议头
    struct usbmux_header {
    	u32 length;	// 消息长度,包括头部
    	u32 version;	// 协议版本号
    	u32 type;       // 消息类型,请求,响应,握手,等
    	u32 tag;	// 消息编号, 用来对应响应
    	char payload;  //请求体
    };
    
    //头部中的type类型
    enum {
    	usbmux_result  = 1,
    	usbmux_connect = 2,
    	usbmux_hello   = 3,
    	usbmux_payload = 8,
    };

##监听

知道了iTunes使用的协议, 那么有没有办法看看iTunes都发了些什么包? 有个简单的办法就是使用`socat`, 类似:

    sudo mv /var/run/usbmuxd /var/run/usbmuxx
    sudo socat -t100 -x -v UNIX-LISTEN:/var/run/usbmuxd,mode=777,reuseaddr,fork UNIX-CONNECT:/var/run/usbmuxx


##包示例:

    > 2013/02/04 00:07:19.567563  length=483 from=0 to=482
     e3 01 00 00 01 00 00 00 08 00 00 00 02 00 00 00  ................
     3c 3f 78 6d 6c 20 76 65 72 73 69 6f 6e 3d 22 31  <?xml version="1
     2e 30 22 20 65 6e 63 6f 64 69 6e 67 3d 22 55 54  .0" encoding="UT
     46 2d 38 22 3f 3e 0a                             F-8"?>.
     3c 21 44 4f 43 54 59 50 45 20 70 6c 69 73 74 20  <!DOCTYPE plist
     50 55 42 4c 49 43 20 22 2d 2f 2f 41 70 70 6c 65  PUBLIC "-//Apple
     2f 2f 44 54 44 20 50 4c 49 53 54 20 31 2e 30 2f  //DTD PLIST 1.0/
     2f 45 4e 22 20 22 68 74 74 70 3a 2f 2f 77 77 77  /EN" "http://www
     2e 61 70 70 6c 65 2e 63 6f 6d 2f 44 54 44 73 2f  .apple.com/DTDs/
     50 72 6f 70 65 72 74 79 4c 69 73 74 2d 31 2e 30  PropertyList-1.0
     2e 64 74 64 22 3e 0a                             .dtd">.
     3c 70 6c 69 73 74 20 76 65 72 73 69 6f 6e 3d 22  <plist version="
     31 2e 30 22 3e 0a                                1.0">.
     3c 64 69 63 74 3e 0a                             <dict>.
     09 3c 6b 65 79 3e 42 75 6e 64 6c 65 49 44 3c 2f  .<key>BundleID</
     6b 65 79 3e 0a                                   key>.
     09 3c 73 74 72 69 6e 67 3e 63 6f 6d 2e 61 70 70  .<string>com.app
     6c 65 2e 69 54 75 6e 65 73 48 65 6c 70 65 72 3c  le.iTunesHelper<
     2f 73 74 72 69 6e 67 3e 0a                       /string>.
     09 3c 6b 65 79 3e 43 6c 69 65 6e 74 56 65 72 73  .<key>ClientVers
     69 6f 6e 53 74 72 69 6e 67 3c 2f 6b 65 79 3e 0a  ionString</key>.
     09 3c 73 74 72 69 6e 67 3e 75 73 62 6d 75 78 64  .<string>usbmuxd
     2d 32 39 36 2e 33 3c 2f 73 74 72 69 6e 67 3e 0a  -296.3</string>.
     09 3c 6b 65 79 3e 4d 65 73 73 61 67 65 54 79 70  .<key>MessageTyp
     65 3c 2f 6b 65 79 3e 0a                          e</key>.
     09 3c 73 74 72 69 6e 67 3e 4c 69 73 74 65 6e 3c  .<string>Listen<
     2f 73 74 72 69 6e 67 3e 0a                       /string>.
     09 3c 6b 65 79 3e 50 72 6f 67 4e 61 6d 65 3c 2f  .<key>ProgName</
     6b 65 79 3e 0a                                   key>.
     09 3c 73 74 72 69 6e 67 3e 69 54 75 6e 65 73 48  .<string>iTunesH
     65 6c 70 65 72 3c 2f 73 74 72 69 6e 67 3e 0a     elper</string>.
     09 3c 6b 65 79 3e 6b 4c 69 62 55 53 42 4d 75 78  .<key>kLibUSBMux
     56 65 72 73 69 6f 6e 3c 2f 6b 65 79 3e 0a        Version</key>.
     09 3c 69 6e 74 65 67 65 72 3e 33 3c 2f 69 6e 74  .<integer>3</int
     65 67 65 72 3e 0a                                eger>.
     3c 2f 64 69 63 74 3e 0a                          </dict>.
     3c 2f 70 6c 69 73 74 3e 0a                       </plist>.
    --
    < 2013/02/04 00:07:19.570319  length=294 from=0 to=293
     26 01 00 00 01 00 00 00 08 00 00 00 02 00 00 00  &...............
     3c 3f 78 6d 6c 20 76 65 72 73 69 6f 6e 3d 22 31  <?xml version="1
     2e 30 22 20 65 6e 63 6f 64 69 6e 67 3d 22 55 54  .0" encoding="UT
     46 2d 38 22 3f 3e 0a                             F-8"?>.
     3c 21 44 4f 43 54 59 50 45 20 70 6c 69 73 74 20  <!DOCTYPE plist
     50 55 42 4c 49 43 20 22 2d 2f 2f 41 70 70 6c 65  PUBLIC "-//Apple
     2f 2f 44 54 44 20 50 4c 49 53 54 20 31 2e 30 2f  //DTD PLIST 1.0/
     2f 45 4e 22 20 22 68 74 74 70 3a 2f 2f 77 77 77  /EN" "http://www
     2e 61 70 70 6c 65 2e 63 6f 6d 2f 44 54 44 73 2f  .apple.com/DTDs/
     50 72 6f 70 65 72 74 79 4c 69 73 74 2d 31 2e 30  PropertyList-1.0
     2e 64 74 64 22 3e 0a                             .dtd">.
     3c 70 6c 69 73 74 20 76 65 72 73 69 6f 6e 3d 22  <plist version="
     31 2e 30 22 3e 0a                                1.0">.
     3c 64 69 63 74 3e 0a                             <dict>.
     09 3c 6b 65 79 3e 4d 65 73 73 61 67 65 54 79 70  .<key>MessageTyp
     65 3c 2f 6b 65 79 3e 0a                          e</key>.
     09 3c 73 74 72 69 6e 67 3e 52 65 73 75 6c 74 3c  .<string>Result<
     2f 73 74 72 69 6e 67 3e 0a                       /string>.
     09 3c 6b 65 79 3e 4e 75 6d 62 65 72 3c 2f 6b 65  .<key>Number</ke
     79 3e 0a                                         y>.
     09 3c 69 6e 74 65 67 65 72 3e 30 3c 2f 69 6e 74  .<integer>0</int
     65 67 65 72 3e 0a                                eger>.
     3c 2f 64 69 63 74 3e 0a                          </dict>.
     3c 2f 70 6c 69 73 74 3e 0a                       </plist>.
    --

第一个iTunes向`/var/run/usbmuxd`发的请求包, 第一行 `e3 01 00 00 01 00 00 00 08 00 00 00 02 00 00 00`是头部
`e3 01 00 00` 即0x01e3 = 483, 包长度   

头部后面紧跟的是payload, 也就是xml格式的请求内容
