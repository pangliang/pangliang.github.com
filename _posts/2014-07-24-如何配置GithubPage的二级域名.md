---
layout : post
categories: [其他]
tags : [github, 博客, blog, github page]
excerpt: 介绍如何配置github page的二级域名, 如何将不同二级域名指向不同的project
---
{% include JB/setup %}

<img width="650px" src="{{ site.image_dir }}/2014/20140724000000.png" alt="...">

#Github Page种类
1. `UserPage`:    用户的整个站点, 这个是最出github支持的类型, 创建一个形如username.github.com的项目就可以

2. `ProjectPage`:   用户创建出来的项目也可以创建站点, 创建一个项目后, 在建立一个名叫`gh-pages`的branch, 这个branch里的文件就是page的站点文件

#UserPage默认域名

用户站点的默认域名是`username.github.io`, 比如笔者的站点就是`liang8305.github.io`

#ProjectPage默认域名

项目的默认域名, 是使用UserPage域名加上二级目录实现的, 比如笔者有个项目叫`cydia`, 那么该项目的站点就是访问 `liang8305.github.io/cydia`

#UserPage自定义域名

我有自己的域名, 如何绑定到UserPage? 比如用`www.zhaoxiaodan.com`替代`liang8305.github.io`他是使用CNAME技术来实现的

具体步骤:

1. 去域名注册商那里, 做一个`CNAME指向`, 将`www.zhaoxiaodan.com` 指向 `liang8305.github.io`, 

2. 在`liang8305/liang8305.github.com`这个项目(也就是page项目)根目录下建一个`CNAME`文件, 里面填写`www.zhaoxiaodan.com`, 然后提交到仓库; 

3. 等10分钟

>CNAME指向之后, 当浏览器访问`www.zhaoxiaodan.com`的时候浏览器就知道`实际上`是访问`liang8305.github.io`  
>添加CNAME 文件之后, 当GithubPage服务器接收到访问`www.zhaoxiaodan.com`的http请求, 就知道, 对应的是这个工程了

#ProjectPage自定义域名

比如用`cydia.zhaoxiaodan.com`替代`liang8305.github.io/cydia`

1. 同样的, 去域名注册商那里, 做一个`CNAME指向`, 将`cydia.zhaoxiaodan.com` 指向 `liang8305.github.io`, 如果以后会有很多二级域名都指过来, 其实可以做一个模糊二级指过来, 比如`*.zhaoxiaodan.com`

2. 在`liang8305/cydia`这个项目(也就是page项目)根目录下建一个`CNAME`文件, 里面填写`cydia.zhaoxiaodan.com`, 然后提交到仓库; 

3. 等10分钟