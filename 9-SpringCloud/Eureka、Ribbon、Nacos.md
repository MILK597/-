# 一、Eureka

## 搭建eureka-server

**1.引入eureka-server依赖**

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    <!--注意这里的依赖名为 xxx-server!!-->
</dependency>
```

**2.在启动类上添加*@EnableEurekaServer*注解，开启eureka的注册中心功能**

**3.application.yml**

```yaml
server:
  port: 10086
spring:
  application:
    name: eureka-server
eureka:
  client:
    service-url: 
      defaultZone: http://127.0.0.1:10086/eureka
```

## 服务注册（服务提供者）

**1）引入eureka-client依赖**

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    <!--注意这里的依赖名为 xxx-client!!-->
</dependency>
```

**2）application.yml文件**

```yaml
spring:
  application:
    name: userservice  #服务提供者的名称
eureka:
  client:
    service-url:
      defaultZone: http://127.0.0.1:10086/eureka #eureka服务端的地址
```

## 服务发现（服务消费者）

前两步同服务注册。只不过application.yml中的 `spring.application.name` 的值为服务消费者的名字。

**3）服务拉取和负载均衡**

在服务消费者的启动类上，给**RestTemplate**这个Bean添加一个**@LoadBalanced注解**：

![image-20210713224049419](img/image-20210713224049419.png)

在服务消费者进行远程调用服务提供者的地方，修改访问的url路径，**用服务名代替ip、端口**：

![image-20210713224245731](img/image-20210713224245731.png)

spring会自动帮助我们从eureka-server端，根据服务提供者的名称（userservice），获取实例列表（ip地址、端口号），而后完成负载均衡，选择一个实例对其发送请求。

# 二、Ribbon负载均衡

SpringCloud底层其实是利用了一个名为**Ribbon的组件，来实现负载均衡功能的**。

Ribbon跟Eureka一样，都是Netflix提供的。

## 基本原理

Ribbon的底层采用了一个拦截器，拦截了RestTemplate发出的请求，对地址做了修改。用一幅图来总结一下：

![image-20210713224724673](img/image-20210713224724673.png)

基本流程如下：

- 拦截我们的RestTemplate请求`http://userservice/user/1`
- **RibbonLoadBalancerClient**会从请求url中获取服务名称，也就是user-service
- **DynamicServerListLoadBalancer**根据user-service到注册中心（eureka）拉取服务列表
- 注册中心（eureka）返回列表：localhost:8081、localhost:8082
- **IRule**利用内置负载均衡规则，从列表中选择一个，例如localhost:8081
- **RibbonLoadBalancerClient**修改请求地址，用localhost:8081替代userservice，得到`http://localhost:8081/user/1`，发起真实请求



## 负载均衡策略

负载均衡的规则都定义在**IRule接口**中，而**IRule**有很多不同的实现类：

| **内置负载均衡规则类**    | **规则描述**                                                 |
| ------------------------- | ------------------------------------------------------------ |
| RoundRobinRule            | 简单轮询服务列表来选择服务器。它是Ribbon默认的负载均衡规则。 |
| AvailabilityFilteringRule | 对以下两种服务器进行忽略：   （1）在默认情况下，这台服务器如果3次连接失败，这台服务器就会被设置为**“短路”状态**。短路状态将持续30秒，如果再次连接失败，短路的持续时间就会几何级地增加。  （2）并发数过高的服务器。如果一个服务器的并发连接数过高，配置了AvailabilityFilteringRule规则的客户端也会将其忽略。并发连接数的上限，可以由客户端的`<clientName>.<clientConfigNameSpace>.ActiveConnectionsLimit`属性进行配置。 |
| WeightedResponseTimeRule  | 为每一个服务器赋予一个权重值。服务器响应时间越长，这个服务器的权重就越小。这个规则会随机选择服务器，这个权重值会影响服务器的选择。 |
| **ZoneAvoidanceRule**     | 以区域可用的服务器为基础进行服务器的选择。使用Zone对服务器进行分类，这个**Zone**可以理解为一个机房、一个机架等。而后再**对Zone内的多个服务做轮询**。 |
| BestAvailableRule         | 忽略那些短路的服务器，并选择并发数较低的服务器。             |
| RandomRule                | 随机选择一个可用的服务器。                                   |
| RetryRule                 | 重试机制的选择逻辑                                           |

