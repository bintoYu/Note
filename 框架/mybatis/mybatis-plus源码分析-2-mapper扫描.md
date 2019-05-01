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

接着，调用processBeanDefinitions方法对获得的beanDefinitions进行处理：该方法主要就是为所有beanDefinition设置了bean的名称、以及之前提到mapperFactoryBeanClass、addToConfig、sqlSessionFactory以及sqlSessionTemplate，不是研究重点，因此不列出代码。

关于ClassPathBeanDefinitionScanner.doScan(basePackages)方法的具体介绍，详见：

<https://www.jianshu.com/p/d5ffdccc4f5d>



### 总结

至此，我们知道了使用@MapperScan注释时，可以达到以下结果：

- 生成一个ClassPathMapperScanner scanner，并设置好annotationClass等属性
- 将basePackages下的所有类都会转化成beanDefinitions并注册到spring容器中



### 存在的困惑

看完代码还有一个问题不明白： 生成ClassPathMapperScanner scanner并设置好annotationClass等属性后，这个scanner用在哪？是直接注入到MapperScan还是？