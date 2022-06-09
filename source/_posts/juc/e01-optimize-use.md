---
title: "并发实战（一）优化与应用"
date: 2021-07-27T17:33:12+08:00
lastmod: 2021-07-27T17:33:12+08:00
categories: ["JUC"]
---

并发实战之代码优化和应用案例。

<!--more-->

## 效率

### 使用多线程充分利用处理器

#### 环境搭建

- 基准测试工具选择，使用了比较靠谱的 JMH，它会执行程序预热，执行多次测试并平均
- CPU 核数限制，有两种思路

	1. 使用虚拟机，分配合适的核
	2. 使用 `msconfifig`，分配合适的核，需要重启比较麻烦

- 并行计算方式的选择

	1. 最初想直接使用 parallel stream，后来发现它有自己的问题
	2. 改为了自己手动控制 thread，实现简单的并行计算

#### 项目生成

使用如下命令生成示例项目代码：

```shell
mvn archetype:generate -DinteractiveMode=false -DarchetypeGroupId=org.openjdk.jmh -DarchetypeArtifactId=jmh-java-benchmark-archetype -DgroupId=e01 -DartifactId=e01 -Dversion=1.0.0
```

#### 案例代码

```java
import org.openjdk.jmh.annotations.*;

import java.util.Arrays;
import java.util.concurrent.FutureTask;
import java.util.concurrent.TimeUnit;

/**
 * 该基准测试测试单线程与多线程情况下对1亿个数进行累加的执行效率
 * <p>
 * multiThreadTest: 4线程每个线程累加2500万次
 * singleThreadTest: 1线程累加1亿次
 * <p>
 * 测试时将该类打包位可执行JAR并使用JAVA命令运行
 */
@Fork(1)
//热身模式使用统计程序最后运行的平均时间
@BenchmarkMode(Mode.AverageTime)
//热身次数
@Warmup(iterations = 3)
//进行的测试次数 这里设置位5轮测试
@Measurement(iterations = 5)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
public class MultiThreadBenchmark {
    static int[] ARRAY = new int[1000_000_00];//声明长度为1亿的整型数组

    static {
        Arrays.fill(ARRAY, 1);//数组每个元素的值为1
    }

    @Benchmark
    public int multiThreadTest() throws Exception {
        int[] array = ARRAY;
        FutureTask<Integer> t1 = new FutureTask<>(() -> {
            int sum = 0;
            for (int i = 0; i < 250_000_00; i++) {
                sum += array[i];
            }
            return sum;
        });
        FutureTask<Integer> t2 = new FutureTask<>(() -> {
            int sum = 0;
            for (int i = 0; i < 250_000_00; i++) {
                sum += array[250_000_00 + i];
            }
            return sum;
        });
        FutureTask<Integer> t3 = new FutureTask<>(() -> {
            int sum = 0;
            for (int i = 0; i < 250_000_00; i++) {
                sum += array[500_000_00 + i];
            }
            return sum;
        });
        FutureTask<Integer> t4 = new FutureTask<>(() -> {
            int sum = 0;
            for (int i = 0; i < 250_000_00; i++) {
                sum += array[750_000_00 + i];
            }
            return sum;
        });
        new Thread(t1).start();
        new Thread(t2).start();
        new Thread(t3).start();
        new Thread(t4).start();
        return t1.get() + t2.get() + t3.get() + t4.get();
    }

    @Benchmark
    public int singleThreadTest() throws Exception {
        int[] array = ARRAY;
        FutureTask<Integer> t1 = new FutureTask<>(() -> {
            int sum = 0;
            for (int i = 0; i < 1000_000_00; i++) {
                sum += array[i];
            }
            return sum;
        });
        new Thread(t1).start();
        return t1.get();
    }
}
```

#### 基准测试

##### 1物理CPU/4核/4逻辑CPU/4线程

