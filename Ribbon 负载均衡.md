## Ribbon 负载均衡

+ 集中式负载均衡：在消费者和服务提供者中间使用独立的代理方式进行负载。比如：Ngnix；
+ 客户端负载：根据自己的请求做负载。比如：Ribbon。

#### 1、Ribbon模块：

+ Ribbon-loadbalancer负载均衡模块：包括负载均衡的算法。
+ Ribbon-eureka：快速、方便的集成Eureka。
+ Ribbon-transport：基于Netty实现多协议的支持。比如：HTTP、TCP、UDP等。
+ Ribbon-httpclient：基于Apache HTTPClient封装的REST客户端。
+ Ribbon-example：Ribbon实例代码。
+ Ribbon-core：比较核心且通用的代码。

#### 2、Ribbon的使用

###### 1、导入依赖

```xml
<!-- Ribbon负载均衡 -->
<dependency>
    <groupId>com.netflix.ribbon</groupId>
    <artifactId>ribbon</artifactId>
</dependency>
<dependency>
    <groupId>com.netflix.ribbon</groupId>
    <artifactId>ribbon-core</artifactId>
</dependency>
<dependency>
    <groupId>com.netflix.ribbon</groupId>
    <artifactId>ribbon-loadbalancer</artifactId>
</dependency>
<dependency>
    <groupId>io.reactivex</groupId>
    <artifactId>rxnetty</artifactId>
    <version>0.4.9</version>
</dependency>
```

###### 2、编写端口测试

```java
public class RibbonTest {


    public static void main(String[] args) {
        Logger logger = LoggerFactory.getLogger(RibbonTest.class);

        //服务列表
        List<Server> serverList = Arrays.asList(new Server("localhost", 8081), new Server("localhost", 8083));

        //构建负载均衡
        BaseLoadBalancer loadBalancer = LoadBalancerBuilder.newBuilder().buildFixedServerListLoadBalancer(serverList);

        loadBalancer.setRule(new RoundRobinRule());

        //调用5次，比较结果
        for (int i = 0; i < 5; i++){
            String result = LoadBalancerCommand.<String>builder().withLoadBalancer(loadBalancer).build()
                            .submit(new ServerOperation<String>() {
                                @Override
                                public Observable<String> call(Server server) {
                                    try{
                                        String addr = "http://" + server.getHost() + ":" + server.getPort() + "/user/hello";
                                        logger.info("调用地址 : {}", addr);
                                        URL url = new URL(addr);
                                        HttpURLConnection conn = (HttpURLConnection)url.openConnection();
                                        conn.setRequestMethod("GET");
                                        conn.connect();
                                        InputStream in = conn.getInputStream();
                                        byte[] data = new byte[in.available()];
                                        in.read(data);
                                        return Observable.just(new String(data));
                                    }catch (Exception e){
                                        return Observable.error(e);
                                    }
                                }
                            }).toBlocking().first();
            logger.info("调用结果 : {}", result);
        }
    }

}
```

运行结果：

![image-20200729184440955](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200729184440955.png)

#### 3、整合Ribbon

###### 1、导入依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
</dependency>
```

###### 2、配置模板

```java
@Bean
@LoadBalanced
public RestTemplate restTemplate(){
    return new RestTemplate();
}
```

###### 3、调用服务

```java
return restTemplate.getForObject("http://eureka-server-provide/user/hello", String.class);
```

###### 4、Ribbon API

通过Ribbon获取对应的服务信息。

```java
@Autowired
private LoadBalancerClient loadBalancerClient;

