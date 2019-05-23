Spring的核心是控制反转（IoC）和面向切面（AOP）
# IoC
为什么叫控制反转？ https://www.cnblogs.com/wang-meng/p/5597490.html



IoC的使用方式：
① 将所有的bean都配置xml中：`<bean id="" class="">`
② 将所有的依赖都使用注解：@Autowired
需要对bean使用@Component，以及在xml中配置扫描器<context:component-scan base-package=" ">



IOC核心源码：<https://yikun.github.io/2015/05/29/Spring-IOC%E6%A0%B8%E5%BF%83%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0/>



Spring IOC 容器源码分析：https://javadoop.com/post/spring-ioc

强烈推荐，内容详尽，而且便于阅读。



IOC的原理（工厂模式+反射技术）
3.1 IOC容器的初始化过程（bean的注册）：
(1)xml读取，解析成beanDefinition，最后在beanFactory中进行注册。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190204140001505.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190204143651795.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)
(2)注册：Spring容器内部有一个HashMap ，存放着bean的信息，以后对 bean 的操作都是围绕这个HashMap 来实现的。![在这里插入图片描述](https://img-blog.csdnimg.cn/20190204144322168.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)
3.2 依赖注入
当Spring IoC容器完成了初始化后，IoC容器中已经管理类Bean定义的相关数据，但是此时IoC容器还没有对所管理的Bean进行依赖注入，在spring IOC设计中，**bean的注册和依赖注入是两个过程**，依赖注入在以下两种情况发生：
 a.用户第一次通过getBean方法向IoC容器索要Bean时，IoC容器触发依赖注入。
 b.当用户在Bean定义资源中为<Bean>元素配置了lazy-init属性，即让容器在解析注册Bean定义时进行预实例化，触发依赖注入。

#### beanFactory和factoryBean：
1. BeanFactory：就是生产bean的工厂，用于生成任意bean。
2. FactoryBean：特殊bean，用于生成特定的bean。FactoryBean在IOC容器的基础上给Bean的实现加上了一个**简单工厂模式和装饰模式**，这样便可以在getObject()方法中对要生成的bean进行灵活配置，比方说生成bean的前后都可以加入一些代码。
   需要注意的是：getBean("factoryBean"); **返回的不是这个工厂bean，而是它生产的具体bean**。
   使用：比方说在生成前加一段话。

```
public class ProductFactory implements FactoryBean<Object> {
	...
	    @Override
    public Object getObject() throws Exception {
        logger.debug("getObject......");
        return new Product();
    }

    @Override
    public Class<?> getObjectType() {
        return Product.class;
    }

    @Override
    public boolean isSingleton() {
        return true;
    }
	...
}
```
配置：

```
 <bean id="ProductFactory" class="com.xxx.ProductFactory"/> 
```

# AOP：
1. 好处：降低耦合度、代码复用
2. 采取横向抽取机制，取代了传统纵向继承
3. 一句话：个人理解，aop就是使用**代理机制**，，对目标类的切入点进行增强。
4. 代理机制有**jdk动态代理和cglib增强**两种

5. AOP术语【掌握】
     1.target：目标类，需要被代理的类。例如：UserService
     2.Joinpoint(连接点):所谓连接点是指那些可能被拦截到的方法。例如：所有的方法
     3.PointCut 切入点：已经被增强的连接点。例如：addUser()
     4.advice 通知/增强，增强代码。例如：after、before
     5.Weaving(织入):是指把增强advice应用到目标对象target来创建新的代理对象proxy的过程.
     6.proxy 代理类
     7.Aspect(切面): 是切入点pointcut和通知advice的结合
     一个线是一个特殊的面。
     一个切入点和一个通知，组成成一个特殊的面。
     ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190204134523801.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)
 6. AOP的两种实现方式：基于配置的（xml）和基于注释（AspectJ）
     基于配置：定义接口和实现类 -> 定义切面 ->　xml配置目标类、切面和aop:config

目标类（接口和实现类）
```
public interface UserService{
    public void addUser();
}

public class UserServiceImpl implements UserService{
    @Override
    public void addUser() {
        System.out.println("addUser! ");
    }
}
```

 切面类：

```java
public class MyAspect implements MethodInterceptor {
	@Override
	public Object invoke(MethodInvocation mi) throws Throwable {
		
		System.out.println("前3");
		//手动执行目标方法
		Object obj = mi.proceed();
		System.out.println("后3");
		return obj;
	}
}
```
xml配置（需要添加schema）：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans 
       					   http://www.springframework.org/schema/beans/spring-beans.xsd
       					   http://www.springframework.org/schema/aop 
       					   http://www.springframework.org/schema/aop/spring-aop.xsd">
	<!-- 1 创建目标类 -->
	<bean id="userServiceId" class="com.itheima.c_spring_aop.UserServiceImpl"></bean>
	<!-- 2 创建切面类（通知） -->
	<bean id="myAspectId" class="com.itheima.c_spring_aop.MyAspect"></bean>
	<!-- 3 aop编程 
		3.1 导入命名空间
		3.2 使用 <aop:config>进行配置
				proxy-target-class="true" 声明时使用cglib代理
			<aop:pointcut> 切入点 ，从目标对象获得具体方法
			<aop:advisor> 特殊的切面，只有一个通知 和 一个切入点
				advice-ref 通知引用
				pointcut-ref 切入点引用
		3.3 切入点表达式
			execution(* com.itheima.c_spring_aop.*.*(..))
			选择方法         返回值任意   包             类名任意   方法名任意   参数任意
		
	-->
	<aop:config proxy-target-class="true">
		<aop:pointcut expression="execution(* com.itheima.c_spring_aop.*.*(..))" id="myPointCut"/>
		<aop:advisor advice-ref="myAspectId" pointcut-ref="myPointCut"/>
	</aop:config>
