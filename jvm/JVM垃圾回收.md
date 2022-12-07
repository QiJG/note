# JVM垃圾回收

## 1. 垃圾回收概述

### 1.1 什么是垃圾

垃圾指的是在运行程序中没有任何指针（或引用）指向的翠香，这个对象就是需要回收的垃圾。

垃圾回收除了释放没有用的对象，也会清除内存里的碎片，碎片整理将占用的堆内存移动到堆的一端，以便JVM整理出内存分配给新的对象。

### 1.2 什么是GC

垃圾回收（Garbage Collection）是一门实用又重要的基数。

思考GC需要完成的事情

1. 哪些内存需要进行回收
2. 什么时候对这些内存进行回收
3. 如何进行回收

GC 主要是处理Java堆Heap

### 1.3 STW

Stop-the-world，简称STW，指的是GC事件发生过程中，会产生应用程序的停顿，由于JVM要执行GC而停止了应用程序（工作线程/用户线程）的执行，除了GC所需的线程外，所有的线程都处于等待状态直到GC任务完成

![image-20221011145015664](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221011145015664.png)

GC优化很多时候就是减少Stop-the-world发生的时间，从而使系统具有高吞吐，低停顿的状态

### 1.4 并行与并发

**并发（Concurrent）**

指的是同一时间段内有几个程序同时处于运行当中，且这几个程序都在同一处理器上运行

**并行（Parallel）**

当系统有一个以上CPU时，当一个CPU执行一个进程时，另外一个CPU可以执行别的进程，其实决定并行的因素并不是CPU的数量，

而是CPU的核心数量，比如一个CPU多个核可以并行

并发：指的是多个事情，在同一时间段内同时发生了

并行：指的是多个事情，在同一时间点上同时发生了

### 1.5 System.gc()

System.gc()或者Runtime.getRuntime().gc()会显示触发Full GC，同时对老年代和新生代进行回收，但是无法保证对垃圾收集器的调用

```java
public class SystemGCTest {
    protected void finalize() throws Throwable{
        super.finalize();
        System.out.println("finalize invoke");
    }

    public static void main(String[] args) {
        new SystemGCTest();
        System.gc();// 该方法并不会同步调用垃圾收集器

    }
}
```

## 1.6安全点与安全区域

**安全点（safePoint）**

程序执行GC并非在所有的地方都能停顿下来开始执行，只有在特定的位置才能停顿下来执行GC，这些位置成为安全点。

安全点过少：可能会导致GC等待时间太长

安全点过多：可能多会导致运行时的性能问题

通常根据“是否具有让程序长时间执行的特征为标准” 选择 方法调用 循环跳转和异常跳转等点

当发生GC时如何停顿：

抢先式中断：（目前没有虚拟机采用）

首先中断所有线程，如果还有线程不在安全点，就恢复线程，让线程跑到安全点

主动式中断：

设置一个中断标志，各个线程运行到安全点后主动轮训这些标志，如果为真，则停止

**安全区域（safe region）**
安全区域是在一段代码片段中，对象的引用关系不会发生变化，在这个区域都是可以进行GC的

### 1.7 GC的分类

![image-20221011152119765](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221011152119765.png)

针对HotSpot VM实现，它里面的GC分类

* Partial GC：并不收集整个GC堆，其中又分为

  * 新生代回收（Minor GC/Young GC）只收集新生代的GC

  * 老年代的回收（Major GC/Old GC）只收集老年代的GC

  * Mixed GC：收集整个young gen以及部分old gen的GC

    只有G1有这个模式

* Full GC：收集整个堆，包括 young gen、old gen、perm gen（如果存在的话）

频繁收集新生代

较少收集老年代

基本不动方法区

### 1.8 GC的触发条件

* 年轻代(Minor GC)触发条件

  一般情况下，所有新对象的生成都放在新生代中，新生代按照内存8:1:1分为 eden和两个survior（S0和S1）

  * 首次GC 当eden区满的话，会将eden存活对象拷贝到from survivor，然后清空eden区
  * 二次GC 将eden区和from survivor区存活对象拷贝 to survivor，然后清空eden和from survivor区
  * 三次GC 将eden区和to survivor区存活对象拷贝到from survivor区，然后清空enden和to survivor区
  * 一般新生代中对象的存活周期超过**15次**GC后会移动到老年代

  ![image-20221011154017329](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221011154017329.png)

