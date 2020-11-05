## MyBatis学习笔记

#### 一、什么是MyBatis

MyBatis 是一款优秀的持久层框架，它支持自定义 SQL、存储过程以及高级映射。MyBatis 免除了几乎所有的 JDBC 代码以及设置参数和获取结果集的工作。MyBatis 可以通过简单的 XML 或注解来配置和映射原始类型、接口和 Java POJO（Plain Old Java Objects，普通老式 Java 对象）为数据库中的记录。

#### 二、Spring Boot整合MyBatis

###### 1、导入依赖

```xml
<dependency>
   <groupId>org.mybatis.spring.boot</groupId>
   <artifactId>mybatis-spring-boot-starter</artifactId>
   <version>2.1.3</version>
</dependency>
```

###### 2、连接数据库配置

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/test?useUnicode=true&characterEncoding=utf-8&serverTimezone=Asia/Shanghai
    username: root
    password: 1234
```

###### 3、mapper接口

```java
@Mapper
public interface UserMapper {

    @Select("select name from user where user_id=#{userId}")
    String queryNameById(Integer userId);

}
```

```java
@Autowired
UserMapper userMapper;

@Test
void contextLoads() {

   Integer userId = 1;
   String name = userMapper.queryNameById(userId);
   System.out.println("name = " + name);
}

