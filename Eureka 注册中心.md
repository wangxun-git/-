## Eureka 注册中心

#### 1、Eureka

Spring Cloud Eureka主要负责实现微服务架构中的服务治理能力。它是基于REST的服务，提供了基于Java的客户端组件，能够非常方便地实现将服务注册到Spring Cloud Eureka中进行统一管理。

#### 2、Eureka比Zookeeper更加适合做注册中心

Eureka是基于AP原理，Zookeeper是基于CP原则构建的

**C：数据一致性  A：服务可用性 P：服务对网络分区故障的容错性**这个三个特性在任何分布式系统中都无法同时满足，最多只能同时满足两个。

#### 3、使用Eureka编写注册中心

###### 1、导入依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

###### 2、添加配置信息

+ 启动类上添加注解：**@EnableEurekaServer**，开启Eureka-server。

```yaml
spring:
  application:
    name: eureka-server
  security:
    user:
      name: wang
      password: 1234  #开启Eureka安全认证
server:
  port: 8761
eureka:
  client:
    register-with-eureka: false #代表不向注册中心注册自己
    fetch-registry: false #表示不去检索服务

```

增加security配置类：

```java
@Configuration
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        //关闭csrf
        http.csrf().disable();
        //支持httpBasic
        http.authorizeRequests()
                .anyRequest()
                .authenticated()
                .and()
                .httpBasic();
    }
}
```

+ 访问地址：http://localhost:8761/login

![image-20200728184527927](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200728184527927.png)![image-20200728184557537](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200728184557537.png)

开启Eureka服务注册

#### 4、使用Eureka编写服务提供者

###### 1、创建新的Spring Boot项目，导入依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

###### 2、添加配置信息

+ 启动类上添加注解：**@EnableDiscoveryClient**，表示该服务是一个Eureka的客户端

```yaml
spring:
  application:
    name: eureka-server-provide

server:
  port: 8081

eureka:
  client:
    serviceUrl:
      defaultZone: http://wang:1234@localhost:8761/eureka/  #注册中心地址
  instance:
    prefer-ip-address: true #开启IP地址认证
    instance-id: ${spring.application.name}:${spring.cloud.client.ip-address}.${server.port}  #定义地址信息
```

运行结果：![image-20200728212419688](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200728212419688.png)

###### 3、编写服务接口

```java
@RestController
public class UserController {

    @GetMapping("/user/hello")
    public String hello(){
        return "hello";
    }
}
```

#### 5、使用Eureka编写服务消费者

###### 1、导入依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```



###### 2、添加配置信息

启动类添加注解：@EnableDiscoveryClient

```yaml
spring:
  application:
    name: eureka-server-consume
server:
  port: 8082
eureka:
  client:
    serviceUrl:
      defaultZone: http://wang:1234@localhost:8761/eureka/
  instance:
    prefer-ip-address: true
    instance-id: ${spring.application.name}:${spring.cloud.client.ip-address}:${server.port}
```

###### 3、编写调用接口

添加配置类：

```java
@Configuration
public class BeanConfiguration {

    @Bean
    public RestTemplate restTemplate(){
        return new RestTemplate();
    }

}
```

接口实现：

```java
@RestController
public class UserConsumeController {

    @Autowired
    private RestTemplate restTemplate;

    @GetMapping("/consume/hello")
    public String hello(){
        return restTemplate.getForObject("http://lcoalhost:8081/user/hello", String.class);
    }

}
```

运行结果：

![image-20200728213639299](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200728213639299.png)

#### 6、Eureka高可用搭建

###### 1、高可用原理

搭建方法：每一个Eureka只需要在配置中指定另外多个Eureka的地址就可以实现。假设用于A主机和B主机：

+ 将A注册到B上面
+ 将B注册到A上面

###### 2、搭建步骤，在本机搭建

创建application-master.yml文件：

```yml
spring:
  application:
    name: eureka-server
server:
  port: 8761
eureka:
  instance:
    hostname: master
  client:
    serviceUrl:
      defaultZone: http://wang:1234@slaveone:8762/eureka/
```

创建application-slaveone.yml文件：

```yml
spring:
  application:
    name: eureka-server
server:
  port: 8762
eureka:
  instance:
    hostname: master
  client:
    serviceUrl:
      defaultZone: http://wang:1234@master:8761/eureka/
```

创建application.yml文件：

```yml
spring:
  application:
    name: eureka-server
  security:
    user:
      name: wang
      password: 1234  #开启Eureka安全认证
  profiles: #指定不同的环境
    active: master

eureka:
  client:
    register-with-eureka: true 
    fetch-registry: true #检索服务
  instance:
    prefer-ip-address: true
    appname: eureka-sever
```

修改文件：C:\Windows\System32\drivers\etc\HOSTS

```bat
127.0.0.1 master
127.0.0.1 slaveone
```

测试：

```bat
java -jar eureka-server-0.0.1-SNAPSHOT.jar --spring.profiles.active=master
```

运行结果：

![image-20200729195352572](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200729195352572.png)

```bat
java -jar eureka-server-0.0.1-SNAPSHOT.jar --spring.profiles.active=master
```

运行结果：

![image-20200729195415862](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200729195415862.png)

#### 7、快速移除已经失效的服务消息

因为Eureka的保护机制，在服务节点下线后，服务消息还会一直存在一Eureka中。

```yml
eureka:  #服务端配置
  server:
    enable-self-preservation: false #关闭自我保护
    eviction-interval-timer-in-ms: 5000 #清理间隔
```

```yaml
eureka:  #客户端配置
    instance-id: ${spring.application.name}:${spring.cloud.client.ip-address}.${server.port}  #定义地址信息
    lease-expiration-duration-in-seconds: 5 #表示服务端等待下一次心跳的时间，如果超时没有收到，则会移除Instance
    lease-renewal-interval-in-seconds: 5 #表示客户端发送心跳给客户端的频率
```

#### 8、服务上下线监控

```java
@Component
public class EurekaStateChangeListener {

    private Logger logger = LoggerFactory.getLogger(EurekaInstanceCanceledEvent.class);

    @EventListener
    public void listen(EurekaInstanceCanceledEvent event){
        logger.info(event.getServerId() + "\t" + event.getAppName() + "服务下线");
    }

    @EventListener
    public void listen(EurekaInstanceRegisteredEvent event){
        logger.info(event.getInstanceInfo().getAppName() + " 进行注册");
    }

    @EventListener
    public void listen(EurekaInstanceRenewedEvent event){
        logger.info(event.getServerId() + "\t" + event.getAppName() + "服务进行续约");
    }

    @EventListener
    public void listen(EurekaRegistryAvailableEvent event){
        logger.info("注册中心启动");
    }

    @EventListener
    public void listen(EurekaServerStartedEvent event){
        logger.info("Eureka Server 启动");
    }

}
```

