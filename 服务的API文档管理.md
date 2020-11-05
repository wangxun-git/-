## 服务的API文档管理

#### 一、Swagger简介

用于生成、描述、调用和可视化RESTful风格的Web服务。对REST API定义了一个标准且和语言无关的接口，可以让人和计算机拥有无须访问源码、文档或网络流量监控就可以发现和理解服务的能力。当使用Swagger进行正确的定义，用户可以理解远程服务并使用最少实现逻辑与远程服务进行交互。

**优势：**

+ 支持API自动生成同步的在线文档，通过代码生成文档，不再需要手动编写接口文档。
+ 提供Web页面在线测试API。

#### 二、集成Swagger管理API文档

##### 1、使用swagger生成文档

###### 1、导入依赖

```xml
<dependency>
   <groupId>com.spring4all</groupId>
   <artifactId>swagger-spring-boot-starter</artifactId>
   <version>1.7.1.RELEASE</version>
</dependency>
```

###### 2、添加配置信息

```java
@ApiOperation(value = "新增用户", notes = "详细描述")
@ApiResponses({
        @ApiResponse(code = 200, message = "OK", response = UserDto.class)
})
@PostMapping("/user")
public UserDto addUser(@ApiParam(value = "新增用户参数", required = true) @RequestBody AddUserParam param){
    System.out.println("param.getName() = " + param.getName());
    return new UserDto();
}
```

```yaml
spring:
  application:
    name: swagger-demo

server:
  port: 8081

eureka:
  client:
    service-url:
      defaultZone: http://wang:1234@localhost:8761/eureka/

  instance:
    prefer-ip-address: true
    instance-id: ${spring.application.name}.${spring.cloud.client.ip-address}.${server.port}
    lease-expiration-duration-in-seconds: 5
    lease-renewal-interval-in-seconds: 5
    status-page-url: http://${spring.cloud.client.ip-address}:${server.port}/swagger-ui.html
```

###### 3、接口调试

http://127.0.0.1:8081/swagger-ui.html

##### 2、Swagger注解

+ **@Api：**用在类上，说明该类的作用，可以标记一个Controller类作为Swagger文档资源

  ![image-20200618182608713](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200618182608713.png)

+ **@ApiModel：**用在类上，表示对类进行说明，用于实体类中的参数接收说明，

+ **@ApiModelProperty：**用于字段，表示对model属性的说明。
+ **@ApiParam：**用于Controller中方法的参数说明

![image-20200618181218009](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200618181218009.png)

+ **@ApiOperation：**用在Controller方法上，说明方法的作用，每一个接口的定义

![image-20200618182618559](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200618182618559.png)

+ **@ApiResponse和@ApiResponses：**用于方法上，说明接口响应的一些信息，**@ApiResponses**组装了很多的**@ApiResponse**

+ **@ApiImplicitParams和@ApiImplicitParam：**用于方法上，为单独的请求参数进行说明。

##### 3、Eureka控制台快速查看Swagger文档

```yaml
eureka:
  client:
    service-url:
      defaultZone: http://wang:1234@localhost:8761/eureka/
  instance:
    status-page-url: http://${spring.cloud.client.ip-address}:${server.port}/swagger-ui.html
```

##### 3、Zuul中聚合多个服务Swagger

###### 1、导入依赖

```xml
<dependency>
   <groupId>org.springframework.cloud</groupId>
   <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
   <version>2.2.2.RELEASE</version>
</dependency>
<dependency>
   <groupId>org.springframework.cloud</groupId>
   <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
   <version>2.2.3.RELEASE</version>
</dependency>

<!-- Swagger -->
<dependency>
   <groupId>io.springfox</groupId>
   <artifactId>springfox-swagger-ui</artifactId>
   <version>2.9.2</version>
</dependency>
<dependency>
   <groupId>io.springfox</groupId>
   <artifactId>springfox-swagger2</artifactId>
   <version>2.9.2</version>
</dependency>
```

###### 2、添加配置信息

```yaml
spring:
  application:
    name: swagger-zuul

server:
  port: 8082

eureka:
  client:
    service-url:
      defaultZone: http://wang:1234@localhost:8761/eureka/
  instance:
    prefer-ip-address: true
    instance-id: ${spring.application.name}.${spring.cloud.client.ip-address}.${server.port}
```

```java
@EnableSwagger2
@Component
@Primary
public class DocumentationConfig implements SwaggerResourcesProvider {

    @Autowired
    private DiscoveryClient discoveryClient;

    @Value("${spring.application.name}")
    private String applicationName;

    @Override
    public List<SwaggerResource> get() {
        List<SwaggerResource> resources = new ArrayList<>();
        //排除自身，将其他服务添加进去
        discoveryClient.getServices().stream().filter(s -> !s.equals(applicationName)).forEach(name ->{
            resources.add(swaggerResource(name, "/" + name + "/v2/api-docs", "2.0"));
        });
        return resources;
    }

    private SwaggerResource swaggerResource(String name, String location, String version){
        SwaggerResource swaggerResource = new SwaggerResource();
        swaggerResource.setName(name);
        swaggerResource.setLocation(location);
        swaggerResource.setSwaggerVersion(version);
        return swaggerResource;
    }
}
```

###### 3、展示

![image-20200618222038389](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200618222038389.png)

```java
 private SwaggerResource swaggerResource(String name, String location, String version){
        SwaggerResource swaggerResource = new SwaggerResource();
        swaggerResource.setName(name);
        swaggerResource.setLocation(location);
        swaggerResource.setSwaggerVersion(version);
        return swaggerResource;
    }
```

![image-20200618222108029](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200618222108029.png)