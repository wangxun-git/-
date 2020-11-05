## Hystrix服务容错处理

#### 一、介绍

在微服务架构中，很多服务之间都会存在依赖，使用Hystrix对依赖的服务进行隔离。服务如果在调用过程中出现故障会出现连锁反应，有可能会让整个系统都不能使用，这种情况称之为服务雪崩效应。Hystrix是Netflix针对微服务分布式系统采用的熔断保护中间件。

#### 二、Hystrix的简单使用

**信号量策略配置**

```java
super(HystrixCommand.Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("MyGroup"))
.andCommandPropertiesDefaults(HystrixCommandProperties.Setter().withExecutionIsolationStrategy(HystrixCommandProperties.ExecutionIsolationStrategy.SEMAPHORE)));
```

**线程隔离策略配置**，系统默认采用

```java
public MyHystrixCommand(String name){
    super(HystrixCommand.Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("wangxun")).andCommandPropertiesDefaults(
            HystrixCommandProperties.Setter().withExecutionIsolationStrategy(HystrixCommandProperties.ExecutionIsolationStrategy.THREAD)
    ).andThreadPoolPropertiesDefaults(HystrixThreadPoolProperties.Setter().withCoreSize(10).withMaxQueueSize(100).withMaximumSize(100))
    );
    this.name = name;
}
```

**结果缓存**

```java
@Override
    protected String getCacheKey() {
        return String.valueOf(this.name);
    }
```

这段代码会把传入进来的name作为缓存的key。

```Caused by: java.lang.IllegalStateException: Request caching is not available. Maybe you need to initialize the HystrixRequestContext?```

缓存取决于请求的上下文，所以必须初始化HystrixRequestContext。

```java
public static void main(String[] args) throws ExecutionException, InterruptedException {

    /*
    请求上下文
     */
    HystrixRequestContext context = HystrixRequestContext.initializeContext();

    String result = new MyHystrixCommand("wangxun").execute();
    System.out.println(result);

    Future<String> future = new MyHystrixCommand("wangxun").queue();
    System.out.println(future.get());
    context.shutdown();
}
```

**清楚缓存**

```java
public class ClearCacheHystrixCommand extends HystrixCommand<String> {

    private final String name;

    private static final HystrixCommandKey GETTER_KEY = HystrixCommandKey.Factory.asKey("wangxun");

    public ClearCacheHystrixCommand(String name){
        super(HystrixCommand.Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("wangxun")).andCommandKey(GETTER_KEY));
        this.name = name;
    }

    public static void fulshCache(String name){
        HystrixRequestCache.getInstance(GETTER_KEY, HystrixConcurrencyStrategyDefault.getInstance()).clear(name);
    }

    @Override
    protected String getCacheKey() {
        return String.valueOf(this.name);
    }

    @Override
    protected String run() throws Exception {
        System.out.println("get data");
        return this.name + ":" + Thread.currentThread().getName();
    }

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        HystrixRequestContext context = HystrixRequestContext.initializeContext();
        String result = new ClearCacheHystrixCommand("wangxun").execute();
        System.out.println(result);
        ClearCacheHystrixCommand.fulshCache("wangxun");
        Future<String> future = new ClearCacheHystrixCommand("wangxun").queue();
        System.out.println(future.get());
    }
}
```

**合并请求**

继承HystrixCollapser

#### 三、在Spring Cloud中使用Hystrix

1、导入依赖

```java
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```

2、启动类上标注@EnableHystrix

3、请求方法

```java
@Autowired
RestTemplate restTemplate;

@GetMapping("/callHello")
@HystrixCommand(fallbackMethod = "defaultCallHello")
public String callHello(){
    String result = restTemplate.getForObject("http://eureka-client-user-server/user/hello", String.class);
    return result;
}

public String defaultCallHello(){
    return "服务调用失败";
}
```

#### 四、Feign整合Hystrix的服务容错

1、开启feign对Hystrix的支持

```yaml
feign:
  hystrix:
    enabled: true
```

2、客户端编写

```java
@FeignClient(value = "eureka-client-user-server", fallbackFactory = UserRemoteClientFallbackFactory.class)
public interface UserRemoteClient {

    @GetMapping("/user/hello")
    String callHello();

}
```

```java
@Component
public class UserRemoteClientFallbackFactory implements FallbackFactory<UserRemoteClient> {

    private Logger logger = LoggerFactory.getLogger(UserRemoteClientFallbackFactory.class);

    @Override
    public UserRemoteClient create(Throwable cause) {
        logger.error("UserRemoteClient调用失败：" + cause);
        return new UserRemoteClient() {
            @Override
            public String callHello() {
                return "服务调用失败了，wang";
            }
        };
    }
}
```

#### 五、Hystrix监控

**官方文档：https://github.com/Netflix/Hystrix/wiki/Metrics-and-Monitoring**

必备条件：

导入Actuator、Hystrix依赖，在启动类添加@EnableHystrix

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```

**访问地址：localhost:8082/actuator/hystrix.stream**；**接着就请求localhost:8082/callHello**

***

**整合Dashboard查看监控数据**
1、导入依赖：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
</dependency>
```

2、标注注解、添加配置

```java
@EnableHystrixDashboard
```

```xml
spring.application.name=hystrix-dashboard-demo
server.port=8081
```

访问 http:localhost:8081/hystrix 

**Turbine聚合集群数据**
1、导入依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-turbine</artifactId>
</dependency>
```

2、启动类添加注解

```java
@EnableTurbine
@EnableDiscoveryClient
```

```xml
eureka.client.serviceUrl.defaultZone=http://wang:1234@localhost:8761/eureka/
turbine.app-config=hystrix-feign-demo
turbine.aggregator.cluster-config=default
turbine.cluster-name-expression=new String("default")
```

