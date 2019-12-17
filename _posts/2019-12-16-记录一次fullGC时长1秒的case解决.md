---
layout:     post
title:      记录一次fullGC时长1秒的case解决
subtitle:   JVM优化篇
date:       2019-12-16
author:     seeyouguo
catalog: true
tags:
    - JVM
---


## 出现问题：
TP90 到 TP6个9 响应时间可以接受，甚至有的很快，但是会出现max 特别大的情况，只要超过500ms系统认定超时
![](https://drive.google.com/open?id=1L8rNP9eZb5WiS7E7uB1QqnYvcfSuI2T9)

trace查看，有响应时间过长，超时的请求存在
![](https://drive.google.com/open?id=1o52kMcJUoBNkDsXTF1RWLPdg5tTmToXu)

## 开始排查：

具体机器，具体traceId，上下文日志，发现这些时间点服务器长达1s（其他时间每10ms都存在日志）的情况
分析qcs-driver服务其他超时方法的日志，存在同样情况
服务器抖动？ 这个直接做否定，因为其它服务没这种情况


该项目的JVM 参数设置情况：
```
CUSTOM_JVM_PROD=" -server
                    -Dfile.encoding=UTF-8
                    -Dsun.jnu.encoding=UTF-8
                    -Djava.net.preferIPv6Addresses=false
                    -Djava.io.tmpdir=/tmp
                    -Duser.timezone=GMT+08
                    -Xmx8g
                    -Xms8g
                    -XX:SurvivorRatio=8
                    -XX:NewRatio=4
                    -XX:MetaspaceSize=256m
                    -XX:MaxMetaspaceSize=512m
                    -XX:+HeapDumpOnOutOfMemoryError
                    -XX:+DisableExplicitGC
                    -XX:+PrintGCDetails
                    -XX:+PrintGCTimeStamps
                    -XX:+PrintCommandLineFlags
                    -XX:+UseConcMarkSweepGC
                    -XX:+UseParNewGC
                    -XX:ParallelCMSThreads=4
                    -XX:+CMSClassUnloadingEnabled
                    -XX:+UseCMSCompactAtFullCollection
                    -XX:CMSFullGCsBeforeCompaction=1
                    -XX:CMSInitiatingOccupancyFraction=72"
```

查看jvm统计数据
jvm full gc count ，3-6天1次
jvm full gc time，1次500ms-1000ms （rpc调用500ms为超时时间）
jvm memory used, 内存使用情况显示从发布那天开始，内存从几百兆开始一直上升到6G，引发full gc

jvm监控:
![full gc情况](https://drive.google.com/open?id=1QpQKzvzww9x8bDa4ZS_zShGq2RV0T3e3)

![内存使用情况](https://drive.google.com/open?id=1S-ehxTOGKLEsnUNoJZYH1ChJrvYUtrq4)

full gc的频率不高，但是每次full gc执行时间过长，超过了系统设置的超时时间(500ms)，服务qps10000左右，tp999为40ms，平均响应时间2ms。这样单次full gc 747ms影响十分巨大

由于设置-XX:+UseCMSCompactAtFullCollection, 可以确定full gc使用的是CMS（java8版本，未配置使用G1）。
查看cms gc日志情况:
- cms failure 的情况没有查到

- cms remark 日志：
![gc日志](https://drive.google.com/open?id=1yGBq0AmJywgQkdsHY27i4fAakZwFXrn8)

可以看到full gc 747ms的情况下CMS-remark 占用时间是740ms, 基本上时间都在做remark。

目前总结的情况：
问题点：
1 full gc 时间过长，内存设置过大，cms-remark时间过长
2 full gc频率是否太长，3-7天才发生一次full gc?

## 解决思路
* 减少jvm 内存
    -Xmx4g
    -Xms4g
* 减少cms-remak时长
    
    最终标记阶段停顿时间过长问题
    CMS的GC停顿时间约80%都在最终标记阶段(Final Remark)，若该阶段停顿时间过长，常见原因是新生代对老年代的无效引用，在上一阶段的并发可取消预清理阶段中，执行阈值时间内未完成循环，来不及触发Young GC，清理这些无效引用
    通过添加参数：-XX:+CMSScavengeBeforeRemark。在执行最终操作之前先触发Young GC，从而减少新生代对老年代的无效引用，降低最终标记阶段的停顿，但如果在上个阶段(并发可取消的预清理)已触发Young GC，也会重复触发Young GC

## 执行

## 效果


## 参考：

- [从实际案例聊聊Java应用的GC优化](https://tech.meituan.com/2017/12/29/jvm-optimize.html)
- [老大难的GC原理及调优](https://juejin.im/post/5b6b986c6fb9a04fd1603f4a)
- [PhantomReference导致CMS GC耗时严重](https://www.jianshu.com/p/6d37afd1f072)