* 老年代（Major GC）触发条件

  * 由Eden 和 from survivor区向 to survivor区复制时，对象的大小大于to survivor可用内存，则把对象转存到老年代，会尝试触发Minor GC，如果空间还是不足，则会触发Major GC
  * 如果Major GC后还是不足，则会OOM
  * 如果发生Major GC 通常会伴随Minor GC但这并不是绝对的。

* Full GC 触发条件

  * System.gc()方法的调用
  * 老年代空间不足
  * 方法区空间不足

## 2. 检测垃圾

#### 2.1 引用计数算法

通过判断对象的引用数量来决定对象是否可以被回收，但是该方法无法解决循环引用的方式

![image-20221011155156431](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221011155156431.png)

```java
package com.custom.check;

/**
 * -Xms10m -Xmx10m -XX:+PrintGCDetails
 *
 * 这个方法用来测试默认垃圾检测算法时 可达性分析算法
 */
public class ReferenceCountGC {
    public Object instance = null;
    private byte[] bigObject = new byte[1024*1024];


    public static void main(String[] args){
        ReferenceCountGC objA = new ReferenceCountGC();
        ReferenceCountGC objB = new ReferenceCountGC();
        objA.instance = objB;
        objB.instance = objA;

        objA = null;
        objB = null;

//        System.gc();// 打开垃圾回收
    }

    /**
     * 未打开gc 时
     * Heap
     *  PSYoungGen      total 2560K, used 1824K [0x00000000ffd00000, 0x0000000100000000, 0x0000000100000000)
     *   eden space 2048K, 89% used [0x00000000ffd00000,0x00000000ffec8030,0x00000000fff00000)
     *   from space 512K, 0% used [0x00000000fff80000,0x00000000fff80000,0x0000000100000000)
     *   to   space 512K, 0% used [0x00000000fff00000,0x00000000fff00000,0x00000000fff80000)
     *  ParOldGen       total 7168K, used 2048K [0x00000000ff600000, 0x00000000ffd00000, 0x00000000ffd00000)
     *   object space 7168K, 28% used [0x00000000ff600000,0x00000000ff800020,0x00000000ffd00000)
     *  Metaspace       used 3154K, capacity 4496K, committed 4864K, reserved 1056768K
     *   class space    used 331K, capacity 388K, committed 512K, reserved 1048576K
     *
     * Process finished with exit code 0
     *
     * 打开gc时
     * [GC (System.gc()) [PSYoungGen: 1780K->488K(2560K)] 3828K->2780K(9728K), 0.0014836 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
     * [Full GC (System.gc()) [PSYoungGen: 488K->0K(2560K)] [ParOldGen: 2292K->661K(7168K)] 2780K->661K(9728K), [Metaspace: 3181K->3181K(1056768K)], 0.0059533 secs] [Times: user=0.01 sys=0.00, real=0.01 secs]
     * Heap
     *  PSYoungGen      total 2560K, used 42K [0x00000000ffd00000, 0x0000000100000000, 0x0000000100000000)
     *   eden space 2048K, 2% used [0x00000000ffd00000,0x00000000ffd0a938,0x00000000fff00000)
     *   from space 512K, 0% used [0x00000000fff00000,0x00000000fff00000,0x00000000fff80000)
     *   to   space 512K, 0% used [0x00000000fff80000,0x00000000fff80000,0x0000000100000000)
     *  ParOldGen       total 7168K, used 661K [0x00000000ff600000, 0x00000000ffd00000, 0x00000000ffd00000)
     *   object space 7168K, 9% used [0x00000000ff600000,0x00000000ff6a5680,0x00000000ffd00000)
     *  Metaspace       used 3206K, capacity 4496K, committed 4864K, reserved 1056768K
     *   class space    used 338K, capacity 388K, committed 512K, reserved 1048576K
     */
}

```

**优点**

* 实现简单，垃圾对象便于标记
* 判定效率高，回收没有延迟性

**缺点**

* 需要单独的字段存储计数器，增加了存储空间的开销
* 每次赋值都要更新计数器，增加了时间的开销
* **致命缺陷，无法处理循环引用的情况**

#### 2.2 可达性分析算法

可达性分析算法时通过判断对象的引用链是否科大来决定对象是否可以被回收。

![image-20221011160321884](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221011160321884.png)

GC Root的对象包括以下几种

