# 进程 VS 线程 ?

Java 从诞生起就明智的支持多线程。线程作为操作系统的调度的最小单元，多个线程合理的并发执行能够明显的提高程序性能。但是不当的使用多程序或者过多的创建线程也会是程序性能降低。所以学习如何使用多线程是每个开发者都必须要面对的过程。<br />

<a name="SMRLQ"></a>
## 什么是进程？
进程是操作系统的基础，是一次的程序执行，是一个程序及其数据在处理机上顺序执行所发生的活动，它是系统进行资源分配的独立单位。一个进程可以包含多个线程，每个线程之间共享操作系统分配给进程的资源。<br />在Windows操作系统中，可以在任务管理器中查看所有的进程(包含用户进程和系统进程); 在类Unix系统中，可以在终端使用PS命令查看。
```shell
# zhoutao @ zhoutaodeMacBook-Pro in ~ [22:42:58]
$ ps -a
  PID TTY           TIME CMD
99351 ttys000    0:00.99 /bin/zsh --login -i
 1802 ttys001    0:00.00 ps -a
32944 ttys001    0:00.03 /Applications/iTerm.app/Contents/MacOS/iTerm2 --server login -fp zhoutao
32945 ttys001    0:00.02 login -fp zhoutao
32946 ttys001    0:06.69 -zsh
38486 ttys004    0:00.05 /Applications/iTerm.app/Contents/MacOS/iTerm2 --server login -fp zhoutao
38487 ttys004    0:00.09 login -fp zhoutao
38488 ttys004    0:12.65 -zsh
73520 ttys005    0:02.37 /bin/zsh --login -i
81952 ttys006    0:01.64 /bin/zsh --login -i
```

- 每个进程都有独立的PID，通过PID可以对进程进行操作。



<a name="GdpMX"></a>
## 什么是线程？
线程是操作系统进行资源调度的最小资源，因此也称之为轻量级进程(Light Weight Process)。每个线程拥有个字的内存区域，计数器以及局部变量等属性。线程在单核CPU上执行的时候是通过分配时间片的方式，在多个线程之间快速切换上下文来模拟并发执行多线程的效果的。<br />
<br />一个简单的hello,world程序，看似没有多线程的参与，其实不然。因为Java天然支持多线程，所以JVM会在main方法启动之前创建一些辅助的线程(称之为守护线程，后文会对此详细介绍)，比如垃圾回收线程等等，通过JMX程序，可以看到这些线程。<br />

```java
ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
// dump 所有线程
ThreadInfo[] threadInfos = threadMXBean.dumpAllThreads(false, false);
for (ThreadInfo threadInfo : threadInfos) {
  String msg = String.format("[%s] %s", threadInfo.getThreadId(), threadInfo.getThreadName());
  System.out.println(msg);
}

// 程序输出结果，每个JVM可能运行结果不一致
[5] Monitor Ctrl-Break // Dump 的线程
[4] Signal Dispatcher //  调度系统中的信令分发器,接收操作系统的信号分发给对应的程序，比如kill的信号
[3] Finalizer  // 要用于在垃圾收集前，调用对象的finalize()方法
[2] Reference Handler // 处理对象引用等垃圾处理的线程，线程优先级最高为 10
[1] main  // 主运行程序
```
<a name="mYuC7"></a>
## 为什么要使用多线程?
这是一个老生常谈的问题。首先需要说明的是没有哪一种编程模型是银弹。多线程的编程也仅使用部分场景。不合理的使用多线程甚至可能造成程序性能的降低。但是在一些场景下，多线程的优点也是非常明显的。<br />

- 高效率利用多核心CPU

随着计算机行业的发展，CPU的核心越来愈多，如果一个程序只能单线程运行，势必会降低计算机的使用效率，相反使用多线程，可以合理的利用多线程资源。<br />

- 更快的响应时间

我们经常会使用MQ的编程模型来处理业务逻辑。MQ的异步处理就是典型的多线程使用，比如使用外卖订餐，在订餐的时候，只需要下单支付，消息的发送，库存的扣减以及订单快照等可以放在MQ的线程中处理，提成程序相应时间。<br />

- 更好地编程模型

Java 提供了非常简便的多线程编程模型，在可以使用多线程的情况下，可以很方便的使用，所以在需要使用多线程操作的时候，可以更好的使用多线程。<br />
<br />

