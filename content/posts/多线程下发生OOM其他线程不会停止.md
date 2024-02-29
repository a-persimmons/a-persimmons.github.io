---
author: ["柿子"]
title: "多线程下发生OOM其他线程不会停止"
date: 2021-06-14 08:03:23
description: "多线程下发生OOM其他线程不会停止."
summary: "关于多线程下发生OOM其他线程不会停止."
tags: ["Java","Thread","JVM","GC"]
---



## 实践得出结论

### 小程序™：

```java
/**
 * @author Chunming Liu
 */
public class HeapOOMTest {
    public static void main(String[] args) {
        new Thread ( () -> {
            ArrayList<byte[]> bytes = new ArrayList<> ();

            while (true) {
                System.out.println ( "[" + LocalDateTime.now () + "] " + Thread.currentThread () );
                bytes.add ( new byte[1024 * 1024] );
                try {
                    Thread.sleep ( 1000 );
                } catch (InterruptedException e) {
                    e.printStackTrace ();
                }
            }
        } ).start ();
        new Thread ( () -> {
            while (true) {
                System.out.println ( LocalDateTime.now () + "==" + Thread.currentThread () );
                try {
                    Thread.sleep ( 1000 );
                } catch (InterruptedException e) {
                    e.printStackTrace ();
                }
            }
        } ).start ();
    }
}

```

### 运行之前添加JVM参数

-Xms16m -Xmx32m

![image-20210614071732239](https://i.loli.net/2021/08/19/ev71FdB8b9KkLW3.png)

### 通过运行期间发生了 OOM但是可以出程序还是在正常运行

![image-20210614071838907](https://i.loli.net/2021/08/19/gkj1s4CrHIGRDBV.png)

### Jconsole监控堆信息

![image-20210614072230423](https://i.loli.net/2021/08/19/V9XzToKbFctm1Ss.png)

### JVM运行时内存情况

![image-20210614073643362](https://i.loli.net/2021/08/19/AcrTsI8LxuNGopj.png)

### 结论

触发堆异常不会影响其他线的运行，通过VM的内存情况，可以看出第一次发生的GC会将堆信息移动到老年代，后面的GC伊甸区活动，猜测和GC的策略有关（当前的GC策略：ParScav:MSC）