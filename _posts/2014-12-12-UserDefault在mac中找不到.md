---
layout : post
categories: [cocos2dx]
tags : [cocos2dx, 游戏开发]
keywords : cocos2dx, 游戏开发, UserDefault
excerpt: 
---
{% include JB/setup %}



现在用cocos2d-x v3 版本, 用CCUserDefault 存储一些值, 在mac电脑使用PrebuildRuntimeLua.app 进行测试的时候发现怎么都找不到 UserDefault.xml 这个文件

google和查询代码之后发现, v3版本 ios 和mac 版本的UserDefault 使用了 apple 原生的 NSUserDefault, 那么存储文件就变成了:

	~/Library/Preferences/org.cocos2dx.PrebuiltRuntimeLua.plist
	
好了, 删掉这个文件, 重启app 发现, 还是能取到删除文件之前存进去的值, 奇葩哦!

再google, 原来mac还会有一个`UserDefault`服务, 临时存着这些`Preferences` 配置, 有点类似cache层一样; 当有需要读取这些配置就会每次都去读文件, 服务直接发回值; 所以清除他们就是删除文件之后还要执行命令:

	defaults read org.cocos2dx.PrebuiltRuntimeLua
	
实际上这是让`UserDefault`服务读取`org.cocos2dx.PrebuiltRuntimeLua`, 但是文件已经删掉了, 那么自然就清空了
	
如果是自己编译了app, 那么文件名和`key`就是你的`Bundle Identifier`, 例如是`com.example.game.helloworld`, 那么, 文件是:

	~/Library/Preferences/com.example.game.helloworld.plist
	
重新读取的domain 是:

	defaults read com.example.game.helloworld

