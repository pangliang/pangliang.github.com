---
layout: post
categories : [lnmp]
tags : [nginx, mysql, drizzle, 性能优化, 高效]
excerpt: drizzle可以让nginx直接访问mysql检索出数据作为resp返回给页面
---
{% include JB/setup %}

1.编译安装 libdrizzle-1.0

        http://agentzh.org/misc/nginx/drizzle7-2011.07.21.tar.gz

        tar xzvf drizzle7-2011.07.21.tar.gz
        cd drizzle7-2011.07.21/
        ./configure –without-server
        make libdrizzle-1.0
        make install-libdrizzle-1.0

2.下载drizzle-nginx-module

        wget https://github.com/chaoslawful/drizzle-nginx-module/tarball/v0.1.2rc7 -O chaoslawful-drizzle-nginx-module-v0.1.2rc7-0-g272cabf.tar.bz2

3.下载rds-json-nginx-module

        wget https://github.com/agentzh/rds-json-nginx-module/tarball/v0.12rc10 -O agentzh-rds-json-nginx-module-v0.12rc10-0-g74c21b3.tar.gz

4.编译安装tengine

        ./configure –prefix=/server/tengine –with-file-aio –with-http_lua_module –with-http_ssl_module –with-http_stub_status_module –with-http_sub_module –with-pcre –add-module=../chaoslawful-drizzle-nginx-module-272cabf –add-module=../agentzh-rds-json-nginx-module-74c21b3
        make && sudo make install

5.配置

    >http节点下:

        upstream cluster {
            drizzle_server 127.0.0.1:3306 dbname=mysql password=liangwei user=root protocol=mysql;
        }

    >server节点下:

        location /mysql {
            set $my_sql 'select * from user';
            drizzle_query $my_sql;
            drizzle_module_header off;
            drizzle_pass cluster;
            rds_json on;
        }

6.简单的压力测试:

        liangwei@liangwei-ubuntu:/server/tengine/conf$ webbench -t 20 -c 10000 http://192.168.56.101/mysql
        Webbench – Simple Web Benchmark 1.5
        Copyright (c) Radim Kolar 1997-2004, GPL Open Source Software.

        Benchmarking: GET http://192.168.56.101/mysql
        10000 clients, running 20 sec.

        Speed=82350 pages/min, 7930495 bytes/sec.
        Requests: 26978 susceed, 472 failed.

        liangwei@liangwei-ubuntu:/server/tengine/conf$ webbench -t 20 -c 10000 http://192.168.56.101/index.php
        Webbench – Simple Web Benchmark 1.5
        Copyright (c) Radim Kolar 1997-2004, GPL Open Source Software.

        Benchmarking: GET http://192.168.56.101/index.php
        10000 clients, running 20 sec.

        Speed=48501 pages/min, 4362751 bytes/sec.
        Requests: 16167 susceed, 0 failed.

7.错误解决

    >1)./configure: error: SSL modules require the OpenSSL library.

        apt-get install libssl-dev

    >2)./configure: error: ngx_http_lua_module requires the Lua library.

        apt-get install liblua5.1-0-dev