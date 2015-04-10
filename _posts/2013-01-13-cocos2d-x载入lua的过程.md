---
layout: post
categories : [cocos2dx]
tags : [cocos2dx, ios, lua]
excerpt: lua脚本读入过程解析, 实现lua脚本的简单加密解密
---
{% include JB/setup %}

入口点在AppDelegate.cpp中:

    //类CCLuaEngine 实现了 cocos2d-x的脚本引擎接口 
    CCScriptEngineProtocol* pEngine = CCLuaEngine::engine();
    //manager中设置脚本引擎
    CCScriptEngineManager::sharedManager()->setScriptEngine(pEngine);
    //通过相对路径获取绝对路径
    string path = CCFileUtils::sharedFileUtils()->fullPathFromRelativePath("script/Main.lua");
    //把文件所在目录添加至lua文件搜索路径
    pEngine->addSearchPath(path.substr(0, path.find_last_of("/")).c_str());
    //运行指定的文件
    pEngine->executeScriptFile(path.c_str());
    
cocos2d-x是可以支持多种脚本引擎的, `CCScriptEngineProtocol`是脚本引擎接口定义    
其LUA版本的实现在`libs/lua/cocos2dx_support/CCLuaEngine.h`,即由`CCLuaEngine`类实现   

追入`executeScriptFile`:

    int CCLuaEngine::executeScriptFile(const char* filename)
    {
        //  读入文件
        int nRet = luaL_dofile(m_state, filename);
        //    lua_gc(m_state, LUA_GCCOLLECT, 0);
    
        if (nRet != 0)
        {
            CCLOG("[LUA ERROR] %s", lua_tostring(m_state, -1));
            lua_pop(m_state, 1);
            return nRet;
        }
        return 0;
    }

追入`luaL_dofile`, 在文件`libs/lua/lua/lauxlib.h`:

    #define luaL_dofile(L, fn) \
        (luaL_loadfile(L, fn) || lua_pcall(L, 0, LUA_MULTRET, 0))
        
`luaL_loadfile`实现在`libs/lua/lua/lauxlib.c`:

    LUALIB_API int luaL_loadfile (lua_State *L, const char *filename) {
      ...
      status = lua_load(L, getF, &lf, lua_tostring(L, -1));
      ...
    }
    
`lua_load`就是lua读入脚本文件的方法.   
其具体实现没太看明白, 简单的说就是通过`getF`方法作为reader(文件读入器)将文件内容读到内存, 然后进行一系列解析

    static const char *getF (lua_State *L, void *ud, size_t *size) {
      LoadF *lf = (LoadF *)ud;
      (void)L;
      if (lf->extraline) {
        lf->extraline = 0;
        *size = 1;
        return "\n";
      }
      if (feof(lf->f)) return NULL;
      memset(lf->buff, 0, sizeof(lf->buff));
      
      //将文件读入LoadF结构体的buff内存, 并将实际读到的长度赋予size, 
      *size = fread(lf->buff, 1, sizeof(lf->buff), lf->f);
      return (*size > 0) ? lf->buff : NULL;
    }
    
`LoadF`结构体:

    typedef struct LoadF {
      int extraline;
      FILE *f;
      unsigned char buff[LUAL_BUFFERSIZE];
    } LoadF;
    
所以, 如果文件是经过加密的文件, 在fread之后把buff的数据进行解密即可, 就像这样:

    static const char *getF (lua_State *L, void *ud, size_t *size) {
      ...
      //将文件读入LoadF结构体的buff内存, 并将实际读到的长度赋予size, 
      *size = fread(lf->buff, 1, sizeof(lf->buff), lf->f);
      //将buff内容解密
      *size = aes256_decrypt(key, lf->buff,*size);
      return (*size > 0) ? lf->buff : NULL;
    }

需要注意的是

1. 文件的大小大于LUAL_BUFFERSIZE时, 会分多次读入
2. LUAL_BUFFERSIZE的大小必须是加密块的整倍数
3. 解密之后必须将size赋值为strlen. 例如块大小为32字节, 加密3B的文件内容`123`后文件大小还是为32B, 解密后buff内容为 0x30,0x31,0x32,0x00....0x00, 会出错

