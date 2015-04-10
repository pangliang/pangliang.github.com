---
layout : post
categories: [cocos2dx]
tags : [cocos2dx, 游戏开发]
keywords : cocos2dx, 游戏开发
excerpt: 
---
{% include JB/setup %}

##问题
`Cocos Code IDE`在lua开发debug的时候, 如果修改了文件会自动reload, 这个功能能方便;

但是如果main引用的文件A又引用了文件B, 当B修改的时候虽然触发reload, 但是B却没有生效

##排查
追了下runtime源码, 是`Cocos Code IDE`在保存文件`xxxx.lua`的时候会给app发送个socket命令: `reload xxxx.lua`, 并且同时`reload main.lua` ; 也就是说 引用 `xxxx.lua`的文件A并没有被reload

##临时解决办法
1. 引用的时候统一使用 `require("src/xxxx")` 格式

2. 在main.lua 的main函数加入如下代码, 把`src`目录下的文件重新reload一遍

```lua

function main()

    for filename,v in pairs(package.loaded) do
        
        if string.find(filename,"src/") == 1 and string.find(filename,"src/main") ~= 1 then
            --卸载旧文件
            package.loaded[filename] = nil
            require(filename)
        end
    end

    -- source

end
```