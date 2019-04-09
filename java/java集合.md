## Map:

### HashMap：

1. HashMap实际上是数组和链表的结合体。HashMap的<font color=#ff0000>底层结构是一个Node<>（实现了Entry<>）数组</font>，，<font color=#ff0000>数组中的每一项是一条链表</font>（**短时使用链表，长时使用红黑树（JDK8以后）**），链表中存的是Node，红黑树存的是TreeNode。这样便继承了数组查找快以及链表寻址修改的优点。
   ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190214165407695.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)
2. HashMap的实例有俩个参数影响其性能：<font color=#ff0000> “初始容量” 和 装填因子</font>。
   初始容量：即Node数组的长度，默认是16，在JDK1.8之后，我们人为设置初始容量的话，**会自动转成2的n次幂**，例如我们设置长为13或16，会转成16，设置成100的话会转成128。这么做的原因是为了加快resize的效率（详见后）。
    装填因子默认是0.75，装填因子与resize有关。

```java
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```

5. HashMap线程不安全。  HashTable线程安全
6. HashMap可以接受null的key和value，而HashTable不行。
   为什么HashTable不允许接受null的key和value？（对于所有并发的map而言都不能，假设能存null的key和value。举例子：一开始读取，线程A读取null，因为没有存，得到null，然后接着线程B往table中写入(null,null)，然后A再去读取null，返回的是null，A以为table没有修改，实际上已经修改了）
7. hash 冲突解决的方法： 
   ①开放定址法：就是一旦发生了冲突，就去寻找下一个空的散列地址，只要散列表足够大，空的散列地址总能找到，并将记录存入。方法1：+1+1的找；方法2：1，-1，2平方, - 2的平方，...
    ②拉链法：
    ③再哈希法：同时构造多个不同的哈希函数，当一个冲突时使用下一个，直到不冲突为止。
