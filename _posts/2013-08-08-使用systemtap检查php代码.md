---
layout: post
categories : [lnmp]
tags : [systemtap]
excerpt: 检查php页响应时间,以前只能自己埋点. 自从有了systemstap...
---
{% include JB/setup %}

需要带debug的php
====

* 根据上一篇文章安装好systemtap, 并补装DTrace support:

        sudo apt-get install systemtap-sdt-dev

* 看下ubuntu 当前的php是哪个版本, 方便装php扩展, 我的是5.4.9, 到时候down源码也down这个版本的源码;

        $ sudo apt-get install php5-cli
        $ php -v

        PHP 5.4.9-4ubuntu2.2 (cli) (built: Jul 15 2013 18:23:35)
        Copyright (c) 1997-2012 The PHP Group
        Zend Engine v2.4.0, Copyright (c) 1998-2012 Zend Technologies

* 下载php源码并编译之, configure的php.ini直接使用ubuntu自带的:

        $ git clone git://github.com/php/php-src php-src
        $ cd php-src
        $ git checkout PHP-5.4

        //或者 到  https://github.com/php/php-src/tree/PHP-5.4 下载 zip包

        $ ./buildconf --force
        $ ./configure --disable-all --enable-dtrace --with-config-file-path=/etc/php5/cli --with-config-file-scan-dir=/etc/php5/cli/conf.d/
        $ make

*  尝试一下并看下php-cli都有哪些函数

        $stap -l 'process.provider("*").mark("*")' -c '/media/win_f/php-src/sapi/cli/php -i'

         process("/media/win_f/php-src/sapi/cli/php").provider("php").mark("compile__file__entry")
         process("/media/win_f/php-src/sapi/cli/php").provider("php").mark("compile__file__return")
         process("/media/win_f/php-src/sapi/cli/php").provider("php").mark("error")
         process("/media/win_f/php-src/sapi/cli/php").provider("php").mark("exception__caught")
         process("/media/win_f/php-src/sapi/cli/php").provider("php").mark("exception__thrown")
         process("/media/win_f/php-src/sapi/cli/php").provider("php").mark("execute__entry")
         process("/media/win_f/php-src/sapi/cli/php").provider("php").mark("execute__return")
         process("/media/win_f/php-src/sapi/cli/php").provider("php").mark("function__entry")
         process("/media/win_f/php-src/sapi/cli/php").provider("php").mark("function__return")
         process("/media/win_f/php-src/sapi/cli/php").provider("php").mark("request__shutdown")
         process("/media/win_f/php-src/sapi/cli/php").provider("php").mark("request__startup")