@GetMapping("/choose")
public Object chooseUrl(){
    ServiceInstance instance = loadBalancerClient.choose("eureka-server-provide");
    return instance;
}
```



```json
{
    "serviceId": "eureka-server-provide",
    "server": {
        "host": "169.254.105.147",
        "port": 8081,
        "scheme": null,
        "id": "169.254.105.147:8081",
        "zone": "defaultZone",
        "readyToServe": true,
        "instanceInfo": {
            "instanceId": "eureka-server-provide:169.254.105.147.8081",
            "app": "EUREKA-SERVER-PROVIDE",
            "appGroupName": null,
            "ipAddr": "169.254.105.147",
            "sid": "na",
            "homePageUrl": "http://169.254.105.147:8081/",
            "statusPageUrl": "http://169.254.105.147:8081/actuator/info",
            "healthCheckUrl": "http://169.254.105.147:8081/actuator/health",
            "secureHealthCheckUrl": null,
            "vipAddress": "eureka-server-provide",
            "secureVipAddress": "eureka-server-provide",
            "countryId": 1,
            "dataCenterInfo": {
                "@class": "com.netflix.appinfo.InstanceInfo$DefaultDataCenterInfo",
                "name": "MyOwn"
            },
            "hostName": "169.254.105.147",
            "status": "UP",
            "overriddenStatus": "UNKNOWN",
            "leaseInfo": {
                "renewalIntervalInSecs": 5,
                "durationInSecs": 5,
                "registrationTimestamp": 1596857194178,
                "lastRenewalTimestamp": 1596867750769,
                "evictionTimestamp": 0,
                "serviceUpTimestamp": 1596857194178
            },
            "isCoordinatingDiscoveryServer": false,
            "metadata": {
                "management.port": "8081"
            },
            "lastUpdatedTimestamp": 1596857194178,
            "lastDirtyTimestamp": 1596857194170,
            "actionType": "ADDED",
            "asgName": null
        },
        "metaInfo": {
            "serviceIdForDiscovery": "eureka-server-provide",
            "serverGroup": null,
            "instanceId": "eureka-server-provide:169.254.105.147.8081",
            "appName": "EUREKA-SERVER-PROVIDE"
        },
        "alive": true,
        "hostPort": "169.254.105.147:8081"
    },
    "secure": false,
    "metadata": {
        "management.port": "8081"
    },
    "scheme": null,
    "host": "169.254.105.147",
    "port": 8081,
    "uri": "http://169.254.105.147:8081",
    "instanceId": "169.254.105.147:8081"
}
```

#### 4、饥饿加载

如果在进行服务调用时出现超时情况的解决方案：

```yaml
ribbon:
  eager-load:
    enabled: true  #开启饥饿加载模式
    clients: eureka-server-provide  #需要饥饿加载的服务名
```

#### 5、负载均衡策略

+ **BestAvailable：**选择一个最小的并发请求的Server，逐个考察Server，如果被标记错误，则跳过，然后选择ActiveRequestCount中最小的Server。
+ **AvailabilityFilteringRule：**检查Status里记录的各个Server的运行状态，过滤掉那些一直连接失败的，过滤掉高并发的后端Server。
+ **ZoneAvoidanceRule：**使用ZoneAvoidancePredicate和AvailabilityPredicate来判断是否选择某个Server，前一个判断一个Zone的运行性能是否可用，剔除不可用的所有Server，后一个用于过滤掉连接数过多的Server。
+ **RandomRule：**随机选择一个Server。
+ **RoundRobbinRule：**默认负载机制，轮询选择。
+ **RetryRule：**对选定的负载均衡策略机上重试机制，当选择某个策略进行请求负载时在一段时间内若选择Server不成功，会一直尝试连接。
+ **WeightedResponseTimeRule：**根据响应时间分配一个Weight(权重)，响应时间越长，Weight越小，被选中的几率越低。

#### 6、自定义负载策略

```java
public class MyRule implements IRule {

    private ILoadBalancer loadBalancer;

    @Override
    public Server choose(Object key) {
        List<Server> servers = loadBalancer.getAllServers();
        for (Server server : servers){
            System.out.println("server = " + server);
        }
        return servers.get(0);
    }

    @Override
    public void setLoadBalancer(ILoadBalancer lb) {
        this.loadBalancer = lb;
    }

    @Override
    public ILoadBalancer getLoadBalancer() {
        return loadBalancer;
    }
}
```

```yaml
ribbon-config-demo:
  ribbon:
    NFLoadBalancerRuleClassName: com.wangxun.eurekaserverconsume.config.MyRule
```

#### 7、重试机制

当Eureka中服务挂掉之后，但是还会在一段时间保存注册信息，之后Ribbon就可能拿到已经失效的信息，可以使用重试机制来避免。

###### 1、需要指定某个服务的负载均衡机制是重试机制。

```yaml
ribbon-config-demo:
  ribbon:
    NFLoadBalancerRuleClassName: com.netfilx.loadbalancer.RetryRule
```

###### 2、导入依赖

```xml
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
</dependency>
```

###### 3、添加配置信息

```yaml
ribbon:
  maxAutoRetries: 1 #对当前实例的重试次数
  maxAutoRetriesNextServer: 3 #切换实例的重试次数
  okToRetryOnAllOperations: true #对所有操作请求都进行重试
  retryableStatusCodes: 500,404,502 #对Http响应码进行重试
```

