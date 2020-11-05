## springboot缓存 cache

#### Java Caching定义了5个接口，分别是

**CachingProvider,CacheManager,Cache,Entry,Expiry**

1. CachingProvider定义了创建、配置、获取、管理和控制多个CacheManager。一个应用可以在运行期访问多个CachingProvider。
2. CacheManager定义了创建、配置、获取、管理和控制多个唯一命名的Cache，这些Cache存在于CacheManager的上下文中。一个CacheManager仅被一个CachingProvider所拥有。
3. Cache是一个类似Map的数据结构并临时存储以Key为索引的值。一个Cache仅被一个CacheManager所拥有。
4. Entry是一个存储在Cache中的key-value对。
5. Expiry 每一个存储在Cache中的条目有一个定义的有效期。一旦超过这个时间，条目为过期的状态。一旦过期，条目将不可访问、更新和删除。缓存有效期可以通过ExpiryPolicy设置

![](C:\Users\Administrator\Desktop\cache.png)

#### 注解

**@Cacheable** 

**@CachePut**1、首先调用目标方法，2、将目标方法的结果缓存起来   但是查询可能会发现没有更新，需要修改设置缓存的id

**@CacheEvict**删除缓存

**@Caching**定义复杂的缓存规则

**@CacheConfig**定义在类

## springbootboot缓存之redis

#### 对象需要序列化，建议使用redis-starter中的

## springboot消息 JMS、AMQP、RabbitMQ

#### 1、消息中间件提升系统异步通信，扩展解耦能力

#### 2、消息服务两个重要概念

1、消息代理，2、目的地

#### 3、消息队列主要有两种形式的目的地

1、队列：点对点通信：消息发送者发送消息，消息代理将消息放入一个队列，消息接收者从队列中获取消息，消息被读取后移出队列。

2、主题：发布/订阅消息：消息发送者发送消息到主题，多个接受者（订阅者）监听（订阅）这个消息。

**RabbitTemplate**实现发送消息。@RabbitListener注解在方法上监听消息代理发布的消息。

#### RabbitMQ安装启动

###### 1、首先安装erlang

因为RabbitMQ是使用Erlang编写。

安装完毕之后，配置环境变量：win+R，cmd，rel；

![image-20200717202921906](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200717202921906.png)

###### 2、安装、启动RabbitMQ

安装完毕之后，进入**D:\RabbitMQ\rabbitmq_server-3.8.4\sbin**，在地址栏输入cmd，输入**rabbitmq-server.bat**

![image-20200717203607994](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200717203607994.png)

访问地址：**http://localhost:15672**

#### 4、开启RabbitMQ服务，访问端口

###### 1、导入依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

###### 2、文件配置

```xml
spring.rabbitmq.host=localhost
spring.rabbitmq.username=guest
spring.rabbitmq.password=guest
spring.rabbitmq.port=5672
```

###### 3、消息发送

```java
          //Message需要自己定制构造一个，定义消息头和消息体
//        rabbitTemplate.send(exchange, routeKey, message);

        Map<String, Object> map = new HashMap<>();
        map.put("wang", "xun");
        map.put("list", Arrays.asList("wang", 123, true));
        String message = "hello RabbitMQ";
        rabbitTemplate.convertAndSend("exchange.direct", "wangxun", map);
```

###### 4、消息接收

```java
Object wangxun = rabbitTemplate.receiveAndConvert("wangxun");
System.out.println(wangxun.getClass());
System.out.println(wangxun);
```

###### 5、消息格式化，json形式

```java
import org.springframework.amqp.support.converter.Jackson2JsonMessageConverter;
import org.springframework.amqp.support.converter.MessageConverter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MyConfig {

    @Bean
    public MessageConverter messageConverter(){
        return new Jackson2JsonMessageConverter();
    }

}
```

###### 6、开启RabbitMQ监听

启动类添加**@EnableRabbit  //开启RabbitMQ的监听**注解

```java
@RabbitListener(queues = "wangxun")
public void bookRabbitListener(Book book){
    System.out.println(book.toString());
}
```

#### amqpAdmin--RabbitMQ系统管理功能组件

```java
@Autowired
AmqpAdmin amqpAdmin;

@Test
public void amqpAdmin(){
    //创建交换器
    amqpAdmin.declareExchange(new DirectExchange("amqpadmin.exchange"));
    //创建队列
    amqpAdmin.declareQueue(new Queue("amqpadmin.queue"));
    //绑定
    amqpAdmin.declareBinding(new Binding("amqpadmin.queue", Binding.DestinationType.QUEUE, "amqpadmin.exchange", "amqpadmin.test", null));

    System.out.println("OK");
}
```

