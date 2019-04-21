[TOC]



### 总结：

模块一：运行时数据区域

模块二：一个类从出生到死亡经历了什么

类加载机制->对象的创建->对象的访问->分代回收算法->对象"死亡"的条件

模块三：垃圾回收器

模块四：双亲委派模型



# 一、运行时数据区域

运行时数据区域：（p39）

![img](https://img-blog.csdnimg.cn/20190112204919410.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

1、程序计数器（线程隔离）：当前线程所执行的字节码的行号指示器，通过改变计数器的值来确定下一条指令，比如循环，分支，跳转，异常处理，线程恢复等都是依赖计数器来完成。

2、栈区：

栈分为java虚拟机栈和本地方法栈

java虚拟机栈（线程隔离）：每个方法在运行时都会创建一个栈帧，栈帧存储局部变量表、操作数栈、动态链接、方法出口等信息。其中最关键的局部变量表（我们通常所说的栈）存着编译期可知的各种基本数据类型（boolean，int）、对象引用等。

本地方法栈（线程隔离）：这部分主要与虚拟机用到的 Native 方法相关，一般情况下， Java 应用程序员并不需要关心这部分的内容。

3、堆区（线程共享）：唯一的目的就是存放对象实例。

java堆是gc的主要区域，通常情况下分为新生代和老年代。更细致分为：Eden、From Survivor、To Survivor空间。

线程共享的java堆中可能划分出多个线程隔离的分配缓冲区

4、方法区（线程共享）：用于存放已被虚拟机加载的类信息，常量（常量池中），静态变量等数据。

被Java虚拟机描述为堆的一个逻辑部分。也被称为“永生代”（permanment generation）

运行时常量池：是方法区的一部分，用于存放编译器生成的各种符号引用，这部分内容将在类加载后放到方法区的运行时常量池中。

注：JDK1.7中，存储在永久代的部分数据就已经转移到了堆中，而到了JDK 1.8 ，使用元空间取代了永生代。

元空间的本质和永久代类似，都是对JVM规范中方法区的实现。不过元空间与永久代之间最大的区别在于：元空间并不在虚拟机中，而是使用本地内存。

堆溢出：java.lang.OutOfMemoryError: Java heap space

```java
public class Test{
    public static void main(String[] args){

        ArrayList list=new ArrayList();
        while(true){
            list.add(new Test());
        }
    }
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

栈溢出：java.lang.StackOverflowError

```java
public class Test{
    public static void main(String[] args){
        new Test().test();
    }

    public void test(){
        test();
    }
}
```



# 二、hotspot虚拟机对象

2.1 对象的创建

1.检查 

虚拟机遇到一条new指令时，首先将去检查这个指令的参数是否能在常量池中定位到一个类的符号引用，并且检查这个符号引用代表的类是否已经被加载、连接和初始化过（类加载机制）。如果没有，那必须先执行相应的类加载过程。

2.分配内存 

接下来将为新生对象分配内存，为对象分配内存空间的任务等同于把一块确定的大小的内存从Java堆中划分出来。

假设Java堆中内存是绝对规整的，所有用过的内存放在一遍，空闲的内存放在另一边，中间放着一个指针作为分界点的指示器，那所分配内存就仅仅是把指向空闲空间的指针挪动一段与对象大小相等的距离，这个分配方式叫做“指针碰撞”

如果Java堆中的内存并不是规整的，已使用的内存和空闲的内存相互交错，那就没办法简单地进行指针碰撞了，虚拟机就必须维护一个列表，记录上哪些内存块是可用的，在分配的时候从列表中找到一块足够大的空间划分给对象实例，并更新列表上的记录，这种分配方式成为“空闲列表”

选择哪种分配方式由Java堆是否规整决定，而Java堆是否规整又由所采用的垃圾收集器是否带有压缩整理功能决定。

\3. Init

执行new指令之后会接着执行Init方法，进行初始化，这样一个对象才算产生出来

2.2 对象的内存布局（p47）

对象分为对象头、实例数据和对齐填充

对象头包括两部分：

a) 对象自身的运行时数据，Hash、GC分带年龄、锁等，长度为32bit或64bit

b) 类型指针，即对象指向类元数据的指针，虚拟机通过这个指针来确定对象是哪个类的实例

![img](https://img-blog.csdnimg.cn/2019020411353523.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

2.3 对象的访问定位（p49）

a）使用句柄访问

![img](https://img-blog.csdn.net/20181015113116170?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

Java堆中将会划分出一块内存来作为句柄池，reference中存储的就是对象的句柄地址，而句柄中包含了实例和类型的指针

优势:在对象被移动(垃圾收集时移动对象是非常普遍的行为)时只会改变句柄中的实例数据指针，而reference本身不需要修改

b）使用直接指针访问

Java堆对象的布局就必须考虑如何访问类型数据的相关信息,而refreence直接存储对象的地址

![img](https://img-blog.csdn.net/20181015145441151?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

优势：速度更快，节省了一次指针定位的时间开销，由于对象的访问在Java中非常频繁，因此这类开销积少成多后也是一项非常可观的执行成本



# 三、OutOfMemoryError 异常（OOM）

出现OOM如何解决：

（1）通过参数 -XX:+HeapDumpOnOutOfMemoryError 让虚拟机在出现OOM异常的时候Dump出内存映像以便于分析。

（2）使用jmap -heap PID >>heap_20110909.log



# 四、垃圾收集

程序计数器、虚拟机栈、本地方法栈3个区域随线程而生，随线程而灭，在这几个区域内就不需要过多考虑回收的问题，因为方法结束或者线程结束时，内存自然就跟随着回收了

### 1.判断对象存活

4.1.1 引用计数器法

给对象添加一个引用计数器，每当由一个地方引用它时，计数器值就加1；当引用失效时，计数器值就减1；任何时刻计数器为0的对象就是不可能再被使用的

4.1.2 可达性分析算法（主流）

通过一系列的称为“GC Roots”的对象作为起始点，从这些节点开始向下搜索，搜索所走过的路径成为引用链，当一个对象到GC ROOTS没有任何引用链相连时，则证明此对象时不可用的

Java语言中GC Roots的对象包括下面几种：

1.虚拟机栈（栈帧中的本地变量表）中引用的对象

2.本地方法栈（Native方法）引用的对象

3.方法区中类静态属性引用的对象

4.方法区中常量引用的对象

### 2.引用

强引用就是在程序代码之中普遍存在的，类似Object obj = new Object() 这类的引用，只要强引用还存在，垃圾收集器永远不会回收掉被引用的对象

软引用用来描述一些还有用但并非必须的元素。在OutOfMemoryError抛出之前，会把这些对象进行回收，如果回收后还没有足够的内存才会抛出内存溢出异常

弱引用：只能生存到下一次垃圾回收发生之前

虚引用的唯一目的就是能在这个对象被收集器回收时收到一个系统通知

### 3.一个对象“死亡”需要满足三点：

① “GC Roots”不可达

② 对象中有finalize()方法，且未被虚拟机调用过（若被调用过，下一次直接gg）

③ 满足①②条件后，对象调用finalize()，且进入F-Queue中等待死亡，若在这个过程中没人捞他一手，就GG

### 4.垃圾收集算法

4.4.1 标记—清除算法

算法分为标记和清除两个阶段：首先标记出所有需要回收的对象，在标记完成后统一回收所有被标记的对象、

不足:一个是效率问题，标记和清除两个过程的效率都不高；另一个是空间问题，标记清楚之后会产生大量不连续的内存碎片，下次程序运行需要分配较大的对象时，无法找到足够的连续内存而不得不提前触发另一次垃圾收集动作

![img](https://img-blog.csdn.net/2018101717364725?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

4.4.2 复制算法

他将可用内存按照容量划分为大小相等的两块，每次只使用其中的一块。当这块的内存用完了，就将还存活着的对象复制到另外一块上面，然后再把已使用过的内存空间一次清理掉。内存分配时只需按顺序分配即可。

不足：将内存缩小为原来的一半

实际中我们并不需要按照1:1比例来划分内存空间，而是将内存分为一块较大的Eden空间和两块较小的Survivor空间，每次使用Eden和其中一块Survivor

当另一个Survivor空间没有足够空间存放上一次新生代收集下来的存活对象时，这些对象将直接通过分配担保机制进入老年代

![img](https://img-blog.csdn.net/20181018104134258?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

4.4.3 标记整理算法

让所有存活的对象都向一端移动，然后直接清理掉端边界以外的内存

![img](https://img-blog.csdn.net/2018101811313088?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

4.4.4 分代收集算法

a)新生代内存按照8:1:1的比例分为一个eden区和两个survivor(survivor0,survivor1)区。

b) 大部分生成的对象都放在eden区(大的对象直接放入老年代)。当eden区满了时，先检查老年代最大连续可用空间 > 新生代所有对象空间？大于的话发生一次MinorGC。小于的话进入分配担保策略。

c) MinorGC的过程：先将eden区存活对象复制到survivor0区，然后清空eden区，当这个survivor0区也存放满了时，则将eden区和survivor0区存活对象复制到另一个survivor1区，然后清空eden和这个survivor0区。（上述回收过程称为Minor GC）。

c) 当survivor1区不足以存放 eden和survivor0的存活对象时，就将存活对象直接存放到老年代。若是老年代也满了就会触发一次Full GC，也就是新生代、老年代都进行回收。

d)Minor GC中长期存活的（默认是15岁）对象将进入老年代

e) 持久代就是方法区。

### 5.分配担保

发生在minorGC之前，先检查老年代最大连续可用空间 > 新生代所有对象空间？

大于的话，正常步骤进行。

小于的话，可以设置XXX（某个参数）来开启分配担保，开启的话，会继续检查 老年代最大连续可用空间 > 历次晋升到老年代对象的平均大小？

大于的话，minor GC，小于的话，先Full GC。

### 5.5. GC是什么时候触发的

5.1 MinorGC：eden区满了。

5.2 Full GC

  有如下原因可能导致Full GC：

a) 年老代（Tenured）被写满；

b) 持久代（Perm）被写满；

c) System.gc()被显式调用；



### 6.内存溢出和内存泄漏

内存溢出指的是内存不够了。

而内存泄漏指的是对象已经不需要了，但仍存在着引用，导致无法GC。

**如果长生命周期的对象持有短生命周期的引用，就很可能会出现内存泄露。**

内存泄漏最简单的例子：

```java
public class Simple {
    Object object;
    public void method1(){
    object = new Object();
    //...其他代码
    }
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

  这里object实例，如果object只在method1()，其他地方不会使用，那么这就是一种内存泄露。因为当method1()方法执行完成后，object对象所分配的内存不会被释放，只有在Simple类创建的对象被释放后才会被释放。

解决方法：

1、将object作为method1()方法中的局部变量。

2、如果一定要这么写，可以改为这样：

```java
public class Simple {
    Object object;
    public void method1(){
    object = new Object();
    //...其他代码
    object = null;
    }
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

### 7.垃圾收集器（JDK1.8默认垃圾收集器Parallel Scavenge（新生代）+Parallel Old（老年代））

注：

新生代 ----> 复制算法，因为每次垃圾收集时都发现有大批对象死去，只有少量存活，只需要付出少量存活对象的复制成本就可以完成收集。

老年代 --->   标记清理或者标记整理算法，因为对象存活率高、没有额外空间。

a)Serial收集器：

1、 是一个单线程的收集器

2、垃圾收集的过程中会Stop The World（服务暂停）（所有serial类和parallel类的收集器都会stop）

![img](https://img-blog.csdn.net/20181021160827138?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

b)ParNew 收集器：

Serial收集器的多线程版本，除了使用了多线程进行收集之外，其余行为和Serial收集器一样

![img](https://img-blog.csdn.net/20181021162058275?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)



c)Parallel Scavenge 

特点：它的关注点与其他收集器不同，其目标是达到一个可控制的吞吐量。

吞吐量=运行用户代码时间/（运行用户代码时间+垃圾收集时间）

![img](https://img-blog.csdn.net/20181021162126429?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

d)Serial Old 收集器：

是Serial收集器的老年代版本,所以使用标记整理算法，是一个单线程收集器，

e)Parallel Old 收集器：

Parallel Old是Paraller Seavenge收集器的老年代版本，所以使用标记整理算法，同时使用多线程。

f)CMS收集器：（concurrent mark sweep）

CMS收集器是基于标记清除（mark sweep）算法实现的，第一次实现了让垃圾收集线程与用户线程（基本上）同时工作。

特点：

1、基于"标记-清除"算法

2、以获取最短回收停顿时间为目标

3、并发收集

整个过程分为4个步骤

![img](https://img-blog.csdn.net/20181021162456142?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

**初始标记（CMS initial mark）** 
初始标记仅仅只是标记一下GC Roots能直接关联到的对象，速度很快，需要“Stop The World”。

**并发标记（CMS concurrent mark）** 
并发标记阶段就是进行GC Roots Tracing的过程。（不会stop the world）

**重新标记（CMS remark）** 
重新标记用户线程并发运行时 出现变动的 地方，这个阶段的停顿时间一般会比初始标记阶段稍长一些，但远比并发标记的时间短，仍然需要“Stop The World”。

**并发清除（CMS concurrent sweep）** 
并发清除阶段会清除对象。  （不会stop the world）

缺点：

1.CMS收集器对CPU资源非常敏感，CMS默认启动的回收线程数是（CPU数量+3）/4，

2.CMS收集器无法处理浮动垃圾

浮动垃圾：由于CMS**并发清理阶段**用户线程所产生的垃圾，这部分垃圾出现在重新标记之后，CMS无法在当次收集中处理掉它们，只好留待下一次GC中再清理。这些垃圾就是“浮动垃圾”。

3.CMS是基于标记清除算法实现的（标记清除算法会产生大量空间碎片）



g)G1收集器：

概念：

1、使用G1收集器时，Java堆的内存布局就与其他收集器有很大差别，它将整个Java堆划分为多个大小相等的独立区域（Region），（虽然还保留有新生代和老年代的概念）。

2、G1跟踪各个Region里面的垃圾堆积的回收价值和成本（回收所获得的空间大小及回收所需时间的经验值），在后台维护一个优先列表，按优先级回收Region

3、 同CMS，G1也可以实现gc线程和用户线程的并发。

![img](https://img-blog.csdnimg.cn/20190221170436740.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

**初始标记、并发标记**：同上。

**最终标记（Final Marking）** 
作用同**重新标记**，同时虚拟机会将对象的变化记录在Logs里面，并且会把Logs的数据合并到Remembered Set中，这阶段需要停顿线程，但是可并行执行。

**筛选回收（Live Data Counting and Evacuation）** 
筛选回收阶段首先对各个Region的回收价值和成本进行排序，按优先级回收Region。（这个阶段其实也可以做到与用户程序一起并发执行，但是因为只回收一部分Region，时间是用户可控制的，而且停顿用户线程将大幅提高收集效率）。



### 8.内存分配与回收策略（了解）

4.6.1 对象优先在Eden分配：

大多数情况对象在新生代Eden区分配，当Eden区没有足够空间进行分配时，虚拟机将发起一次Minor GC

4.6.2 大对象直接进入老年代：

所谓大对象就是指需要大量连续内存空间的Java对象，最典型的大对象就是那种很长的字符串以及数组。这样做的目的是避免Eden区及两个Servivor之间发生大量的内存复制

4.6.3长期存活的对象将进入老年代

如果对象在Eden区出生并且尽力过一次Minor GC后仍然存活，并且能够被Servivor容纳，将被移动到Servivor空间中，并且把对象年龄设置成为1.对象在Servivor区中每熬过一次Minor GC，年龄就增加1岁，当它的年龄增加到一定程度（默认15岁），就将会被晋级到老年代中

4.6.4动态对象年龄判定

为了更好地适应不同程序的内存状况，虚拟机并不是永远地要求对象的年龄必须达到了MaxTenuringThreshold才能晋级到老年代，如果在Servivor空间中相同年龄所有对象的大小总和大于Survivor空间的一半，年龄大于等于该年龄的对象就可以直接进入到老年代，无须登到MaxTenuringThreshold中要求的年龄

4.6.4 空间分配担保：

Minor GC 之前 ---> 老年代最大可用的连续空间 > 新生代所有对象总空间？Minor GC安全：情况A；

A：HandlePromotionFailure开启？情况B：FULL GC；

B：老年代最大可用的连续空间 > 历次晋级到老年代对象的平均大小：冒险Minor GC：FULL GC;



# 五、类加载机制（重要）

虚拟机把描述类的数据从Class文件加载到内存，并对数据进行校验、转换解析和初始化，最终形成可以被虚拟机直接使用的Java类型，这就是虚拟机的类加载机制

![img](https://img-blog.csdn.net/20181024093343810?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)



### 5.1 类加载的时机

类的整个生命周期包括：加载、连接（验证、准备、解析）、初始化、使用和卸载7个阶段

而类的加载包括：加载、连接（验证、准备、解析）、初始化

### 5.2 类加载的过程（重要）

5.2.1 加载

1)通过类的全限定名查找该类的字节流文件

2)将这字节流所代表的静态存储结构转化为方法区运行时数据结构（静->动）

3)在内存中生成Class对象，作为方法区这个类的各种数据的访问入口


**数组类**本身不通过类加载器创建，它是由Java虚拟机直接创建的

（数组类的创建过程遵循以下规则：

1)如果数组的组件类型(指的是数组去掉一个维度的类型)是引用类型，那就递归单个类加载过程去加载，数组C将在加载该组件类型的类加载器的类名称空间上被标识

2)如果数组的组件类型不是引用类型(列如int[]组数)，Java虚拟机将会把数组C标识为与引导类加载器关联

3)数组类的可见性与它的组件类型的可见性一致，如果组件类型不是引用类型，那数组类的可见性将默认为public）

5.2.2 验证

验证阶段会完成下面4个阶段的检验动作：文件格式验证，元数据验证，字节码验证，符号引用验证

1.文件格式验证

**第一阶段要验证字节流是否符合Class文件格式的规范**，并且能被当前版本的虚拟机处理,如验证魔数是否0xCAFEBABE。

**这个阶段的验证是基于二进制字节流进行的，只有通过验证后，字节流才会进入内存的方法区进行存储，所以后面的3个验证阶段全部是基于方法区的存储结构进行的，不会再直接操作字节流**

2.元数据验证（即与父类的关系）

1.这个类**是否有父类**(除了java.lang.Object之外,所有的类都应当有父类)

2.如果这个类不是抽象类，**是否实现了父类**或接口之中要求实现的所有方法

3.类中的字段、**方法是否与父类产生矛盾**

3.字节码验证

第三阶段是整个验证过程中最复杂的一个阶段，主要目的是通过数据流和控制流分析，确定程序语言是否是合法的、符合逻辑的。在第二阶段对元数据信息中的数据类型做完校验后，这个阶段将对类的方法体进行校验分析，保证被校验类的方法在运行时不会做出危害虚拟机安全的事件。



5.2.3 准备

为类中的所有静态变量 分配内存空间，并为其设置一个初始值。注意：这里不包括实例变量，**实例变量将会在对象实例化时随着对象一起分配在Java堆中**。

假设public static int value = 123；

那**变量value在准备阶段过后的初始值为0而不是123**，因为这时候尚未开始执行任何Java方法，而把value赋值为123的putstatic指令是程序被编译后，存放于类构造器<clinit>()方法之中，所以**把value赋值为123的动作将在初始化阶段才会执行**，但是**如果使用final修饰，则在这个阶段其初始值设置为123**

5.2.4解析

**解析阶段是虚拟机将常量池内符号引用替换为直接引用的过程**

符号引用：**用一组符号来描述所引用的目标**

直接引用：**可以使直接指向目标的指针、相对偏移量或是一个能间接定位到目标的句柄。**

5.2.5 初始化

类的初始化阶段是类加载过程的最后一步，**是执行类构造器<client>()方法的过程**。

**只有主动引用的时候才会触发类的初始化操作。**

（一个类在初始化的时候要求其父类全部初始化了，但是一个接口初始化的时候不要求其父接口全部都初始化了）

在连接的准备阶段，类变量已赋过一次系统要求的初始值，而在初始化阶段，则是根据程序员自己写的逻辑去初始化类变量和其他资源，举个例子如下：

```java
    public static int value1  = 5;
    public static int value2  = 6;
    static{
        value2 = 66;
    }
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

在准备阶段value1和value2都等于0；

在初始化阶段value1和value2分别等于5和66；





**主动引用：**

虚拟机规范规定有且只有5种情况（即主动引用）必须立即对类进行初始化：

1.遇到**new、getstatic、putstatic或invokestatic这4条字节码指令**时，如果类没有进行过初始化，则需要触发其初始化。生成这4条指令的最常见的Java代码场景是：使用new关键字实例化对象的时候、读取或设置一个类的静态字段（被final修饰、已在编译期把结果放入常量池的静态字段除外）的时候，以及调用一个类的静态方法的时候

2.使用java.lang.reflect包的方法**对类进行反射调用**的时候，如果类没有进行过初始化，则需要先触发其初始化

3.当初始化一个类的时候，如果发现**其父类还没有进行过初始化**，则需要**先触发其父类的初始化**

4.当虚拟机启动时候，虚拟机会**先初始化****包含main()方法的那个类**

5.当使用JDK1.7的动态语言支持时，如果一个java.lang.invoke.MethodHandle实例最后的解析结果REF_getStatic、REF_putStatic、REF_invokeStatic的方法句柄，并且这个方法句柄所对应的类没有进行过初始化，则需要先触发其初始化

**被动引用：**

1.通过子类引用父类的静态字段，不会导致子类初始化

![img](https://img-blog.csdn.net/20181024141941315?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

2.通过数组定义来引用类，不会触发此类的初始化

![img](https://img-blog.csdn.net/2018102414192575?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

3.常量不会触发定义常量的类的初始化，因为常量在编译阶段会存入调用类的常量池中，本质上并没有直接引用到定义常量的类。

![img](https://img-blog.csdn.net/20181024142055422?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

**接口的初始化**：

接口在初始化时，并不要求其父接口全部完成类初始化，只有在真正使用到父接口的时候（如引用接口中定义的常量）才会初始化



### 5.3 类的加载器（重要）

5.3.1 类与类加载器

比较**两个类是否“相等”**，只有在这**两个类是由同一个类加载器加载的前提下**才有意义。

5.3.2 双亲委派模型：

只存在两种不同的类加载器：**启动类加载器**（Bootstrap ClassLoader），使用C++实现，是虚拟机自身的一部分。另一种**其他**类加载器，使用JAVA实现，独立于JVM，并且全部继承自抽象类java.lang.ClassLoader.

启动类加载器（Bootstrap ClassLoader），负责将存放在<JAVA+HOME>\lib目录中的，或者被-Xbootclasspath参数所制定的路径中的，并且是JVM识别的（仅按照文件名识别，如rt.jar，如果名字不符合，即使放在lib目录中也不会被加载），加载到虚拟机内存中，启动类加载器无法被JAVA程序直接引用。

扩展类加载器，由sun.misc.Launcher$ExtClassLoader实现，负责加载<JAVA_HOME>\lib\ext目录中的，或者被java.ext.dirs系统变量所指定的路径中的所有类库，开发者可以直接使用扩展类加载器。

应用程序类加载器（Application ClassLoader），由sun.misc.Launcher$AppClassLoader来实现。由于这个类加载器是ClassLoader中的getSystemClassLoader()方法的返回值，所以一般称它为**系统类加载器**。**负责加载用户类路径（ClassPath）上所指定的类库**，开发者可以直接使用这个类加载器，如果应用程序中没有自定义过自己的类加载器，一般情况下这个就是程序中**默认的类加载器**。

![img](https://img-blog.csdnimg.cn/20181026153327852.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_27,color_FFFFFF,t_70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

这张图表示类加载器的双亲委派模型（Parents Delegation model）. 双亲委派模型要求除了顶层的启动加载类外，其余的类加载器都应当有自己的父类加载器。，这里类加载器之间的父子关系一般不会以继承的关系来实现，而是使用组合关系来复用父类加载器的代码。

双亲委派模型的工作过程是：

**如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把这个请求委派给父类加载器去完成，**只有当父类加载器反馈自己无法完成这个加载请求（它的搜索范围中没有找到所需的类）时,夫加载器会抛出ClassNotFoundException然后给子加载器，子加载器才会尝试自己去加载。

这样做的好处就是：

**1、**Java类随着它的类加载器一起具备了一种带有优先级的层次关系**，可以避免类的重复加载**。例如类java.lang.Object,它存放在rt.jar中，无论哪一个类加载器要加载这个类，最终都是委派给处于模型**最顶端的启动类加载器**进行加载，因此Object类在程序的各种类加载器环境中都是同一个类。

2、**保证安全**，**如果没有使用双亲委派模型**，如果用户**自己编写了一个java.lang.object的类**，并放在ClassPath中，那系统会出现多个Object类，**应用程序会变得一片混乱**。



# 六、Java内存模型与线程

注：JAVA内存模型中所说的“变量”与java编程的变量有所不同，不包括局部变量和方法参数，因为它们是线程私有的。

### 6.1主内存与工作内存

Java内存模型规定所有的变量存储在主内存中，每条线程还有自己的工作内存。



### 6.2 内存间的交互操作

![img](https://img-blog.csdnimg.cn/20190117173702538.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

Java内存模型定义了以下八种操作来完成：（理解，不背）

lock（锁定）：作用于**主内存变量**，把一个变量标识为一条**线程独占**状态。

unlock（解锁）：作用于**主内存变量**，**解锁才可以被其他线程锁定**。

read（读取）：作用于**主内存变量**，把一个变量值**从主内存传输到线程的工作内存**中，以便随后的load动作使用

load（载入）：作用于**工作内存的变量**，它把**read操作得到的变量值放入工作内存的变量副本**中。

use（使用）：作用于工作内存的变量，把工作内存中的一个**变量值传递给执行引擎**，每当虚拟机遇到一个需要使用变量的值的字节码指令时将会执行这个操作。

assign（赋值）：作用于工作内存的变量，它把一个**从执行引擎接收到的值赋值给工作内存的变量**，每当虚拟机遇到一个给变量赋值的字节码指令时执行这个操作。

store（存储）：作用于工作内存的变量，把**工作内存中的变量值传送到主内存**中。（只是负责传输）

write（写入）：作用于**主内存的变量**，它把**store操作从工作内存中得到的变量的值放入到主内存的变量**中。

如果要**把一个变量从主内存中复制到工作内存，就需要按顺序德执行read和load操作**， 如果**把变量从工作内存中同步回主内存中，就要按顺序地执行store和write操作**。注意，**这里只要求上述操作必须按顺序执行，而没有保证必须是连续执行**。也就是**read和load之间， store和write之间是可以插入其他指令的**，如对主内存中的变量a、b进行访问时，可能的顺 序是read a，read b，load b， load a。

Java内存模型还规定了在执行上述八种基本操作时，必须满足如下规则：（理解）

1、不允许read和load、store和write操作之一单独出现

2、不允许一个线程丢弃它的最近的assign的操作，即变量在工作内存中改变了之后必须同步到主内存中。

3、不允许一个线程无原因地（没有发生过任何assign操作）把数据从工作内存同步回主内存中。

4、一个新的变量只能在主内存中诞生，不允许在工作内存中直接使用一个未被初始化（load或assign）的变量。即就是对一个变量实施use和store操作之前，必须先执行过了assign和load操作。

5、一个变量在同一时刻只允许一条线程对其进行lock操作，但lock操作可以被同一条线程重复执行多次，多次执行lock后，只有执行相同次数的unlock操作，变量才会被解锁。

6、如果对一个变量执行lock操作，将会清空工作内存中此变量的值

7、如果一个变量事先没有被lock操作锁定，则不允许对它执行unlock操作；也不允许去unlock一个被其他线程锁定的变量。

8、对一个变量执行unlock操作之前，必须先把此变量同步到主内存中（执行store和write操作）。



### 七、jvm调优：

\1. 使用ps aux | grep java 或 jps -vl 获取当前运行的java进程号(PID)

\2. 提取GC信息() ：jstat –gc PID >> gc_20110909.log

可以显示gc的信息，查看gc的次数，及时间。其中最后五项，分别是young gc的次数，young gc的时间，full gc的次数，full gc的时间，gc的总时间。 

4、 提取Heap区信息（间隔10分钟提取一次）：jmap -heap PID >>heap_20110909.log

5、提取对象信息（间隔10分钟提取一次）jmap -histo PID > histo_*.log(文件较大,*按序号生成输入)

6、查找cpu占有率极高的java线程出现的原因：（注意这里有分进程pid和线程pid，是两个东西）

\- 找到java进程pid，输出top，假设获得的是`23344。`

![img](https://img-blog.csdnimg.cn/20190226101506928.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

`- top -Hp 23344`可以查看该进程下各个线程的cpu使用情况；找到最高的一个对应的线程pid

![img](https://img-blog.csdnimg.cn/20190226101453407.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

\- jstack 进程pid 获取该进程下所有线程的状态，查找之前占有率最高的线程所在的日志位置，查找方法为，将上面找到的线程pid（25077）转化成16进制-->0x246c，这就是线程的nid

![img](https://img-blog.csdnimg.cn/20190226101917216.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

在nid=0x246c的线程调用栈中，发现该线程一直在执行JstackCase类第33行的calculate方法，得到这个信息，就可以检查对应的代码是否有问题。

b)      宕机日志(在domain目录,如果没有宕机,条件允许的情况下可使用 kill -3 PID(此操作会杀掉进程))