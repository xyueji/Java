# 垃圾回收机制
垃圾回收包含的内容不少，但顺着下面的顺序捋清知识也并不难。首先要

搞清垃圾回收的范围（栈需要GC去回收吗？），然后就是回收的前提条件

如何判断一个对象已经可以被回收（这里只重点学习根搜索算法就行了），

之后便是建立在根搜索基础上的三种回收策略，最后便是JVM中对这三种

策略的具体实现。

### 1.范围：要回收哪些区域？
Java方法栈、本地方法栈以及PC计数器随方法或线程的结束而自然被回收，

所以这些区域不需要考虑回收问题。Java堆和方法区是GC回收的重点区域，

因为一个接口的多个实现类需要的内存不一样，一个方法的多个分支需要

的内存可能也不一样，而这两个区域又对立于栈可能随时都会有对象不再

被引用，因此这部分内存的分配和回收都是动态的。
### 2.前提：如何判断对象已死？
1. 引用计数法
引用计数法就是通过一个计数器记录该对象被引用的次数，方法简单高效，

但是解决不了循环引用的问题。比如对象A包含指向对象B的引用，对象B

也包含指向对象A的引用，但没有引用指向A和B，这时当前回收如果采用的

是引用计数法，那么对象A和B的被引用次数都为1，都不会被回收。<br>



下面是循环引用的例子，在Hotspot JVM下可以被正常回收，可以证实JVM

采用的不是简单的引用计数法。通过-XX:+PrintGCDetails输出GC日志。<br>
```java
package com.cdai.jvm.gc;  
public class ReferenceCount {  
    final static int MB = 1024 * 1024;  
    byte[] size = new byte[2 * MB];  
    Object ref;  
	
    public static void main(String[] args) {  
        ReferenceCount objA = new ReferenceCount();  
        ReferenceCount objB = new ReferenceCount();  

        objA.ref = objB;  
        objB.ref = objA;  
         
        objA = null;  
        objB = null;  

        System.gc();  
        System.gc();  
    }
}  
//[Full GC (System) [Tenured: 2048K->366K(10944K), 0.0046272 secs] 4604K->366K(15872K), [Perm : 154K->154K(12288K)], 0.0046751 secs] [Times: user=0.02 sys=0.00, real=0.00 secs]
``` 
2. 根搜索
通过选取一些根对象作为起始点，开始向下搜索，如果一个对象到根对象

不可达时，则说明此对象已经没有被引用，是可以被回收的。可以作为根的

对象有：栈中变量引用的对象，类静态属性引用的对象，常量引用的对象等。

因为每个线程都有一个栈，所以我们需要选取多个根对象。<br>
在根搜索中得到的不可达对象并不是立即就被标记成可回收的，而是先进行一次

标记放入F-Queue等待执行对象的finalize()方法，执行后GC将进行二次标记，复活

的对象之后将不会被回收。因此，使对象复活的唯一办法就是重写finalize()方法，

并使对象重新被引用。
```java
package com.cdai.jvm.gc;  
public class DeadToRebirth {  
    private static DeadToRebirth hook;   
    @Override  

    public void finalize() throws Throwable {  
        super.finalize();  
        DeadToRebirth.hook = this;  
    }  
    public static void main(String[] args) throws Exception {  
        DeadToRebirth.hook = new DeadToRebirth();  
        DeadToRebirth.hook = null;  
        System.gc();  
        Thread.sleep(500);  
        if (DeadToRebirth.hook != null)  
            System.out.println("Rebirth!");  
        else  
            System.out.println("Dead!"); 
        DeadToRebirth.hook = null;  
        System.gc();  
        Thread.sleep(500);  
        if (DeadToRebirth.hook != null)  
            System.out.println("Rebirth!");  
        else  
            System.out.println("Dead!");  
    }
}  
/**
要注意的两点是：
第一，finalize()方法只会被执行一次，所以对象只有一次复活的机会。
第二，执行GC后，要停顿半秒等待优先级很低的finalize()执行完毕。
**/
```
### 策略：垃圾回收的算法
1. 标记-清除
没错，这里的标记指的就是之前我们介绍过的两次标记过程。标记完成后就可以

对标记为垃圾的对象进行回收了。怎么样，简单吧。但是这种策略的缺点很明显，

回收后内存碎片很多，如果之后程序运行时申请大内存，可能会又导致一次GC。

虽然缺点明显，这种策略却是后两种策略的基础。正因为它的缺点，所以促成了

后两种策略的产生。
2. 标记-复制
将内存分为两块，标记完成开始回收时，将一块内存中保留的对象全部复制到另

一块空闲内存中。实现起来也很简单，当大部分对象都被回收时这种策略也很高效。

但这种策略也有缺点，可用内存变为一半了！<br>
怎样解决呢？聪明的程序员们总是办法多过问题的。可以将堆不按1:1的比例分离，

而是按8:1:1分成一块Eden和两小块Survivor区，每次将Eden和Survivor中存活的对象

