---
layout : post
categories: [其他]
tags : [RDS, ssh tunnel,内网]
keywords : 
excerpt: 
---
{% include JB/setup %}

项目最初觉得没有必要生产环境开发环境这么搞, 直接用rds做开发库了; 阿里云的数据库后台登录又必须先登录阿里云帐号; 不适合把帐号给多个人

google了下, 网上有如下几种方案

- 用tengine做反向代理
- 用mysql-proxy做代理
- 用iptable做封包转发
- 利用ssh tunnel

其实前面三种, 跟直接把mysql开出公网来没啥区别, 只不过稍微灵活一点, 用的时候开开, 不用关掉;
但是万一`忘记`关了呢? 而且`控制权`还是在服务器管理者手里;

一般都给开发发放ssh权限, 利用ssh tunnel还是方便, 够`临时`, 连不连又都是开发自己选择;

使用命令:

```bash
ssh -N -L 本地端口号:rds实例地址:rds实例端口号 ecs用户名@ecsip
```

ssh tunnel的意思就是说, 建立一个本地端口, 把这个本地端口的数据包通过ssh服务发到`ecsip`这个服务器, 再通过服务器上的ssh服务转发给指定的`rds实例地址:rds实例端口号`, 那也就打开了一个`本地->ECS服务器->RDS`的通道了

这个东西的优势在于, 我只需要保证ssh服务的健壮, 就可以了; 

同样的, 这个方式还适合其他的不便于`让公网可直接访问`的内网服务, 比如redis什么的

##注意

ssh服务必须打开`GatewayPorts`选项, 具体办法是修改`/etc/ssh/sshd_config`配置文件, 添加一行

```
GatewayPorts yes
```





