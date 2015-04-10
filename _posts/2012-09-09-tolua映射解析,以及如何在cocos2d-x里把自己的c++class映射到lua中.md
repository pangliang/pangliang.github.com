---
layout: post
categories : [cocos2dx]
tags : [cocos2dx, ios, httpclient, lua]
excerpt: tolua解析, 实现自定义c++ 类库. 并映射到cocos2d-x的lua中使用
---
{% include JB/setup %}

不管引擎多强大,总会自己拓展一些类. 或者使用一些有源码包. 用c++实现然后再映射到lua里.

1.映射

>这个代码还是比较好懂,整个函数就是把类和所继承的类和对应的lua做一个映射.
然后做函数的映射;也就是把MyHttpClient的doGet函数映射到tolua_liangwei_MyHttpClient_doGet00函数里去,
由 tolua_liangwei_MyHttpClient_doGet00 去做 调用时参数的获取和转换,
然后再由tolua_liangwei_MyHttpClient_doGet00真正的调用MyHttpClient的doGet函数;

    {% highlight cpp linenos %}
        TOLUA_API int tolua_liangwei_extension_open (lua_State* tolua_S)
        {
            tolua_open(tolua_S);

            //注册类型名
            tolua_usertype(tolua_S,"MyHttpClient");

            tolua_module(tolua_S,NULL,0);
            tolua_beginmodule(tolua_S,NULL);

            //注册函数和类
            tolua_cclass(tolua_S, "MyHttpClient", "MyHttpClient", "CCObject", NULL);
            tolua_beginmodule(tolua_S,"MyHttpClient");
            tolua_function(tolua_S,"doGet",tolua_liangwei_MyHttpClient_doGet00);
            tolua_endmodule(tolua_S);

            tolua_endmodule(tolua_S);
            return 1;
        }
    {% endhighlight %}

2.然后来看tolua_liangwei_MyHttpClient_doGet00

    {% highlight cpp linenos %}
        static int tolua_liangwei_MyHttpClient_doGet00(lua_State* tolua_S)
        {
            tolua_Error tolua_err;
            if (
                    //函数类型检测, 如果不满足就goto tolua_lerror报错
                    !tolua_isusertable(tolua_S,1,"MyHttpClient",0,&tolua_err) ||
                            !tolua_isstring(tolua_S,2,0,&tolua_err) ||
                            !tolua_isfunction(tolua_S,3,&tolua_err) ||
                            !tolua_isnoobj(tolua_S,4,&tolua_err)
                    )
                goto tolua_lerror;
            else
            {
                //满足, 取出第一个参数. tolua_tostring, 顾名思义 ,lua类型转换成c++的string(char *);
                const char* url = ((const char*)  tolua_tostring(tolua_S,2,0));
                //取出第二个参数, lua的function是一个int型的"描述符"
                int funcID = (tolua_ref_function(tolua_S,3,0));
                {
                    //真正调用方法;
                    MyHttpClient::doGet(url, funcID);
                }
            }
            return 1;
            tolua_lerror:
                    tolua_error(tolua_S,"#ferror in function 'node'.",&tolua_err);
            return 0;

        }
    {% endhighlight %}

3.那么tolua_liangwei_extension_open 是什么时候”被”调用?一般在register lua engine 完成之后调用; 也就是在AppDelegate::applicationDidFinishLaunching() 中:

    {% highlight cpp linenos %}
        // register lua engine
        CCScriptEngineProtocol* pEngine = CCLuaEngine::engine();
        CCScriptEngineManager::sharedManager()->setScriptEngine(pEngine);

        //注册自己的扩展
        lua_State* L = pEngine->getLuaState();
        tolua_liangwei_extension_open(L);
    {% endhighlight %}

4.MyHttpClient 的原型:

    {% highlight cpp linenos %}
        class MyHttpClient : public cocos2d::CCObject
        {
        public:
            static void doGet(const char* url,int handler);
            virtual void onHttpRequestCompleted(cocos2d::CCNode *sender, void *data);
            virtual void executeFunction(int responseCode, const char* resp);

        private:
            int m_nHandler;
        };
    {% endhighlight %}

onHttpRequestCompleted和executeFunction 都不需要暴露给lua, 所以就不需要映射;
