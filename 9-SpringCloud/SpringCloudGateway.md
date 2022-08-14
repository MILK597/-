网关的**核心功能特性**：

- 请求路由和负载均衡
- 权限控制
- 限流

## 1. Maven依赖

引入 SpringCloud的dependencyManagement：

```xml
<dependencyManagement>
    <dependencies>
        <!--由于我们使用Nacos作为注册中心，所以也需要引入这个-->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-alibaba-dependencies</artifactId>
            <version>2.2.6.RELEASE</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <!--SpringCloud-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>Hoxton.SR8</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

引入SpringCloudGateway依赖：

```xml
<!--网关依赖-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
<!--还需要引入Nacos服务发现依赖-->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```



## 2. 路由配置

创建**application.yml文件**，内容如下：

```yaml
server:
  port: 10010 # 网关端口
spring:
  application:
    name: gateway # 网关服务名称
  cloud:
    nacos:
      server-addr: localhost:8848 # nacos地址
    gateway:
      routes: # 网关路由配置
        - id: user-service # 路由id，自定义，只要唯一即可
          # uri: http://127.0.0.1:8081 # 路由的目标地址 http就是固定地址
          uri: lb://userservice # 路由的目标地址 lb就是负载均衡，后面跟服务名称
          predicates: # 路由断言，也就是判断请求是否符合路由规则的条件
            - Path=/user/** # 这个是按照路径匹配，只要以/user/开头就符合要求
```

本例中，我们将 `/user/**`开头的请求，代理到`lb://userservice`，lb是负载均衡，根据服务名拉取服务列表，实现负载均衡。



路由配置包括：

1. **路由id**：路由的唯一标示

2. **路由目标（uri）**：路由的目标地址，前缀使用`http`代表固定地址，前缀使用`lb`代表根据服务名负载均衡

3. **路由断言（predicates）**：判断路由的规则，

4. **路由过滤器（filters）**：对请求或响应做处理



### 路由断言

路由断言就是当请求满足一定条件时，网关才会把请求转发到相应的微服务。

我们在配置文件中写的断言规则只是字符串，这些字符串会被**Predicate Factory（断言工厂）**读取并处理，转变为路由判断的条件。

还有一些断言规则如下：

| **名称**   | **说明**                       | **示例**                                                     |
| ---------- | ------------------------------ | ------------------------------------------------------------ |
| After      | 是某个时间点后的请求           | `- After=2037-01-20T17:42:47.789-07:00[America/Denver]`      |
| Before     | 是某个时间点之前的请求         | `- Before=2031-04-13T15:14:47.433+08:00[Asia/Shanghai]`      |
| Between    | 是某两个时间点之前的请求       | `- Between=2037-01-20T17:42:47.789-07:00[America/Denver],  2037-01-21T17:42:47.789-07:00[America/Denver]` |
| Cookie     | 请求必须包含某些cookie         | `- Cookie=chocolate, ch.p`                                   |
| Header     | 请求必须包含某些header         | `- Header=X-Request-Id, \d+`                                 |
| Host       | 请求必须是访问某个host（域名） | `- Host=**.somehost.org,**.anotherhost.org`                  |
| Method     | 请求方式必须是指定方式         | `- Method=GET,POST`                                          |
| **Path**   | 请求路径必须符合指定规`则`     | `- Path=/red/{segment},/blue/**`                             |
| Query      | 请求参数必须包含指定参数       | `- Query=name,Jack或者- Query=name`                          |
| RemoteAddr | 请求者的ip必须是指定范围       | `- RemoteAddr=192.168.1.1/24`                                |



### 路由过滤器

路由过滤器，可以对进入网关的**请求**和微服务返回的**响应**做处理，比如添加请求头、添加响应头等。

**注：过滤器是在路由之后执行：**

![image-20220721101140600](img/image-20220721101140600.png)



常用的过滤器种类：

| **名称**                 | **说明**                     |
| ------------------------ | ---------------------------- |
| **AddRequestHeader**     | 给当前请求添加一个请求头     |
| **RemoveRequestHeader**  | 移除请求中的一个请求头       |
| **AddResponseHeader**    | 给响应结果中添加一个响应头   |
| **RemoveResponseHeader** | 从响应结果中移除有一个响应头 |
| **RequestRateLimiter**   | 限制请求的流量               |



**配置过滤器（以AddRequestHeader为例）**

只需要修改gateway服务的application.yml文件，添加路由过滤即可：

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service 
          uri: lb://userservice 
          predicates: 
            - Path=/user/** 
          filters: # 过滤器
            - AddRequestHeader=Truth, Itcast is freaking awesome! # 添加请求头，这个表示添加了一个 Truth：Itcast is freaking awesome! 的请求头
```

**当前过滤器写在user-service路由下，因此仅仅对访问userservice的请求有效**。

如果要**对所有的路由都生效**，则可以将过滤器工厂写到**default-filters**下。格式如下：

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service 
          uri: lb://userservice 
          predicates: 
            - Path=/user/**
      default-filters: # 默认过滤项。这样就对所有的路由都会添加请求头
        - AddRequestHeader=Truth, Itcast is freaking awesome! 
```



## 3. 全局过滤器

上面的路由过滤器是SpringCloudGateway提供的，处理逻辑是固定的。我们如果想要自定义过滤逻辑，就需要编写代码实现。

实现方式是实现**GlobalFilter接口**：

```java
public interface GlobalFilter {
    /**
     *  处理当前请求，有必要的话通过GatewayFilterChain将请求交给下一个过滤器处理
     *
     * @param exchange 请求上下文，里面可以获取Request、Response等信息
     * @param chain 用来把请求委托给下一个过滤器 
     * @return  Mono<Void> 返回标示当前过滤器业务结束
     */
    Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain);
}
```

我们可以在filter方法中编写自定义逻辑，实现下列功能：

- 登录状态判断
- 权限校验
- 请求限流



例子：

定义全局过滤器，拦截请求，判断请求的参数是否满足下面条件：

- 参数中是否有authorization，

- authorization参数值是否为admin

如果同时满足则放行，否则拦截

实现方式：创建一个类实现 GlobalFilter接口，并把它加入到Spring容器中：

```java
@Order(-1)		//需要加上这两个注解，Order注解的值指定该过滤器的优先级，数值越小优先级越高。也可以不添加该注解，而让该全局过滤器实现 Ordered接口，在其getOrder()方法中返回优先数
@Component
public class AuthorizeFilter implements GlobalFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 1.获取请求参数
        MultiValueMap<String, String> params = exchange.getRequest().getQueryParams();
        // 2.获取authorization参数
        String auth = params.getFirst("authorization");
        // 3.校验
        if ("admin".equals(auth)) {
            // 放行
            return chain.filter(exchange);
        }
        // 4.拦截
        // 4.1.禁止访问，设置状态码
        exchange.getResponse().setStatusCode(HttpStatus.FORBIDDEN);
        // 4.2.结束处理
        return exchange.getResponse().setComplete();
    }
}
```



### 过滤器执行顺序

请求进入网关会碰到三类过滤器：**当前路由的过滤器、DefaultFilter、GlobalFilter**

**请求路由后**，会将当前路由过滤器和DefaultFilter、GlobalFilter，合并到一个**过滤器链（集合）**中，**排序后**依次执行每个过滤器：

![image-20210714214228409](img/image-20210714214228409.png)



**排序的规则**是什么呢？

- 每一个过滤器都必须指定一个int类型的order值，**order值越小，优先级越高，执行顺序越靠前**。
- **GlobalFilter**通过实现**Ordered接口**，或者添加**@Order注解**来指定order值，由我们自己指定
- **路由过滤器**和**defaultFilter**的order由Spring指定，**默认是按照声明顺序从1递增**。
- 当以上三种过滤器的order值一样时，会按照 `defaultFilter > 路由过滤器 > GlobalFilter`的顺序执行。



## 4. 跨域问题配置

```yaml
spring:
  cloud:
    gateway:
      # 。。。
      globalcors: # 全局的跨域处理
        add-to-simple-url-handler-mapping: true # 解决options请求被拦截问题
        corsConfigurations:
          '[/**]':
            allowedOrigins: # 允许哪些网站的跨域请求 
              - http://localhost:8090
            allowedMethods: # 允许的跨域ajax的请求方式
              - GET
              - POST
              - DELETE
              - PUT
              - OPTIONS
            allowedHeaders: "*" # 允许在请求中携带的头信息
            allowCredentials: true # 是否允许携带cookie
            maxAge: 360000 # 这次跨域检测的有效期
```





