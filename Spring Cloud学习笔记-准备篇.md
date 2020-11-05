## Spring Cloud学习笔记-准备篇

#### 1、Spring Cloud与微服务概述

###### 1、传统的单体应用

+ 单体应用程序：把所有的功能堆积在一起。会出现牵一发而动全身的效应。
+ Dubbo基于Netty的TCP及二进制的数据传输，Spring Cloud基于HTTP。Http传输每一次都需要建立连接，传输的也是文本内容。所以在性能方面要比Dubbo框架弱。但是HTTP风格的API交互，在不同的语言中是通用的。

###### 2、什么是微服务

+ 一种框架风格，将单体划分为小型的服务单元，微服务之间通过HTTP的API进行资源访问与操作。

###### 3、微服务的优势和劣势

优势：

+ 服务的独立部署
+ 服务快速启动
+ 更加适合敏捷开发：敏捷开发以**用户的需求进化**为核心，采用迭代、循序渐进的方法进行。
+ 职责专一
+ 服务可以动态按需扩容
+ 代码复用

劣势：

+ 分布式部署，调用的复杂性高：通过HTTP调用，会产生很多问题，比如网络问题、容错问题、调用关系等。
+ 独立的数据库，分布式事务的挑战：每个微服务都有自己的数据库，去中心化的数据管理。事务的解决：柔性事务中的最终一致性。
+ 测试的难度提升：API文档管理。
+ 运维难度的提升：

###### 4、什么是 Spring Cloud

Spring Cloud是一系列框架的有序集合。

#### 2、实战前的准备工作

###### 1、Spring Boot入门

Spring Cloud基于Spring Boot搭建。

###### 2、Profiles多环境配置

```xml
spring.profiles.active=dev  #激活不同环境下的配置
```

| 配置文件                    | 环境                 |
| --------------------------- | -------------------- |
| application.properties      | 通用配置，不区分环境 |
| application-dev.properties  | 开发环境             |
| application-test.properties | 测试环境             |
| application-prod.properties | 生产环境             |

###### 3、热部署

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
</dependency>
```

只要有保存操作，Spring Boot就会自动重新加载被修改的Class文件。

###### 4、actuator监控

Spring Boot提供了一个用于监控和管理自身应用信息的模块。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

**UP：**表示应用处于健康状态，**DOWN**表示处于不健康状态。

```yaml
management:
  endpoints:
    web:
      exposure:
        include: "*"   #让所有的端点都暴露
  endpoint:
    health:
      show-details: always  #可以让一些健康信息的详细显示出来
```

###### 4.1、自动以actuator端点

如果需要对应用的健康状态增加一些其他维度的数据，可以通过集成**AbstractHealthIndicator**来实现自己的业务逻辑。

```java
@Component
public class UserHealthIndicator extends AbstractHealthIndicator {
    @Override
    protected void doHealthCheck(Health.Builder builder) throws Exception {
        builder.up().withDetail("status", "false");
    }
}
```

开发一个全新的端点，通过@Endpoint注解实现。

```java
@Component
@Endpoint(id = "user")
public class Userpoint {

    @ReadOperation
    public List<Map<String, Object>> health(){
        List<Map<String, Object>> list = new ArrayList<>();
        Map<String, Object> map = new HashMap<>();
        map.put("userId", 1001);
        map.put("userName", "wang");
        list.add(map);
        return list;
    }
}
```

请求地址：http://localhost:8080/actuator/user

###### 5、统一异常处理

在处理错误时返回自定义的格式，定义一个异常处理类。

```java
//全局异常处理  还可以做：1、全局数据处理  2、全局数据预处理
@ControllerAdvice
public class GlobalExceptionHandler {

    Logger logger = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    @ExceptionHandler(value = Exception.class)
    @ResponseBody
    public ResponseData defaultErrorHandler(HttpServletRequest request, Exception e) throws Exception{
        logger.error("{}", e);
        ResponseData responseData = new ResponseData();
        responseData.setMessage(e.getMessage());
        if (e instanceof org.springframework.web.servlet.NoHandlerFoundException){
            responseData.setCode(404);
        }else {
            responseData.setCode(500);
        }
        responseData.setData(null);
        responseData.setStatus(null);
        return responseData;
    }

}
```

```yaml
spring:
  mvc:
    throw-exception-if-no-handler-found: true  #出现错误，直接抛出异常
  resources:
    add-mappings: false #不为工程中的资源文件建立映射
```

###### 6、异步执行

+ 异步调用是不用等待结果的返回就可以执行后面的逻辑；同步调用则需要等待结果再执行后面的逻辑。

+ 通常使用异步操作时都会创建一个线程执行一段逻辑，然后把线程丢到线程池。

在Spring中有一种比较简单的方式来实现异步的操作，使用@Async注解。

```java
@Async
public void saveLog(){
    System.out.println("线程名称 = " + Thread.currentThread().getName());
}
```

该方法需要被外部类调用才可以。在启动类上添加注解：@EnableAsync。

###### 7、自定义starter

+ 创建一个配置类

```java
@Data
@ConfigurationProperties("spring.user")
public class UserProperties {

    private String name;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

```java
public class UserClient {

    private UserProperties userProperties;

    public UserClient(){}

    public UserClient(UserProperties p) {
        this.userProperties = p;
    }

    public String getName() {
        return userProperties.getName();
    }

}
```

+ 自动创建客户端

```java
@Configuration
@EnableConfigurationProperties(UserProperties.class)
public class UserAutoConfigure {

    @Bean
    @ConditionalOnProperty(prefix = "spring.user", value = "enabled", havingValue = "true")
    public UserClient userClient(UserProperties userProperties) {
        return new UserClient(userProperties);
    }
}
```

+ 在其他工程中引入本地jar

```xml
<dependency>
    <groupId>com.mylearn</groupId>
    <artifactId>spring-boot-starter-demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
</dependency>
```

```java
@Autowired
UserClient userClient;

@GetMapping("/hello")
public String hello(){
    String name = userClient.getName();
    return name;
}
```

