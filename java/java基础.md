
# 一、基础
#### 1. java八大原生类型

byte(1)、boolean(1)、short(2)、char(2)、int(4)、long(64)、float(32)、double(64)

#### 2.  值传递和引用传递

下面代码中结果i还是0，因为<font color=#ff0000>八大原生类型作为形参时是值传递</font>

```java
public void change(int i) {	i++;	}
int i = 0;
change(i);
```

而下面的代码中，是将引用进行传递，所以age会改变

```java
public void change(Person p){	p.age++;	}
Person p;
change(p);
System.out.println(p.age);
```


#### 3. 面向对象和面向过程的区别

##### 面向过程

**优点：** 性能比面向对象高，因为类调用时需要实例化，开销比较大，比较消耗资源;比如单片机、嵌入式开发、Linux/Unix等一般采用面向过程开发，性能是最重要的因素。

**缺点：** 没有面向对象易维护、易复用、易扩展

##### 面向对象

**优点：** 易维护、易复用、易扩展，由于面向对象有封装、继承、多态性的特性，可以设计出低耦合的系统，使系统更加灵活、更加易于维护

**缺点：** 性能比面向过程低



#### 4. 重载与重写的区别：

***重载（OverLoad）**：方法中名字相同,参数不同（参数的数量或参数的类型）
**重写（子父类之间）**：子类重写了父类的方法
 注: 

&emsp;(1)子类中不能重写父类中的final方法

&emsp;(2)子类中必须重写父类中的abstract方法 

#### 5. Java 面向对象编程三大特性: 封装 继承 多态

##### 封装

封装把一个对象的属性私有化，同时提供一些可以被外界访问的属性的方法，如果属性不想被外界访问，我们大可不必提供方法给外界访问。但是如果一个类没有提供给外界访问的方法，那么这个类也没有什么意义了。

##### 继承

继承是使用已存在的类的定义作为基础建立新类的技术，新类的定义可以增加新的数据或新的功能，也可以用父类的功能，但不能选择性地继承父类。通过使用继承我们能够非常方便地复用以前的代码。

**关于继承如下 3 点请记住：**

1. 子类拥有父类非 private 的属性和方法。
2. 子类可以拥有自己属性和方法，即子类可以对父类进行扩展。
3. 子类可以用自己的方式实现父类的方法。（以后介绍）。

##### 多态

所谓多态就是指程序中定义的引用变量所指向的具体类型和通过该引用变量发出的方法调用在编程时并不确定，而是在程序运行期间才确定，即一个引用变量到底会指向哪个类的实例对象，该引用变量发出的方法调用到底是哪个类中实现的方法，必须在由程序运行期间才能决定。

在Java中有两种形式可以实现多态：继承（多个子类对同一方法的重写）和接口（实现接口并覆盖接口中同一方法）。

#### 6. 变量修饰符

