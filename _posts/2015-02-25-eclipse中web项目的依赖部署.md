---
layout : post
categories: [Java]
tags : [tomcat, 游戏开发]
excerpt: 
---
{% include JB/setup %}



WEB工程有引用其它JAVA工程中的类,在TOMCAT运行时,找不到依赖工程中的JAVA类.
比如工程A引用工程B, 一般情况下是把工程B编译成jar包然后放到`WEB-INF/lib`目录下
但是开发的时候一方面不能热部署, 另一个查看源码时也无法跳转

解决办法是:

1. 解决编译依赖: 项目`Properties`的`Java Build Path`中加入引用的`Project`
2. 解决部署依赖: 项目`Properties`的`Deloyment Assmbly`中加入引用的`Project`