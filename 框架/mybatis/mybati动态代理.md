@[toc]
### mybatis mapper的使用
首先，我们来看看使用mybatis的mapper的步骤：
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

### 疑问点
1. UserMapper是一个接口，明明我们也没有编写实现类，为什么就能直接使用？
2. 可以看出UserMapper使用时调用了UserMapper.xml中的sql语句，这一过程是怎么实现的？

对于第一点，我们使用Mapper代理类来对UserMapper这个接口进行代理，实现接口中定义的方法。
对于第二点，实际上mybatis在初始化的过程中便会对xml文件进行解析，并将解析结果存入sqlSessionFactory中。

### 总流程图
实际上，不管怎么个动态代理，最后都是由sqlSession来执行sql语句的。所以最后还是委托给sqlSession的
总流程图：（个人理解）
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190319140731198.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)

### mapper.xml解析
mapper.xml的解析属于sqlSessionFactory生成的其中一部分。
下面给出mybatis版的xml解析：
https://www.cnblogs.com/wangjiuyong/articles/6720501.html

mybatis-plus版的sqlSessionFactory生成:
https://blog.csdn.net/bintoYu/article/details/89748464
或 https://github.com/bintoYu/Note/blob/master/%E6%A1%86%E6%9E%B6/mybatis/mybatis-plus%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-1-sqlSessionFactory%E7%9A%84%E7%94%9F%E6%88%90.md

### getMapper(type)方法获得mapper代理类
将之前存放的工厂取出，然后生成mapper代理类

```java
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