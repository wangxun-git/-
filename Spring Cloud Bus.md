## Spring Cloud Bus

+ 消息总线：使用轻量级的消息代理来构建一个公用的消息主题使系统中所有的微服务都连接起来，该主题上所有的消息会被所有实例监听和消费。 
+ 消息代理：是一种消息验证、传输、路由的架构模式。在应用程序之间起到通信调度并最小化应用之间的依赖的作用，使用应用可以高效的解耦通信过程。消息代理是中间件产品，核心是消息的路由程序，用来实现接收和分发消息。并通过设定好的消息处理流来转发给正确的应用。目的是为了能够从应用程序中传入消息，并执行特殊的操作。
+ 通过分布式的启动器对Spring boot应用进行扩展，也可以用来建立一个或多个应用之间的通信频道。

#### 1、Spring Cloud Bus拉取GitHub读取配置

##### 1、将配置文件上传至GitHub

![image-20200816184632663](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200816184632663.png)

##### 2、创建config服务

###### 1、导入依赖

```yaml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
```

###### 2、添加配置信息

```yaml
server:
  port: 8080
spring:
  application:
    name: config-server
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
  cloud:
    config:
      server:
        git:
          uri: https://github.com/wangxun-git/MyGit.git/
          username: #github账号
          password: #密码
          search-paths: /
management:
  endpoints:
    web:
      exposure:
        include: bus,bus-refresh

eureka:
  client:
    serviceUrl:
      defaultZone: http://wang:1234@localhost:8761/eureka/
  instance: #采用IP注册
    prefer-ip-address: true
    instance-id: ${spring.application.name}:${spring.cloud.client.ip-address}:${server.port}
```

##### 3、创建config客户端

###### 1、导入依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-client</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
```

###### 2、创建bootstrap.yml文件

```yaml
spring:
  cloud:
    config:
      uri: http://localhost:8080
      name: cloud-config
      profile: dev
```

###### 3、创建application.yml文件

```yaml
spring:
  application:
    name: config-server
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
eureka:
  client:
    serviceUrl:
      defaultZone: http://wang:1234@localhost:8761/eureka/
  instance: #采用IP注册
    prefer-ip-address: true
    instance-id: ${spring.application.name}:${spring.cloud.client.ip-address}:${server.port}
server:
  port: 8082
```

###### 4、编写测试

```java
@RestController
public class HelloController {

    @Value("${spring.cloud}")
    private String context;

    @Autowired
    RabbitTemplate rabbitTemplate;

    @GetMapping("/hello")
    public String hello(){
        rabbitTemplate.convertAndSend("hello", context);
        return "Spring cloud : " + context;
    }

}
```

###### 5、运行结果

![image-20200816185235876](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200816185235876.png)

![image-20200816185255471](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200816185255471.png)

#### 2、RabbitMQ实现消息总线

+ RabbitMQ是实现了高级消息队列协议(AMQP)，被称为面向消息的中间件。

+ Broker：消息队列服务器的实体，是一个中间件应用，负责接收消息生产者的消息，然后将消息发送给消息接受者或者其他的Broker。
+ Exchange：消息交换机，是消息第一个到达的地方，消息通过它指定的路由规则，分发到不同的消息队列中去。
+ Queue：消息队列，消息通过发送和路由之后最终到达的地方，到达Queue的信息即进入逻辑上等待消费的状态，每个消息都会被发送到一个或多个队列。
+ Binding：绑定，把Exchange和Queue按照路由规则绑定起来。
+ Channel：信道，多路复用中的一条独立的双向数据流通道，AMQP信息都是通过信道实现发送和接受的。

**步骤：**

+ 1、客户端连接到消息队列服务器，打开一个Channel。
+ 2、客户端声明一个Exchange，并设置相关属性。
+ 3、客户端声明一个Queue，并设置相关属性。
+ 4、客户端使用Routing Key，在Exchange和Queue中建立好绑定关系。
+ 5、客户端投递消息到Exchange。
+ 6、Exchange接收消息之后，根据信息的Key和已经设置的Binding，进行消息的路由，将信息投递到一个或多个的Queue里。

**Exchange的几种类型：**

+ Direct交换机：完全根据Key进行投递。
+ Topic交换机：对Key进行模式匹配后进行投递，可以使用符号**#**匹配一个或多个词，符号*匹配正好一个词。
+ Fanout交换机：不需要任何Key，他才去广播的模式，一个消息进来时，投递到与该交换机绑定的所有队列。

**RabbitMQ支持消息的持久化：**

+ Exchange持久化：指定durable => 1。	
+ Queue持久化：指定durable => 1。

##### 1、Spring Boot整合RabbitMQ

###### 1、导入依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

###### 2、添加配置信息

```yaml
spring:
  application:
    name: rabbitmq-hello
  rabbitmq:
    host: localhost
    port: 15672
    username: guest
    password: guest
