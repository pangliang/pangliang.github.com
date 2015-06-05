---
layout : post
categories: [其他]
tags : [github, githubpage, blog, baidu, 爬虫]
keywords : 
excerpt: 
---
{% include JB/setup %}


虽然咱的博客也就是简单的做个笔记, 但是还是奇怪于, 访问统计中百度的来源少的可怜, 基本没有; 一直也没管; 因为我就是做笔记, 给自己看的;

今天群里聊天, 才知道原来github 把百度爬虫给屏蔽了.

好吧, 那怎么办?

###使用国内的gitcafe 做个镜像

- 注册个gitcafe帐号,最好跟github用户名一致
- gitcafe创建一个跟用户名一样的项目
- 修改原来github page的git配置, 添加多一个remote, 名字叫`gitcafe`, 注意修改为自己的用户名和项目名

```
#原来的master只关联github remote
[branch "master"]
        remote = github
        merge = refs/heads/master
        
#添加gitcafe remote
[remote "gitcafe"]
        url = https://gitcafe.com/pangliang/pangliang.git
        fetch = +refs/heads/*:refs/remotes/gitcafe/*
```

- 把本地的`master`分支push到gitcafe的`gitcafe-pages`分支

```bash
git push gitcafe master:gitcafe-pages
```

- 修改gitcafe上的cname
- 把域名添加多几个A记录, 按`海外`区分, `海外`的就走github好了, 国内走gitcafe, 然后单独再加个给baidu的索引, 就像这样:

![]({{ site.image_dir }}/2015/20150605053529.png)

### 效果

在国内访问, 就自动走了gitcafe的ip, 就像这样:

![]({{ site.image_dir }}/2015/20150605054151.png)

### 百度也能抓去成功了

![]({{ site.image_dir }}/2015/20150605055448.png)

### 怎么检查github page 是否对?

人在国内, 怎么检查, 两个办法, 把dns指向8.8.8.8, 或者添加hosts强制指向`192.30.252.153`

![]({{ site.image_dir }}/2015/20150605054917.png)
