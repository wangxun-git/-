## MyBatis-PLUS

#### 1、使用

###### 1、导入依赖

```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.0.5</version>
</dependency>
```

###### 2、添加配置

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mybatis_plus?serverTimezone=GMT%2B8
    username: root
    password: 1234
    driver-class-name: com.mysql.cj.jdbc.Driver
mybatis-plus:  #mybatis日志
  configuration:
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
```

###### 3、实体类

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class User {

    @TableId(type = IdType.AUTO)
    private long id;

    private String name;

    private int age;

    private String email;
}
```

###### 4、mapper接口

```java
@Mapper
public interface UserMapper extends BaseMapper<User> {

}
```

###### 5、测试

```java
@Test
void contextLoads()  {
    List<User> userList = userMapper.selectList(null);
    userList.forEach(System.out::println);
}
```

```java
User(id=1, name=Jone, age=18, email=test1@baomidou.com)
User(id=2, name=Jack, age=20, email=test2@baomidou.com)
User(id=3, name=Tom, age=28, email=test3@baomidou.com)
User(id=4, name=Sandy, age=21, email=test4@baomidou.com)
User(id=5, name=Billie, age=24, email=test5@baomidou.com)
User(id=6, name=wang, age=23, email=test6@baomidou.com)
```

#### 2、update

```java
@Test
void update(){
    User user = new User();
    user.setId(5);
    user.setAge(28);
    int count = userMapper.updateById(user);
    System.out.println("count = " + count);
}
```

```json
==>  Preparing: UPDATE user SET age=? WHERE id=? 
==> Parameters: 28(Integer), 5(Long)
<==    Updates: 1
```

#### 3、自动填充

+ 例如：自动填充创建时间、更新时间等。

```java
import com.baomidou.mybatisplus.annotation.FieldFill;
import com.baomidou.mybatisplus.annotation.TableField;

@TableField(fill = FieldFill.INSERT)
private Date createTime;

@TableField(fill = FieldFill.INSERT_UPDATE)
private Date updateTime;
```

```java
@Component
public class MyMetaObjectHandler implements MetaObjectHandler {

    private static final Logger logger = LoggerFactory.getLogger(MyMetaObjectHandler.class);

    @Override
    public void insertFill(MetaObject metaObject) {
        logger.info("---------start insert fill----------");
        this.setFieldValByName("createTime", new Date(), metaObject);
        this.setFieldValByName("updateTime", new Date(), metaObject);
    }

    @Override
    public void updateFill(MetaObject metaObject) {
        logger.info("---------start update fill---------");
        this.setFieldValByName("updateTime", new Date(), metaObject);
    }
}
```

**测试**

#### 4、乐观锁

```java
@Version
@TableField(fill = FieldFill.INSERT_UPDATE)
private Integer version;
```

```java
@EnableTransactionManagement
@Configuration
//@MapperScan("com.atguigu.mybatis_plus.mapper")
public class MybatisPlusConfig {

    /**
     * 乐观锁插件
     */
    @Bean
    public OptimisticLockerInterceptor optimisticLockerInterceptor(){
        return new OptimisticLockerInterceptor();
    }
}
```

#### 5、select

###### 1、通过多个id批量查询

```java
@Test
void testSelectBatchIds(){
    List<User> userList = userMapper.selectBatchIds(Arrays.asList(1, 2, 3));
    userList.forEach(System.out::println);
}
```

#### 6、分页

```java
/**
 * 分页插件
 */
@Bean
public PaginationInterceptor paginationInterceptor(){
    return new PaginationInterceptor();
}
```

```java
@Test
void testSelectPage(){
    Page<User> page = new Page<>(1, 5);
    IPage<User> userIPage = userMapper.selectPage(page, null);
    System.out.println("userIPage = " + userIPage);

    page.getRecords().forEach(System.out::println);
    System.out.println(page.getCurrent());
    System.out.println("pages" + page.getPages());
    System.out.println(page.getSize());
    System.out.println(page.getTotal());
    System.out.println(page.hasNext());
    System.out.println(page.hasPrevious());
}
```