1. 虚拟机栈中引用的对象（方法中参数，局部变量）
2. 方法区中静态属性引用的静态变量
3. 方法区中常量引用的对象
4. 本地方法栈中Native方法引用的对象
5. 所有被同步锁synchronized持有的对象
6. java虚拟机内部对象（class对象，异常对象）

**如果一个指针保存了堆内存的对象地址，但是又不在堆内存中**

**对象的finalization机制**

finalize方法允许在子类中被重写，用于对象回收时进行资源释放，但是永远不要主动调用该方法

* finalize方法可能导致对象复活
* finalize方法执行时间没有保证
* 一个糟糕的finalize方法会严重影响GC性能

虚拟机中对象可能处于三种状态

* 可触及的：从根节点开始，可以达到这个对象
* 可复活的：对象的所有引用被释放，但对象可能在finalize方法复活
* 不可触及的：finalize方法被调用，并且没有复活，那么就会进入不可触及的状态，不可触及的对象不可能被复活，因为finalize方法只会调用一次

![image-20221011162436252](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221011162436252.png)

测试代码：

```java
package com.custom.check;

public class CanReliveObj {
    public static CanReliveObj obj;
    protected void finalize() throws  Throwable{
        super.finalize();
        System.out.println("invoke finalize method");
        obj = this; // 让对象状态变为可复活的状态
    }

    /*
        first gc
        invoke finalize method
        obj is still alive
        second gc
        obj is dead
     */
    public static void main(String[] args) {
        try{
            obj = new CanReliveObj();
            obj = null;
            System.gc();
            System.out.println("first gc");
            Thread.sleep(2000);
            if(obj == null){
                System.out.println("obj is dead");
            } else {
                System.out.println("obj is still alive");
            }
            System.out.println("second gc");
            obj = null;
            System.gc();
            Thread.sleep(2000);
            if(obj == null){
                System.out.println("obj is dead");
            } else {
                System.out.println("obj is still alive");
            }
        }catch (InterruptedException e){

        }
    }
}
```

可以通过 MAT 查看 GC Root对象

> https://www.eclipse.org/mat/downloads.php

## 3. 垃圾收集算法

### 3.1 标记清除算法

标记清除算法分为标记和清除两个阶段

标记：GC从引用根节点开始遍历，标记所有引用的对象，一般是在对象的Header中记录

清除：GC从堆内存中从头到尾进行线性遍历，如果发现某个对象没有被标记则将其回收

![image-20221011165536217](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221011165536217.png)

缺点：

* 效率问题：标记和清除两个过程的效率都不高
* 在进行GC时需要停止整个应用
* 空间的问题，会产生大量的空间碎片

### 3.2 复制算法

复制算法将可用内存按容量划分为大小相等的两块i，每次只使用其中的一块。当这一块的内存用完后就将存活的对象复制到另一块上边，然后再把已使用过得内存空间一次清理掉

优点：

* 没有标记和清除过程，实现简单，运行高效
* 复制过去后保证空间的连续性

缺点

* 需要两倍的内存空间
* 如果对象的存活率很高，复制这已工作所花费的时间不可忽视

### 3.3 标记整理算法

标记整理算法的标记过程类似标记清除算法，但是后续步骤不是直接对可回收对象进行清理，而是将所有存活的对象都向一端移动，然后直接清理掉端边界意外的内存（适用于对象存活率高的老年代）

![image-20221011171925556](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221011171925556.png)

优点：

* 不存在内存碎片
* 消除了复制算法中内存两倍内存的弊端

缺点：

* 从效率上来说 标记整理算法要低于复制算法
* 移动对象的同时，如果对象被其他对象引用还要调整引用地址
* STW

### 3.4 分代收集算法

分代收集算法基于 不同的对象生命周期是不一样的，因此不同的生命周期对象采用不同的生命周期，一般将java堆分为新生代和老年代，这样可以根据各个年代的特点使用不同的回收算法

#### 3.4.1 对象分配过程：

![image-20221011173625740](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221011173625740.png)

总结：

* 针对幸存者S0、S1的总结，复制之后有交换，谁空谁是To
* 关于垃圾回收：频繁在新生代，很少在老年代，几乎不在永久代/元空间收集

#### 3.4.2 特殊对象创建过程

![image-20221011173953330](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221011173953330.png)

#### 3.4.3 各分代特点

**1.新生代**

区域相对老年代较小，对象生命周期短，存活率第，回收频繁。

