---
layout : post
categories: [android]
tags : [hook, 勾子]
keywords : 
excerpt: 
---



之前学习了如何做一个简单android的函数勾子, 而这个勾子是用native 的函数去hook java函数, 现在来学习如何封装让他可以实现java hook java

不过不管怎么说, 这里已经不算是`原理`了, 因为`原理`就是`改accessFlags并设置nativeFunc`, 实际的hook 函数还是个native函数, 所以说这个是用这个`原理`来封装

我们一般要hook一个方法, 有可能希望在三个时间点进行处理:

* 原方法执行前
* 替换原方法
* 原方法执行后

先来做个最简单的, 在原方法执行之前 执行, 那么就需要如下的Handler 用来hook 原函数, 并且这个java层的hook 可以获得原函数所在类的对象实例和所有参数值

```java
public interface Handler
{
	/**
	 * 在原方法执行之前执行
	 * @param params 原方法参数
	 */
	public void before(Object instance, Object[] params);
}
```

所以在jni 函数hook 里将原函数的nativeFunc 修改为 forwardToHander, 然后forwardToHander里 解析原函数的参数值等信息之后, 再调用 Handler的before方法, 将原函数所在类实例和参数值传递给handler的before回调方法

具体代码:

```c
#include <jni.h>
#include "log.h"
#include "Dalvik.h"

/**
 * dvmCallMethod 需要的是 ArrayObject*, 而 nativeFunction 传入的参数是 const u4*
 * 做个转换
 */
ArrayObject* argsToArrayObject(const char* argsTypeDesc, const u4* args)
{
    //全部参数都转成Object型
    ClassObject* objectArrayClass = dvmFindArrayClass("[Ljava/lang/Object;", NULL);

    // 参数个数, 类型描述符都是一个字符, 所以 argsTypeDesc 字符串长度就是参数个数
    int argsSize = strlen(argsTypeDesc);

    // 分配空间
    ArrayObject* argArray = dvmAllocArrayByClass(objectArrayClass,argsSize ,ALLOC_DEFAULT);

    // 挨个赋值
    // 因为 args 并不是一个真正意义上的"数组", 当long 或者 double 形, 会占用 args[i]和args[i+1]
    // 所以需要多一个下标来指向args内存
    int pArgs = 0;
    for(int i=0;i<argsSize;i++)
    {
        // 当前参数类型描述符
        const char descChar = argsTypeDesc[i];

        JValue value;
        Object* obj;

        switch(descChar)
        {

            case 'Z':
            case 'C':
            case 'F':
            case 'B':
            case 'S':
            case 'I':

                //如果是基本数据类型, 则使用java的"自动装配", 也就是 int 转 Integer, long 转 Long, 这样就都转成Object了
                value.i = args[pArgs++];
                obj = (Object*) dvmBoxPrimitive(value, dvmFindPrimitiveClass(descChar));
                dvmReleaseTrackedAlloc(obj, dvmThreadSelf());
                break;
            case 'D':
            case 'J':

                // 基本数据类型中, double 和 long 占用两个字节, 所以 pArgs 需要 +2
                value.j = dvmGetArgLong(args, pArgs++);
                pArgs++;
                obj = (Object*) dvmBoxPrimitive(value, dvmFindPrimitiveClass(descChar));
                dvmReleaseTrackedAlloc(obj, dvmThreadSelf());
                break;
            case '[':
            case 'L':
                obj  = (Object*) args[pArgs++];
                break;
            default:
                LOGE("Unknown method signature description character: %c\n", descChar);
                obj = NULL;
        }

        // 塞到ArrayObject 中
        dvmSetObjectArrayElement(argArray,i,obj);
    }

    return argArray;
}

/**
 * 被hook的函数统一调用这个native方法, 由他来装配之后跳转到Handler
 */
static void forwardToHander(const u4* args, JValue* pResult,
                          const Method* method, struct Thread* self) {

    // 之前放进去的 "附加参数"
    Object* handlerObject = const_cast<Object*>(reinterpret_cast<const Object*>(method->insns));

    // 在原方法执行之前, 我们是要调用 Handler 的 before 这个hook
    Method* handlerBeforeMethod = dvmFindInterfaceMethodHierByDescriptor(handlerObject->clazz,"before","(Ljava/lang/Object;[Ljava/lang/Object;)V");

    if(nullptr != handlerBeforeMethod)
    {
        LOGD("handlerBeforeMethod name:%s,desc:%s",handlerBeforeMethod->name,handlerBeforeMethod->shorty);
        JValue* result = new JValue();

        // method->shorty[0] 表示返回值类型, 比如"ILJ" 说明原函数有一个Object型参数和一个long型参数, 返回值为 整形
        // 注意自动装配类型, int long double, 如果是Integer, Long, Double, 则描述符为Ljava/lang/Integer,Ljava/lang/Long和Ljava/lang/Double
        const char returnTypeDesc = method->shorty[0];
        const char* argsTypeDesc = &(method->shorty[1]);

        // 转一下参数格式
        ArrayObject* argsArray ;
        Object* originalInstance;

        // 同样的, 如果原函数不是static 的, args[0] 函数是所在类的实例
        if(dvmIsStaticMethod(method))
        {
            argsArray = argsToArrayObject(argsTypeDesc,&(args[0]));
            originalInstance = nullptr;
        }else{
            argsArray = argsToArrayObject(argsTypeDesc,&(args[1]));
            originalInstance = (Object*)(args[0]);
        }


        dvmCallMethod(self,handlerBeforeMethod, handlerObject, result, originalInstance ,argsArray);
    }else{
        LOGE("method no found....");
    }

    // before 前置调用先不修改原返回值了
    pResult->l = nullptr;


}


/**
 * 设置hook, 将原函数调用转发给nativeFunc forwardToHander 进行处理
 */
extern "C" JNIEXPORT void JNICALL
Java_com_zhaoxiaodan_hookdemo_Demo2_hook(JNIEnv *env, jobject instance, jobject clazzToHook,
                                                 jstring methodName_, jstring methodSig_,
                                                 jobject handler) {
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
        Method* method = reinterpret_cast<Method*>(methodIDToHook);

        LOGD("hook function %s, args desc:%s",method->name,method->shorty);

        SET_METHOD_FLAG(method, ACC_NATIVE);
        //转发
        method->nativeFunc = &forwardToHander;
        method->registersSize=method->insSize;
        method->outsSize=0;

        // "附加参数" 功能
        // 在hook 的时候传进来的 handler 实现实例, 我们放到Method 结构体的insns 属性中
        // 到时候dvm 调用 nativeFunc 的时候会带回到 nativeFunc 中, 我们就能取到了
        method->insns = reinterpret_cast<const u2*>(dvmDecodeIndirectRef(dvmThreadSelf(),handler));
    }

    env->ReleaseStringUTFChars(methodName_, methodName);
    env->ReleaseStringUTFChars(methodSig_, methodSig);
}
```