#### 7、逻辑删除

```java
/**
 * 逻辑删除
 */
@Bean
public ISqlInjector sqlInjector(){
    return new LogicSqlInjector();
}
```

```java
@TableLogic
@TableField(fill = FieldFill.INSERT)
private Integer deleted;
```

```yaml
mybatis-plus:  
  global-config:
    db-config:
      logic-not-delete-value: 0
      logic-delete-value: 1
```

```java
@Test
void testLogicDelete(){
    int count = userMapper.deleteById(1L);
    System.out.println("count = " + count);

    List<User> userList = userMapper.selectList(null);
    userList.forEach(System.out::println);
}
```

#### 二、条件构造器

###### 1、Wrapper

![image-20200824214011272](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200824214011272.png)

+ **Wrapper：**条件构造抽象类，最顶端父类。
+ **AbstractWrapper：**用于查询条件封装，生成SQL的where条件。
+ **QueryWrapper：**Entity对象封装操作类，不是用lambda语法。
+ **UpdateWrapper：**Update 条件封装，用于Entity对象更新操作。
+ **AbstractLambdaWrapper：**Lambda 语法使用 Wrapper统一处理解析 lambda 获取 column。
+ **LambdaQueryWrapper：**用于Lambda语法使用的查询Wrapper。
+ **LambdaUpdateWrapper：**Lambda 更新封装Wrapper。

###### 1、ge

```java
@Test
void testDelete(){
    QueryWrapper<User> queryWrapper = new QueryWrapper<>();
    queryWrapper
            .isNull("name")
            .ge("age", 22)
            .isNotNull("email");
    int delete = userMapper.delete(queryWrapper);
    System.out.println("delete = " + delete);
}
```

###### 2、eq、ne

```java
@Test
void testSelectOne(){
    QueryWrapper<User> queryWrapper = new QueryWrapper<>();
    queryWrapper.eq("name", "Tom");
    User user = userMapper.selectOne(queryWrapper);  //返回的是一条实体记录，当出现多条时会报错
    System.out.println("user = " + user);
}
```

###### 3、between、notBetween

```java
@Test
void testSelectCount(){
    QueryWrapper<User> queryWrapper = new QueryWrapper<>();
    queryWrapper.between("age", 20, 30);
    Integer count = userMapper.selectCount(queryWrapper);
    System.out.println("count = " + count);
}
```

###### 4、allEq

```java
@Test
void testSelectList(){
    QueryWrapper<User> queryWrapper = new QueryWrapper<>();
    Map<String, Object> map = new HashMap<>();
    map.put("id", 2);
    map.put("name", "Jack");
    map.put("age", 20);
    queryWrapper.allEq(map);
    List<User> userList = userMapper.selectList(queryWrapper);
    userList.forEach(System.out::println);
}
```

```sql
SELECT id,name,age,email,create_time,update_time,version,deleted FROM user WHERE deleted=0 AND name = ? AND id = ? AND age = ?
```

###### 5、like、notLike、likeLeft、likeRight

```java
@Test
void testSelectMaps(){
    QueryWrapper<User> queryWrapper = new QueryWrapper<>();
    queryWrapper
            .notLike("name", "e")
            .likeRight("email", "t");
    List<Map<String, Object>> list = userMapper.selectMaps(queryWrapper);
    list.forEach(System.out::println);
}
```

```sql
SELECT id,name,age,email,create_time,update_time,version,deleted FROM user WHERE deleted=0 AND name NOT LIKE ? AND email LIKE ? 
%e%(String), t%(String)
```

###### 6、in、notIn、inSql、notinSql、exists、notExists