这种情况比较适合复制算法

**2.老年代**

区域较大，对象的生命周期长，存活率高，回收不及新生代频繁。

这种情况比较适合标记清除/标记整理

### 3.5 增量式垃圾回收

增量式垃圾回收的算法仍然是传统的 标记清除/复制算法，允许垃圾收集线程以分阶段的方式完成，标记、清理、复制工作

缺点：由于在垃圾回收过程中间歇执行应用程序代码，所以减少系统卡顿时间，但是会造成系统吞吐量下降

### 3.6 分区算法

分区算法将整个堆空间划分成连续的不同小区间，每个小区间都独立使用，独立回收**

![image-20221011174908021](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221011174908021.png)

## 4. 垃圾收集器

### 4.1 垃圾收集器的分类

* 按线程分： 串行 并行
* 按模式分：独占式 并发式
* 按碎片处理方式：压缩式垃圾回收机 非压缩式垃圾回收器
* 按工作的内存空间分：新生代垃圾回收器 ，老年代垃圾回收器

### 4.2 评估GC性能的指标

* 吞吐量：运行用户代码的时间占总运行时间的比例
* 暂停时间：执行垃圾收集时，程序的工作线程被暂停的时间
* 内存占用：java堆区所占的内存大小

### 4.3 经典垃圾回收器

串行回收器：Serial 、Serial Old

并行回收器：ParNew，Prallel Scavenge，Parallel Old

并发回收器：CMS，G1

### 4.4 垃圾回收器的组合关系

![image-20221011181402879](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221011181402879.png)

如何查看默认的垃圾回收器

* -XX:+PrintCommandLineFlags：查看命令行相关参数
* 使用命令行指令：jinfo -flag 相关的垃圾回收参数 进程ID

```java
package com.custom.check;

import java.util.ArrayList;
import java.util.List;

/**
 * 用于查看默认的垃圾收集器
 * -XX:+PrintCommandLineFlags -XX:+UseConcMarkSweepGC
 * jinfo -flag 相关垃圾回收器参数 进程ID
 *
 */
public class GCUseTest {
    public static void main(String[] args) {
        List<byte[]> list = new ArrayList<>();
        while (true){
            byte[] bytes = new byte[100];
            list.add(bytes);
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

```



### 4.5 Serial(复制算法)

![image-20221011182812166](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221011182812166.png)

优点：简单高效

缺点：会给用户带来停顿

可以使用 -XX:+UseSerialGC 参数指定 新生代和老年代都是用串行收集器

### 4.6 Serial Old(标记整理算法)

老年代的单线程收集器，可以作为CMS收集器的后备预案

### 4.7 ParNew(复制算法)

新生代的并行收集器，ParNew收集器是Serial收集器的多线程版本

![image-20221011183256597](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221011183256597.png)

使用场景：

多核服务器，与CMS收集器搭配使用，除Serial外 目前只有ParNewGC能与CMS收集器配合工作

参数：

使用： -XX:+UseConcMarkSweepGC选择使用CMS作为老年代的收集器时，新生代的默认收集器就是ParNew

也可以使用 -XX:+UseParNewGC 指定使用 ParNew 作为新生代收集器

### 4.8 Parallel Scavenge(复制算法)

Parallel Scavenge 是基于并行，同样采取了并行算法，但是和ParNew收集器不同，Parallel Scavenge的目标是达到一个可控制的吞吐量，因此它也被成为吞吐量优先的垃圾收集器

![image-20221011183935240](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221011183935240.png)

参数配置：

-XX:+UseParallelGC 来选择Parallel Scavenge作为新生代收集器

-XX:+UseParallelOldGC 手动指定老年代都是使用并行回收收集器

* 默认JDK8 是开启的
* 上面两个参数，开启一个 另外一个也会开启

### 4.9 Parallel Old(标记整理算法)

老年代并行收集器

![image-20221011185412862](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221011185412862.png)

### 4.10 CMS(Concurrent Mark Sweep) 收集器(标记清除算法)

CMS 第一次实现了垃圾收集线程与用户线程同时工作，他是一种并发收集器，采用的是标记清除算法

#### 4.10.1 收集过程