使用代码为:

```java
public class Demo2
{
	String TAG = "===[hookdemo]===";

	public interface Handler
	{
		/**
		 * 在原方法执行之前执行
		 * @param params 原方法参数
		 */
		public void before(Object instance, Object[] params);
	}

	public String test(String param1, long param2, int[] param3)
	{
		return param1;
	}

	public static String staticTest(String param1, Long param2, int[] param3)
	{
		return param1;
	}
	public void demo()
	{
		String param1 = "param1";

		/**
		 * 设置hook
		 * hook 住Demo2 这个类的 test 方法
		 * 实现一个Handler类来实现 回调函数 before
		 */
		hook(Demo2.class, "test", "(Ljava/lang/String;J[I)Ljava/lang/String;", new Handler()
		{
			@Override
			public void before(Object instance, Object[] params)
			{
				Log.d(TAG, "===========Handler before,param len: "+params.length+",instance:"+instance);
				for (int i = 0; i < params.length; i++)
				{
					Log.d(TAG, "p" + i + "=" + params[i]);
				}
			}
		});
		Log.d(TAG, "===========call test" + this.test(param1,999L,new int[]{1,2,3}));


		/**
		 * 测试静态函数的hook, 则回调函数参数中 instance 为 null
		 */
		hook(Demo2.class, "staticTest", "(Ljava/lang/String;Ljava/lang/Long;[I)Ljava/lang/String;", new Handler()
		{
			@Override
			public void before(Object instance, Object[] params)
			{
				Log.d(TAG, "===========Handler before,param len: "+params.length+",instance:"+instance);
				for (int i = 0; i < params.length; i++)
				{
					Log.d(TAG, "p" + i + "=" + params[i]);
				}
			}
		});
		Log.d(TAG, "===========call test" + this.staticTest(param1, 999L, new int[]{1, 2, 3}));
	}

	/**
	 * hook 函数, 是个jni 函数
	 * @param clazzToHook
	 * @param methodName
	 * @param methodSig
	 * @param handler
	 */
	private native void hook(Class<?> clazzToHook, String methodName, String methodSig, Handler handler);
}
```

但是这个封装其实也不好, 只是学习如何在nativeFunc 通过dvm 调用回java层代码而已

它并没有真正的转发处理, 还是native来调用具体的方法; 也就是说如果hooker 需要做其他逻辑的话, 比如, 我是不是设置了before是不是有replace之类的, native代码就多了

最好的就是, 我把原函数改掉, 然后获取信息, 然后转发给一个java层处理器, 告诉它, 被hook的是这个函数, 对象是这个, 参数是这些, 然后勾子是这个, 就可以了, Handler 勾子的调用都由这个java层处理器来做, 而不是natviceFunc里,  native里不需要做太多逻辑


具体实现就不在做了, 因为[alibaba/dexposed](https://github.com/alibaba/dexposed)项目已经做的很好了, 具体可以看它的实现

我自己也做了些注释:

> [pangliang/dexposed](https://github.com/alibaba/dexposed)