```
C:\Users\Administrator>systeminfo

主机名:           M4A103006
OS 名称:          Microsoft Windows 10 企业版
OS 版本:          10.0.17763 暂缺 Build 17763
OS 制造商:        Microsoft Corporation
OS 配置:          独立工作站
OS 构建类型:      Multiprocessor Free
注册的所有人:     test
注册的组织:
产品 ID:          00328-90000-00000-AAOEM
初始安装日期:     2019/9/2, 9:38:59
系统启动时间:     2021/2/22, 23:51:50
系统制造商:       Gigabyte Technology Co., Ltd.
系统型号:         B150M-SC-SI
系统类型:         x64-based PC
处理器:           安装了 1 个处理器。
```



```shell
C:\Users\Administrator>wmic
wmic:root\cli>cpu get *
AddressWidth  Architecture  AssetTag                Availability  Caption                                Characteristics  ConfigManagerErrorCode  ConfigManagerUserConfig  CpuStatus  CreationClassName  CurrentClockSpeed  CurrentVoltage  DataWidth  Description                            DeviceID  ErrorCleared  Er
64            9             To Be Filled By O.E.M.  3             Intel64 Family 6 Model 165 Stepping 3  252                                                               1          Win32_Processor    3096               8               64         Intel64 Family 6 Model 165 Stepping 3  CPU0
```

基准测试结果如下：

```shell
PS D:\project\x\juc-learning\e01\target> java -jar .\benchmark.jar
# VM invoker: D:\lib\jdk1.8.0_291\jre\bin\java.exe
# VM options: <none>
# Warmup: 3 iterations, 1 s each
# Measurement: 5 iterations, 1 s each
# Threads: 1 thread, will synchronize iterations
# Benchmark mode: Average time, time/op
# Benchmark: e01.MultiThreadBenchmark.multiThreadTest

# Run progress: 0.00% complete, ETA 00:00:16
# Fork: 1 of 1
# Warmup Iteration   1: 29.682 ms/op
# Warmup Iteration   2: 29.392 ms/op
# Warmup Iteration   3: 30.204 ms/op
Iteration   1: 31.925 ms/op
Iteration   2: 32.760 ms/op
Iteration   3: 31.405 ms/op
Iteration   4: 29.112 ms/op
Iteration   5: 33.795 ms/op


Result: 31.799 ±(99.9%) 6.752 ms/op [Average]
  Statistics: (min, avg, max) = (29.112, 31.799, 33.795), stdev = 1.754
  Confidence interval (99.9%): [25.047, 38.551]


# VM invoker: D:\lib\jdk1.8.0_291\jre\bin\java.exe
# VM options: <none>
# Warmup: 3 iterations, 1 s each
# Measurement: 5 iterations, 1 s each
# Threads: 1 thread, will synchronize iterations
# Benchmark mode: Average time, time/op
# Benchmark: e01.MultiThreadBenchmark.singleThreadTest

# Run progress: 50.00% complete, ETA 00:00:10
# Fork: 1 of 1
# Warmup Iteration   1: 41.810 ms/op
# Warmup Iteration   2: 48.859 ms/op
# Warmup Iteration   3: 47.528 ms/op
Iteration   1: 40.586 ms/op
Iteration   2: 40.174 ms/op
Iteration   3: 47.591 ms/op
Iteration   4: 41.265 ms/op
Iteration   5: 47.850 ms/op


Result: 43.493 ±(99.9%) 14.940 ms/op [Average]
  Statistics: (min, avg, max) = (40.174, 43.493, 47.850), stdev = 3.880
  Confidence interval (99.9%): [28.554, 58.433]


# Run complete. Total time: 00:00:20

Benchmark                                  Mode  Samples   Score  Score error  Units
e.MultiThreadBenchmark.multiThreadTest     avgt        5  31.799        6.752  ms/op
e.MultiThreadBenchmark.singleThreadTest    avgt        5  43.493       14.940  ms/op
```

##### 单核CPU

基准测试结果如下：