1. 初始标记：在这个阶段中，程序中所有的用户线程都会因为STW机制出现短暂的暂停，主要任务是标记以下 GC Roots能直接关联的对象，速度非常快
2. 并发标记：从GC Roots的直接关联对象开始遍历整个对象图的过程中，标记出全部的垃圾对象，这个过程耗时长，但是不需要暂停用户线程，可以垃圾收集线程与用户线程并发执行
3. 重新标记：修正并发标记期间，因用户线程继续运作而导致标记产生变动的哪一部分对象的标记记录，这个阶段的停顿时间比初始标记稍长
4. 并发清除：清理删除标记阶段判断已经死亡的对象
5. ![image-20221011190943763](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221011190943763.png)

由于耗时最长的并发标记与并发清除阶段 都不需要暂停工作，所以整体的回收是低停顿的

由于垃圾收集阶段，用户线程没有中断，因此CMS回收过程中还要保证用户线程有足够的内存空间，

当堆内存使用率达到某一阈值时便开始进行回收，如果内存未满足需要则会启动 Serial Old收集器重新进行老年代收集

#### 4.10.2 采用空闲列表分配内存

#### 4.10.3 特性

优点：

* 并发收集
* 低停顿

缺点

* CMS收集器对CPU资源非常敏感
* CMS收集器无法处理浮动垃圾
* 产生内存碎片

#### 4.10.4 参数设置

-XX:+UseConcMarkSweepGC 手动执行CMS收集器执行内存回收任务，默认新生代会采用 ParNew

### 4.11 G1(Garbage First)收集器(区域化分代式)

G1收集器 基于标记-整理算法实现，也就是说不会产生内存碎片，G1垃圾收集器不同于之间的收集器的一个重要特点是：G1回收的范围是整个Java堆（包括新生代，老年代），而前六种收集器回收的范围仅限于新生代或老年代。

#### 4.11.1 为什么叫G1呢

因为G1是一个并行回收器，他用堆分割为很多不想管的(Region)，使用不同的Region来表示Eden survivor,old等

G1有计划的避免在整个java堆中进行全区域垃圾回收。G1跟踪各个Region里面的垃圾堆积的价值大小，在后台维护一个优先列表，每次根据允许的收集时间，优先回收价值最大的region，所以叫G1

![image-20221011194428649](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221011194428649.png)

#### 4.11.2 G1收集器的特点

1. 并行与并发
2. 分代收集
3. 可预测的停顿时间模型

#### 4.11.3 收集过程

1. 初始标记：标记出GC Roots直接关联的对象，这个阶段速度比较快，需要停止用户线程，单线程执行
2. 并发标记：从GC Root开始对堆中对象进行可达性分析，找出存活对象，这个阶段耗时较长，但是可以和用户线程并发执行
3. 最终标记：修正并发标记阶段由于用户程序执行而产生变动的标记记录
4. 筛选回收：筛选回收阶段会对各个Region的回收价值和成本进行排序，根据用户所希望的GC停顿时间来指定回收计划，这里为了提高效率，并没有采用用户线程和回收线程并发的操作

![image-20221011195026808](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221011195026808.png)

年轻代GC

 应⽤程序分配内存，当年轻代的Eden区⽤尽时开始年轻代回收过程；G1的年轻代收集阶段是⼀个

**并⾏的独占式收集器**。在年轻代回收期，G1 GC暂停所有的应⽤程序线程，启动多线程执⾏年轻代回

收。然后从年轻代区间移动存活对象到Survivor区间或者⽼年代区间，也有可能是两个区间都会涉及。

**⽼年代并发标记(Concurrent Marking)**

 当堆内存使⽤达到⼀定值(默认是45%)时，开始⽼年代并发标记过程。

**混合回收(Mixed GC)**

 标记完成⻢上开始混合回收过程。对于⼀个混合回收期，G1 GC从⽼年代移动存活对象到空闲区

间，这些空闲区间也就成为了⽼年代的⼀部分。和年轻代不同，⽼年代的G1回收器和其他GC不同，G1

的⽼年代回收器不需要整个⽼年代被回收，⼀次主要扫描/回收⼀⼩部分⽼年代Region就可以了。同

时，这个⽼年代Region是和年轻代⼀起被回收的。

 举个示例：⼀个Web服务器，java进程最⼤堆内存为4G，每分钟响应1500个请求，每45秒钟会新

分配⼤约2G的内存。G1会每45秒钟进⾏⼀次年轻代回收，每31个⼩时整个堆的使⽤率会达到45%。会

开始⽼年代并发标记过程，标记完成后开始四到五次的混合回收。