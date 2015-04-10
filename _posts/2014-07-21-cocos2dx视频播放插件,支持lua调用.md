---
layout : post
categories: [cocos2dx]
tags : [cocos2dx, 游戏开发, 视频播放, 插件]
excerpt: 项目中需要播放视频, 编写了个视频播放插件, 并绑定到lua中供调用
---
{% include JB/setup %}

###git:
	
	https://github.com/liang8305/cocos2dx_videoview_extends

###添加步骤:

1. VideoView.h VideoView.cpp 放到c++ 的Classes 目录

2. 修改jni 的 Android.mk  添加VideoView.cpp 编译

		LOCAL_SRC_FILES := ... \
			   ../../Classes/VideoView.cpp \
	           ...

3. AppDelegate.cpp   添加lua 扩展

		#include "VideoView.h"

		bool AppDelegate::applicationDidFinishLaunching()
		{
		    ...

		    auto engine = LuaEngine::getInstance();
		    ScriptEngineManager::getInstance()->setScriptEngine(engine);

		    //添加lua 扩展
		    lua_State* L = engine->getLuaStack()->getLuaState();
		    tolua_videoview_extension_open(L);

		    if (engine->executeScriptFile("src/main.lua")) {
		        return false;
		    }

		    return true;
		}

4. cn.sharedream.game.VideoView.java  放到android java src 目录


5. 在lua中使用

		-- 播放完之后的回调函数
		local function videoFinish()
	        print("================videoFinish")
	        --恢复游戏, 和原来的声音
	        cc.Director:getInstance():startAnimation();
	        cc.SimpleAudioEngine:getInstance():pauseMusic();
	    end

	    -- 原游戏, 声音暂停
	    cc.Director:getInstance():stopAnimation();
		cc.SimpleAudioEngine:getInstance():pauseMusic();
		VideoView:play("res/video2.mp4",videoFinish)