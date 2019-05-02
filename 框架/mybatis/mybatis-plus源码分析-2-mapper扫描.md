[TOC]



### 预备知识：

在spring整合mybatis时，我们需要配置以下3个东西：

- dataSource（数据源）
- sqlSessionFactory
- MapperScanner（Mapper扫描器）

其中对于MapperScanner，最合适的配置方式是使用MapperScannerConfigurer来进行配置。例如下面这个xml配置：

```xml
<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
   <property name="basePackage" value="com.study.mybatis.mapper" />
</bean>

```

这样 `MapperScannerConfigurer` 就会扫描指定basePackage下面的所有接口，并把它们注册为一个个 `MapperFactoryBean` 对象。

与此同时，MapperScannerConfigurer 还为我们提供了另外两个可以缩小搜索和注册范围的属性。一个是 annotationClass，另一个是 markerInterface 。 

- annotationClass： 当指定了annotationClass 的时候，MapperScannerConfigurer将只注册使用了 annotationClass 注解标记的接口。 -
- markerInterface： 当指定了 markerInterface 之后，MapperScannerConfigurer 将只注册继承自 markerInterface 的接口。
- 注：如果上述两个属性都指定了的话，那么 MapperScannerConfigurer 将取它们的并集，而不是交集。

示例xml配置：

```xml
<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
   <property name="basePackage" value="com.study.mybatis.mapper" />
   <property name="markerInterface" value="com.study.mybatis.mapper.SuperMapper"/>
</bean>
或
<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
   <property name="basePackage" value="com.study.mybatis.mapper" />
   <property name="annotationClass" value="com.study.mybatis.annotation.MybatisMapper"/>
</bean>

```

### 入口：

![image](<https://github.com/bintoYu/Note/tree/master/picture/mybatis-plus/3.png>)

点击进入MapperScan类：

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
@Documented
@Import({MapperScannerRegistrar.class})
@Repeatable(MapperScans.class)
public @interface MapperScan {
	...
}
```

从上面的代码中可以看到MapperScan包含了多个注解，其中最关键的是第四个注解@Import({MapperScannerRegistrar.class})，这是一个非常重要的扫描类，稍后会进行详细讲解。（注：import的作用请见：<https://www.jianshu.com/p/afd2c49394c2>）

MapperScan的内部实现：

```java
public @interface MapperScan {

  String[] value() default {};

  String[] basePackages() default {};

  Class<?>[] basePackageClasses() default {};

  Class<? extends BeanNameGenerator> nameGenerator() default BeanNameGenerator.class;

  Class<? extends Annotation> annotationClass() default Annotation.class;

  Class<?> markerInterface() default Class.class;

  String sqlSessionTemplateRef() default "";

  String sqlSessionFactoryRef() default "";

  Class<? extends MapperFactoryBean> factoryBean() default MapperFactoryBean.class;

}

```

可以看到，MapperScan由以下部分组成：

- value

- basePackages：基包，见预备知识

- basePackageClasses的Class类型

- BeanNameGenerator（未知）

- annotationClass：见预备知识

- markerInterface：见预备知识

- sqlSessionTemplateRef、sqlSessionFactoryRef

- MapperFactoryBean：mapper工厂bean，用于生成mapper工厂，请注意**默认是MapperFactoryBean.class**

  

### MapperScannerRegistrar

##### 先看MapperScannerRegistrar的声明

```java
public class MapperScannerRegistrar implements ImportBeanDefinitionRegistrar, ResourceLoaderAware 
```

- 可以看出该类实现了ImportBeanDefinitionRegistrar和ResourceLoaderAware接口。

- ImportBeanDefinitionRegistrar：实现该接口则会**自动调用registerBeanDefinitions**方法,来向Spring容器注入bean，下面会详细讲解该方法。

- ResourceLoaderAware：实现该接口，spring容器会自动注入一个resourseLoader的实现类，经过调试发现注入的ResourceLoader实际的实例是AnnotationConfigEmbeddedWebApplication这个类

  

##### registerBeanDefinitions方法

registerBeanDefinitions方法的主要功能有三：

- 从注释中获取MapperScan的各个属性的值（如果在注释中有声明的话），并设置到ClassPathMapperScanner scanner中。
- 从注释中获取value、basePackages、basePackageClasses，并存入List<String> basePackages中。
- 其**核心功能**：使用Spring框架自带的```ClassPathBeanDefinitionScanner.doScan(basePackages)```方法，将上一步获得的basePackages包下的类转化成BeanDefinition，并注册到spring容器中。

##### 具体实现：

```java
 @Override
  public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
    AnnotationAttributes mapperScanAttrs = AnnotationAttributes
        .fromMap(importingClassMetadata.getAnnotationAttributes(MapperScan.class.getName()));
    if (mapperScanAttrs != null) {
      registerBeanDefinitions(mapperScanAttrs, registry);
    }
  }

  void registerBeanDefinitions(AnnotationAttributes annoAttrs, BeanDefinitionRegistry registry) {

    ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);

    // this check is needed in Spring 3.1
    Optional.ofNullable(resourceLoader).ifPresent(scanner::setResourceLoader);

    //从注释中获取MapperScan的各个属性的值（如果在注释中有声明的话），并设置到ClassPathMapperScanner scanner中。
    Class<? extends Annotation> annotationClass = annoAttrs.getClass("annotationClass");
    if (!Annotation.class.equals(annotationClass)) {
      scanner.setAnnotationClass(annotationClass);
    }
    Class<?> markerInterface = annoAttrs.getClass("markerInterface");
    if (!Class.class.equals(markerInterface)) {
      scanner.setMarkerInterface(markerInterface);
    }
    Class<? extends BeanNameGenerator> generatorClass = annoAttrs.getClass("nameGenerator");
    if (!BeanNameGenerator.class.equals(generatorClass)) {
      scanner.setBeanNameGenerator(BeanUtils.instantiateClass(generatorClass));
    }
    Class<? extends MapperFactoryBean> mapperFactoryBeanClass = annoAttrs.getClass("factoryBean");
    if (!MapperFactoryBean.class.equals(mapperFactoryBeanClass)) {
      scanner.setMapperFactoryBeanClass(mapperFactoryBeanClass);
    }
    scanner.setSqlSessionTemplateBeanName(annoAttrs.getString("sqlSessionTemplateRef"));
    scanner.setSqlSessionFactoryBeanName(annoAttrs.getString("sqlSessionFactoryRef"));

    //从注释中获取value、basePackages、basePackageClasses，并存入List<String> basePackages中。
    List<String> basePackages = new ArrayList<>();
    basePackages.addAll(
        Arrays.stream(annoAttrs.getStringArray("value"))
            .filter(StringUtils::hasText)
            .collect(Collectors.toList()));

    basePackages.addAll(
        Arrays.stream(annoAttrs.getStringArray("basePackages"))
            .filter(StringUtils::hasText)
            .collect(Collectors.toList()));

    basePackages.addAll(
        Arrays.stream(annoAttrs.getClassArray("basePackageClasses"))
            .map(ClassUtils::getPackageName)
            .collect(Collectors.toList()));

    scanner.registerFilters();
    
    //核心代码，祥见下
    scanner.doScan(StringUtils.toStringArray(basePackages));
  }