```

###### 3、创建生产者Sender

```java
@Component
public class Sender {

    @Autowired
    private AmqpTemplate amqpTemplate;

    public void send(){
        String context = "hello" + new Date();
        System.out.println("context = " + context);
        this.amqpTemplate.convertAndSend("hello", context);
    }

    @Bean
    public Queue helloQueue(){
        return new Queue("hello");
    }

}
```

###### 4、创建消费者

```java
@Service
@RabbitListener(queues = "hello")
public class Receiver {

    //搭配@RabbitListener使用，当@RabbitListener接收到消息交给@RabbitHandler来处理
    @RabbitHandler 
    public void hello1(String context){
        System.out.println("context = " + context);
    }

    @RabbitHandler  
    public void hello2(byte[] message){
        System.out.println("message = " + new String(message));
    }

}
```

###### 5、创建测试

```java
@Autowired
Sender sender;

@Test
void contextLoads() {
    sender.send();
}
```

#### 3、Kafka实现消息总线

+ 分布式：支持消息分区以及分布式消费，并保证分区内的消息顺序。
+ 跨平台：支持不同技术平台的 客户端。
+ 实时性：支持实时数据处理和离线数据处理。
+ 伸缩性：支持水平扩展。

基本概念：

+ Broker：Kafka集群包含一个或多个的服务器。
+ Topic：逻辑上同RabbitMQ中的Queue队列相似，每条发布到Kafka集群中的消息都必须有一个Topic。
+ Producer：消息生产者，负责生产消息并发送到Kafka Broker。
+ Consumer：消息消费者，向Kafka Broker读取消息并处理的客户端。
+ Consumer Group：每个Consumer属于一个特定的组(可以为每一个Consumer指定属于一个组，若不指定则属于默认组)，组可以用来实现一条数据被组内多个成员消费等功能。

##### 1、Window下搭建Kafka运行环境

https://www.cnblogs.com/mh-study/p/9537970.html

![image-20200816160234678](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200816160234678.png)

##### 2、Spring Boot整合Kafka

###### 1、导入依赖

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka-test</artifactId>
    <scope>test</scope>
</dependency>

<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

###### 2、添加配置信息

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092 #制定kafka代理地址
    producer:
      retries: 0 #消息发送失败重试次数
      batch-size: 1000 #每次批量发送消息的数量
      buffer-memory: 5000 #每次批量发送消息的缓冲区大小
      key-serializer: org.apache.kafka.common.serialization.StringSerializer # 指定消息key和消息体的编解码方式
      value-serializer: org.apache.kafka.common.serialization.StringSerializer

    consumer:
      group-id: user-log-group # 指定默认消费者group id
      auto-offset-reset: earliest
      enable-auto-commit: true
      auto-commit-interval: 100
      key-serializer: org.apache.kafka.common.serialization.StringSerializer # 指定消息key和消息体的编解码方式
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
```

###### 3、编写测试

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Userlog {

    private String username;

    private String userId;

    private String state;

}
```

```java
/**
 * 消息生产者
 */
@Slf4j
@Component
public class UserLogProducer {

    @Autowired
    private KafkaTemplate kafkaTemplate;

    public void snedlog(String userId){
        Userlog userlog = new Userlog();
        userlog.setUserId(userId);
        userlog.setState("1");
        userlog.setUsername("wangxun");

        log.info("-----------userlog : {} " , userlog);
        kafkaTemplate.send("testDemo", JSON.toJSONString(userlog));
    }

}
```

```java
/**
 * 消息消费者
 */
@Slf4j
@Component
public class UserLogConsumer {

    @KafkaListener(topics = "testDemo")
    public void consumer(ConsumerRecord consumerRecord){
        Optional kafkaMsg = Optional.ofNullable(consumerRecord.value());
        if (kafkaMsg.isPresent()){
            Object msg = kafkaMsg.get();
            log.info("---------msg : {}", msg);
        }
    }

}
```

```java
/**
* 启动类中编写
*/
@Autowired
private UserLogProducer userLogProducer;

@PostConstruct
public void init(){
    for (int i = 0; i < 10; i++){
        userLogProducer.snedlog(String.valueOf(i));
    }
}
```

###### 4、运行启动类测试

![image-20200816162634167](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200816162634167.png)

#### 4、事件驱动模型

+ 事件监听者：Spring中定义了事件监听者**ApplicationListener**。
+ 事件发布者：Spring中定义了ApplicationEventPublisher和ApplicationEventMulticaster。
