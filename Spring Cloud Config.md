## Spring Cloud Config

+ 用来为分布式系统中的基础设施和微服务应用提供集中化的配置支持；分为服务端和客户端；服务端被称为分布式配置中心，是一个独立的微服务应用，用来连接配置仓库并为客户端提供获取配置信息、加密/解密信息等访问接口。客户端是微服务架构中各个微服务应用或基础设施，通过指定的配置中心来管理应用资源与业务相关的配置内容，并在启动时从配置中心获取和加载配置的信息。

#### 1、构建配置中心

###### 1、导入依赖：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

2、配置相关信息

```yaml
server:
  port: 3344
spring:
  application:
    name: config-server
  cloud:
    config:
      server:
        git:
          uri: https://github.com/wangxun-git/gms
          username: wangxun
          password: wx971013.
          search-paths: blob/master/src/main/resources/application.properties
```