```

##### scan.doScan

```java
  @Override
  public Set<BeanDefinitionHolder> doScan(String... basePackages) {
     //核心super.doScan()：扫描basePackages，得到package中所有类的beanDefinitions，并将beanDefinitions注册到spring容器中。
    Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);

    if (beanDefinitions.isEmpty()) {
      LOGGER.warn(() -> "No MyBatis mapper was found in '" + Arrays.toString(basePackages) + "' package. Please check your configuration.");
    } else {
        
      //为所有beanDefinition设置了bean的名称、以及之前提到mapperFactoryBeanClass、addToConfig、sqlSessionFactory以及sqlSessionTemplate
      processBeanDefinitions(beanDefinitions);
    }

    return beanDefinitions;
  }


```

首先，ClassPathBeanDefinitionScanner.doScan(basePackages)方法，该方法会对之前存入的List<String> basePackages进行扫描，得到packages中所有类的beanDefinitions，并将beanDefinitions注册到容器中。

关于ClassPathBeanDefinitionScanner.doScan(basePackages)方法的具体介绍，详见：

<https://www.jianshu.com/p/d5ffdccc4f5d>

接着，调用processBeanDefinitions方法对获得的beanDefinitions进行处理：

```java
  private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
    GenericBeanDefinition definition;
    for (BeanDefinitionHolder holder : beanDefinitions) {
      definition = (GenericBeanDefinition) holder.getBeanDefinition();
      String beanClassName = definition.getBeanClassName();

      //最最最关键的两行代码：
      //设置构造器的参数为beanClassName，通过构造器注入接口字段为beanClassName（即设置为实际的标注了@Mapper的接口）
      definition.getConstructorArgumentValues().addGenericArgumentValue(beanClassName);
      //设置beanClass为mapperFactoryBean.class，这个mapperFactoryBean尤为重要
      //当spring注入这个definition的时候，实际上调用的是该MapperFactoryBean的getObject()方法来获得特定的mapper实例
      //这个mapper接口具体的实现类是由beanClassName来设置的。
      definition.setBeanClass(this.mapperFactoryBeanClass);

      //设置addToConfig，sqlSessionFactory，sqlSessionTemplate
       ...
           
       if (!explicitFactoryUsed) {
        definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
      }
    }
```

该方法主要就是为所有beanDefinition设置了之前提到的各种属性，最关键的是中间两行代码。

做的事情其实是重写BeanDefinition的BeanClass字段为MapperFactoryBean.class，并且将beanClass其实也就是MapperFactoryBean的构造器参数设置为实际的标注了@Mapper的接口。

这么做的原因为：当我们如下图注入Mapper接口时

```java
    @Autowired
    private UserMapper userMapper;

