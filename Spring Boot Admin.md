# Spring Boot Admin

可以用于监控和管理，能够将**actuator**中的信息进行界面化的展示，也可以监控所有的Spring Boot应用的健康情况。

主要功能点：

+ 显示应用程序的监控状态
+ 应用程序上下文监控
+ 查看JVM、线程信息
+ 可视化的查看日志以及下载日志文件
+ 动态切换日志级别
+ Http请求信息跟踪......

#### 一、Spring Boot Admin的使用方法

###### 1、导入依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>spring-boot-admin-starter-server</artifactId>
</dependency>
```

###### 2、开启注解，配置

```java
@EnableAdminServer
```

```properties
server.port=9091
```

###### 3、访问端口

#### 二、将服务注册到Spring Boot Admin中

###### 1、导入依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>spring-boot-admin-starter-client</artifactId>
</dependency>
```

###### 2、添加配置信息

```properties
server.port=9092
spring.boot.admin.client.url=http://localhost:9091/
management.endpoints.web.exposure.include=*
```

#### 三、开启服务的安全认证

###### 1、导入依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

###### 2、添加配置信息，服务端

```java
@Configuration
public class SecurityPermitAllConfig extends WebSecurityConfigurerAdapter {

    private final String adminContextPath;

    public SecurityPermitAllConfig(AdminServerProperties adminServerProperties) {
        this.adminContextPath = adminServerProperties.getContextPath();
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        SavedRequestAwareAuthenticationSuccessHandler successHandler = new SavedRequestAwareAuthenticationSuccessHandler();
        successHandler.setTargetUrlParameter("radirectTo");
        //静态资源和登录界面可以不用认证
        http.authorizeRequests().antMatchers(adminContextPath + "/assets/**").permitAll()
                .antMatchers(adminContextPath + "/login").permitAll()
                //其他请求必须认证
                .anyRequest().authenticated()
                //自定义登录和退出
                .and().formLogin()
                .loginPage(adminContextPath + "/login")
                .successHandler(successHandler).and().logout()
                .logoutUrl(adminContextPath + "/logout")
                //启用HTTP-Basic，用于Spring Boot Admin注册
        .and().httpBasic().and().csrf().disable();
    }
}
```

```properties
spring.security.user.name=wang
spring.security.user.password=1234
```

服务端开启了安全认证，客户端也需要配置相同的用户名和密码，否则注册失败

```properties
spring.boot.admin.client.username=wang
spring.boot.admin.client.password=1234
```

#### 三、集成Eureka

###### 1、导入依赖

```xml
<dependency>
   <groupId>de.codecentric</groupId>
   <artifactId>spring-boot-admin-starter-server</artifactId>
</dependency>

<dependency>
   <groupId>org.springframework.cloud</groupId>
   <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
   <version>2.2.3.RELEASE</version>
</dependency>

<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

###### 2、添加配置信息，开启注解

```yaml
server:
  port: 9091
spring:
  security:
    user:
      name: wang
      password: 1234
  application:
    name: spring-boot-admin-eureka

eureka:
  client:
    service-url:
      defaultZone: http://wang:1234@localhost:8761/eureka/

  instance:
    prefer-ip-address: true
    instance-id: ${spring.application.name}.${spring.cloud.client.ip-address}.${server.port}
    lease-expiration-duration-in-seconds: 5
    lease-renewal-interval-in-seconds: 5

management:
  endpoints:
    web:
      exposure:
        include: "*"
```

```java
@EnableAdminServer
@EnableDiscoveryClient
```

#### 四、监控服务

通过**Spring Boot Admin**连接注册中心来查看服务状态，只能在页面查看。为了实现自动监控，通过邮件告警，只需要配置一些邮件的信息就可以使用了。

###### 1、邮件报警：导入依赖

```xml
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-mail</artifactId>
</dependency>
```

###### 2、添加配置信息

```yaml
spring:
#邮件服务
  mail:
    host: smtp.163.com
    username: wx971126@163.com
    password: wx971126
    properties:
      mail.debug: false
      mail.smtp.auth: true
  boot:
    admin:
      notify:
        mail:
          to: 945097967@qq.com  #发送给谁
          from: wx971126@163.com
```

```java
@SpringBootApplication
@EnableAdminServer
@EnableDiscoveryClient
public class SpringBootAdminEurekaApplication {

   public static void main(String[] args) {
      SpringApplication.run(SpringBootAdminEurekaApplication.class, args);
   }

   @Configuration
   public static class SecurityPermitAllConfig extends WebSecurityConfigurerAdapter{
      private final String adminContextPath;


      public SecurityPermitAllConfig(AdminServerProperties adminServerProperties) {
         this.adminContextPath = adminServerProperties.getContextPath();
      }

      @Override
      protected void configure(HttpSecurity http) throws Exception {
         SavedRequestAwareAuthenticationSuccessHandler successHandler = new SavedRequestAwareAuthenticationSuccessHandler();
         successHandler.setTargetUrlParameter("redirectTo");
         // 静态资源和登录页面可以不用认证
         http.authorizeRequests().antMatchers(adminContextPath + "/assets/**").permitAll()
               .antMatchers(adminContextPath + "/login").permitAll()
               // 其他请求必须认证
               .anyRequest().authenticated()
               // 自定义登录和退出
               .and().formLogin()
               .loginPage(adminContextPath + "/login").successHandler(successHandler).and().logout()
               .logoutUrl(adminContextPath + "/logout")
               // 启用HTTP-Basic，用于Spring Boot Admin Client注册
               .and().httpBasic()
               .and().csrf().disable();
      }
   }

}
```

