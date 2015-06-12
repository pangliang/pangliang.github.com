---
layout: post
categories : [lnmp]
tags : [redis, php, session, 并发一致性]
excerpt: 通过phpredis模块一个并发问题,分析php session的机制
---
{% include JB/setup %}

session 在php中的生命周期，已文件存储为例

    * session_start()：打开session文件，将session的数据读出来，反序列化后存入$_SESSION变量
    * 改变$_SESSION的key值时并不写入 session文件
    * session_write_close()后，或者php文件退出后，才将$_SESSION序列化后存入文件

这样$_SESSION不管有多少个KEY，都是整体读出，整体写入的。那么这些key的值就会有一致性问题；


例如,原来key1=1111， key2=2222，程序流程：

    线程1：读取session -> 其他操作 --- > 修改key1 为 3333  -> 保存
    线程2：读取session -> 修改key2为4444 -> 保存

那么此时的结果会是   key1=3333 ，key2=2222

在文件为session的存储方法时，php源码在fopen了session_file_handle之后，会对这个session_file_handle做一个flock(,LOCK_EX)，也就是加一个写锁；将并发变成顺序

    线程1：获取锁 -> 读取session -> 其他操作- > 修改key1 为 3333  -> 保存

    线程2：等待锁 ————————————————————————-> 读取session -> key2为4444 -> 保存

而phpredis使用redis作为存储介质, 没有做这个锁, 那么会有这个并发一致性问题存在

使用下面的代码即可重现这个问题:

* 先执行one.php
* 再执行two.php
* 不等two.php返回立刻three.php
* 等two.php返回后four.php输出最后结果


测试代码:

*   one.php:

        {% highlight php %}
        <?php
            require_once("redis.php");
    
            session_start();
    
            $_SESSION['key1']=11111111;
            $_SESSION['key2']=22222222;
    
            print "session_id:".session_id()."<br>";
            print $_SESSION['key1']."|".$_SESSION['key2'];
        ?>
        {% endhighlight %}

*   two.php:

        {% highlight php %}
        <?php
    
            require_once("redis.php");
    
            session_start();
    
            sleep(10);
            $_SESSION['key2']=444444444;
    
            print "session_id:".session_id()."<br>";
        ?>
        {% endhighlight %}

*   three.php:

        {% highlight php %}
            <?php
    
                require_once("redis.php");
    
                session_start();
    
                $_SESSION['key1']=333333333;
    
                print "session_id:".session_id()."<br>";
                print $_SESSION['key1']."|".$_SESSION['key2'];
            ?>
        {% endhighlight %}

*   four.php:

        {% highlight php %}
            <?php
    
                require_once("redis.php");
    
                session_start();
    
                print "session_id:".session_id()."<br>";
                print $_SESSION['key1']."|".$_SESSION['key2'];
            ?>
        {% endhighlight %}

*   redis.php:

        {% highlight php %}
            <?php
                ini_set("session.save_handler","redis");
                ini_set("session.save_path","tcp://127.0.0.1:6379?timeout=1");
            ?>
        {% endhighlight %}