注：protected<font color=#ff0000>对于子女、朋友（同一package下）来说，就是public的</font>；而对于其他外部class，protected就变成private。
 ![image](https://github.com/bintoYu/Note/blob/master/picture/1554799361778.png)
 
#### 7.接口和抽象类的区别
**语法层面**上的区别：

- 抽象类可以含有具体方法
- 抽象类中的成员变量可以是各种类型的，而接口中的成员变量只能是public static final类型的；
- 抽象类可以有静态代码块和静态方法，接口中不能含有静态代码块以及静态方法；
- <font color=#ff0000>一个类只能继承一个类（包括抽象类），而一个类却可以实现多个接口。</font>



**设计层面**上的区别：

- 抽象类体现的是一种继承关系，父类和子类在概念上是一致的，是“is-a”的关系；接口仅仅是“like-a”的关系。-
- <font color=#ff0000>抽象类的设计目的，是代码复用。</font>
  而接口约束了行为的有无，但是不对行为的实现进行限制，serviceImpl中的具体实现service接口并没有理会。
    
#### 8. 浅拷贝、深拷贝：
对于基本数据类型而言，就没区别。
对于对象而言，浅拷贝实际上还是对象、深拷贝就是clone方法，返回的是一个新创建的副本。

#### 9. final,finalize,finally关键字的区别：（常考）
这是三个完全不同的东西。
final：用来声明变量或类不可变。
finally：用于异常处理。
finalize：垃圾回收的方法。
#### 10.多继承问题

java类是单继承的。classB Extends classA

java接口可以多继承。**Interface3 Extends Interface0, Interface1, interface……**

不允许类多重继承的主要原因是，**如果A同时继承B和C，而B和C同时有一个D方法，A如何决定该继承那一个呢？**

但接口不存在这样的问题，接口全都是抽象方法继承谁都无所谓，所以接口可以继承多个接口。

Spring IOC 容器源码分析：https://javadoop.com/post/spring-ioc

强烈推荐，内容详尽，而且便于阅读。


## 二、 String类
#### 1. String类是final类，初始化后是不可变的(immutable)，<font color=#ff0000>相关的任何change操作都会生成新的对象。</font>
#### 2. StringPool（字符串常量池）
 每当我们创建字符串常量时，JVM会首先检查StringPool，如果该字符串已经存在池中，那么就直接返回常量池中的实例引用。如果不存在，就会实例化该字符串再将其放到常量池中。
 #### 3. 字符串的创建方式
    3.1 单独使用""引号创建的字符串都是常量，编译期就已经确定存储到String Pool（字符串常量池）中；
    3.2 使用 new String("")：常量池（如果没有）和堆中都会生成对象，然后返回堆中的地址。
    3.3 使用只包含常量的字符串连接符如"aa" + "aa"创建的也是常量,编译期就能确定,已经确定存储到String Pool中；
    3.4 使用包含变量的字符串连接符如"aa" + s1创建的对象是运行期才创建的,存储在heap中；
#### 4. 关于equals和==：String中 == 比较的是两个对象的地址，equals才是比较对象中的值。
#### 5. String为什么是final的？
保证安全性：String非常常用，如果不是final的话，便可以编写一个恶意的类来继承String。
#### 6. String c = "xx" + "yy " + a + "zz" + "mm" + b; 实质上的实现过程是： String c = new StringBuilder("xxyy ").append(a).append("zz").append("mm").append(b).toString();

### 7. String、StringBuffer、StringBuilder：
速度上：StringBuilder > StringBuffer > String（String最慢的原因在于String为常量，另外两个为变量）
    String：适用于少量的字符串操作的情况
    StringBuilder：适用于单线程下在字符缓冲区进行大量操作的情况
    StringBuffer：适用多线程下在字符缓冲区进行大量操作的情况

## 三、装箱和拆箱
Integer i = 10;  //装箱
int n = i;   //拆箱
 1. 下面这段代码的输出结果是什么？

```java
        Integer i1 = 100;
        Integer i2 = 100;
        Integer i3 = 200;
        Integer i4 = 200;
         
        System.out.println(i1==i2);
        System.out.println(i3==i4);
```
 结果：true，false。原因：在通过valueOf方法创建Integer对象的时候，<font color=#ff0000>如果数值在[-128,127]之间</font>，就直接返回对象的引用，否则new一个对象返回。
 2. 下面代码的结果为false,false，因为在某个范围内的整数是有限的，而浮点数却不是。因此每次都会new一个对象。

```java
        Double i1 = 100.0;
        Double i2 = 100.0;
        Double i3 = 200.0;
        Double i4 = 200.0;
         
        System.out.println(i1==i2);
        System.out.println(i3==i4);
```

 3. 对于Boolean而言，只要相同一定是true。
 
 ##  四、 hashCode 与 equals (重要)

面试官可能会问你：“你重写过 hashcode 和 equals 么，为什么重写equals时必须重写hashCode方法？”

- 首先，我们要确认一点，**如果两个对象相同（equals方法为true），那么他们的hashCode也是相同的**。
因为不重写的话，就会用Object.hashCode()方法，而**Object.hashCode返回的是地址本身**。
就可能出现重写了equals后使得两个对象的equals为true，但hashcode不相同的情况。

## 五、异常
 1. 所有的异常都是通过Throwable衍生出来的。
 2. Java中的检查型异常和非检查型异常有什么区别？(常考）
检查型异常和非检查型异常的主要区别在于其处理方式。检查型异常需要使用try, catch和finally关键字在编译期进行处理，否则编译器会报错。对于非检查型异常则不需要这样做。
继承java.lang.Exception  -->检查型异常
继承RuntimeException    --> 非检查型异常。
注：常见非检查型有NullPointerException和ArrayIndexOutOfBoundException等
3. 在Java异常处理的过程中，你遵循的那些最好的实践是什么？（常考）
&emsp;a）调用方法的时候<font color=#ff0000>返回布尔值来代替返回null</font>，这样可以尽可能地避免 NullPointerException。
&emsp;b）catch块里别不写代码。通常你起码要打印出异常信息，当然你最好根据需求对异常信息进行处理。
&emsp;c）<font color=#ff0000>能抛检查型异常就尽量不抛非检查型异常</font>
&emsp;d）一定要在数据库操作后，在finally块中调用close()方法
4. 为什么Java中还存在检查型异常?
因为有些异常还是需要你自己来优雅地进行处理。
5. throw 和 throws这两个关键字在java中有什么不同?
throw是用来抛出任意异常的，在方法中。而throws是在函数头中，用来标明方法可能抛出的各种异常, 
6. 什么是“异常链”？
一个异常后跑出另外一个异常，并且希望把异常原始信息保存下来，这被称为异常链。
```java
 try {
            g();
     } catch (MyException1 e) {
       e.printStackTrace();

       throw new MyException2(e);
     }
```

7. 如果执行finally代码块之前方法返回了结果，或者JVM退出了，finally块中还会执行吗？
这个问题也可以换个方式问：“**如果在try或者finally的代码块中调用了System.exit()，结果会是怎样”**。事实上，<font color=#ff0000>只有在try里面是有System.exit(0)来退出JVM的情况下finally块中的代码才不会执行</font>。

## 六、反射
#### 1. 定义:反射是Java语言的一个特性，它允许程序<font color=#ff0000>在运行时（注意不是编译的时候）</font>动态获取类的所有信息。
#### 2. 原理：
Java中创建一个类时，JVM就会创建对应的Class对象，最后加载到内存中（方法区）。
#### 3. 反射的优缺点：
优点：反射提高了程序的灵活性和扩展性。
缺点：性能不好。

#### 4. 哪里用到反射机制？
spring框架中就大量地运用了反射的机制，例如DI。

#### 5. 常用代码：
```java
 //获取class
 c = Class.forName("cn.classes.Person");
 c = Person.class

//利用class实例化对象
Person p = (Person)c.newInstance();
//有参构造
Constructor<?>[] cons = c.getConstructors();
Person p = (Person)cons[0].newInstance("xy",20);
//调用方法
c.getMethod("add").invoke(xx);
```

#### 6. 反射的优缺点:
优点：实现动态创建对象和编译，体现出很大的灵活性。
缺点：对性能有影响。


## 七、代理
#### 静态代理（即代理模式）

#### 动态代理：（spring的aop使用的就是动态代理）
1. 个人理解：client -->  代理  --> 目标object
即client不直接访问目标object，而是通过代理实现访问，代理起着中介的作用。
2. 两种方式：
（1）java动态代理： 委托类必须实现接口，只能代理<font color=#ff0000>接口</font>。通过<font color=#ff0000>反射机制</font>实现。
使用方法：
&emsp;a、创建被代理的类以及接口
&emsp;b、创建一个实现接口InvacationHandler的类，它必须实现invoke方法。
&emsp;c、通过Proxy.newProxyInstance(...)创建一个代理
&emsp;d、通过代理调用方法

```java
public interface UserService{
	void addUser();
}
public class UserServiceImpl implements UserService{
	@Override
	public void addUser(){
		System.out.println("addUser()");
	}
}
public class JDKDynamicProxy implements InvocationHandler
{
    private Object target;

    public JDKDynamicProxy(Object target) {
        this.target = target;
    }

    /**
     * 获取被代理接口实例对象
     * @param <T>
     * @return
     */
    public <T> T getProxy() {
        return (T) Proxy.newProxyInstance(target.getClass().getClassLoader(), target.getClass().getInterfaces(), this);
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("Do something before");
        Object result = method.invoke(target, args);
        System.out.println("Do something after");
        return result;
    }
}
public class Test
{
	public static void main(String[] args)
	{
	    // jdk动态代理测试
        UserService userService = new JDKDynamicProxy(new UserServiceImpl()).getProxy();
        userService.addUser();
	}
}
```

（2）cglib动态代理：对类进行代理,但需要创建一个子类对象。
####  正反向代理：
1. 正向代理（为客户端服务）：
![在这里插入图片描述](https://img-blog.csdnimg.cn/2019020210510679.png)
用途：
a、翻墙
b、学校的上网认证
c、作为缓存

2.反向代理：（反过来为内网服务，客户端是无法感知代理的存在的，nginx使用的就是反向代理技术）
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190202105217317.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_15,color_FFFFFF,t_70)
 作用：
 a、保护内网的安全
 b、负载均衡


 
## 八、补充：
#### 1.Java序列化中如果有些字段不想进行序列化 怎么办

对于不想进行序列化的变量，使用transient关键字修饰。
transient关键字的作用是：阻止实例中那些用此关键字修饰的的变量序列化；当对象被反序列化时，被transient修饰的变量值不会被持久化和恢复。transient只能修饰变量，不能修饰类和方法。

#### 2. JDK1.8 的新特性
a）HashMap中新添加了红黑树，concurrentHashMap结构改变。
b）Lambda表达式：
个人理解：目前个人喜欢用lambda来代替内部类或者内部方法，格式一般为  参数或(参数) -> (单或多条语句);
```java
//		PriorityQueue<Integer> queue = new PriorityQueue<Integer>(k, new Comparator<Integer>() {
//			  
//	        @Override
//	        public int compare(Integer o1, Integer o2) {
//	            return o2.compareTo(o1);
//	        }
//	    });

		//使用lamda表达式
		PriorityQueue<Integer> queue = new PriorityQueue<>(k,(o1,o2) -> o2.compareTo(o1));
```

c）Stream API： 详见 https://www.cnblogs.com/aoeiuv/p/5911692.html
大家可以把Stream当成一个装饰后的Iterator。
最简单例子：统计num中不为空的个数

```java 
//Lists是Guava中的一个工具类
List<Integer> nums = Lists.newArrayList(1,null,3,4,null,6);
nums.stream().filter(num -> num != null).count();
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/2019022411174939.png)
而转换Stream有以下几种方式：（不背）
① distinct
② filter
③ map
④ flatMap
⑤ peek
⑥ limit
⑦ skip
例子：

```java
List<Integer> nums = Lists.newArrayList(1,1,null,2,3,4,null,5,6,7,8,9,10);
System.out.println(“sum is:”+nums.stream().filter(num -> num != null).distinct().mapToInt(num -> num * 2).peek(System.out::println).skip(2).limit(4).sum());