运行结果：name = wang
```

###### 4、使用xml文件

```yaml
mybatis:
  configuration:
    map-underscore-to-camel-case: true  #开启驼峰命名规范
  mapper-locations: classpath:mybatis/*.xml  #映射文件路径
```

###### 5、定义接口

```java
public interface GradeMapper {

    Grade queryByGradeId(Map map);
    
    User queryUser(Map map);

}
```

###### 6、xml文件

```xml
<mapper namespace="com.example.myBatisdemo.mapper.GradeMapper">

    <select id="queryByGradeId" parameterType="java.util.Map" resultType="com.example.myBatisdemo.entity.Grade">
        select * from grade where grade_id=#{gradeId}
        <if test="gradeName != null and gradeName !=''">
            and
            grade_name = #{gradeName}
        </if>
    </select>
</mapper>

	<select id="queryUser" parameterType="java.util.Map" resultType="com.example.myBatisdemo.entity.User">
        <include refid="select-all"/>
        from user where user_id=#{userId}
        <if test="name != null and name != ''">
            and
            name = #{name}
        </if>
    </select>
```

###### 7、Junit测试

```java
@Test   //测试mapper
void contextLoads() {

   Integer userId = 1;
   String name = userMapper.queryNameById(userId);
   System.out.println("name = " + name);
}

@Test   //测试xml文件形式
void testMapper(){
   Map<String, Object> map = new HashMap<>();
   map.put("gradeId",1);
   map.put("gradeName","java");

   Grade grade = gradeMapper.queryByGradeId(map);
   System.out.println("grade = " + grade);
}

@Test
void testQueryUser(){
    Map<String, Object> map = new HashMap<>();
    map.put("userId", 1);

    User user = gradeMapper.queryUser(map);
    System.out.println("user = " + user);
}
```

#### 三、SqlSessionFactroy和SqlSession

+ **SqlSessionFactory**的实例可以通过**SqlSessionFactoryBuilder** 获得。
+ SqlSessionFactoryBuilder 则可以从 XML 配置文件或一个预先配置的 Configuration 实例来构建出 SqlSessionFactory 实例。

#### 四、MyBatis的一级缓存和二级缓存

+ MyBatis内置了一个强大的事务型查询缓存机制，可以非常方便地配置和定制。
+ 默认情况下，只开启了本地的会话缓存。
+ 如果要启用全局的二级缓存，需要在SQL映射文件配置**<cache/>**，作用：
  + 映射语句文件中所有的**select**语句的结果都会被缓存。
  + 映射语句文件中的所有的**insert、update、delete**语句都会刷新缓存。
  + 缓存会使用最近最少使用算法(LRU,Least,Recently Used)算法清除不需要的缓存。
  + 缓存不会定时进行刷新，不会进行定时刷新。
  + 缓存会保存列表或对象(无论查询返回的哪种结果)的1024个引用。
  + 缓存会被视为读/写缓存，这意味获取到的对象并不是共享的，可以安全的被调用者修改，而不干扰其他调用者或线程所做的潜在修改。
+ 清除策略：
  + **LRU：**最近最少使用：移除最长时间不被使用的对象。
  + **FIFO：**先进先出：按对象进入缓存的顺序来移除它们。
  + **SOFT：**软引用：基于垃圾回收器状态和软引用规则移除对象。
  + **WEAK：**弱引用：更积极的基于垃圾收集器状态和弱引用规则移除对象。
+ 缓存只作用**于cache标签所在的映射文件中语句**。如果混合使用 Java API 和 XML 映射文件，在共用接口中的语句将不会被默认缓存。需要使用 **@CacheNamespaceRef 注解指定缓存作用域**。
+ **二级缓存是事务性的**。当SqlSession完成提交时，或是完成并回滚，但是没有执行**flushCache = true**的**insert、update、delete**语句时，缓存会获得更新。

###### 1、一级缓存配置

+ 当多个**SqlSession**操作同一行的数据时，会出现脏读的情况。

###### 2、二级缓存

+ 默认情况下，MyBatis开启了二级缓存，但是它并没有生效，因为二级缓存的作用域是**namespace**，所以需要配置**<cache/>**标签。

```xml
<cache
        eviction="FIFO"
        flushInterval="60000"
        size="520"
        readOnly="true"
/>
```

+ 第二次查询会命中缓存，取消了访问数据库的操作。

**多表联查的二级缓存：**

```yaml
xml文件:
  UserMapper.xml     ClassMapper.xml
```

**UserMapper.xml**

```xml
 <cache
            eviction="FIFO"
            flushInterval="60000"
            size="520"
            readOnly="true"
    />
    
    <select id="getClassName" parameterType="java.lang.Long" resultType="java.lang.String">
        select class_name from class c join user u on c.id = u.id where u.id=#{id}
    </select>
```

**ClassMapper.xml**

```xml
<cache
        eviction="FIFO"
        flushInterval="60000"
        size="520"
        readOnly="true"
/>

<cache-ref namespace="com.wangxun.mybatiscache.mapper.UserMapper"/>

<insert id="insertClass">
    insert into class(class_id, class_name, id)
    value (#{classId}, #{className}, #{id})
</insert>
```

```java 
@Test
void testSecondCache(){
    List<String> className1 = userMapper.getClassName(0L);
    className1.forEach(System.out::println);
    List<String> className2 = userMapper.getClassName(0L);
    className2.forEach(System.out::println);
    //插入数据，进行测试
    Map<String, Object> map = new HashMap<>();
    map.put("classId", 2);
    map.put("className", "teach");
    map.put("id", 0L);
    classMapper.insertClass(map);
    List<String> className3 = userMapper.getClassName(0L);
    className3.forEach(System.out::println);
}
```

**运行结果：**

```sql
2020-08-26 21:35:52.964 DEBUG 4244 --- [           main] c.w.mybatiscache.mapper.UserMapper       : Cache Hit Ratio [com.wangxun.mybatiscache.mapper.UserMapper]: 0.0
2020-08-26 21:35:52.972  INFO 4244 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...
2020-08-26 21:35:53.227  INFO 4244 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.
2020-08-26 21:35:53.236 DEBUG 4244 --- [           main] c.w.m.mapper.UserMapper.getClassName     : ==>  Preparing: select class_name from class c join user u on c.id = u.id where u.id=?
2020-08-26 21:35:53.276 DEBUG 4244 --- [           main] c.w.m.mapper.UserMapper.getClassName     : ==> Parameters: 0(Long)
2020-08-26 21:35:53.305 DEBUG 4244 --- [           main] c.w.m.mapper.UserMapper.getClassName     : <==      Total: 1
信管
2020-08-26 21:35:53.312 DEBUG 4244 --- [           main] c.w.mybatiscache.mapper.UserMapper       : Cache Hit Ratio [com.wangxun.mybatiscache.mapper.UserMapper]: 0.5
信管
2020-08-26 21:35:53.314 DEBUG 4244 --- [           main] c.w.m.mapper.ClassMapper.insertClass     : ==>  Preparing: insert into class(class_id, class_name, id) value (?, ?, ?)
2020-08-26 21:35:53.315 DEBUG 4244 --- [           main] c.w.m.mapper.ClassMapper.insertClass     : ==> Parameters: 2(Integer), teach(String), 0(Long)
2020-08-26 21:35:53.322 DEBUG 4244 --- [           main] c.w.m.mapper.ClassMapper.insertClass     : <==    Updates: 1
2020-08-26 21:35:53.323 DEBUG 4244 --- [           main] c.w.mybatiscache.mapper.UserMapper       : Cache Hit Ratio [com.wangxun.mybatiscache.mapper.UserMapper]: 0.3333333333333333
2020-08-26 21:35:53.323 DEBUG 4244 --- [           main] c.w.m.mapper.UserMapper.getClassName     : ==>  Preparing: select class_name from class c join user u on c.id = u.id where u.id=?
2020-08-26 21:35:53.324 DEBUG 4244 --- [           main] c.w.m.mapper.UserMapper.getClassName     : ==> Parameters: 0(Long)
2020-08-26 21:35:53.325 DEBUG 4244 --- [           main] c.w.m.mapper.UserMapper.getClassName     : <==      Total: 2
信管
teach
```

