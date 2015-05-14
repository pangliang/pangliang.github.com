---
layout : post
categories: [android]
tags : [黑屏, 闪退, 生命周期]
keywords : 
excerpt: 
---
{% include JB/setup %}

这次做个移动MM的支付SDK, 提交测试总是打回来说:

>启动该应用，触发任意计费点，在支付界面进行切换后台或锁屏解锁操作，返回后应用报错退出

查来查去, 发现每次切到桌面, sdk总是又被初始化一次; 而sdk初始话是在Activity的onCreate() 中进行的; 那也就说Activity在切Home的时候被onDestroy()了

于是去检查 Cocos2dxActivity 的各个onXXX方法, 没见啥异常

参考[横屏切换竖屏Activity的生命周期及configChanges](http://blog.csdn.net/imdxt1986/article/details/7339711)

- 不设置Activity的android:configChanges时，切屏会重新调用各个生命周期，切横屏时会执行一次，切竖屏时会执行两次

- 设置Activity的android:configChanges="orientation"时，切屏还是会重新调用各个生命周期，切横、竖屏时只会执行一次

- 设置Activity的android:configChanges="keyboardHidden"时，切屏还是会重新调用各个生命周期，切横、竖屏时只会执行一次

- 设置Activity的android:configChanges="orientation|screenSize|keyboardHidden"时，切屏不会重新调用各个生命周期，只会执行onConfigurationChanged方法

于是设置之, 提交
