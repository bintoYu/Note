**注：本篇本人看的较为混乱，而且个人觉得实际收获不是很大，因此可以选择跳过，直接去看源码分析-2-MapperScanner。**



### 预备知识：

在spring整合mybatis时，我们需要配置以下3个东西：

- dataSource（数据源）
- sqlSessionFactory
- MapperScanner（Mapper扫描器）

本次将介绍的是sqlSessionFactory的生成，在讲解具体代码之前，先列出mybatis-plus的SqlSessionFactory包含的6种重要属性，以方便后续代码理解：

```xml
<bean id="sqlSessionFactory"  
 class="com.baomidou.mybatisplus.spring.MybatisSqlSessionFactoryBean">  
 <!-- 配置数据源 -->  
 <property name="dataSource" ref="dataSource" />  
 <!-- 自动扫描 Xml 文件位置 -->  
 <property name="mapperLocations" value="classpath*:com/ds/orm/mapper/**/*.xml" />  
 <!-- 配置 Mybatis 配置文件（可无） -->  
 <property name="configLocation" value="classpath:mybatis-config.xml" />  
 <!-- 配置包别名，支持通配符 * 或者 ; 分割 -->  
 <property name="typeAliasesPackage" value="com.ds.orm.model" />  
 <!-- 枚举属性配置扫描，支持通配符 * 或者 ; 分割 -->  
 <property name="typeEnumsPackage" value="com.baomidou.springmvc.entity.*.enums" /> 

 <!-- MP 全局配置注入 -->  
 <property name="globalConfig" ref="globalConfig" />  
</bean>  
```



### 提前总结：流程图

