# Volatile的应用和实现

<a name="V7hrm"></a>
## 问题引入

<br />在多线程编程中，如果我们使用下面的格式的代码：

```java
package com.taoes;

public class RunnableDemo implements Runnable {

    private boolean allowableRun = true;

    public void setAllowableRun(boolean allowableRun) {
        this.allowableRun = allowableRun;
    }


    @Override
    public void run() {
        while (allowableRun) {
            // 省略代码
        }
        System.out.println("程序退出");
    }

    static class Main {
        public static void main(String[] args) throws InterruptedException {
            RunnableDemo runnableDemo = new RunnableDemo();
            Thread runThread = new Thread(runnableDemo);
            runThread.start();
            Thread.sleep(1000);
            runnableDemo.setAllowableRun(false);
            System.out.println("allowableRun已被设置为False");
        }
    }
}
```

阅读上面的代码，我们会发现这与我们期待的结果并不相同，我们期待，在main方法调用 ` runnableDemo.setAllowableRun(false);` 后，线程应该输出 '程序退出' ， 但是程序并没有输出该结果，而是仍然在死循环中执行，这时候我们已经将 `allowableRun` 设置为false 但是程序为什么依旧在while循环中运行呢？<br />这个问题的原因就在于 `私有堆栈中的值和公共堆栈的值不一致造成的` , 我们可以通过使用关键字 `volatile` 来修饰 allowableRun 字段，这样就可以保证allowableRun字段在其他线程的修改在当前线程能够读取到，即线程之间的 `可见性`。 所谓的可见性就是 `当一个线程修改一个共享变量时，另外一个线程能够读到这个修改的值`。 在一些使用恰当的场合，使用volatile 相当于synchronized 更高的性能，因为volatile 减少了线程之间上下文切换的损耗。
<a name="33VGt"></a>
## Voliatile 的实现原理
 在讨论这个问题之前，我们先来了解几个术语：

- [x] 内存屏蔽: 一组处理器的指令，用于实现对内存操作的顺序限制
- [x] 缓冲行: CPU高速缓冲器的最小存储单元，处理区在处理缓存的时候会加载整个缓冲行
- [x] 原子操作：不可终端的一个或一个系列的操作
- [x] 缓存行填充：处理器识别到从内存读取到的操作数是可缓存的，处理器会读取整个缓存行到适当的缓存（L1,L2获取其他缓存区）
- [x] 缓存命中: 如果高速缓存行中填充的数据的内存位置仍然是上次处理器放的位置时候，处理器将从缓存中读取数据，而非内存中
- [x] 写命中：类似于缓存命中，当然操作数准备写入内存的识货，处理器会检查缓存行中是否存在有效的缓存行，如果存在，回将操作数缓存到

那么，volatile是如何保证可见性的呢？通过JIT编译成汇编语言 ( `这里不做详细的讨论，如何进行JIT编译，另外也可以通过 hsdis 和 jitwatch实现` ) 后可以看到，在每次写入被volatile 修饰的变量的时候，都会多一个指令，这个指令为lock指令。<br />这也是因为CPU并非和内存直接交互数据，而是之间隔离Cache，数据发生改变，首先写入Cache，然后同步保存到Memory区域，但是这个保存内存区域并不知道是何时保存的。所以lock指令会将修改的数据即刻保存到内存区域，其他CPU通过 ‘嗅探’的方, 从总线传播数据得到自己缓存的数据无效，在对这个数据进行读写的时候，就会从内存中获取新的数据。
> 在一些文章和视频教学中，有部分错误的观点认为在数据修改后，处理器会将更新的数据推送到其他CPU的地方，这是错误的理论。


<a name="lQuIB"></a>
### lock 指令的功能描述
lock指令会实现两个功能:

- 向当前缓存行的数据写入内存数据
- 使其他CPU的缓存的相同内存地址的数据无效

<a name="CU1fz"></a>
### volatile的优化示例
** **<br />LinkedTransferQueue 对象会考虑到缓冲行的长度的问题，进行字节最佳，以提高性能，这是因为部分新的CPU，如酷睿等其高速缓冲器的宽度为64位，如果该队列的数据数据并非占用超过64位，那么CPU会将其放在同一个缓冲行中，每一次的操作都会更新缓冲行，造成性能的问题，所以在在设计这个类的时候，并发编程专家_ Doug Lea  _使用的了追加字节的方式来实现。<br />但是并非所有的使用的volatile都需要补充到64位数据，当CPU的高级缓存区的宽度为32位的时候或者共享变量并不会被频繁的写入都没有必要去追加字节！

