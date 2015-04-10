---
layout: post
categories : [android]
tags : [源码阅读]
excerpt:
---
{% include JB/setup %}

##NativeActivity 机制

刚使用cocos2dx V3的最大感受就是没有 Cocos2dxActivity 这个java了...为啥? 就是cocos2dx V3 是真正的Native了;

{% highlight java linenos %}
public class testgame extends Cocos2dxActivity {
        protected void onCreate(Bundle savedInstanceState) {
                super.onCreate(savedInstanceState);
        }

        public Cocos2dxGLSurfaceView onCreateView() {
                return new LuaGLSurfaceView(this);
        }

        static {
                System.loadLibrary("cocos2dlua");
        }
}
{% endhighlight %}

相对的, 是android原来编写应用的方式: JavaActivity;   
而NativeActivity是Android SDK提供的编写Native应用的辅助类, 它帮助管理Android 框架和应用之间的事件通信;   
查看 $NDK_ROOT/platforms/android-14/arch-x86/usr/include/android/native_activity.h,这个是NativeActivity的机制说明

<!-- more -->

要编写一个NativeActivity, 你可以使用最原始的方法:

1.  实现NativeActivity创建和注册方法:

        typedef void ANativeActivity_createFunc(ANativeActivity* activity,
                void* savedState, size_t savedStateSize);

        //默认的ANativeActivity_createFunc
        extern ANativeActivity_createFunc ANativeActivity_onCreate;

1.  设置callback

{% highlight cpp linenos %}
typedef struct ANativeActivity {
    struct ANativeActivityCallbacks* callbacks;

    ......

} ANativeActivity;

typedef struct ANativeActivityCallbacks {
    /**
     * 当NativeActivity 启动时回调.  就像原来 JAVAActivity.onStart()一样
     */
    void (*onStart)(ANativeActivity* activity);

    //类似的还有
    void (*onResume)(ANativeActivity* activity);
    void* (*onSaveInstanceState)(ANativeActivity* activity, size_t* outSize);
    void (*onPause)(ANativeActivity* activity);
    void (*onStop)(ANativeActivity* activity);
    void (*onDestroy)(ANativeActivity* activity);

    ......

} ANativeActivityCallbacks;
{% endhighlight %}

一个简单的 NativeActivity就像这样:

{% highlight cpp linenos %}
static void onStart(ANativeActivity* activity) {
    printf("hello world");
}

static void onDestroy(ANativeActivity* activity) {
    printf("bye...");
}

//实现入口点函数, 就像main一样
void ANativeActivity_onCreate(ANativeActivity* activity,void* savedState, size_t savedStateSize) {
    //实现设置回调函数
    activity->callbacks->onDestroy = onDestroy;
    activity->callbacks->onStart = onStart;
}

{% endhighlight %}

就什么简单: ANativeActivity_onCreate就相当于main函数, 然后作为app它有生命周期和,你就需要监听一下生命周期事件, 完事!

##android_native_app_glue
编写Native应用有两种方式,   

    1.  自己实现native_activity.h中的回调函数; 
    2.  利用android_native_app_glue 

android_native_app_glue它将input事件单独起了一个线程,将实现使用pipe管道进行流水处理, 避免事件处理造成的ui hang住   
cocos2dx就是使用第二种   

$NDK_ROOT/sources/android/native_app_glue/android_native_app_glue.c

{% highlight cpp linenos %}
//实现入口点回调函数
void ANativeActivity_onCreate(ANativeActivity* activity,
        void* savedState, size_t savedStateSize) {
    LOGV("Creating: %p\n", activity);

    //设置activity回调函数
    activity->callbacks->onDestroy = onDestroy;
    activity->callbacks->onStart = onStart;
    ......
    activity->callbacks->onInputQueueCreated = onInputQueueCreated;
    activity->callbacks->onInputQueueDestroyed = onInputQueueDestroyed;

    //生成android_app
    activity->instance = android_app_create(activity, savedState, savedStateSize);
}

static struct android_app* android_app_create(ANativeActivity* activity,
        void* savedState, size_t savedStateSize) {

    ......

    //声明事件管道
    android_app->msgread = msgpipe[0];
    android_app->msgwrite = msgpipe[1];

    //在新线程中让app_entry
    pthread_attr_t attr;
    pthread_attr_init(&attr);
    pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);
    pthread_create(&android_app->thread, &attr, android_app_entry, android_app);

    ......
}

static void* android_app_entry(void* param) {
    struct android_app* android_app = (struct android_app*)param;

    android_app->config = AConfiguration_new();
    AConfiguration_fromAssetManager(android_app->config, android_app->activity->assetManager);

    print_cur_config(android_app);

    //设置消息"循环器"来源
    android_app->cmdPollSource.id = LOOPER_ID_MAIN;
    android_app->cmdPollSource.app = android_app;
    //设置消息辅助类的事件处理回调
    android_app->cmdPollSource.process = process_cmd;

    /**
     *  这里使用了ALooper辅助类, 实现源码暂时找不到, 推测就是会监听刚才声明的`事件管道`的读取端
     *  当有消息来的时候就调用cmdPollSource.process指定的回调函数process_cmd
     */
    ALooper* looper = ALooper_prepare(ALOOPER_PREPARE_ALLOW_NON_CALLBACKS);
    ALooper_addFd(looper, android_app->msgread, LOOPER_ID_MAIN, ALOOPER_EVENT_INPUT, NULL,
            &android_app->cmdPollSource);
    android_app->looper = looper;

    //给个标志给主线程, 让主线程知道我们已经初始化完毕, 可以继续下一步了
    pthread_mutex_lock(&android_app->mutex);
    android_app->running = 1;
    pthread_cond_broadcast(&android_app->cond);
    pthread_mutex_unlock(&android_app->mutex);

    //调用app实现的android_main
    android_main(android_app);

    android_app_destroy(android_app);
    return NULL;
}

static void process_cmd(struct android_app* app, struct android_poll_source* source) {
    int8_t cmd = android_app_read_cmd(app);
    android_app_pre_exec_cmd(app, cmd);
    //调用app实现的onAppCmd
    if (app->onAppCmd != NULL) app->onAppCmd(app, cmd);
    android_app_post_exec_cmd(app, cmd);
}
{% endhighlight %}

##为什么

按照原来Java Activity的方式, JavaActivity里去load一下jni的.so库,然后JavaActivity.onCreate()里再调用jni里的onCreate()之类的东西;   
然后JavaActivity.onTouchXXX 什么的时间都要通过jni转一下, 编写很麻烦, 对于jni这种中转依赖的很重;   
而使用NativeActivity, 就完全是Native的; 所有onXXX()都是直接C++里实现就好;   








