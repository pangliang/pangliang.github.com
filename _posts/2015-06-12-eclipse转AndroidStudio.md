---
layout : post
categories: [android]
tags : [eclipse, adt, android studio, ndk]
keywords : 
excerpt: 
---
{% include JB/setup %}

最近搞ndk比较多, eclipse的ndk还算不错, 但是cdt有点傻傻的, 我本来加了c++11的特性, 配置好了build也没有任何问题, cdt偏偏说认不出`std::thread`之类的

而且google官方已经把adt改成android studio了, adt基于eclipse, 而android studio基于IntelliJ IDEA. 一直都觉得idea 比eclipse用起来舒服, 就因为需要搞android一直没有换; 得了, 现在可以换过去了

## ndk配置

- local.properties

> ndk.dir=/server/android-ndk-r10e

- build.gradle

```
android {
	...
	
   defaultConfig {
        ...

        ndk {
            moduleName "android_anti_debug"
            cFlags "-std=c++11 -pthread -fexceptions -frtti"
            ldLibs "log", "z", "android", "atomic"
            stl "gnustl_static"
            abiFilters "x86", "armeabi"
        }
    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
}
```

## 编译慢

AndroidStudio -> Preferences -> BuildTools -> Gradle -> 勾选`Offline work`















