---
layout : post
categories: [java, android]
tags : [加密, 解密, 反编译, 调试, natvie, jni]
keywords : 
excerpt: 
---
{% include JB/setup %}


## 思考

之前研究了下如何调试和尝试反一个别人加密的东西, 所以现在的体会就是:

> 其实重点不是你如何加密, 重点是如何不让别人知道你怎么加密的

因为像这种`自己加密的资源运行的时候自己解密之后拿来用`的程序, 我甚至根本不用关心你到底怎么加密, 加密算法是啥, 我只需要知道, 你解密完了之后, 那个资源的内存块在哪, 写个dumper就全拿到了;

加密并不能防止被破解, 只是增加破解的难度和门槛, 加密解密是一个相互博弈的过程

把资源加个密, 那么对于那些技术比较初级, 又想简单拷贝的人, 他们就拿你没办法了; 但是对于那些懂一点原理懂一点IDA的人, 并没有什么卵用

那么好, 现在我要研究一下, 怎么样再增加那么一点点门槛

之前解密的一个重要前提就是调试, 所以, 最直接想到的增加的门槛就是`反调试`.

## 测试

先建个android 工程, 并加入native支持

> [android-anti-debug](https://github.com/pangliang/android-anti-debug)


先打出pid和父pid出来看看

```c++
#include <jni.h>
#include <stdio.h>
#include <sys/ptrace.h>
#include <unistd.h>

#include "log.h"

void anti_debug()
{
	int pid = getpid();
	int ppid = getppid();

	LOGD("pid:%d,ppid:%d",pid,ppid);

}

jint JNI_OnLoad(JavaVM* vm, void* reserved)
{
	anti_debug();
	JNIEnv* env;
	if (vm->GetEnv(reinterpret_cast<void**>(&env), JNI_VERSION_1_6) != JNI_OK)
	{
		return -1;
	}

	return JNI_VERSION_1_6;
}
```

```
06-11 06:42:24.411: D/[android-anti-debug](1451): pid:1451,ppid:192
```

pid 192 是:

```
root      192   1     492204 38400 ffffffff b756598c S zygote
```
> 在Android系统中，所有的应用程序进程以及系统服务进程SystemServer都是由Zygote进程孕育（fork）出来的，这也许就是为什么要把它称为Zygote（受精卵）的原因吧
> 
> 具体参见:[Android系统进程Zygote启动过程的源代码分析](http://blog.csdn.net/luoshengyang/article/details/6768304)

看下 `/proc/1451/stat`

```
root@mx3:/data/local/tmp # cat /proc/17204//status
Name:	ndroidantidebug
State:	S (sleeping)
Tgid:	17204
Pid:	17204
PPid:	2146
TracerPid:	0
Uid:	10058	10058	10058	10058
Gid:	10058	10058	10058	10058
FDSize:	256
...
```

用gdbserver 去attach一下看看发生什么:

```
root@mx3:/data/local/tmp # ps |grep blog
u0_a58    30678 2146  907884 59720 ffffffff 4005778c S com.zhaoxiaodan.blog.androidantidebug
root@mx3:/data/local/tmp # ./gdbserver --attach 127.0.0.1:1234 30678
Attached; pid = 30678
Listening on port 1234

root@mx3:/data/local/tmp # cat /proc/17204//status
Name:	ndroidantidebug
State:	t (tracing stop)
Tgid:	17204
Pid:	17204
PPid:	2146
TracerPid:	20337
Uid:	10058	10058	10058	10058
Gid:	10058	10058	10058	10058
```

发现`TracerPid`行由0 变为了 20337


ida这些调试工具其实都是使用`ptrace`进行的, ptrace有一个很重要的特定：

> 一个进程只能被一个进程调试。

所以, 最简单的办法就是在`JNI_OnLoad`里直接`ptrace(PTRACE_TRACEME, 0, 0, 0);`

## 方法1, 直接`ptrace(PTRACE_TRACEME, 0, 0, 0);`

```c++
#include <jni.h>
#include <stdio.h>
#include <sys/ptrace.h>
#include <unistd.h>

#include "log.h"

void anti_debug()
{
	ptrace(PTRACE_TRACEME, 0, 0, 0);
}

jint JNI_OnLoad(JavaVM* vm, void* reserved)
{
	anti_debug();
	JNIEnv* env;
	if (vm->GetEnv(reinterpret_cast<void**>(&env), JNI_VERSION_1_6) != JNI_OK)
	{
		return -1;
	}

	return JNI_VERSION_1_6;
}
```

然后再用gdbserver 去attach:

```
root@mx3:/data/local/tmp # ./gdbserver --attach 127.0.0.1:1234 31092
Cannot attach to lwp 31092: Operation not permitted (1)

Exiting
```

好了, 挂不上了

但是这种方法, 用反编译打开, 很容易就找到调用`ptrace`的地方, 不知道修改下`汇编指令`(比如改为nilnil), 就跳过这个调用了

## 方法2, 暗桩

根据上面说的`/proc/$pid/status`中`TracerPid`行显示调试程序的`pid`的原理, 可以写一个方法检查下这个值, 如果!=0就退出程序

检查函数如下:

```c++
void be_attached_check()
{
	try
	{
		const int bufsize = 1024;
		char filename[bufsize];
		char line[bufsize];
		int pid = getpid();
		sprintf(filename, "/proc/%d/status", pid);
		FILE* fd = fopen(filename, "r");
		if (fd != nullptr)
		{
			while (fgets(line, bufsize, fd))
			{
				if (strncmp(line, "TracerPid", 9) == 0)
				{
					int statue = atoi(&line[10]);
					LOGD("%s", line);
					if (statue != 0)
					{
						LOGD("be attached !! kill %d", pid);
						fclose(fd);
						int ret = kill(pid, SIGKILL);
					}
					break;
				}
			}
			fclose(fd);
		} else
		{
			LOGD("open %s fail...", filename);
		}
	} catch (...)
	{

	}

}
```

可以把这个函数搞成一个宏, 然后写个程序随机的把这个宏插入到源码的各个地方, 随着代码的不断执行, 会遇到各个这样的检查点

其实也没什么卵用, 只不过桩子多了, 你拔起来就麻烦点咯

下面这个只是用线程模拟检查的过程:

```c++
#include "android-anti-debug.h"
#include <string>
#include <sys/ptrace.h>
#include <unistd.h>
#include <stdlib.h>
#include <chrono>
#include <thread>
#include "log.h"

void be_attached_check()
{
	try
	{
		const int bufsize = 1024;
		char filename[bufsize];
		char line[bufsize];
		int pid = getpid();
		sprintf(filename, "/proc/%d/status", pid);
		FILE* fd = fopen(filename, "r");
		if (fd != nullptr)
		{
			while (fgets(line, bufsize, fd))
			{
				if (strncmp(line, "TracerPid", 9) == 0)
				{
					int statue = atoi(&line[10]);
					LOGD("%s", line);
					if (statue != 0)
					{
						LOGD("be attached !! kill %d", pid);
						fclose(fd);
						int ret = kill(pid, SIGKILL);
					}
					break;
				}
			}
			fclose(fd);
		} else
		{
			LOGD("open %s fail...", filename);
		}
	} catch (...)
	{

	}

}

//检查线程, 每秒检查一下
void thread_task(int n)
{
	while (true)
	{
		LOGD("start be_attached_check...");
		be_attached_check();
		std::this_thread::sleep_for(std::chrono::seconds(n));
	}
}

void anti_debug()
{
//	ptrace(PTRACE_TRACEME, 0, 0, 0);
	auto checkThread = std::thread(thread_task, 1);
	checkThread.detach();
}

jint JNI_OnLoad(JavaVM* vm, void* reserved)
{
	anti_debug();
	JNIEnv* env;
	if (vm->GetEnv(reinterpret_cast<void**>(&env), JNI_VERSION_1_6) != JNI_OK)
	{
		return -1;
	}

	return JNI_VERSION_1_6;
}
```













