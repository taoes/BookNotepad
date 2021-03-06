# 第1章 走进Java


## JVM 中的 Exact Memory  Management


### 支持的VM

- [x] Exact VM
- [x] HotSpot VM

## JVM 的 HotSpot VM


### 热点探测技术

热点探测技术的目的是寻找到最有价值的代码，通知JIT以方法的单位编译，如果一个方法或者代码块被触发多次(超过一定的阈值)，将会分别触发标准编译以及OSR(栈上替换)编译动作, 通过编译器和解释器恰当的协同工作，在程序响应时间和代码执行性能中找到平衡点，无需本地代码输出才能执行程序，JIT即时编译器的压力也相对较小.


#### 实现方式

实现热点探测技术有两种方式:

- 基于采样的探测方式

> VM会周期性的检测各个线程的栈顶，若某个方法经常出现在栈顶，那么VM 会认为其是一个'热点方法'. 这种实现方法简单，但是不够精准，容器收到线程阻塞等问题的干扰，提高采样率会造成程序执行的性能问题.


- 基于方法的计数器的探测方式

> VM会给每一个方法甚至代码块创建一个计数器，统计其访问的次数，超过一定的阈值, 就会被认为是热点方法，即HotSpot


#### HotSpot VM 的方案

HotSpot VM 使用的是基于计数器的方式，其内存在两种计数器: 方法调用计数器以及汇编计数器。

- 方法调动计数器
> 主要用户计算方法的调用次数，在Client模式下为1500次，在Service下为10000次。方法计数器呈现衰减的特性，在一段时间内调用次数减少，方法计数器就会减少.

- 汇编计数器
> 主要用于统计循环体内的代码块的调用次数

