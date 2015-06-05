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
- 修改原来github page的git配置, 添加多一个remote, 注意修改为自己的用户名和项目名

```
[remote "gitcafe"]
url = https://gitcafe.com/pangliang/pangliang.git
fetch = +refs/heads/*:refs/remotes/gitcafe/*
pushurl = https://gitcafe.com/pangliang/pangliang.git
```

- 把本地的`master`分支push到gitcafe的`gitcafe-pages`分支

```bash
git push -v gitcafe refs/heads/master:refs/heads/gitcafe-pages
```

- 如果原来有`CNAME`的话, 在gitcafe上删掉它
- 把`_config.yml`中的`production_url`修改为gitcafe的地址:

> production_url : http://pangliang.gitcafe.io