复制到另一块空闲的Survivor中。这三块区域并不是堆的全部，而是构成了新生代。<br>
在GC开始的时候，对象只会存在于Eden区和名为“From”的Survivor区，Survivor区“To”是空的。紧接着进行GC，Eden区中所有存活的对象都会被复制到“To”，而在“From”区中，仍存活的对象会根据他们的年龄值来决定去向。年龄达到一定值(年龄阈值，可以通过-XX:MaxTenuringThreshold来设置)的对象会被移动到年老代中，没有达到阈值的对象会被复制到“To”区域。经过这次GC后，Eden区和From区已经被清空。这个时候，“From”和“To”会交换他们的角色，也就是新的“To”就是上次GC前的“From”，新的“From”就是上次GC前的“To”。不管怎样，都会保证名为To的Survivor区域是空的。Minor GC会一直重复这样的过程，直到“To”区被填满，“To”区被填满之后，会将所有对象移动到年老代中。<br>
<img src="img/young_gc.png">
* 一个对象的这一辈子
我是一个普通的Java对象，我出生在Eden区，在Eden区我还看到和我长的很像的小兄弟，我们在Eden区中玩了挺长时间。有一天Eden区中的人实在是太多了，我就被迫去了Survivor区的“From”区，自从去了Survivor区，我就开始漂了，有时候在Survivor的“From”区，有时候在Survivor的“To”区，居无定所。直到我18岁的时候，爸爸说我成人了，该去社会上闯闯了。于是我就去了年老代那边，年老代里，人很多，并且年龄都挺大的，我在这里也认识了很多人。在年老代里，我生活了20年(每次GC加一岁)，然后被回收。<br>
* 有关年轻代的JVM参数
1)-XX:NewSize和-XX:MaxNewSize<br>

用于设置年轻代的大小，建议设为整个堆大小的1/3或者1/4,两个值设为一样大。<br>

2)-XX:SurvivorRatio<br>

用于设置Eden和其中一个Survivor的比值，这个值也比较重要。<br>

3)-XX:+PrintTenuringDistribution<br>

这个参数用于显示每次Minor GC时Survivor区中各个年龄段的对象的大小。<br>

4).-XX:InitialTenuringThreshol和-XX:MaxTenuringThreshold<br>

用于设置晋升到老年代的对象年龄的最小值和最大值，每个对象在坚持过一次Minor GC之后，年龄就加1。<br>

3. 标记-整理
标记整理算法的“标记”过程和标记-清除算法一致，只是后面并不是直接对可回收对象进行整理，而是让所有存活的对象都向一段移动，然后直接清理掉端边界意外的内存。
4. 实现：虚拟机中的收集器
（1）新生代上的GC实现<br>

Serial：单线程的收集器，只使用一个线程进行收集，并且收集时会暂停其他所有

工作线程（Stop the world）。它是Client模式下的默认新生代收集器。<br>

ParNew：Serial收集器的多线程版本。在单CPU甚至两个CPU的环境下，由于线程

交互的开销，无法保证性能超越Serial收集器。<br>

Parallel Scavenge：也是多线程收集器，与ParNew的区别是，它是吞吐量优先

收集器。吞吐量=运行用户代码时间/(运行用户代码+垃圾收集时间)。另一点区别

是配置-XX:+UseAdaptiveSizePolicy后，虚拟机会自动调整Eden/Survivor等参数来

提供用户所需的吞吐量。我们需要配置的就是内存大小-Xmx和吞吐量GCTimeRatio。<br>
（2）老年代上的GC实现

Serial Old：Serial收集器的老年代版本。<br>

Parallel Old：Parallel Scavenge的老年代版本。此前，如果新生代采用PS GC的话，<br>

老年代只有Serial Old能与之配合。现在有了Parallel Old与之配合，可以在注重吞吐量

及CPU资源敏感的场合使用了。<br>

CMS：采用的是标记-清除而非标记-整理，是一款并发低停顿的收集器。但是由于

采用标记-清除，内存碎片问题不可避免。可以使用-XX:CMSFullGCsBeforeCompaction

设置执行几次CMS回收后，跟着来一次内存碎片整理。<br>

5. 触发：何时开始GC？
Minor GC（新生代回收）的触发条件比较简单，Eden空间不足就开始进行Minor GC

回收新生代。<br>
而Full GC（老年代回收，一般伴随一次Minor GC）则有几种触发条件：

* 老年代空间不足

* PermSpace空间不足

* 统计得到的Minor GC晋升到老年代的平均大小大于老年代的剩余空间

这里注意一点：PermSpace并不等同于方法区，只不过是Hotspot JVM用PermSpace来

实现方法区而已，有些虚拟机没有PermSpace而用其他机制来实现方法区。

6. 补充：对象的空间分配和晋升

* 对象优先在Eden上分配

* 大对象直接进入老年代

虚拟机提供了-XX:PretenureSizeThreshold参数，大于这个参数值的对象将直接分配到

老年代中。因为新生代采用的是标记-复制策略，在Eden中分配大对象将会导致Eden区

和两个Survivor区之间大量的内存拷贝。

* 长期存活的对象将进入老年代

对象在Survivor区中每熬过一次Minor GC，年龄就增加1岁，当它的年龄增加到一定程度

（默认为15岁）时，就会晋升到老年代中。