![1556618422137](<https://github.com/bintoYu/Note/raw/master/picture/mybatis-plus/sqlSessionFactory.png>)



### 正式开始

第一步：项目中引入mybatis-plus：

> ```xml
> pom.xml
> 	<dependency>
>   <groupId>com.baomidou</groupId>
>   <artifactId>mybatis-plus-boot-starter</artifactId>
>   <version>3.1.1</version>
>   </dependency>
> ```
>
> 第二步：可以从扩展库（external Libraries）中查看到mybatis-plus-boot-starter的源码。

![image](<https://github.com/bintoYu/Note/raw/master/picture/mybatis-plus/1.png>)

红框部分spring.factories中记录了spring-boot-starter类项目的入口为MybatisPlusAutoConfiguration：

```properties
spring.factories:
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.baomidou.mybatisplus.autoconfigure.MybatisPlusAutoConfiguration
```



接下来查看MybatisPlusAutoConfiguration.java是如何完成初始化配置的。

从注解的角度看：

![image](https://github.com/bintoYu/Note/raw/master/picture/mybatis-plus/2.png)

- @Configuration是将该类加入spring容器当中，

- @ConditionalOnClass({SqlSessionFactory.class, MybatisSqlSessionFactoryBean.class})

  SqlSessionFactory，MybatisSqlSessionFactoryBean类的依赖存在。

- @ConditionalOnSingleCandidate(DataSource.class)  DataSource这个实例必须存在

- @AutoConfigureAfter(DataSourceAutoConfiguration.class) 其他的类加载完之后，再加载DataSourceAutoConfiguration这个类，它主要是完成数据配置初始化。

  

阅读代码后发现，该类的核心便是配置SqlSessionFactory，因此接下来进行具体了解：

> ### 配置SqlSessionFactory
>
> 代码如下：
>
> ```java
> @Bean
> @ConditionalOnMissingBean
> public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
> // TODO 使用 MybatisSqlSessionFactoryBean 而不是 SqlSessionFactoryBean
> MybatisSqlSessionFactoryBean factory = new MybatisSqlSessionFactoryBean();
> factory.setDataSource(dataSource);
> factory.setVfs(SpringBootVFS.class);
> //设置configurationLocation到factory中
> if (StringUtils.hasText(this.properties.getConfigLocation())) {
>    factory.setConfigLocation(this.resourceLoader.getResource(this.properties.getConfigLocation()));
> }
> 
> //设置configuration到factory中，后面会用到！
> applyConfiguration(factory);
> 
> if (this.properties.getConfigurationProperties() != null) {
>    factory.setConfigurationProperties(this.properties.getConfigurationProperties());
> 
>    
> //设置TypeAliasesPackage、TypeAliasesSuperType、TypeHandlersPackage、MapperLocations
>   if (StringUtils.hasLength(this.properties.getTypeAliasesPackage())) {
>       factory.setTypeAliasesPackage(this.properties.getTypeAliasesPackage());
>   }
>   if (this.properties.getTypeAliasesSuperType() != null) {
>       factory.setTypeAliasesSuperType(this.properties.getTypeAliasesSuperType());
>   }
>   if (StringUtils.hasLength(this.properties.getTypeHandlersPackage())) {
>       factory.setTypeHandlersPackage(this.properties.getTypeHandlersPackage());
>   }
>   if (!ObjectUtils.isEmpty(this.properties.resolveMapperLocations())) {
>       factory.setMapperLocations(this.properties.resolveMapperLocations());
>   }         
>    
> //...设置其他属性到factory中，代码略
> 	 ...
> ...    
> 
>    
> factory.setGlobalConfig(globalConfig);
> 	return factory.getObject();
> }
> ```

先看注释和方法声明：@ConditionalOnMissingBean是在Spring容器中缺少bean的时候,创建SqlSessionFactory这个对象，DataSource这个对象会在这个方法中会自动注入进来，这是Spring的IOC来完成的。

再看具体实现：创建一个MybatisSqlSessionFactoryBean的实例，它是实现Spring中FactoryBean接口的类，然后在这个实例中设置DataSource,VFS,ConfigLocation,MybatisConfiguraition等属性。

```java
    private void applyConfiguration(MybatisSqlSessionFactoryBean factory) {
        // 取出configuration
        MybatisConfiguration configuration = this.properties.getConfiguration();
        //若configuration为空，new或者自定义生成。
        if (configuration == null && !StringUtils.hasText(this.properties.getConfigLocation())) {
            configuration = new MybatisConfiguration();
        }
        if (configuration != null && !CollectionUtils.isEmpty(this.configurationCustomizers)) {
            for (ConfigurationCustomizer customizer : this.configurationCustomizers) {
                customizer.customize(configuration);
            }
        }
        //设置到factory中
        factory.setConfiguration(configuration);
    }
```

接下来就是设置各种属性。（代码略）

需要注意的是，该方法返回的值是 ``` return factory.getObject(); ``` 因此我们接着查看 factory.getObject()方法。

```java
    public SqlSessionFactory getObject() throws Exception {
        if (this.sqlSessionFactory == null) {
            this.afterPropertiesSet();
        }

        return this.sqlSessionFactory;
    }
```

可以看到getObject获取SqlSessionFacoty,会调用afterPropertiesSet()，代码如下：

```java
    public void afterPropertiesSet() throws Exception {
        Assert.notNull(this.dataSource, "Property 'dataSource' is required");
        Assert.notNull(this.sqlSessionFactoryBuilder, "Property 'sqlSessionFactoryBuilder' is required");
        Assert.state(this.configuration == null && this.configLocation == null || this.configuration == null || this.configLocation == null, "Property 'configuration' and 'configLocation' can not specified with together");
        this.sqlSessionFactory = this.buildSqlSessionFactory();
    }
```

可以看到afterPropertiesSet紧接着会调用 buildSqlSessionFactory()方法，该方法便用来构造SqlSessionFactory，代码很长，我们一点一点分析：



#### 构建SqlSessionFactory

下面的代码

```java
protected SqlSessionFactory buildSqlSessionFactory() throws Exception {
    	//第一部分：将configuration、configurationProperties以及globalConfig设置到targetConfiguration中
        MybatisXMLConfigBuilder xmlConfigBuilder = null;
        MybatisConfiguration targetConfiguration;
        //因为前面有设置configuration，所以一定会进入这一个if
    	if (this.configuration != null) 
            targetConfiguration = this.configuration;
            if (targetConfiguration.getVariables() == null) {
                targetConfiguration.setVariables(this.configurationProperties);
            } else if (this.configurationProperties != null) {
                targetConfiguration.getVariables().putAll(this.configurationProperties);
            }
        } else if (this.configLocation != null) {
            xmlConfigBuilder = new MybatisXMLConfigBuilder(this.configLocation.getInputStream(), (String)null, this.configurationProperties);
            targetConfiguration = xmlConfigBuilder.getConfiguration();
        } else {
            //略
        }
    
        this.globalConfig = (GlobalConfig)Optional.ofNullable(this.globalConfig).orElseGet(GlobalConfigUtils::defaults);
        this.globalConfig.setDbConfig((DbConfig)Optional.ofNullable(this.globalConfig.getDbConfig()).orElseGet(DbConfig::new));
        targetConfiguration.setGlobalConfig(this.globalConfig);
        if (targetConfiguration.isMapUnderscoreToCamelCase()) {
            targetConfiguration.setObjectWrapperFactory(new MybatisMapWrapperFactory());
        }
```

这一整块代码的唯一功能：将configuration、configurationProperties以及globalConfig设置到targetConfiguration中。

因为前文提到了configuration已经设置到factory中，因此一定会进入第一个if，而第一个if的作用是将configuration和configurationProperties设置到targetConfiguration中。



下面一段代码主要功能为：获取枚举属性（typeEnumsPackage）以及包别名（typeAliasesPackage），并进行**注册**。

枚举的注册，就是把其存入Map<JdbcType, TypeHandler<?>> map这一内存中。

别名的注册，是将别名（例如user）及其类型(例如User.class）存入 Map<String, Class<?>> typeAliases这一内存中。

```java
        if (StringUtils.hasLength(this.typeEnumsPackage)) {
            //这一整个if-else的功能：生成typeEnumsPackage的class文件，并存入Set<Class<?>> classes中。
            Object classes;
            //假如包名包含*或,或; 例如：com.study.*
            if (this.typeEnumsPackage.contains("*") && !this.typeEnumsPackage.contains(",") && !this.typeEnumsPackage.contains(";")) {
                //1.将包名com.study.*转化成具体路径：classpath*: com/study/*.class
                //2.根据具体路径生成Class类，然后存入Set<Class<?>>中，返回该Set
                classes = PackageHelper.scanTypePackage(this.typeEnumsPackage);
                if (((Set)classes).isEmpty()) {
                    LOGGER.warn(() -> {
                        return "Can't find class in '[" + this.typeEnumsPackage + "]' package. Please check your configuration.";
                    });
                }
            } else {
                //1.把报名按照",; \t\n"进行切割。
                //2.切割后步骤同上：转化成具体路径然后存入Set<Class<?>>中
                String[] typeEnumsPackageArray = StringUtils.tokenizeToStringArray(this.typeEnumsPackage, ",; \t\n");
                com.baomidou.mybatisplus.core.toolkit.Assert.notNull(typeEnumsPackageArray, "not find typeEnumsPackage:" + this.typeEnumsPackage, new Object[0]);
                classes = new HashSet();
                Stream.of(typeEnumsPackageArray).forEach((typePackage) -> {
                    Set<Class<?>> scanTypePackage = PackageHelper.scanTypePackage(typePackage);
                    if (scanTypePackage.isEmpty()) {
                        LOGGER.warn(() -> {
                            return "Can't find class in '[" + typePackage + "]' package. Please check your configuration.";
                        });
                    } else {
                        classes.addAll(PackageHelper.scanTypePackage(typePackage));
                    }

                });
            }

            //遍历Set<Class<?>> classes，并注册到typeHandlerRegistry中。
            //所谓的注册，就是把其存入并注册到typeHandlerRegistry中的Map<JdbcType, TypeHandler<?>> map这一内存中。
            TypeHandlerRegistry typeHandlerRegistry = targetConfiguration.getTypeHandlerRegistry();
            ((Set)classes).stream().filter(Class::isEnum).filter((cls) -> {
                return IEnum.class.isAssignableFrom(cls) || EnumTypeHandler.dealEnumType(cls).isPresent();
            }).forEach((cls) -> {
                typeHandlerRegistry.register(cls, EnumTypeHandler.class);
            });
        }

		//生成包别名（typeAliasesPackage）
        if (StringUtils.hasLength(this.typeAliasesPackage)) {
            Set var10000 = this.scanClasses(this.typeAliasesPackage, this.typeAliasesSuperType);
            TypeAliasRegistry var10001 = targetConfiguration.getTypeAliasRegistry();
            var10000.forEach(var10001::registerAlias);
        }

		//注册别名到TypeAliasRegistry中
        if (!ObjectUtils.isEmpty(this.typeAliases)) {
            Stream.of(this.typeAliases).forEach((typeAlias) -> {
                targetConfiguration.getTypeAliasRegistry().registerAlias(typeAlias);
                LOGGER.debug(() -> {
                    return "Registered type alias: '" + typeAlias + "'";
                });
            });
        }

```

​		

紧接着配置plugins、typeHandlersPackage、typeHandlers、DatabaseId以及事物工厂TransactionFactory到targetConfiguration中。（代码略）

最后对于属性mapperLocation，使用了**ibatis自带的XMLMapperBuilder.parse()方法来验证mapperLocation的合法性**

```java
        if (this.mapperLocations != null) {
            if (this.mapperLocations.length == 0) {
                LOGGER.warn(() -> {
                    return "Property 'mapperLocations' was specified but matching resources are not found.";
                });
            } else {
                Resource[] var50 = this.mapperLocations;
                int var54 = var50.length;

                for(int var5 = 0; var5 < var54; ++var5) {
                    Resource mapperLocation = var50[var5];
                    if (mapperLocation != null) {
                        try {
                            //核心代码：用到的是ibatis自带的XMLMapperBuilder.parse()方法来验证mapperLocation的合法性
                            XMLMapperBuilder xmlMapperBuilder = new XMLMapperBuilder(mapperLocation.getInputStream(), targetConfiguration, mapperLocation.toString(), targetConfiguration.getSqlFragments());
                            xmlMapperBuilder.parse();
                        } catch (Exception var41) {
                            throw new NestedIOException("Failed to parse mapping resource: '" + mapperLocation + "'", var41);
                        } finally {
                            ErrorContext.instance().reset();
                        }

                        LOGGER.debug(() -> {
                            return "Parsed mapper file: '" + mapperLocation + "'";
                        });
                    }
                }
            }
        } 

```



