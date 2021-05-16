# Spring Cloud微服务网关Zuul基础篇
文章首发：[Spring Cloud微服务网关Zuul基础篇](https://www.maishuren.top/archives/springcloud%E5%BE%AE%E6%9C%8D%E5%8A%A1%E7%BD%91%E5%85%B3zuul%E5%9F%BA%E7%A1%80%E7%AF%87)

# 一、概述

Zuul是从设备和网络到后端应用程序所有请求的后门，为内部服务提供可配置的对外URL到服务的映射关系，基于JVM的后端路由器。具有一下的功能：

* 认证与授权
* 压力测试
* 金丝雀测试
* 动态路由
* 负载削减
* 静态相应处理
* 主动流量管理

其底层是基于Servlet，本质就是一系列的Filter所构成的责任链

# 二、入门案例

## 2.1创建父级pom工程

```xml
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.2.3.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
        <spring.cloud-version>Hoxton.SR3</spring.cloud-version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring.cloud-version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <!-- springboot web -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <exclusions>
                <exclusion>
                    <artifactId>spring-boot-starter-tomcat</artifactId>
                    <groupId>org.springframework.boot</groupId>
                </exclusion>
            </exclusions>
        </dependency>
        <!--不用Tomcat,使用undertow -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-undertow</artifactId>
        </dependency>
        <dependency>
            <groupId>io.undertow</groupId>
            <artifactId>undertow-servlet</artifactId>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
```

## 2.2创建Eureka Server

```xml
 	<dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
```

启动类

```java
@EnableEurekaServer
@SpringBootApplication
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

配置文件

```yaml
server:
  port: 8671
eureka:
  instance:
    hostname: localhost
  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```

## 2.3创建Zuul Server

```xml
	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-zuul</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
		</dependency>
	</dependencies>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>
```

创建启动类，`@EnableZuulProxy`注解开启zuul

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableZuulProxy
public class ZuulServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ZuulServerApplication.class, args);
    }
}
```

配置文件

```java
spring:
  application:
    name: zuul-server
server:
  port: 88
eureka:
  client:
    serviceUrl:
      defaultZone: http://${eureka.host:127.0.0.1}:${eureka.port:8671}/eureka/
  instance:
    prefer-ip-address: true
zuul:
  routes:
    service-a:
      path: /client/**
      serviceId: client-a
```

## 2.4创建一个普通服务

```xml
	<dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
    </dependencies>
```

启动类

```java
@SpringBootApplication
@EnableDiscoveryClient
public class ServiceAApplication {

    public static void main(String[] args) {
        SpringApplication.run(ServiceAApplication.class, args);
    }
}
```

controller

```java
@RestController
public class TestController {

	@GetMapping("/add")
	public Integer add(Integer a, Integer b){
		return a + b;
	}
}
```

配置文件

```yaml
server:
  port: 8080
spring:
  application:
    name: service-a
eureka:
  client:
    serviceUrl:
      defaultZone: http://${eureka.host:127.0.0.1}:${eureka.port:8671}/eureka/
  instance:
    prefer-ip-address: true
```

运行Eureka Server、Zuul Server和Service A

> 分别访问：
>
> http://localhost:8080/add?a=1&b=2
>
> http://localhost:88/client/add?a=1&b=2
>
> 返回一样的结果，都是被service-a的controller处理了，8080端口的是service-a，88端口的是Zuul Server。说明Zuul成功把请求转发到了service-a去了。

# 三、Zuul的典型配置

## 3.1路由配置

**1、路由配置简化与规则**

（1）单实例serviceId映射，如下，是一个从/client/**到服务client-a的一个映射规则。也有简化的写法

```yaml
zuul:
  routes:
    service-a:
      path: /client/**
      serviceId: client-a
```

简化的写法：

```yaml
zuul:
  routes:
    client-a: /client/** 
```

更加简化的写法：

```yaml
zuul:
  routes:
    client-a:
```

相当于：

```yaml
zuul:
  routes:
    service-a:
      path: /client-a/**
      serviceId: client-a
```

（2）单实例url映射

除了路由到服务外，还能路由到物理地址，将serviceId替换成url即可

```yaml
zuul:
  routes:
    service-a:
      path: /client-a/**
      url: http://localhost:8080 # client-a的地址
```

（3）多实例路由

在默认的情况下，zuul会使用Eureka中集成的Ribbon的基本负载均衡功能，如果想要使用Ribbon的负载均衡的功能，就需要指定一个serviceId，此操作需要禁止Ribbon使用Eureka，在E版本之后，新增了负载均衡策略的配置，下面是脱离Eureka使用Ribbon的负载均衡，如果本来就是用Eureka作为注册中心的话，不用下面的配置，因为Eureka已经集成了Ribbon默认是轮询的负载均衡的策略。

```yaml
########################## 脱离eureka让zuul结合ribbon实现路由负载均衡  ##########################

zuul:
  routes:
    ribbon-route:
      path: /ribbon/**
      serviceId: ribbon-route

ribbon:
  eureka:
    enabled: false  #禁止Ribbon使用Eureka

ribbon-route:
  ribbon:
    NIWSServerListClassName: com.netflix.loadbalancer.ConfigurationBasedServerList
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule     #Ribbon LB Strategy
    listOfServers: localhost:7070,localhost:7071     #client services for Ribbon LB
```

（4）forward本地跳转

有时候我们在zuul会做一些逻辑处理，在网关中写好一个接口

```java
@RestController
public class TestController {

	@GetMapping("/forward")
	public Integer add(Integer a, Integer b){
		return "跳转"+a + b;
	}
}
```

在访问/client接口的时候跳转到这个方法处理，因此需要在zuul里配置本地跳转

```yaml
zuul:
  routes:
    service-a:
      path: /client-a/**
      serviceId: forward:/forward
```

（5）相同路径的加载规则

有一种特殊的情况，为映射路径指定多个serviceId，那么它该加载哪个服务

```yaml
zuul:
  routes:
    service-a:
      path: /client/**
      serviceId: client-a
    service-b: 
      path: /client/**
      serviceId: client-b
```

上面的配置，它总是会路由到后面的哪个服务，也就是client-b。在yml解析器工作的时候，如果同一个映射路径对应多个服务，按照加载顺序，最末尾加载的映射规则会把之前的映射规则覆盖掉。

**2.路由通配符**

| 规则 | 解释                     | 示例                                      |
| ---- | ------------------------ | ----------------------------------------- |
| /**  | 匹配任意数量的路径和字符 | /order/add、/order/query、/order/detail/1 |
| /*   | 匹配任意数量的字符       | /order/add、/order/query                  |
| /?   | 匹配单个字符             | /order/a、/order/b、/order/c              |

## 3.2功能配置

**1.路由前缀**

在配置路由规则的时候，我们可以配置一个统一的代理前缀。

```yaml
zuul:
  prefix: /api #使用api指定前缀
  routes:
    client-a:
      path: /client/**
      serviceId: client-a 
```

例如在访问client-a服务的接口时要加上`/api`前缀：`/api/client/add`。也可以使用`stripPrefix=false`关闭此功能。如下，这样请求client-a 服务的接口时，就不需要带上`/api`

```yaml
zuul:
  prefix: /api #使用api指定前缀
  routes:
    client-a:
      path: /client/**
      serviceId: client-a 
      stripPrefix: false
```

**2.服务屏蔽与路径屏蔽**

有时候为了避免某些服务或者路径的入侵，可以将他们屏蔽掉。加上了ignored-service和ignored-patterns之后，zuul在拉去服务列表，创建映射规则的时候，就会忽略掉client-b服务和`/**/div/**`接口。

```yaml
zuul:
  ignored-service: client-b   #忽略的服务，防服务入侵
  ignored-patterns: /**/div/** #忽略的接口，屏蔽接口
  routes:
    client-a:
      path: /client/**
      serviceId: client-a
```

**3.敏感头信息**

在构建系统时，使用HTTP的header传值是很方便，协议的一些认证信息默认也在header里面，例如token，cookies。在zuul的配置里面可以指定敏感头，切断它和下层服务之间的交互。

```yaml
zuul:
  routes:
    client-a:
      path: /client/**
      serviceId: client-a
      sensitiveHeaders: Cookie,Set-Cookie,Authorization
```



**4.重定向问题**

在通过网关访问认证服务时，认证服务返回重定向地址，这时候host会重定向到了认证服务的地址，这样就会导致认证服务的地址暴露，这很明显不是我们想要的。所有需要设置一下重定向的，在配置文件中配置就可以解决。

![](https://cdn.jsdelivr.net/gh/MaiSR9527/blog-pic/springcloud/zuul-01.png)

```yaml
zuul:
  add-host-header: true
  routes:
    client-a:
      path: /client/**
      serviceId: client-a
```



**5.重试机制**

在生产环境中，由于各种原因，可能使得一次的请求偶然失败，考虑到用户体验，不能通过有感知的操作来触发重试，这时候就需要用到重试机制了。在配置中开启重试功能，并且此功能要慎用，一些接口要保证幂等性。

```yaml
zuul:
  retryable: true
 ribbon:
   MaxAutoRetries: 1 #同一个服务重试的次数（除去首次）
   MaxAutoRetriesNextServer: 1 #切换相同服务数量
```