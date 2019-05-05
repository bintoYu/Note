#### #和{}
Mybatis在处理#{}时，会将sql中的#{}替换为?号，调用PreparedStatement的set方法来赋值；
Mybatis在处理${}时，就是把${}替换成变量的值。
使用#{}可以有效的防止SQL注入，提高系统安全性。


#### 基础
1. 使用sqlSession查询：主要要close！
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190319172210578.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)
2. 使用sqlSession插入：**注意要commit和close！**
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190319172314819.png)![在这里插入图片描述](https://img-blog.csdnimg.cn/20190319172702275.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)
3. 更新

```xml
<update id="updateMember" parameterType="com.zbh.entity.Member">
        update Member
        <set>
            <if test="memberName != null">memberName=#{memberName},</if>
            <if test="memberAccount != null">memberAccount=#{memberAccount},</if>
            <if test="address != null">address=#{address},</if>
            <if test="sex != null">sex=#{sex}</if>
        </set>
        where memberId=#{memberId}
</update>
```
4. 主键返回自增主键： ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190319173445198.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)
5. 主键返回UUID：
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190319173604485.png)
6. if标签/where标签
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190321141714927.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)
7. foreach标签
  批量查询：
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190321141945893.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/2019032114193376.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)
  批量插入：（这里用的mapper）
  service层
```java
    public Integer submitItem(List<Item> list ){
        return researchMapper.submitItem(list);
    }
```

```xml

 <insert id="submitItem"  parameterType="java.util.List">
        insert into ITEM (
        ITEM_CODE,
        ITEM_NAME,
        ITEM_VALUE,
        ITEM_CATAGORY
        )
        values
        <foreach collection="list" item="item" open="(" separator="," close=")">
            '${item.itemCode}',
            '#{item.itemName,jdbcType=VARCHAR}',
            '#{item.itemValue,jdbcType=VARCHAR}',
            '#{item.itemCategory,jdbcType=VARCHAR}'
        </foreach>
    </insert>

```
#### mybatis常见面试
https://blog.csdn.net/a745233700/article/details/80977133


### mybatis延迟加载，缓存，其他（看看word）
使用：
1. mybatis配置文件中配置：

```xml
 <settings>
     <!--开启延迟加载-->
     <setting name="lazyLoadingEnabled" value="true"/>
 </settings>
```
2. resultMap中配置association：

```xml
 <!--查询订单和创建订单的用户，使用延迟加载-->
 <resultMap id="OrderAndUserLazyLoad" type="Orders">
     <id column="id" property="id"/>
     <result column="user_id" property="userId" />
     <result column="number" property="number" />
     <result column="createtime" property="createtime" />
     <result column="note" property="note" />
     <!--
     select:要延迟加载的statement的id
     colunm：关联两张表的那个列的列名
      -->
     <association property="user" javaType="User" select="findUser" column="user_id">
     </association>
 </resultMap>
```

3.  需要配置两个查询语句：

```xml
 <select id="findOrdersByLazyLoad" resultMap="OrderAndUserLazyLoad">
     SELECT * FROM orders
 </select>

 <select id="findUser" parameterType="int" resultType="User">
     SELECT * FROM User WHERE id = #{value}
 </select>
```
4. 
    ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190321143717945.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)

#### Mybatis的缓存
Mybatis的缓存，包括一级缓存**（默认开启）**和二级缓存（手动开启）
一级缓存是在SqlSession 层面进行缓存的。即，同一个SqlSession ，多次调用同一个Mapper和同一个方法的同一个参数，只会进行一次数据库查询，然后把数据缓存到缓冲中，以后直接先从缓存中取出数据，不会直接去查数据库。
二级缓存指的就是同一个namespace下的mapper。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190321144033670.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)
一、 一级缓存：
![在这里插入图片描述](https://img-blog.csdnimg.cn/201903211455570.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)
测试1：不会进行两次查询
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190321145637430.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)
测试2：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190321145649865.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190321145702620.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)
二、二级缓存：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190321150317379.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)
开启二级缓存：
1、mybatis配置文件中：`<setting name="cacheEnabled" value="true"/>`
2、mapper映射文件中：默认使用了PerpetualCache：`<cache/>`
测试：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190321150701438.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)

#### mybatis执行sql
个人理解：mybatis执行sql，都是交给sqlSession来执行sql语句。
例：一个dao层未使用动态代理的方法

```java
	private SqlSession session = ...;
	
	public String getAppSizeByAppId(AppType appType,OriginType originType, Long appId){
		String size;
		Map<String, Object> condition = new HashMap<String, Object>();
		condition.put("tableName", DbUtils.getAppTablePrefix(appType, originType) + "visualization");
		condition.put("appId", appId);
		//关键代码
		size = session.selectOne(NAMESPACE + ".selectAppSizeByAppId", condition);
		condition = null;
		return size;
	}
```

https://www.cnblogs.com/cfas/p/7604668.html