```

**实际调用的是MapperFactoryBean中的getObject()获取特定的mapper实例**。因此接下来我们便会对MapperFactoryBean进行讲解。



### 小总结及困惑

至此，我们知道了使用@MapperScan注释时，可以达到以下结果：

- 生成一个ClassPathMapperScanner scanner，并设置好annotationClass等属性
- 将basePackages下的所有mapper类都会转化成beanDefinitions并注册到spring容器中

存在的困惑：

- 生成ClassPathMapperScanner scanner并设置好annotationClass等属性后，这个scanner用在哪？是直接注入到MapperScan还是？(未解决)

### MapperFactoryBean

本专题涉及的类的结构图如下：

![image](<https://github.com/bintoYu/Note/tree/master/picture/mybatis-plus/4.png>)

##### 声明：

```java
public class MapperFactoryBean<T> extends SqlSessionDaoSupport implements FactoryBean<T> {
    private Class<T> mapperInterface;
    ...

```



它是继承自SqlSessionSupport这个类并实现FactoryBean这个接口，而参数mapperInterface实际上就是通过上面的```definition.getConstructorArgumentValues().addGenericArgumentValue(beanClassName);```这段代码注入的。

##### getObject()获得mapper实例

```java
@Override
public T getObject() throws Exception {
  return getSqlSession().getMapper(this.mapperInterface);
}

```

从这里看出是通过getSqlSession().getMapper方法得到的mapper实例，而getSqlSession()实际上是类SqlSessionDaoSupport的方法。

```java
public abstract class SqlSessionDaoSupport extends DaoSupport {

   //注：这个是类SqlSessionDaoSupport的方法
   public SqlSession getSqlSession() {
   	  return this.sqlSessionTemplate;
   }
}

```

可以看出，getSqlSession()就是获得sqlSessionTemplate，而sqlSessionTemplate从何而来？

```java
private SqlSessionTemplate sqlSessionTemplate;

//sqlSessionTemplate是通过sqlSessionFactory获得。
public void setSqlSessionFactory(SqlSessionFactory sqlSessionFactory) {
  if (this.sqlSessionTemplate == null || sqlSessionFactory != this.sqlSessionTemplate.getSqlSessionFactory()) {
    this.sqlSessionTemplate = createSqlSessionTemplate(sqlSessionFactory);
  }
}

```

可以看到，sqlSessionTemplate是通过sqlSessionFactory所获得。对于sqlSessionFactory的产生，请见上一篇博客**“mybatis-plus源码分析-1-sqlSessionFactory的生成”**



当我们接着看getSqlSession().getMapper()方法时 ```  <T> T getMapper(Class<T> var1);```  ，发现SqlSession是一个接口，getMapper只是一个定义。

作者本人没有找到getMapper的具体实现，但在下面所讲的mapper的注册中有了一些猜测，猜测是从getSqlSession().getConfiguration()中获取的mapper。



##### 将mapper进行注册

从上图中可以看到MapperFactoryBean实现了InitializingBean方法，因此在初始化MapperFactoryBean时，会调用afterPropertiesSet这个方法，然而MapperFactoryBean并没有这个方法，因此一直向上翻代码，发现其祖父DaoSupport实现了afterPropertiesSet()：

```java
public abstract class DaoSupport implements InitializingBean {
	...
    public final void afterPropertiesSet() throws IllegalArgumentException, BeanInitializationException {
        this.checkDaoConfig();

        try {
            this.initDao();
        } catch (Exception var2) {
            throw new BeanInitializationException("Initialization of DAO failed", var2);
        }
    }
	...
}

```

可以看到，在初始化MapperFactoryBean时，会调用checkDaoConfig方法，接下来看该方法：

```java
  @Override
  protected void checkDaoConfig() {
    super.checkDaoConfig();

    notNull(this.mapperInterface, "Property 'mapperInterface' is required");

    Configuration configuration = getSqlSession().getConfiguration();
    if (this.addToConfig && !configuration.hasMapper(this.mapperInterface)) {
      try {
        //最核心的地方！！！！
        configuration.addMapper(this.mapperInterface);
      } catch (Exception e) {
        logger.error("Error while adding the mapper '" + this.mapperInterface + "' to configuration.", e);
        throw new IllegalArgumentException(e);
      } finally {
        ErrorContext.instance().reset();
      }
    }
  }

```

**可以看到，在该方法中，它将MapperFactoryBean中的mapperInterface存入到了SqlSession的Configuration里！**

这也是上文本人猜测是从getSqlSession().getConfiguration()中获取mapper实例的依据。

对于initDao()方法，本人并未找到有代码的具体实现。



##### MapperFactoryBean的总结及困惑

总结如图：

![image](<https://github.com/bintoYu/Note/tree/master/picture/mybatis-plus/MapperFactoryBean.png>)

困惑：

- mapperInterface是如何注入进去的？