```shell
[root@localhost ~]# java -jar -Xmx1G benchmark.jar 
# VM invoker: /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.282.b08-1.el7_9.x86_64/jre/bin/java
# VM options: -Xmx1G
# Warmup: 3 iterations, 1 s each
# Measurement: 5 iterations, 1 s each
# Threads: 1 thread, will synchronize iterations
# Benchmark mode: Average time, time/op
# Benchmark: e01.MultiThreadBenchmark.multiThreadTest

# Run progress: 0.00% complete, ETA 00:00:16
# Fork: 1 of 1
# Warmup Iteration   1: 55.879 ms/op
# Warmup Iteration   2: 47.744 ms/op
# Warmup Iteration   3: 46.071 ms/op
Iteration   1: 46.260 ms/op
Iteration   2: 45.865 ms/op
Iteration   3: 47.366 ms/op
Iteration   4: 47.519 ms/op
Iteration   5: 49.032 ms/op


Result: 47.208 ±(99.9%) 4.776 ms/op [Average]
  Statistics: (min, avg, max) = (45.865, 47.208, 49.032), stdev = 1.240
  Confidence interval (99.9%): [42.432, 51.984]


# VM invoker: /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.282.b08-1.el7_9.x86_64/jre/bin/java
# VM options: -Xmx1G
# Warmup: 3 iterations, 1 s each
# Measurement: 5 iterations, 1 s each
# Threads: 1 thread, will synchronize iterations
# Benchmark mode: Average time, time/op
# Benchmark: e01.MultiThreadBenchmark.singleThreadTest

# Run progress: 50.00% complete, ETA 00:00:14
# Fork: 1 of 1
# Warmup Iteration   1: 47.382 ms/op
# Warmup Iteration   2: 45.322 ms/op
# Warmup Iteration   3: 47.355 ms/op
Iteration   1: 45.308 ms/op
Iteration   2: 46.224 ms/op
Iteration   3: 51.987 ms/op
Iteration   4: 46.791 ms/op
Iteration   5: 46.782 ms/op


Result: 47.418 ±(99.9%) 10.106 ms/op [Average]
  Statistics: (min, avg, max) = (45.308, 47.418, 51.987), stdev = 2.625
  Confidence interval (99.9%): [37.312, 57.525]


# Run complete. Total time: 00:00:24

Benchmark                                  Mode  Samples   Score  Score error  Units
e.MultiThreadBenchmark.multiThreadTest     avgt        5  47.208        4.776  ms/op
e.MultiThreadBenchmark.singleThreadTest    avgt        5  47.418       10.106  ms/op
```

#### 总结

从上面多核与单核的基准测试结果来看：

1. 多核处理器多线程执行效率明显高于单线程。
2. 单核处理器使用多线程时执行效率反而不如单线程执行，导致这其中的差异是因为线程切换上下文时需要花费时间。
3. 虽然单核处理器使用多线程在执行效率上弱于单线程，但并不意味单核处理器上使用多线程就没有意义：
   + 如果在单核处理器上只使用单线程进行处理，那么 CPU 只能执行一项任务
   + 而使用多线程时，虽然效率有些微的降低，但仍能同时执行多项任务

## 限制

### 限制对 CPU 的使用

#### sleep 方式实现

以下代码均在单核CPU下进行测试，多核CPU下无法达到利用率100%：

```java
/**
 * 在单核CPU下进行测试，多核CPU下无法达到利用率100%
 */
public class CPULimitWithSleep {
    public static void main(String[] args) {
        while (true) {
            //空转CPU 将导致CPU占用高达100% 无法执行其他程序
        }
    }
}
```

在没有利用 CPU 来计算时，不要让 `while(true)` 空转浪费 CPU ，这时可以使用 `yield` 或 `sleep` 来让出 CPU 的使用权给其他程序

```java
public class CPULimitWithSleep {
    public static void main(String[] args) {
        while (true) {
            try {
                Thread.sleep(50);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

- 可以用 `wait` 或 条件变量达到类似的效果
- 不同的是，后两种都需要加锁，并且需要相应的唤醒操作，一般适用于要进行同步的场景

- `sleep` 适用于无需锁同步的场景

