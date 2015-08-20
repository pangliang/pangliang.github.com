---
layout : post
categories: [android]
tags : []
keywords : 
excerpt: 
---
{% include JB/setup %}

[dexposed](https://github.com/alibaba/dexposed) 这个项目相当不错, 之前就想着怎么动态替换jvm中的代码, 一直没有思路; 现在好好学习一下


## 准备源码库

因为`dexposed`其实是用了dvm和art调用class的方式来做的, 而dvm和art的头文件什么的在android源码中, 所以下一份源码, 具体办法见上一个博文: [准备android源码库](http://www.zhaoxiaodan.com/android/%E5%87%86%E5%A4%87android%E6%BA%90%E7%A0%81%E5%BA%93.html)

## 最简单的hook

Demo1中有个test函数, 在调用hook之前正常返回”11111”; 
调用hook之后, 却返回”newTestMethod”, 被我们给`修改`了

```java
public class Demo1
{
	String TAG = "===[hookdemo]===";

	public static String staticTest(String param1)
	{
		return "staticTest";
	}

	public String test(String param1)
	{
		return "11111";
	}

	public void demo()
	{
		String param1 = "param1";
		Log.d(TAG, "===========before hook test:" + this.test(param1));
		hook(Demo1.class, "test", "(Ljava/lang/String;)Ljava/lang/String;");
		Log.d(TAG, "===========after hook test:" + this.test(param1));

		Log.d(TAG, "===========before hook staticTest:" + this.staticTest(param1));
		hook(Demo1.class, "staticTest", "(Ljava/lang/String;)Ljava/lang/String;");
		Log.d(TAG, "===========after hook staticTest:" + this.staticTest(param1));
	}

	private native void hook(Class<?> clazzToHook, String methodName, String methodSig);
}
```

ndk 中的部分

```c
#include <jni.h>
#include "log.h"
#include "Dalvik.h"

static void showMethodInfo(const Method* method)
{
    //看看method的各个属性都是啥:
    LOGD("accessFlags:%d",method->accessFlags);
    LOGD("clazz->descriptor:%s",method->clazz->descriptor);
    LOGD("clazz->sourceFile:%s",method->clazz->sourceFile);
    LOGD("methodIndex:%d",method->methodIndex);
    LOGD("name:%s",method->name);
    LOGD("shorty:%s",method->shorty);
}

/**
 * 使用jni GetMethodID 方法获取jmethodID 强制转为 Method 的hook 方法 示例
 */
static void newTestMethod(const u4* args, JValue* pResult,
                          const Method* method, struct Thread* self) {

    // args 是原来函数的参数数组, 原来test函数只有一个String型参数
    // 并且要注意, 如果是不是static函数, 下标0 是函数所在类的实例obj
    // 在dvm中Object,  jni 中的jobject 和 java 中的 Object类 都不是同一个东西
    // String类对应StringObject
    // 取出参数打印出来看看
    StringObject* param1 = NULL;

    if(dvmIsStaticMethod(method))
        param1 = (StringObject*)args[0];
    else
        param1 = (StringObject*)args[1];
    LOGD("param1:%s",dvmCreateCstrFromString(param1));

    //JValue 是个union ,要返回int 就 pResult->i=1; 返回Object对象就 pResult->l = ojb;
    // 但是, 在dvm中的Object,  jni 中的jobject 和 java 中的 Object类 都不是同一个东西
    // 所以, 我们这里使用dvm的函数来创建一个StringObject*
    pResult->l = dvmCreateStringFromCstr("newTestMethod");

    // 一般情况应该使用宏 : RETURN_XXX(result);
    return;
}

extern "C" JNIEXPORT void JNICALL
Java_com_zhaoxiaodan_hookdemo_Demo1_hook(JNIEnv *env, jobject instance, jobject clazzToHook,
                                                jstring methodName_, jstring methodSig_) {

    const char *methodName = env->GetStringUTFChars(methodName_, 0);
    const char *methodSig = env->GetStringUTFChars(methodSig_, 0);

    jmethodID methodIDToHook = env->GetMethodID((jclass) clazzToHook,methodName,methodSig);

    // 找不到有可能是个static
    if(nullptr == methodIDToHook){
        env->ExceptionClear();
        methodIDToHook = env->GetStaticMethodID((jclass) clazzToHook,methodName,methodSig);
    }


    if(methodIDToHook != nullptr)
    {
        //主要在这里替换
        //jmethodID 在dvm里实际上就是个Method 结构体
        Method* method = (Method*) methodIDToHook;

        //看看method的各个属性都是啥:
        showMethodInfo(method);

        //设置Method 的 accessFlags 为 枚举型
        // ACC_NATIVE 表示 这个method 切换成了一个native 方法
        // 这个枚举 在 dalvik/libdex/DexFile.h
        // 类似:
        // ACC_PUBLIC       = 0x00000001,       // class, field, method, ic
        // ACC_PRIVATE      = 0x00000002,       // field, method, ic
        // ACC_PROTECTED    = 0x00000004,       // field, method, ic
        SET_METHOD_FLAG(method, ACC_NATIVE);

        //既然是一个native方法, 那就把 nativeFunc 指针指向我们的hook, 用来替换test的新方法
        method->nativeFunc = &newTestMethod;

        // registersSize是函数栈大小, insSize是参数占用大小
        // 如果是native方法, 就没有额外开销了
        // 所有开销就是参数占用, 所以把它设置成跟参数占用空间
        method->registersSize=method->insSize;

        //未知
        method->outsSize=0;
    }

    env->ReleaseStringUTFChars(methodName_, methodName);
    env->ReleaseStringUTFChars(methodSig_, methodSig);
}
```

运行之后得到:

```
/===[hookdemo]===﹕ ===========before hook test:11111
/[---hookdemo---]﹕ accessFlags:1
/[---hookdemo---]﹕ clazz->descriptor:Lcom/zhaoxiaodan/hookdemo/MainActivity;
/[---hookdemo---]﹕ clazz->sourceFile:MainActivity.java
/[---hookdemo---]﹕ methodIndex:334
/[---hookdemo---]﹕ name:test
/[---hookdemo---]﹕ shorty:LL
/[---hookdemo---]﹕ param1:param1
/===[hookdemo]===﹕ ===========after hook test:newTestMethod
/===[hookdemo]===﹕ ===========before hook staticTest:staticTest
/dalvikvm﹕ GetMethodID: not returning static method Lcom/zhaoxiaodan/hookdemo/MainActivity;.staticTest (Ljava/lang/String;)Ljava/lang/String;
/[---hookdemo---]﹕ accessFlags:9
/[---hookdemo---]﹕ clazz->descriptor:Lcom/zhaoxiaodan/hookdemo/MainActivity;
/[---hookdemo---]﹕ clazz->sourceFile:MainActivity.java
/[---hookdemo---]﹕ methodIndex:0
/[---hookdemo---]﹕ name:staticTest
/[---hookdemo---]﹕ shorty:LL
/[---hookdemo---]﹕ param1:param1
/===[hookdemo]===﹕ ===========after hook staticTest:newTestMethod
```

原理就是, Method结构体表示了一个java层函数, 而其中的accessFlags属性如果是ACC_NATIVE , dvm在call 原java层函数的时候, 则会转向调用 属性nativeFunc 所指向的函数

所以我们把不是native的java层函数的accessFlags强制改为ACC_NATIVE, 然后把nativeFunc 指向我们的新函数, 则完成了方法的修改

只不过, 这里使用native方法替换了java层的原方法




## 参考文章

* [编译屏障和内存屏障](http://blog.csdn.net/u013234805/article/details/24796503) : 需要设置的 ANDROID_SMP 是啥 ?
* [Dalvik虚拟机JNI方法的注册过程分析](http://blog.csdn.net/luoshengyang/article/details/8923483) 