### 修改负载均衡策略

1）方式一：

在微服务（服务消费者）的启动类中（或者自己写一个配置类），定义一个新的IRule：

```java
@Bean
public IRule randomRule(){
    return new RandomRule();
}
```

这种方式是**全局生效**，也就是对于该微服务中的所有服务调用都采用这种负载均衡规则

2）方式二：

在微服务（服务消费者）的**application.yml文件**中，添加如下配置：

```yaml
userservice: # 给某个服务提供者配置负载均衡规则，这里是userservice服务
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule # 负载均衡规则 
```

以上方式声明在**访问userservice这个微服务时**使用的负载均衡规则

## 饥饿加载

Ribbon默认是采用**懒加载**，即第一次发起请求时才会去创建LoadBalancerClient，造成第一次请求响应时间会很长。

而**饥饿加载**则会在项目启动时创建，降低第一次访问的耗时

在服务消费者（order-service）如下配置饥饿加载：

```yaml
ribbon:
  eager-load:
    enabled: true	#开启饥饿加载
    clients: userservice	#指定饥饿加载的服务名称
```

以上配置说明 在order-service（服务消费者）中 访问userservice（服务提供者）这个微服务时使用饥饿加载

**注意：**

​	上述配置中 `clients` 是一个集合，如果有多个微服务,那么需要如下配置：

```yml
ribbon:
  eager-load:
    enabled: true	#开启饥饿加载
    clients: 
      - service1
      - service2
```



# 三、Nacos

Nacos是配置中心、注册中心等。Nacos服务端已经搭建好了，只需要从官网下载即可，所以我们在项目中使用Nacos，我们的微服务就是Nacos的客户端。

## 1. 依赖

引入依赖管理：

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-alibaba-dependencies</artifactId>
            <version>2.2.6.RELEASE</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

引入nacos-discovery依赖

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```



## 2. 配置Nacos服务地址

```yml
spring:
  cloud:
    nacos:
      server-addr: localhost:8848	#Nacos默认端口号 8848
```



## 3. 服务分级存储模型

![image-20220720203529535](img/image-20220720203529535.png)

一个微服务可以包含多个集群，每个集群有多个实例，Nacos将同一机房内的实例划分为一个**集群**。

微服务互相访问时，应该尽可能访问同集群实例，因为本地访问速度更快。当本集群内不可用时，才访问其它集群。

### 3.1 配置集群

在微服务项目的 application.yml 中配置：

```yml
spring:
  cloud:
    nacos:
      server-addr: localhost:8848
      discovery:
        cluster-name: HZ # 集群名称
```

这样，当前项目就是 `HZ` 集群中的一个实例。

### 3.2 同集群优先的负载均衡

Nacos中提供了一个 `IRule` 的实现类，可以**优先从同集群中挑选实例**。如果要想实现同集群优先的负载均衡，修改负载均衡策略即可：

```yml
userservice:
  ribbon:
    NFLoadBalancerRuleClassName: com.alibaba.cloud.nacos.ribbon.NacosRule #改为 NacosRule
```

**NacosRule负载均衡策略**

①优先选择同集群服务实例列表

②本地集群找不到提供者，才到其他集群寻找，并且会输出警告信息

③确定了可用实例列表后，再采用**随机负载均衡规则**选择实例

### 3.3 配置负载均衡权重

**默认情况下NacosRule是同集群内随机挑选**。但是可以通过 Nacos控制台页面配置不同实例的权重，**权重值在 0~1之间，如果权重修改为0，则该实例永远不会被访问**



## 4. 环境隔离

Nacos提供了**namespace（命名空间）**来实现**环境隔离功能**。

- nacos中可以有多个**namespace**
- namespace下可以有**group**、**service**等
- 不同namespace之间相互隔离，例如**不同namespace的服务互相不可见**

**可以通过Nacos控制台来创建 namespace**

### 给微服务配置namespace

application.yml：

```yaml
spring:
  cloud:
    nacos:
      server-addr: localhost:8848
      discovery:
        cluster-name: HZ
        namespace: 492a7d5d-237b-46a1-a99a-fa8e98e4b0f9 # 命名空间，填命名空间的ID，这个ID可以从Nacos控制台获取
