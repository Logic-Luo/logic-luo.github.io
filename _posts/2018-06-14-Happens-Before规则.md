---
layout:     post
title:      Happens-Before规则
subtitle:   ""
date:       2018-06-14
author:     Logic
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - jvm
    - Java
---

#### Happens-Before规则

- **程序循序规则**。如果程序中操作A在操作B之前，那么在线程中A操作将在B操作之前执行。
- **监视器锁规则**。在监视器锁上的解锁操作必须在同一个监视器锁上的加锁之前执行。
- **volatile变量规则**。对volatile变量的写入操作必须在对该变量的读操作之前执行。
- **线程启动规则**。在线程上对Thread.Start的调用必须在该线程中执行任何操作之前执行。
- **线程结束规则**。线程中的任何操作都必须在其他线程检测到该线程已经结束之前执行，或者从Thread.join中成功返回，或者在调用Thread.isAlive时返回false。
- **中断规则**。当一个线程在另一个线程上调用interrupt时，必须在被中断线程检测到interrupt调用之前执行（通过抛出InterruptedException，或者调用isInterrupted和interrupted）。
- **终结器规则**。对象的构造函数必修在启动该对象的终结器执行之前完成。
- **传递性**。如果操作A在操作B之前执行，并且操作B在操作C之前执行，那么操作A必须在操作C之前完成。

#### 在类库中的提供的其他Happens-Before排序
- 将一个元素放入线程安全容器的操作将在另一个线程从该容器中获得这个元素的操作之前执行。
- 在CountDownLatch上的倒数操作将在线程从闭锁上的await方法返回之前执行。
- 释放Semaphore许可的操作将在从该Semaphore上获得一个许可之前执行。
- Future表示的任务的所有操作将在从Future.get中返回之前执行。
- 向Executor提交一个Runnable或Callable的操作将在任务开始执行之前执行。
- 一个线程到达CyclicBarrier或Exchanger的操作将在其他到达该栅栏或交换点的线程被释放之前执行。如果CyclicBarrier使用一个栅栏操作，那么到达栅栏的操作将在栅栏操作之前执行，而栅栏操作又会在线程从展览中释放之前执行。