8. HashMap的工作原理：
   get（）具体原理：
   ![get方法详解](https://img-blog.csdnimg.cn/20190129194000329.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)
   put的具体原理：（最好便说便写）
   a）对 Key 求 Hash 值，找到数组中对应的位置。
   b）如果对应bucket为空，直接放入；如果不为空，使用equals()判断key是否存在，如果存在，替换旧值，如果不存在，存入bucket中。
   c）如果链表长度超过阀值（TREEIFY THRESHOLD==8），如果桶长度小于64：resize，否则将链表转成红黑树，链表长度低于6，就把红黑树转回链表
   d）如果桶满了（容量16 * 加载因子0.75），就需要 resize（扩容2倍后重排）
9. map的遍历方式：
   a） 使用Map.Entry<X,X> ( 两个都需要时用，最常见）：

```java
Map<Integer, Integer> map = new HashMap<Integer, Integer>(); 
for (Map.Entry<Integer, Integer> entry : map.entrySet()) 
  	System.out.println("Key = " + entry.getKey() + ", Value = " + entry.getValue()); 
```

&emsp;&emsp;b） 在for-each循环中遍历keySet或values（只需要其中一个时用）：

```java
for (Integer key : map.keySet()) { 
  System.out.println("Key = " + key); 
} 

for (Integer value : map.values()) { 
  System.out.println("Value = " + value); 
}
```

&emsp;&emsp;c） 使用Iterator遍历（边遍历边删除时用）：

```java
Iterator iter= map.entrySet().iterator(); 
while (iter.hasNext()) { 
  iter.remove();
}
```

8. 能否使用任何类作为 Map 的 key？  能否使用任何类作为 Map 的 key？
    可以使用任何类作为 Map 的 key，然而在使用之前，需要考虑以下几点：
   &emsp;&emsp;- **如果类重写了 equals() 方法，也应该重写 hashCode() 方法。**
   &emsp;&emsp;- 用户自定义 Key 类最好是不可变类。
   原因在于：如果有一个类 MyKey，在 HashMap 中使用它：

```java
HashMap<MyKey, String> myHashMap = new HashMap<MyKey, String>();

//传递给 MyKey 的 name 参数被用于 equals() 和 hashCode() 中
MyKey key = new MyKey("Pankaj"); // 假设 hashCode=1234
myHashMap.put(key, "Value");

// 以下的代码会改变 key 的 hashCode() 和 equals() 值
key.setName("Amit"); // 假设新的 hashCode=7890

//下面会返回 null，因为 HashMap 会尝试查找存储同样索引的 key，而 key 已被改变了，匹配失败，返回 null
System.out.println(myHashMap.get(new MyKey("Pankaj")));
```

9. 为什么重写了equals()方法，一定要重写hashCode()方法？
   首先，我们要确认一点，如果两个对象相同（equals方法为true），那么他们的hashCode也是相同的。
   因为不重写的话，就会用Object.hashCode()方法，而Object.hashCode返回的是地址本身。
   就可能出现重写了equals后使得两个对象的equals为true，但hashcode不相同的情况。



11. 有什么方法可以减少碰撞？有什么方法可以减少碰撞？
    a）使用扰动函数。
    b）使用不可变、final 的对象
12. 为什么 String、Integer 这样的 wrapper 类适合作为键？
    因为 String，Integer  是不可变类。
13. hash函数的实现：（前16位和后16位亦或，然后与桶长度-1进行&运算）
    （如果hashCode之后直接&(n-1)，很容易产生碰撞。因此java设计者权衡了speed, utility, and quality，决定将高16位与低16位异或来减少这种影响）

```java   
	h = key.hashCode()；返回散列值也就是hashcode
    // >>>:无符号右移，忽略符号位，空位都以0补齐
    //其中n是数组的长度，即Map的数组部分初始化长度
    //hash结果为:h和它的后16位亦或
    //然后再求index：hash &(桶长度-1)
    return (n-1)&(h ^ (h >>> 16));
```

10. 链表过深时，为什么不用二叉查找树代替而选择红黑树？为什么不一直使用红黑树？
    之所以选择红黑树是为了解决二叉查找树的缺陷：<font color=#ff0000>二叉查找树在特殊情况下会变成一条线性结构</font>（这就跟原来使用链表结构一样了，造成层次很深的问题），遍历查找会非常慢。
    而之所以不一直使用红黑树，是因为红黑树属于平衡二叉树，<font color=#ff0000>为了保持“平衡”是需要付出代价的</font>，但是该代价所损耗的资源要比遍历线性链表要少。如果链表长度很短的话，根本不需要引入红黑树，引入反而会慢。
11. 说说你对红黑树的见解？
    &emsp;&emsp;a）每个节点非红即黑
    &emsp;&emsp;b）根节点总是黑色的
    &emsp;&emsp;c）如果节点是红色的，则它的子节点必须是黑色的（反之不一定）
    &emsp;&emsp;d）每个叶子节点都是黑色的空节点（NIL节点）
    &emsp;&emsp;e）从根节点到叶节点或空子节点的每条路径，必须包含相同数目的黑色节点（即相同的黑色高度）
    &emsp;&emsp;f）红黑树能够以<font color=#ff0000>O(log2(N))</font>的时间复杂度进行增删查操作。
    &emsp;&emsp;g）红黑树不像 avl 树（平衡树）一样追求绝对的平衡，他允许局部很少的不完全平衡，这样对于效率影响不大，但省去了很多没有必要的调平衡操作。
    ![红黑树](https://img-blog.csdnimg.cn/20190129204547164.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_8,color_FFFFFF,t_70)
12. 如果 HashMap 的大小超过了负载因子（load factor）定义的容量怎么办？如果 HashMap 的大小超过了负载因子（load factor）定义的容量怎么办？
    HashMap 默认的负载因子大小为0.75。也就是说，当一个 Map 填满了75%的 Node 时候，和其它集合类一样（如 ArrayList 等），将会创建原来大小的两倍的 bucket 数组。
    然后会将原数组中的数据必须重新计算其在新数组中的位置，并放进去（此过程最消耗性能）。
    注： 在JDK1.8之后，resize计算新位置时有优化：(了解即可）

```java
					//如果这个oldTab[j]就一个元素，那么就直接放到newTab里面
                    if (e.next == null) 
                        newTab[e.hash & (newCap - 1)] = e;
                     else if ...
                     else 
                     {
 							 /*这里的操作就是 (e.hash & oldCap) == 0 这一句，
                            这一句如果是true，表明(e.hash & (newCap - 1))还会和
                            e.hash & (oldCap - 1)一样。因为oldCap和newCap是2的次幂，
                            并且newCap是oldCap的两倍，就相当于oldCap的唯一
                            一个二进制的1向高位移动了一位
                            (e.hash & oldCap) == 0就代表了(e.hash & (newCap - 1))还会和
                            e.hash & (oldCap - 1)一样。

                            比如原来容量是16，那么就相当于e.hash & 0x1111 
                            （0x1111就是oldCap - 1 = 16 - 1 = 15），
                            现在容量扩大了一倍，就是32，那么rehash定位就等于
                            e.hash & 0x11111 （0x11111就是newCap - 1 = 32 - 1 = 31）
                            现在(e.hash & oldCap) == 0就表明了
                            e.hash & 0x10000 == 0，这样的话，不就是
                            已知： e.hash &  0x1111 = hash定位值Value
                             并且  e.hash & 0x10000 = 0
                            那么   e.hash & 0x11111 不也就是
                            原来的hash定位值Value吗？
                            */
                         //初始化两个链表，低位为原本位置oldCap，高位在j + oldCap
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {
								//加入低位链表
                            }
                            else {
								//加入高位链表
                            }
						//两个链表的头放入数组对应位置
						...
```

13. 重新调整 HashMap 大小存在什么问题吗？重新调整 HashMap 大小存在什么问题吗？
    如果两个线程都发现 HashMap 需要重新调整大小了，它们会同时试着调整大小。JDK1.7中rehash的时候，旧链表迁移新链表的时候，如果在新表的数组索引位置相同，则链表元素会倒置（JDK1.8不会倒置）。如果条件竞争发生了，那么就死循环了。多线程的环境下不使用 HashMap。

### HashTable：

1. 数组 + 链表方式存储，默认容量：11（质数为宜）
2. put操作：首先进行索引计算 （key.hashCode() & 0x7FFFFFFF）% table.length；若在链表中找到了，则替换旧值，若未找到则继续；当总元素个数超过 容量 * 加载因子 时，扩容为old*2+1；将新元素加到链表头部。
3. 对修改 Hashtable 内部共享数据的方法添加了 synchronized，保证线程安全
4. HashMap 与 HashTable 区别：
   &emsp;- 默认容量不同，扩容不同（HashMap为old*2,HashTable为old*2+1）
   &emsp;- 线程安全性：HashTable 安全
   &emsp;- 效率不同：HashTable 要慢，因为加锁
   &emsp;- HashMap可以接受null的key和value，而HashTable不行。
5. 可以使用 CocurrentHashMap 来代替 Hashtable 吗？
   ConcurrentHashMap 当然可以代替 HashTable，但是 HashTable 提供更强的线程安全性

### CocurrentHashMap（JDK 1.7）：

1. CocurrentHashMap 是由 Segment 数组和 HashEntry 数组和链表组成。
2. Segment：一个ReentrantLock,也就是说segment扮演者锁的角色。
3. 核心数据如 value，以及链表都是 volatile 修饰的，保证了获取时的可见性
4. 虽然 HashEntry 中的 value 是用 volatile 关键词修饰的，但是并不能保证并发的原子性，所以 put 操作时仍然需要加锁处理
5. ConcurrentHashMap会对数据hash后 对摘要值进行二次hash，其目的是减少hash冲突，使元素均匀分布。

### CocurrentHashMap（JDK 1.8） （读不加锁、写加锁）

1. JDK1.8 的实现已经摒弃了Segment的概念，<font color=#ff0000>恢复成HashMap的结构</font>（bucket数组+链表+红黑树），<font color=#ff0000>并发控制使用 Synchronized 和 CAS 来操作</font>，整个看起来就像是优化过且线程安全的HashMap.
2. JDK1.7版本锁的粒度是基于Segment的，包含多个HashEntry，而JDK1.8锁的粒度就是HashEntry。
3. JDK1.8使用内置锁synchronized来代替重入锁ReentrantLock
4. 因为JDK1.8中，hashMap引入了红黑树，所以cocurrentHashMap也同样使用了红黑树。
5. CocurrentHashMap（1.8）中get操作为什么不加锁？
   get操作可以无锁是由于Node的元素val和指针next是用volatile修饰的，这样可保证多线程对node的可见性。
6. CocurrentHashMap（1.8）中put操作为什么加锁？
   正因为node中元素是用volatile修饰，并不能保证原子性，所以写操作需要加锁来保证原子性。

##### 源码解析：

①table初始化：
table初始化操作会延缓到第一次put行为。但是put是可以并发执行的，所以初始化还是要考虑并发。
sizeCtl默认为0，非默认sizeCtl会是一个2的幂次方的值（详见hashMap）。
具体并发控制操作请看代码注释：

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
		//如果一个线程发现sizeCtl<0，意味着另外的线程执行CAS操作成功，当前线程只需要让出cpu时间片
        if ((sc = sizeCtl) < 0) 
            Thread.yield(); 
        //尝试用CAS将sizeCtl设置成-1，成功的话说明由该线程来进行初始化
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
        //设置成功，进行初始化
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

put操作：
put操作采用CAS+synchronize来控制
其中table中某一格为空，也就是第一次插入时采用CAS，而之后链表或者红黑树的插入就采用synchronize

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
	//关键在这一块！CAS操作的条件！
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        //下面使用synchronize来插入链表或红黑树（代码略）
    }
    addCount(1L, binCount);
    return null;
}
```

### LinkedHashMap（HashMap + 双向链表）

双向链表中存放着Entry<>。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190225110737772.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)
LinkedHashMap实现双向链表的核心在于重新定义了Entry<>,里面新添了after和before

```java
private static class Entry<K,V> extends HashMap.Entry<K,V> {
    // These fields comprise the doubly linked list used for iteration.
    Entry<K,V> before, after;

    Entry(int hash, K key, V value, HashMap.Entry<K,V> next) {
        super(hash, key, value, next);
    }
    ...
}
```

注：LinkedHashMap可以选择是**插入顺序**（不改变顺序）或者是**访问顺序**（改变顺序，可用于LRUCache）。默认是插入，如果想使用访问排序，方法：`new LinkedHashMap(10, 0.75, true);`前两个参数是HashMap的参数，最后的true表明使用访问排序。

### TreeMap 

TreeMap,其实就是一个**红黑树**，树的**每一个节点都是一个Entry<>**, 内部声明了一个Comparator的局部变量。

```java
public class TreeMap<K,V>  extends ...
{
    private final Comparator<? super K> comparator;

    private transient Entry<K,V> root;
}
```

put操作会使用比较器进行比较，小于放左边，大于放右边。

```java
public V put(K key, V value) {
		//前面略
		
        Comparator<? super K> cpr = comparator;
        if (cpr != null) {
            do {
                parent = t;
                cmp = cpr.compare(key, t.key);
                if (cmp < 0)
                    t = t.left;
                else if (cmp > 0)
                    t = t.right;
                else
                    return t.setValue(value);
            } while (t != null);
        }	
		
		//后面略
}

```

# List

### ArrayList（动态数组）

- 查找O(1),但插入与删除耗时。
- 初始化：
  （1）ArrayList()构造一个<font color=#ff0000>初始容量为 10 </font>的空数组，不够可以扩充。
   （2）ArrayList(int initialCapacity)构造一个具有指定初始容量的空数组。
- 当元素超出数组内容，会产生一个新数组（容量是<font color=#ff0000>原来的1.5倍</font>），然后使用Arrays.copyOf()将元素放入新数组中.
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190226162527245.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)
- 非线程安全
- 多线程场景下如何使用 ArrayList？
  最常用的方法是通过 **Collections 的 synchronizedList 方法**将 ArrayList 转换成线程安全的容器后再使用。

```java
List<Object> list =Collections.synchronizedList(new ArrayList<Object>);
```

- 为什么 ArrayList 的 elementData 加上 transient 修饰？
  ArrayList 中的数组定义如下：

```java
private transient Object[] elementData;
```

transient 的作用是说不希望 elementData 数组被序列化。

```java
private void writeObject(java.io.ObjectOutputStream s) throws java.io.IOException{
    s.defaultWriteObject();

    for (int i=0; i<size; i++)
        s.writeObject(elementData[i]);
    }
```

每次序列化时，先调用 defaultWriteObject() 方法序列化 ArrayList 中的非 transient 元素，然后遍历 elementData，只序列化已存入的元素，这样既加快了序列化的速度，又减小了序列化之后的文件大小。

- 遍历一个 List 有哪些不同的方式？每种方法的实现原理是什么？Java 中 List 遍历的最佳实践是什么？
  &emsp;&emsp;a）for 循环遍历。
  &emsp;&emsp;b）迭代器遍历，Iterator。
  &emsp;&emsp;c）foreach 循环遍历。（不能在遍历过程中操作数据集合）
  注：如果一个数据集合**实现了Random Access**接口，例如ArrayList，那么按位置读取元素的平均时间复杂度为 O(1)，**建议使用for 循环遍历**。否则建议使用Iterator 或 foreach 遍历。

### LinkedList（链表）

查找O(n)，插入和删除O（i）

### 阻塞队列：（详见并发编程.md）

### Vector

- 与 ArrayList 类型一样，内部也是使用数组来存储对象，<font color=#ff0000>但是线程安全的</font>。
- 不建议使用（现在也基本不使用），因为性能消耗。

## Set：

- Set集合的实现依赖于Map的实现，通过Map的 key值唯一性来实现。
- 补充：list和set都可以存放null。

## fail-fast机制（了解）

- fail-fast 是Java集合的一种错误检测机制，当多个线程对集合进行结构上的改变的时，有可能会产生fail-fast机制。
- fail-fast产生的原因就在于程序在对 collection 进行迭代时，某个线程对该 collection 在结构上对其做了修改，这时迭代器就会抛出 ConcurrentModificationException 异常信息，从而产生 fail-fast

#### 补充：

- 如何边遍历边移除 Collection 中的元素？
  边遍历边修改 Collection 的**唯一正确**方式是使用 **Iterator.remove()** 方法，如下：

```java
Iterator<Integer> it = list.iterator();
while(it.hasNext()){
    // do something
    it.remove();
}
```
