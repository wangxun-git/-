## Feign

#### 1、feign

Feign是声明式的REST客户端，可以让Feign调用更加的简单。Feign完全代理Http请求，我们只需要像调用方法一样去调用它就可以。

#### 2、feign的使用

###### 1、服务调用者方面：

导入依赖：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

编写feign调用接口

```java
@FeignClient("eureka-client-user-server")
public interface IUserService {

    @GetMapping("/user/hello")
    public String hello();

}
```

@FeignClient，该注解表示当前是一个feign的客户端，value属性表示对应的服务，也就是在eurek注册中心注册服务名称。

在Controller层：

```java
@Autowired
IUserService userService;

@GetMapping("/call/hello")
public String Hello(){
    return userService.hello();
}
```

出现的问题：

**Read timed out**

解决办法：

```yaml
#配置ribbon立即加载
ribbon:
  eager-load:
    enabled: true
    clients: ribbon-eureka-demo,eureka-client-user-server
```

#### 3、自定义Feign的配置

###### 日志配置 查看错误信息，日志等信息。.

```java
@Configuration
public class FeignConfiguration {

    @Bean
    Logger.Level feignLoggerLevel(){
        return Logger.Level.FULL;
    }
}
```

+ NONE：不输出日志
+ BASIC：只输出请求方法的URL和响应的状态以及接口执行的时间
+ HEADERS：将BASIC信息和请求头信息输出
+ FULL：输出完整的请求信息

```java
configuration = FeignConfiguration.class
```

#### 4、继承特性

导入本地项目依赖：需要

```xml
<build>
   <plugins>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
    </plugins>
</build>
```

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <configuration>
                <source>1.8</source>
                <target>1.8</target>
            </configuration>
        </plugin>
    </plugins>
</build>
```

###### 步骤：

###### 1、编写公共api   feign-inherit-api

```java
public interface UserRemoteClient {

    @GetMapping("/user/name")
    String getName();

}
```

###### 2、编写服务提供者 feign-inherit-provide

```java
@RestController
public class DemoController implements com.fegin.feigninheritapi.service.UserRemoteClient{

    @Override
    public String getName() {
        System.out.println("调用服务提供者");
        return "wang";
    }
}
```

###### 3、编写服务消费者 feign-inherit-consume

```java
@FeignClient("feign-inherit-provide")
public interface IUserDmo extends UserRemoteClient {

}
```

```java
@RestController
public class DemoController {

    @Autowired
    IUserDmo userDmo;

    @GetMapping("/call")
    public String callHello(){
        System.out.println("调用服务消费者");
        return userDmo.getName();
    }
   
}
```

