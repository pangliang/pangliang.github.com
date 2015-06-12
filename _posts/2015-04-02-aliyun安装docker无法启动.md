---
layout : post
categories: [其他]
tags : []
keywords : aliyun, 阿里云, 无法启动
excerpt: 
---
{% include JB/setup %}

很简单, 因为阿里云服务器默认占用了172.16.x.x 网段, 删掉这个路由表即可

	sudo route del -net 172.16.0.0 netmask 255.240.0.0

