---
layout: post
categories : [lnmp]
tags : [nginx, tengine, nginx_tcp_proxy_module, mysql, 负载均衡]
excerpt: this is sample excerpt....
---
{% include JB/setup %}

LVS 太重了, 自己在php里rand() slave又没办法健康检查, 自动上下线挂掉的mysql
整个轻量的mysql负载均衡方案

在42.121.16.232 和 192.168.56.101 分别都有一个mysql

在192.168.56.101 上放 nginx ,也就是 和mysql 在同一台机器

nginx 占用了mysql 原来的3306端口 ,则本机的mysql 挪到端口 3307

1. 下载 module

        wget https://github.com/yaoweibin/nginx_tcp_proxy_module/tarball/v0.26 -O yaoweibin-nginx_tcp_proxy_module-v0.26-0-g9677e00.tar.gz
        tar -xvf yaoweibin-nginx_tcp_proxy_module-v0.26-0-g9677e00.tar.gz

2. 给tengine打补丁

>将一下内容存为tcp.patch

    diff -Nur tengine-1.3.0_bak/src/core/ngx_log.c tengine-1.3.0/src/core/ngx_log.c
    --- tengine-1.3.0_bak/src/core/ngx_log.c 2012-07-19 14:52:55.929140710 +0800
    +++ tengine-1.3.0/src/core/ngx_log.c 2012-07-19 14:53:32.900674183 +0800
    @@ -67,7 +67,7 @@</code>

    static const char *debug_levels[] = {
    "debug_core", "debug_alloc", "debug_mutex", "debug_event",
    - "debug_http", "debug_mail", "debug_mysql"
    + "debug_http", "debug_mail", "debug_mysql","debug_tcp"
    };

    diff -Nur tengine-1.3.0_bak/src/core/ngx_log.h tengine-1.3.0/src/core/ngx_log.h
    --- tengine-1.3.0_bak/src/core/ngx_log.h 2012-07-19 14:52:55.929140710 +0800
    +++ tengine-1.3.0/src/core/ngx_log.h 2012-07-19 14:55:46.099001142 +0800
    @@ -30,6 +30,7 @@
    #define NGX_LOG_DEBUG_HTTP 0x100
    #define NGX_LOG_DEBUG_MAIL 0x200
    #define NGX_LOG_DEBUG_MYSQL 0x400
    +#define NGX_LOG_DEBUG_TCP 0x800

    /*
    * do not forget to update debug_levels[] in src/core/ngx_log.c
    @@ -37,7 +38,7 @@
    */

    #define NGX_LOG_DEBUG_FIRST NGX_LOG_DEBUG_CORE
    -#define NGX_LOG_DEBUG_LAST NGX_LOG_DEBUG_MYSQL
    +#define NGX_LOG_DEBUG_LAST NGX_LOG_DEBUG_TCP
    #define NGX_LOG_DEBUG_CONNECTION 0x80000000
    #define NGX_LOG_DEBUG_ALL 0x7ffffff0

    diff -Nur tengine-1.3.0_bak/src/event/ngx_event_connect.h tengine-1.3.0/src/event/ngx_event_connect.h
    --- tengine-1.3.0_bak/src/event/ngx_event_connect.h 2012-07-19 14:52:56.093138648 +0800
    +++ tengine-1.3.0/src/event/ngx_event_connect.h 2012-07-19 14:57:56.645359897 +0800
    @@ -33,6 +33,7 @@
    void *data);
    #endif

    +#define NGX_INVALED_INDEX (-1)

    struct ngx_peer_connection_s {
    ngx_connection_t *connection;
    @@ -43,6 +44,8 @@

    ngx_uint_t tries;

    + ngx_uint_t check_index;
    +
    ngx_event_get_peer_pt get;
    ngx_event_free_peer_pt free;
    void *data;

>打补丁

    cd tengine-1.3.0
    patch -p1

3.编译 tengine

        ./configure --prefix=/server/tengine --with-file-aio --with-http_lua_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-pcre --add-module=../chaoslawful-drizzle-nginx-module-272cabf --add-module=../agentzh-rds-json-nginx-module-74c21b3 --add-module=../yaoweibin-nginx_tcp_proxy_module-b83e5a6
        make &amp;&amp; sudo make install

4.配置

>顶节点:

    tcp {

        upstream mysql_cluster {
            # simple round-robin
            server 42.121.16.232:3306;
            server 192.168.56.101:3307;
            check interval=3000 rise=2 fall=5 timeout=10000 type=mysql;
        }

        server {
            listen 3306;
            proxy_pass mysql_cluster;
        }
    }

问题:
------
因为mysql中默认的max_connect_errors是10，
发空包到mysql的端口, mysql认为连接出错，
超过10次,当在访问的时候就被锁住了, 并屏蔽主机的进一步连接请求。

另:
有人在github上给作者提过这个问题, 也没解决

猜测作者测试时用127.0.0.1 来监控  ,
但mysql不会把”本机”给屏蔽掉
many connection errors 的错误就不会重现

可以用telnet 192.168.XX.XX 3306  和telnet 127.0.0.1 3306 两种方式来测
127的怎么telnet都不会有问题  , 192的多试几次就会:

    liangwei@liangwei-ubuntu:~$ telnet 192.168.56.101 3307
    Trying 192.168.56.101...
    Connected to 192.168.56.101.
    Escape character is '^]'.
    Host '192.168.56.101' is blocked because of many connection errors; unblock with 'mysqladmin flush-hosts'Connection closed by foreign host.

解决方案设想:

发送 mysql login packet: