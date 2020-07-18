# 4、常见的JVM 异常演示

<a name="s92W1"></a>
### 4.1 JVM 堆内存溢出展示

```java
public class OutOfMemoryException {

  public static void main(String[] args) {
    List<OutOfMemoryException> bufList = new ArrayList<>();
    for (; ; ) {
      bufList.add(new OutOfMemoryException());
    }
  }
}
```

> 需要注意的是，由于现在计算机配置都非常高，因此直接让计算机内存溢出是一件非常困难的操作，因此这里我们将虚拟机的配置更改下，需要在启动参数中添加 **`**_-Xms1m -Xmx1m -XX:+HeapDumpOnOutOfMemoryError_**` ** 标识初始化内存1M，最大拓展内存1M，并且dump 异常信息当出现OOM的时候


执行代码，等待一段时间之后，控制台出现以下错误信息**并且项目的根目录出现后缀名为： hprof 如:**java_pid92363.hprof 的文件

```
java.lang.OutOfMemoryError: Java heap space
Dumping heap to java_pid92363.hprof ...
Heap dump file created [2571260 bytes in 0.011 secs]
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at java.util.Arrays.copyOf(Arrays.java:3210)
	at java.util.Arrays.copyOf(Arrays.java:3181)
	at java.util.ArrayList.grow(ArrayList.java:265)
	at java.util.ArrayList.ensureExplicitCapacity(ArrayList.java:239)
	at java.util.ArrayList.ensureCapacityInternal(ArrayList.java:231)
	at java.util.ArrayList.add(ArrayList.java:462)
	at com.zhoutao.memory.OutOfMemoryException.main(OutOfMemoryException.java:17)

Process finished with exit code 1
```


我们可以在控制台输入命令  `jvisualvm`   启动 JVisualVM 工具  ,在文件中载入 java_pid92363.hprof 文件，在文件选项卡中载入此hprof文件即可<br />![image.png](https://cdn.nlark.com/yuque/0/2020/png/437981/1584877193189-c91b7565-e3a0-474a-8daf-59c618d50da1.png#align=left&display=inline&height=629&name=image.png&originHeight=1258&originWidth=1210&size=186910&status=done&style=none&width=605)<br />
<br />可以看到报错信息中显示在main线程中出现OOM，**标识OOM的异常信息是在main线程出现的，在类选项卡中，我们可以看到，**OutOfMemoryException 类的实例达到4万之多，导致OOM<br />**![image.png](https://cdn.nlark.com/yuque/0/2020/png/437981/1584877279463-4eae0b6a-8c90-4ddb-bdcd-46dc88d6b311.png#align=left&display=inline&height=288&name=image.png&originHeight=650&originWidth=2734&size=174974&status=done&style=stroke&width=1213)**<br />**<br />**
<a name="pUXrd"></a>
### 4.2 栈内存溢出异常

```java
/**
 * 使用递归的方式测试 JVM 的栈内存溢出
 *
 * <p>设置栈内存大小: -Xss100k
 */
public class StackOverflowException {

  public static void main(String[] args) {
    StackOverExample example = new StackOverExample();
    example.test();
  }
}

class StackOverExample {

  private int length;

  void test() {
    this.length++;
    try {
      test();
    } catch (Throwable throwable) {
      System.out.println("栈深度：" + this.length);
      throwable.printStackTrace();
    }
  }
}
```


输出数据:

```
栈深度：9925
java.lang.StackOverflowError
	at sun.misc.Unsafe.compareAndSwapLong(Native Method)
	at java.util.concurrent.ConcurrentHashMap.addCount(ConcurrentHashMap.java:2259)
	at java.util.concurrent.ConcurrentHashMap.putVal(ConcurrentHashMap.java:1070)
	at java.util.concurrent.ConcurrentHashMap.putIfAbsent(ConcurrentHashMap.java:1535)
	at java.lang.ClassLoader.getClassLoadingLock(ClassLoader.java:463)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:404)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:411)
	at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:349)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:357)
	at com.zhoutao.memory.StackOverExample.test(StackOverflowException.java:28)
	at com.zhoutao.memory.StackOverExample.test(StackOverflowException.java:26)
```

可以看到栈深度达到9925 的时候出现了栈内存溢出异常的错误,使用JVisuaVM 获取线程Dump的数据可以看到一下信息, 当前线程状态为** **TIMED_WAITING ,处于等待休眠状态。

```
"main" #1 prio=5 os_prio=31 tid=0x00007fcce4801800 nid=0x1103 sleeping[0x000070000d0c9000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
        at java.lang.Thread.sleep(Native Method)
        at java.lang.Thread.sleep(Thread.java:340)
        at java.util.concurrent.TimeUnit.sleep(TimeUnit.java:386)
        at com.zhoutao.memory.StackOverExample.test(StackOverflowException.java:27)
        at com.zhoutao.memory.StackOverExample.test(StackOverflowException.java:28)
        at com.zhoutao.memory.StackOverExample.test(StackOverflowException.java:28)
        at com.zhoutao.memory.StackOverExample.test(StackOverflowException.java:28)
        at com.zhoutao.memory.StackOverExample.test(StackOverflowException.java:28)
        at com.zhoutao.memory.StackOverExample.test(StackOverflowException.java:28)
        at com.zhoutao.memory.StackOverExample.test(StackOverflowException.java:28)
        at com.zhoutao.memory.StackOverExample.test(StackOverflowException.java:28)
        at com.zhoutao.memory.StackOverExample.test(StackOverflowException.java:28)
        at com.zhoutao.memory.StackOverExample.test(StackOverflowException.java:28)
        at com.zhoutao.memory.StackOverExample.test(StackOverflowException.java:28)
        at com.zhoutao.memory.StackOverExample.test(StackOverflowException.java:28)
        at com.zhoutao.memory.StackOverExample.test(StackOverflowException.java:28)
        at com.zhoutao.memory.StackOverExample.test(StackOverflowException.java:28)
        at com.zhoutao.memory.StackOverExample.test(StackOverflowException.java:28)
        at com.zhoutao.memory.StackOverExample.test(StackOverflowException.java:28)
        at com.zhoutao.memory.StackOverExample.test(StackOverflowException.java:28)
        at com.zhoutao.memory.StackOverExample.test(StackOverflowException.java:28)
        at com.zhoutao.memory.StackOverExample.test(StackOverflowException.java:28)
        at com.zhoutao.memory.StackOverExample.test(StackOverflowException.java:28)
        at com.zhoutao.memory.StackOverExample.test(StackOverflowException.java:28)
        at com.zhoutao.memory.StackOverExample.test(StackOverflowException.java:28)
        at com.zhoutao.memory.StackOverExample.test(StackOverflowException.java:28)
        at com.zhoutao.memory.StackOverExample.test(StackOverflowException.java:28)
        at com.zhoutao.memory.StackOverExample.test(StackOverflowException.java:28)
        at com.zhoutao.memory.StackOverExample.test(StackOverflowException.java:28)
        at com.zhoutao.memory.StackOverExample.test(StackOverflowException.java:28)

```


<a name="TSGMs"></a>
### 4.3 死锁异常信息

程序分析:

- 线程A 已经获取到对象 lockObject1 的锁，然后延时2秒
- 线程2 已经获取到对象 lockObject2的锁，然后延时1秒
- 由于延时的时间差异，线程B首先执行完成，这时候B尝试获取 lockObject1的锁，但是A线程持有lockObject1的锁，所以B线程在等待获取lockObject1的锁
- 在B线程等待的期间，A线程执行完成，尝试获取对象lockObject2的锁，此时就会出现死锁问题

```java
public class DeadLockException {

  public static void main(String[] args) throws Exception {

    final Object lockObject1 = new String();
    final Object lockObject2 = new StringBuffer();

    new Thread(
            () -> {
              System.out.println("DeadLockException.run");
              synchronized (lockObject1) {
                System.out.println("1 get 1's lock");
                try {
                  TimeUnit.SECONDS.sleep(2);
                } catch (InterruptedException e) {
                  e.printStackTrace();
                }

                synchronized (lockObject2) {
                  System.out.println("1 get 2's lock");
                }
              }
            })
        .start();

    new Thread(
            () -> {
              System.out.println("DeadLockException.run");
              synchronized (lockObject2) {
                System.out.println("2 get 2's lock");
                try {
                  TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                  e.printStackTrace();
                }
                synchronized (lockObject1) {
                  System.out.println("2 get 1's lock");
                }
              }
            })
        .start();
  }
}
```

程序执行结果:

```
DeadLockException.run
1 get 1's lock
DeadLockException.run
2 get 2's lock
```


使用JvisualVM 工具分析可见到 图4.3.1 ,JvisualVM 已经发现了死锁，这里dump线程，可以再次看到：

![image.png](https://cdn.nlark.com/yuque/0/2020/png/437981/1584881598630-f903ec4a-c25a-484b-9f24-c7c438a7b9bf.png#align=left&display=inline&height=256&name=image.png&originHeight=944&originWidth=2750&size=231011&status=done&style=stroke&width=746)<br />图 4.3.1 <br />
<br />
<br />
<br />![image.png](https://cdn.nlark.com/yuque/0/2020/png/437981/1584881774460-83d39c1b-54ef-43d6-8693-e1354c7379e2.png#align=left&display=inline&height=217&name=image.png&originHeight=752&originWidth=2584&size=244327&status=done&style=stroke&width=746)<br />
<br />图 4.3.2 线程Dump 分析<br />
<br />
<br />
<br />
<br />![image.png](https://cdn.nlark.com/yuque/0/2020/png/437981/1584882200031-bd8e9577-a373-4bc7-a80f-bf9027577738.png#align=left&display=inline&height=530&name=image.png&originHeight=1060&originWidth=986&size=104937&status=done&style=none&width=493)<br />
<br />图 4.3.3 使用JProfile 查看是否存在死锁<br />
<br />
<br />

<a name="nvs3X"></a>
### 4.4 Metaspace 元空间内存溢出


从Java1.8开始，永久代的概念被移除，取而代之的是在本地内存中的元空间(Metaspace) ，默认大小20.75M，可以通过 -XX:MaxMatespaceSize=1000m,来修改元空间大小，可以通过使用Cglib 不同的创建类来实现元空间的方法溢出。通过需要修改VM 参数 -XX:MaxMetaspaceSize=2M 

```java
public class JavaMethodAreaOOM {

	static class OOMObject{
	}
	
	public static void main(String[] args) {
		while (true) {
			Enhancer enhancer = new Enhancer();
			enhancer.setSuperclass(OOMObject.class);
			enhancer.setUseCache(false);
			enhancer.setCallback(new MethodInterceptor() {
				@Override
				public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
					return proxy.invoke(obj, args);
				}
			});
			enhancer.create();
		}
	}
}
```


> 输出结果:
> Exception in thread “main” java.lang.OutOfMemoryError: Metaspace
> at net.sf.cglib.core.AbstractClassGenerator.create(AbstractClassGenerator.java:237)

> at net.sf.cglib.proxy.Enhancer.createHelper(Enhancer.java:377)

> at net.sf.cglib.proxy.Enhancer.create(Enhancer.java:285)

> at com.seeyon.oom.JavaMethodAreaOOM.main(JavaMethodAreaOOM.java:42)



<a name="KvzYN"></a>
### 4.5 其他监控工具

<a name="btQSB"></a>
#### Jconsole

![image.png](https://cdn.nlark.com/yuque/0/2020/png/437981/1584879904388-fcd430a9-b7ba-4766-bfb8-d10d263f2ef9.png#align=left&display=inline&height=750&name=image.png&originHeight=1500&originWidth=1800&size=297148&status=done&style=none&width=900)

<a name="P3r8G"></a>
#### Jprofile
![image.png](https://cdn.nlark.com/yuque/0/2020/png/437981/1584879954948-b1cef266-5fd4-4c84-886b-c48dbbfd422a.png#align=left&display=inline&height=989&name=image.png&originHeight=1978&originWidth=3360&size=467590&status=done&style=none&width=1680)<br />

