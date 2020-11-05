## Sleuth服务跟踪

微服务架构下，服务之间的调用关系越来越复杂，通过Zuul转发到具体的业务接口，一个接口中会涉及多个服务的交互，只要某个服务出现问题，整个请求都会失败。如何快速定位问题所在，需要用到链路跟踪。每个请求都是一条完整的服务调用链。

#### 一、Spring Cloud集成Sleuth

###### 1、导入依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>
```

###### 2、访问接口

###### 3、窗口结果

![image-20200607212453649](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200607212453649.png)

内容由**[appname,traceId,spanId,exportable]**组成。

**appname：**服务的名称。

**traceId：**整个请求的唯一id。

**spanId：**基本的工作单元，发起一次远程调用就是一个span。

**exporttable：**决定是否导入数据到**ZipKin**中

#### 二、整合Logstash

引入日志分析系统，比如ELK，将各个机器上的日志整合起来，然后通过traceId来搜索出对应的请求链路信息。

###### 1、ELK简介

**Elasticsearch：**开源分布式搜索引擎，特点：分布式、零配置、自动发现、restful风格接口、自动搜索负载等。

**Logstash：**完全开源的工具。对日志进行收集、分析并存储以供以后使用。

**kibana：**可以为**Elasticsearch**和**Logstash**提供web界面。

###### 2、输出JSON格式

可以通过**Logstash**输出、收集JSON格式的日志，然后存储在**ELasticsearch**，然后在**kibana**中查看。

**导入依赖：**

```xml
<!-- 输出 Json 格式日志 -->
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>5.2</version>
</dependency>
```

#### 三、整合ZipKin

ZipKIin致力于手机所有服务的监控数据的分布式跟踪系统，它提供收集数据和查询数据两大接口服务。

###### 1、命令开启服务

E:\Zipkin>java -jar zipkin.jar

###### 2、访问地址

**http://localhost:9411/zipkin/**

###### 3、项目集成Zipkin发送调用链数据

导入依赖和添加配置：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zipkin</artifactId>
</dependency>
```

```yaml
spring:
  zipkin:
    base-url: http://localhost:9411  #配置ZipKin Server 的地址
```

![image-20200609185033129](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200609185033129.png)

###### 4、抽样采集数据

在实际使用中可能调用很多接口，但是ZipKin中只有一条数据，这是因为手机信息是有一定比例的。ZipKin中的数据条数与调用接口次数默认比例是0.1。

```yaml
spring:
  sleuth:
    sampler:
      probability: 1.0  #抽样比例
```

在高并发情况下，大量的请求如果都采集，很影响性能，采取抽样做法可以减少一部分数据量。

###### 5、TracingFilter

**TracingFilter**是负责处理请求和响应的组件，可以实现扩展性的需求。

```java
@Component
@Order(TraceWebServletAutoConfiguration.TRACING_FILTER_ORDER + 1)
public class MyFilter extends GenericFilterBean {

    private final Tracer tracer;

    MyFilter(Tracer tracer){
        this.tracer = tracer;
    }

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        Span currentSpan = this.tracer.currentSpan();
        if (currentSpan == null){
            filterChain.doFilter(servletRequest,servletResponse);
            return;
        }
        ((HttpServletResponse) servletResponse).addHeader("ZIPKIN-TRACE-ID", currentSpan.context().traceIdString());
        currentSpan.tag("custom", "tag");
        filterChain.doFilter(servletRequest, servletResponse);
    }
}
```

![image-20200609191026985](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200609191026985.png)

##### 6、使用RabbitMq代替Http发送调用链数据

利用消息队列来提高发送性能，保证数据不丢失。
导入依赖

```xml
<dependency>
    <groupId>org.springframework.amqp</groupId>
    <artifactId>spring-rabbit</artifactId>
</dependency>
```

```yaml
spring:
  application:
    name: sleuth-article-service
  zipkin:
    sender:
      type: rabbit #发送调用链数据的方式

  rabbitmq:
    addresses: amqp://192.168.2.173:5672
    username: wang
    password: 1234
```

**开启ZipKin服务**

**java -DRABBIT_ADDRESSES=192.168.2.173:5672 - DRABBIT_USER=wang -DRABBIT_PASSWORD=1324 -jar zipkin.jar** 

###### 7、使用Elasticsearch存储调用链数据

ZipKin收集的数据会存储在ZipKin服务的内存中，容易丢失，可以使用Elasticsearch来存储数据，实现数据的持久化。