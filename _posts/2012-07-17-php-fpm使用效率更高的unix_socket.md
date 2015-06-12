---
layout: post
categories : [lnmp]
tags : [什么是, 性能优化]
excerpt: php-fpm使用UDS效率更高
---
{% include JB/setup %}

通过上个文章 , 本来觉得 在php-fpm 与mysql 通信时也需要开新的端口

但是检查发现, 其实php-fpm 并没有通过打开TCP端口和mysql的3306端口进行通信, 查看my.conf, 发现

        socket = /var/run/mysqld/mysqld.sock

这是一个unix socket, 查资料说是unix socket 比tcp 效率更高,

>什么是UNIX SOCKET?
>
>socket API原本是为网络通讯设计的，但后来在socket的框架上发展出一种IPC机制，就是UNIX Domain Socket。虽然网络socket也可用于同一台主机的进程间通讯（通过loopback地址127.0.0.1），但是UNIX Domain Socket用于IPC更有效率：不需要经过网络协议栈，不需要打包拆包、计算校验和、维护序号和应答等，只是将应用层数据从一个进程拷贝到另一个进程。这是因为，IPC机制本质上是可靠的通讯，而网络协议是为不可靠的通讯设计的。UNIX Domain Socket也提供面向流和面向数据包两种API接口，类似于TCP和UDP，但是面向消息的UNIX Domain Socket也是可靠的，消息既不会丢失也不会顺序错乱。



那么何不让 nginx 访问 php-fpm时 也采用这种方式?

修改 php-fpm.conf(pool.d/www.conf)

        listen = 127.0.0.1:9000

改为:

        listen = /dev/shm/php.socket

修改nginx 的php-fpm 转发配置:

        location ~ \.php$ {
                    fastcgi_pass   127.0.0.1:9000;
                    fastcgi_index  index.php;
                    fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
                    include        fastcgi_params;
         }

中的

        fastcgi_pass   127.0.0.1:9000;

改为:

        fastcgi_pass unix:/dev/shm/php.socket;





