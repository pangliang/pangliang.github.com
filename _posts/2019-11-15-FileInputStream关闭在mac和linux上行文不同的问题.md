---
layout : post
categories: [java]
tags : []
keywords :
excerpt:
sequence: yes
---

```java
import java.io.InputStream;
import java.util.concurrent.CountDownLatch;

public class Test {
    public static void main(String[] args) throws Exception {
        CountDownLatch threadStartLock = new CountDownLatch(1);
        CountDownLatch mainLock = new CountDownLatch(1);

        InputStream in = System.in;

        new Thread(() -> {
            System.out.println("thread start...");
            threadStartLock.countDown();
            try {
                byte[] buf = new byte[1024];
                int len = 0;
                while((len = in.read(buf, 0, buf.length)) > 0) {
                    System.out.println(new String(buf, 0, len));
                }

                System.out.println("thread return..." + len);
            }catch (Exception e) {
                e.printStackTrace();
            }finally {
                mainLock.countDown();
            }
        }).start();

        threadStartLock.await();

        Thread.sleep(3 * 1000);
        System.out.println("closeing...");
        in.close();
        System.out.println("closed...");

        mainLock.await();
        System.out.println("exit");
    }
}
```

* InputStream 在一个线程中读, 当无输入时会阻塞在 `read()` 方法
* 在另一个线程中主动close关闭

我们一般都会认为, `inputstream.read()` 方法的阻塞会中断并返回-1或抛异常; SocketInputStream 也确实就是这样的行为(抛异常)

但是

`FileInputStream` 在 `mac` 和 `linux` 上会有不同的行为

* 在mac上, close后, read的阻塞中断且返回-1
* 在linux上, close后, 如果一直没有读到数据, 一直阻塞, 读到数据, 返回-1

也就是说, 在linux上, 如果没有可读数据, read会一直阻塞, close也无法中断

这样的场景下, 正确做法

```java
byte[] buf = new byte[1024];
int len = 0;
do {
	while((len = in.available()) == 0){
		Thread.sleep(50);
	}
	len = in.read(buf, 0, len);
	System.out.println(new String(buf, 0, len));
}while(len > 0);
```

`in.available()` EOF和无可读都返回0, 因此无法区分两者; 只能使用在`无限流`的场景