```

## 5. 实例类型

Nacos的服务实例分为两种类型：

- **临时实例**：如果实例宕机超过一定时间，会从服务列表剔除。**默认的类型**。

- **非临时实例**：如果实例宕机，不会从服务列表剔除，也可以叫**永久实例**。

**配置一个服务实例为永久实例**：

```yaml
spring:
  cloud:
    nacos:
      discovery:
        ephemeral: false # 设置为非临时实例
```



## 6. 配置管理

Nacos还可以用作配置中心，微服务可以从Nacos拉取配置。

### 6.1 添加配置

可以在Nacos控制台中给微服务添加配置：

![image-20210714164742924](img/image-20210714164742924.png)

然后在弹出的表单中，填写配置信息：

![image-20210714164856664](img/image-20210714164856664.png)

**注意：**项目的核心配置，**需要热更新的配置才有放到nacos管理的必要**。基本不会变更的一些配置还是保存在微服务本地比较好。



### 6.2 微服务拉取配置

**微服务要拉取nacos中管理的配置，并且与本地的application.yml配置合并，才能完成项目启动。** 但是 Nacos的地址是写在application.yml中的，如果尚未读取application.yml，又如何得知nacos地址呢？

因此Spring引入了一个新的配置文件：**bootstrap.yaml文件**，会在application.yml之前被读取，流程如下：

![img](img/L0iFYNF.png)

所以我们可以把Nacos地址写在 **bootstrap.yml文件**中。

#### 1）**nacos-config**依赖

```xml
<!--nacos配置管理依赖-->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
```

#### 2）bootstrap.yaml

在微服务中添加一个bootstrap.yaml文件，内容如下：

```yaml
spring:
  application:
    name: userservice # 1.服务名称
  profiles:
    active: dev #2.开发环境，这里是dev 
  cloud:
    nacos:
      server-addr: localhost:8848 # Nacos地址
      config:
        file-extension: yaml #3.Nacos管理的配置文件后缀名
```

微服务启动时会根据`spring.cloud.nacos.server-addr`获取nacos地址，再根据

`${spring.application.name}-${spring.profiles.active}.${spring.cloud.nacos.config.file-extension}`作为文件id，来拉取Nacos中管理的配置。

本例中，就是拉取 `userservice-dev.yaml`。



### 6.3 配置热更新

我们最终的目的，是希望修改nacos中的配置后，**微服务中无需重启即可让配置生效**，也就是**配置热更新**。

要实现配置热更新，可以使用两种方式：

**方式一**：

在@Value注入的变量所在类上添加注解**@RefreshScope**：

![image-20210714171036335](img/image-20210714171036335.png)

> 上图中@Value读取的配置项就是Nacos中的配置文件中的。

**方式二（推荐）**：

使用**@ConfigurationProperties注解** 把配置信息封装成一个配置类。然后在需要读取配置信息的地方注入该配置类即可。

例如：

```java
@Component
@Data
@ConfigurationProperties(prefix = "pattern")
public class PatternProperties {
    private String dateformat;
}
```

**注意：** 使用**@ConfigurationProperties** 需要在配置类上添加 **@Component注解** 或者在启动类上添加 **@EnableConfigurationProperties注解** 。并且配置类一定要有 setter方法。



### 6.4 多环境配置文件共享配置

有时候多个环境都需要有一些相同的配置。可以把这些各个环境相同的配置另外写在一个配置文件中供微服务拉取。

其实微服务启动时，会去nacos读取两个配置文件，例如：

1. 带环境的配置文件：`[spring.application.name]-[spring.profiles.active].yaml`，例如：`userservice-dev.yaml`

2. 不带环境的配置文件：`[spring.application.name].yaml`，例如：`userservice.yaml`

而`[spring.application.name].yaml`不包含环境，因此可以被多个环境共享，所以多环境共享配置可以写入该文件。



当nacos、服务本地等同时出现相同属性时，优先级有高低之分：

![image-20210714174623557](img/image-20210714174623557.png)

也就是说，当上述三种配置文件中都有一个相同的属性时，最终的属性值以优先级最高的配置文件中的属性值为准。

## 7. Nacos集群

搭建Nacos集群见笔记。