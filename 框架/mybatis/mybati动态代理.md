
@[toc]

### mybatis mapper的使用

首先，我们来看看，对于原生的mybatis，使用mapper的步骤：
1、编写xxxMapper.xml以及xxxMapper接口

```xml
<mapper namespace="com.test.UserMapper">  
<!-- 注意这里的 namespace必须对应着map接口的全类名-->  
    <select id="findUserById" parameterType="int" resultType="user">  
        select * from user where id = #{id}  
    </select>  

    <select id="findUserByUsername" parameterType="java.lang.String"  
        resultType="user">  
        select * from user where username like '%${value}%'  
    </select>  

    <insert id="insertUser" parameterType="user">  
        <selectKey keyProperty="id" order="AFTER" resultType="java.lang.Integer">  
            select LAST_INSERT_ID()  
        </selectKey>  
        insert into user(username,birthday,sex,address)  
        values(#{username},#{birthday},#{sex},#{address})  
    </insert>  
</mapper> 
```

```java
public interface UserMapper {
    public User findUserById(int id);

	public User findUserByUsername(String userName);

    public void insertUser(User user);
}
```

2、在mybatis配置文件中配置mapper.xml的位置

```xml
    <mappers>
        <mapper resource="com/test/UserMapper.xml"/>
    </mappers>
```

3、获得sqlSession
4、sqlSession.getMapper(type)获取mapper接口，然后调用方法。

```java
public User findUserById(Integer id) {
        SqlSession sqlSession = MyBatisSqlSessionFactory.getSqlSession();
        try {
            UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
            return userMapper.findUserById(id);
        } finally {
            sqlSession.close();
        }
    }
```

### 提出问题

1. UserMapper是一个接口，明明我们也没有编写实现类，为什么就能直接使用？
2. 可以看出UserMapper使用时调用了UserMapper.xml中的sql语句，这一过程是怎么实现的？

对于第一点，我们使用Mapper代理类来对UserMapper这个接口进行代理，实现接口中定义的方法。
对于第二点，实际上mybatis在初始化的过程中便会对xml文件进行解析，并将解析结果存入sqlSessionFactory中。

### 总流程图