```java
@Test
void testSelectObjs(){
    QueryWrapper<User> queryWrapper = new QueryWrapper<>();
    queryWrapper.in("id", 1, 2, 3);
    List<Object> objects = userMapper.selectObjs(queryWrapper);
    objects.forEach(System.out::println);
}
```

```sql
SELECT id,name,age,email,create_time,update_time,version,deleted FROM user WHERE deleted=0 AND id IN (?,?,?)
```

```java 
   @Test
    void testSelectObjs(){
        QueryWrapper<User> queryWrapper = new QueryWrapper<>();
        queryWrapper.inSql("id", "elect id from user where id < 3");
        List<Object> objects = userMapper.selectObjs(queryWrapper);
        objects.forEach(System.out::println);
    }
```

```sql
SELECT id,name,age,email,create_time,update_time,version,deleted FROM user WHERE deleted=0 AND id IN (select id from user where id < 3) 
```

###### 7、or、and

```java
@Test
void testUpdate1(){
    User user = new User();
    user.setAge(99);
    user.setName("Andy");

    UpdateWrapper<User> updateWrapper = new UpdateWrapper<>();
    updateWrapper
            .like("name", "h")
            .or()
            .between("age", "20", "22");
    int update = userMapper.update(user, updateWrapper);
    System.out.println("update = " + update);
}
```

```sql
UPDATE user SET name=?, age=?, update_time=?, version=? WHERE deleted=0 AND name LIKE ? OR age BETWEEN ? AND ? 
```

###### 8、嵌套or、嵌套and

```java
@Test
void testUpdate2(){
    User user = new User();
    user.setAge(99);
    user.setName("Andy");

    UpdateWrapper<User> updateWrapper = new UpdateWrapper<>();
    updateWrapper
            .like("name", "h")
            .or(i -> i.eq("name", "李白").ne("age", 20));
    int update = userMapper.update(user, updateWrapper);
    System.out.println("update = " + update);
}
```

```sql
UPDATE user SET name=?, age=?, update_time=?, version=? WHERE deleted=0 AND name LIKE ? OR ( name = ? AND age <> ? )
```

###### 9、orderBy、orderByDesc、orderByAsc

```java
@Test
void testSelectListOrderBy(){
    QueryWrapper<User> queryWrapper = new QueryWrapper<>();
    queryWrapper.orderByDesc("id");
    List<User> userList = userMapper.selectList(queryWrapper);
    userList.forEach(System.out::println);
}
```

```sql
SELECT id,name,age,email,create_time,update_time,version,deleted FROM user WHERE deleted=0 ORDER BY id DESC
```

###### 10、last

+ 直接拼接到 sql 的最后

```java
@Test
void testSelectListLast(){
    QueryWrapper<User> queryWrapper = new QueryWrapper<>();
    queryWrapper.last("limit 1");
    List<User> userList = userMapper.selectList(queryWrapper);
    userList.forEach(System.out::println);
}
```

```sql
SELECT id,name,age,email,create_time,update_time,version,deleted FROM user WHERE deleted=0 limit 1
```

###### 11、指定要查询的列

```java
@Test
void testSelectListColumn(){
    QueryWrapper<User> queryWrapper = new QueryWrapper<>();
    queryWrapper.select("id", "name", "age");
    List<User> userList = userMapper.selectList(queryWrapper);
    userList.forEach(System.out::println);
}
```

```sql
SELECT id,name,age FROM user WHERE deleted=0 
```

###### 12、set、setSql

```java
@Test
void testUpdateSet(){
    User user = new User();
    user.setAge(99);
    UpdateWrapper<User> updateWrapper = new UpdateWrapper<>();
    updateWrapper
            .like("name", "il")
            .set("name", "悟空")
            .setSql("email = test5@baomidou.com");
    int update = userMapper.update(user, updateWrapper);
    System.out.println("update = " + update);
}
```

```sql
UPDATE user SET age=?, update_time=?, version=?, name=?,email = 'test5@baomidou.com' WHERE deleted=0 AND name LIKE ? 
```

