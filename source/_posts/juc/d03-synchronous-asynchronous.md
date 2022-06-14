---
title: "并发编程（三）同步与异步"
date: 2021-04-11T01:12:27+08:00
lastmod: 2021-04-11T15:35:42+08:00
categories: ["JUC"]
---

同步与异步的概念。

<!--more-->

## 概念

从方法调用的角度来看，同步和异步的概念：

- 如果需要等待结果返回才能继续运行的话就是同步。
- 如果不需要等待结果返回就能继续运行就是异步。

## 案例

同步读取文件：

```java
@Slf4j(topic = "d03.SynchronousReadFile")
public class SynchronousReadFile {
    public static void main(String[] args) {
        ResourceReadUtil.read(Constants.ULTRA_MAN_MP4_RES_PATH);

        if(log.isDebugEnabled()){
            log.debug("开始做其他事情...");
        }
    }
}
```

异步读取文件：

```java
@Slf4j(topic = "d03.AsynchronousReadFile")
public class AsynchronousReadFile {
    public static void main(String[] args) {
        new Thread(() -> ResourceReadUtil.read(Constants.ULTRA_MAN_MP4_RES_PATH)).start();

        if (log.isDebugEnabled()) {
            log.debug("开始做其他事情...");
        }
    }
}
```

多线程可以使方法的执行变成异步的（即不用干巴巴的等着），比如说读取磁盘文件时，假设读取操作花费了5秒，如果没有线程的调度机制，这5秒调用者什么都做不了，其他代码都得暂停……

## 场景

- 比如在项目中，视频文件需要转换格式等操作比较费时，这时开一个新线程处理视频转换，避免阻塞主线程。
- Tomcat 的异步 Servlet （Servlet3.0）也是类似的目的，让用户线程处理耗时较长的操作，避免阻塞 Tomcat 的工作线程。

- UI 程序中，开线程进行其他操作，避免阻塞 UI 线程。

## 应用

[使用多线程充分利用处理器](./e01-optimize-use/#使用多线程充分利用处理器)
