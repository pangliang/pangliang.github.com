---
layout: post
categories : [lnmp]
tags : [nginx, tengine, nginx_tcp_proxy_module, mysql, 负载均衡]
excerpt: this is sample excerpt....
---
{% include JB/setup %}

上个文章用nginx_tcp_proxy_module 来给mysql 做均衡负载 ,
在此模块对mysql 做健康检查时 发送空包 ,被max_connect_errors block 的问题 ,
今天简单弄了下 ,让它发送 一个 user=root ,password 为空 的 mysql login packet .
测试了3000多次检查 , 即使登陆不成功 ,也不会被block掉

至于带密码的login packet ,涉及到跟mysql 的握手步骤 ,还有待研究
不过感觉就健康检查没必要真的login 进去 ,
而且即使mysql有login 失败的block ,专门给nginx 所在IP 开一个 密码为空的用户也是可以接受的方案

下面是针对nginx_tcp_proxy_module-v0.26 的patch

    diff -urN yaoweibin-nginx_tcp_proxy_module-b83e5a6/ngx_tcp_upstream_check.c yaoweibin-nginx_tcp_proxy_module-b83e5a6_new/ngx_tcp_upstream_check.c
    --- yaoweibin-nginx_tcp_proxy_module-b83e5a6/ngx_tcp_upstream_check.c   2011-12-30 10:43:33.000000000 +0800
    +++ yaoweibin-nginx_tcp_proxy_module-b83e5a6_new/ngx_tcp_upstream_check.c       2012-07-20 14:40:00.888077351 +0800
    @@ -77,6 +77,18 @@
            "\x00"                /* Compression Type    : 0x00 = NULL compression   */
     };

    +const char mysql_login_pkt[] = {
    +       "\x3c\x00\x00"          /* Packet Length ,      :       封包长度 */
    +       "\x01"                  /* Packet Number        :       封包序列号 ,login packet 固定为 0x01 */
    +       "\x05\xa6"              /* Client Capabilities  :       不懂啥意思 */
    +       "\x0f\x00"              /* Extended Client Capabilities :               */
    +       "\x00\x00\x00\x01"      /* Max packet           :       =0x01000000 */
    +       "\x21"                  /* Character set        :       默认字符集  */
    +       "\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00" /* Unknown */
    +       "\x72\x6f\x6f\x74\x00"  /* User Name    :       0x726f6f74 = 'root' ,字符串的 acsii 码 ,'\0'结尾, 长度变化时,上面的 Packet Length 要跟着变      */
    +       "\x00"                  /* Password     :       空密码 , */
    +       "\x6d\x79\x73\x71\x6c\x5f\x6e\x61\x74\x69\x76\x65\x5f\x70\x61\x73\x73\x77\x6f\x72\x64\x00"      /* 结尾固定字符串 "mysql_native_password" */
    +};

     #define HANDSHAKE    0x16
     #define SERVER_HELLO 0x02
    @@ -133,7 +145,7 @@
         {
             NGX_TCP_CHECK_MYSQL,
             "mysql",
    -        ngx_null_string,
    +        ngx_string(mysql_login_pkt),
             0,
             ngx_tcp_check_send_handler,
             ngx_tcp_check_recv_handler,
    @@ -1094,6 +1106,9 @@
         ctx->send.start = ctx->send.pos = (u_char *)uscf->send.data;
         ctx->send.end = ctx->send.last = ctx->send.start + uscf->send.len;

    +    ngx_log_debug1(NGX_LOG_DEBUG_TCP, ngx_cycle->log, 0,
    +            "mysql_init: send:%V", &uscf->send);
    +
         ctx->recv.start = ctx->recv.pos = NULL;
         ctx->recv.end = ctx->recv.last = NULL;
