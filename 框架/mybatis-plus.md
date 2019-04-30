> ### mybatis-plus 快速搭建工程
>
> <https://mp.baomidou.com/guide/quick-start.html#%E5%88%9D%E5%A7%8B%E5%8C%96%E5%B7%A5%E7%A8%8B>



> ### 增删查改(CRUD)
>
> 链接中给出了所有的CRUD接口：
>
> <https://mp.baomidou.com/guide/crud-interface.html#selectbymap>
>
> ##### 通用使用示例：
>
> 1. 增：
>
>    ```java
>        public void testInsert(){
>            Employee employee = new Employee();
>            employee.setLastName("东方不败");
>            employee.setEmail("dfbb@163.com");
>            employee.setGender(1);
>            employee.setAge(20);
>            emplopyeeDao.insert(employee);
>            //mybatisplus会自动把当前插入对象在数据库中的id写回到该实体中
>            System.out.println(employee.getId());
>        }
>    ```
>
> 2. 删：
>
>    ```java
>    //1.根据id删除
>    emplopyeeDao.deleteById(1);
>    
>    //2.根据Map删除
>    Map<String,Object> columnMap = new HashMap<>();
>    columnMap.put("gender",0);
>    columnMap.put("age",18);
>    emplopyeeDao.deleteByMap(columnMap);
>    
>    //3.根据id批量删除
>    List<Integer> idList = new ArrayList<>();
>    idList.add(1);
>    idList.add(2);
>    emplopyeeDao.deleteBatchIds(idList);
>    ```
>
>    
>
> 3. 查：
>
>    ```java
>    //1.根据id查询
>    Employee employee = emplopyeeDao.selectById(1);
>    
>    //2. 3. 略
>    //4. 根据id批量查询
>    List<Integer> idList = new ArrayList<>();
>    idList.add(1);
>    idList.add(2);
>    idList.add(3);
>    List<Employee> employees = emplopyeeDao.selectBatchIds(idList);
>    System.out.println(employees);
>    ```
>
>    
>
> 4. 改：
>
>    ```java
>    public void testUpdate(){
>            Employee employee = new Employee();
>            employee.setId(1);
>            employee.setLastName("更新测试");
>            //emplopyeeDao.updateById(employee);//根据id进行更新，没有传值的属性就不会更新
>            emplopyeeDao.updateAllColumnById(employee);//根据id进行更新，没传值的属性就更新为null
>    }
>    ```
>
> ##### 条件构造器(QueryWrapper以及UpdateWrapper)
>
> 实例：
>
> **1、分页查询年龄在18 - 50且gender为0、姓名为tom的用户：**
>
> ```java
> List<Employee> employees = emplopyeeDao.selectPage(new Page<Employee>(1,3),
>      new QueryWrapper<Employee>()
>         .between("age",18,50)
>         .eq("gender",0)
>         .eq("last_name","tom")
> );
> ```

**2、查询gender为0且名字中带有老师、或者邮箱中带有a的用户：**

```java
List<Employee> employees = emplopyeeDao.selectList(
                new QueryWrapper<Employee>()
               .eq("gender",0)
               .like("last_name","老师")
                //.or()//和or new 区别不大
               .orNew()
               .like("email","a")
);
```

**3、查询gender为0，根据age排序，简单分页：**

```java
List<Employee> employees = emplopyeeDao.selectList(
                new QueryWrapper<Employee>()
                .eq("gender",0)
                .orderBy("age")//直接orderby 是升序，asc
                .last("desc limit 1,3")//在sql语句后面追加last里面的内容(改为降序，同时分页)
);
```

**4、根据条件更新：**

```java
//该案例表示把last_name为tom，age为25的所有用户的信息更新为employee中设置的信息。
public void testQueryWrapperUpdate(){
        Employee employee = new Employee();
        employee.setLastName("苍老师");
        employee.setEmail("cjk@sina.com");
        employee.setGender(0);
        emplopyeeDao.update(employee,
                new UpdateWrapper<Employee>()
                .eq("last_name","tom")
                .eq("age",25)
        );
}
```