未学习mapper方法时，我们是使用sqlSession来执行sql语句。（详见mybatis基础）
实际上，只要是mybatis，最后的最后一定还是由sqlSession来执行sql语句的。
所以从流程图中可以看到，动态代理的最后还是委托给了sqlSession。
总流程图：（个人理解）
![在这里插入图片描述](<https://github.com/bintoYu/Note/raw/master/picture/mybatis%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86.png>)

### mapper.xml解析

mapper.xml解析使用到了XMLMapperBuilder.parse()方法:

```java
    public void parse() {
        if (!this.configuration.isResourceLoaded(this.resource)) {
            //关注mapper.xml的解析、里面解析了MappedStatement
            this.configurationElement(this.parser.evalNode("/mapper"));
            this.configuration.addLoadedResource(this.resource);
             //这里注册mapper动态代理对象。并且注册注解的方法（@select等）成MappedStatement、详情看下面
            this.bindMapperForNamespace();
        }

        this.parsePendingResultMaps();
        this.parsePendingCacheRefs();
        this.parsePendingStatements();
    }
 
```

配置xml中的各种属性：

```java
	//xml中的标签属性设置到this中
	private void configurationElement(XNode context) {
        try {
            String namespace = context.getStringAttribute("namespace");
            if (namespace != null && !namespace.equals("")) {
                this.builderAssistant.setCurrentNamespace(namespace);
                this.cacheRefElement(context.evalNode("cache-ref"));
                this.cacheElement(context.evalNode("cache"));
                this.parameterMapElement(context.evalNodes("/mapper/parameterMap"));
                this.resultMapElements(context.evalNodes("/mapper/resultMap"));
                this.sqlElement(context.evalNodes("/mapper/sql"));
                //把上面的（select|insert|update|delete）标签解析成MappedStatement对象、用于执行，代码略
                this.buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
            } else {
                throw new BuilderException("Mapper's namespace cannot be empty");
            }
        } catch (Exception var3) {
            throw new BuilderException("Error parsing Mapper XML. The XML location is '" + this.resource + "'. Cause: " + var3, var3);
        }
    }

```

注册mapper对象：

```java
private void bindMapperForNamespace() {
        String namespace = this.builderAssistant.getCurrentNamespace();
        if (namespace != null) {
            Class boundType = null;

            try {
                boundType = Resources.classForName(namespace);
            } catch (ClassNotFoundException var4) {
                ;
            }

            if (boundType != null && !this.configuration.hasMapper(boundType)) {
                this.configuration.addLoadedResource("namespace:" + namespace);
                //最关键的代码！可以看到根据boundType将mapper进行了注册！
                this.configuration.addMapper(boundType);
            }
        }
```

### 将mapper注册到mybatis容器中

接着看```Configuration的addMapper()```方法代码：

```java
//Class Configuration:
   protected final MapperRegistry mapperRegistry;
    
    public <T> void addMapper(Class<T> type) {
        this.mapperRegistry.addMapper(type);
    }
```

可以看到，**mapperInterface实际上是存放到了MapperRegistry中！**

```MapperRegistry的addMapper()```代码如下：

```java
//Class MapperRegistry:
 private final Map<Class<?>, MapperProxyFactory<?>> knownMappers = new HashMap();

 public <T> void addMapper(Class<T> type) {
        if (type.isInterface()) {
            if (this.hasMapper(type)) {
                throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
            }

            boolean loadCompleted = false;

            try {
            	//关键在这
                //将mapper的类型作为key，mapper代理工厂作为value存入HashMap knownMappers中
                this.knownMappers.put(type, new MapperProxyFactory(type));
                //生成解析器并解析（略）
                MapperAnnotationBuilder parser = new MapperAnnotationBuilder(this.config, type);
                parser.parse();
                loadCompleted = true;
            } finally {
                if (!loadCompleted) {
                    this.knownMappers.remove(type);
                }

            }
        }

    }
```

该方法功能主要有：

- 将mapper的class文件作为key，mapper代理工厂作为value存入HashMap knownMappers中
- 生成解析器并解析

**到这里，我们便确定了，mapper类最终最终是以类型为key，其代理工厂为value注册到了MapperRegistry中。**



### getMapper(type)方法获得mapper代理类

getMapper()方法同样在MapperRegistry类中

```java
//Class MapperRegistry:
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
      return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
  }
```

### 为什么要使用mapper代理类？

Mapper代理类可以对UserMapper这个接口进行代理，实现接口中定义的方法。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190317125757119.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)
MapperProxy实现InvocationHandler，使用了JDK动态代理技术代理了接口xxxMapper，因此可以拦截xxxMapper里的方法。
拦截的意义在于在method.invoke()方法后置入代码，会根据方法名获取MapperMethod。

```java
//Class MapperProxy:
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    if (Object.class.equals(method.getDeclaringClass())) {
      try {
        return method.invoke(this, args);
      } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
      }
    }
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    return mapperMethod.execute(sqlSession, args);
  }
```

### mapperMethod.execute委托给SqlSession去执行sql

```java
public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    if (SqlCommandType.INSERT == command.getType()) {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.insert(command.getName(), param));
    } else if (SqlCommandType.UPDATE == command.getType()) {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.update(command.getName(), param));
    } else if (SqlCommandType.DELETE == command.getType()) {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.delete(command.getName(), param));
    } else if (SqlCommandType.SELECT == command.getType()) {
      if (method.returnsVoid() && method.hasResultHandler()) {
        executeWithResultHandler(sqlSession, args);
        result = null;
      } else if (method.returnsMany()) {
        result = executeForMany(sqlSession, args);
      } else if (method.returnsMap()) {
        result = executeForMap(sqlSession, args);
      } else {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = sqlSession.selectOne(command.getName(), param);
      }
    } else if (SqlCommandType.FLUSH == command.getType()) {
        result = sqlSession.flushStatements();
    } else {
      throw new BindingException("Unknown execution method for: " + command.getName());
    }
    if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
      throw new BindingException("Mapper method '" + command.getName() 
          + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
    }
    return result;
  }
```