</beans>

```

基于注释：定义接口和实现类 -> 定义切面：使用@Aspect，并直接在类中定义切入点和增强

```java
@Component
@Aspect
public class Operator {
    
    @Pointcut("execution(* com.aijava.springcode.service..*.*(..))")
    public void pointCut(){}
    
    @Before("pointCut()")
    public void doBefore(JoinPoint joinPoint){
        System.out.println("AOP Before Advice...");
    }
    
    @After("pointCut()")
    public void doAfter(JoinPoint joinPoint){
        System.out.println("AOP After Advice...");
    }
    
    @AfterReturning(pointcut="pointCut()",returning="returnVal")
    public void afterReturn(JoinPoint joinPoint,Object returnVal){
        System.out.println("AOP AfterReturning Advice:" + returnVal);
    }
    
    @AfterThrowing(pointcut="pointCut()",throwing="error")
    public void afterThrowing(JoinPoint joinPoint,Throwable error){
        System.out.println("AOP AfterThrowing Advice..." + error);
        System.out.println("AfterThrowing...");
    }
    
    @Around("pointCut()")
    public void around(ProceedingJoinPoint pjp){
        System.out.println("AOP Aronud before...");
        try {
            pjp.proceed();
        } catch (Throwable e) {
            e.printStackTrace();
        }
        System.out.println("AOP Aronud after...");
    }
    
}
```

  5. 通知类型：
  前置通知（Before advice）：在某连接点之前执行的通知，但这个通知不能阻止连接点之前的执行流程（除非它抛出一个异常）。
  后置通知（After returning advice）：在某连接点正常完成后执行的通知：例如，一个方法没有抛出任何异常，正常返回。
  环绕通知（Around Advice）：包围一个连接点的通知，如方法调用。这是最强大的一种通知类型。环绕通知可以在方法调用前后完成自定义的行为。它也会选择是否继续执行连接点或直接返回它自己的返回值或抛出异常来结束执行。
  异常通知（After throwing advice）：在方法抛出异常退出时执行的通知。
  最终通知（After (finally) advice）：当某连接点退出的时候执行的通知（不论是正常返回还是异常退出）。

## AOP之事物（了解）：

1. 一个事务过程中只会有一个Connetion对象！

2. 事务的传播属性（**一般选择Required，读操作选择Supports**）
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190204171132745.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)
3. 使用（对service层进行aop增强）

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:context="http://www.springframework.org/schema/context" xmlns:p="http://www.springframework.org/schema/p"
	xmlns:aop="http://www.springframework.org/schema/aop" xmlns:tx="http://www.springframework.org/schema/tx"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.0.xsd
	http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.0.xsd
	http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-4.0.xsd http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx-4.0.xsd
	http://www.springframework.org/schema/util http://www.springframework.org/schema/util/spring-util-4.0.xsd">

	<!-- 事务配置 -->
	<!-- 事务管理器 -->
	<bean id="transactionManager"
		class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
		<!-- 数据源 -->
		<property name="dataSource" ref="dataSource" />
	</bean>
	<!-- 通知 -->
	<tx:advice id="txAdvice" transaction-manager="transactionManager">
		<tx:attributes>
			<!-- 传播行为 -->
			<tx:method name="save*" propagation="REQUIRED" />
			<tx:method name="insert*" propagation="REQUIRED" />
			<tx:method name="add*" propagation="REQUIRED" />
			<tx:method name="create*" propagation="REQUIRED" />
			<tx:method name="delete*" propagation="REQUIRED" />
			<tx:method name="update*" propagation="REQUIRED" />
			<tx:method name="find*" propagation="SUPPORTS" read-only="true" />
			<tx:method name="select*" propagation="SUPPORTS" read-only="true" />
			<tx:method name="get*" propagation="SUPPORTS" read-only="true" />
		</tx:attributes>
	</tx:advice>
	<!-- 切面 -->
	<aop:config>
		<aop:advisor advice-ref="txAdvice"
			pointcut="execution(* com.YYSchedule.store.service.*.*(..))" />
	</aop:config>
</beans>
```

#### Springmvc

主要流程是：
处理器映射器（请求获取Action）  --> 处理器适配器（执行action，返回modelandview）  --> 试图解析器（解析成view） -->  view层（渲染）  -->用户 
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190308163732531.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)