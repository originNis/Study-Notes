# SpringCloud

[TOC]



# 0 - 认识微服务

### 单体架构

在微服务出现前，原始的项目可能更多采用单体架构，即业务的所有功能模块放在一个项目中，整体打包并部署。其优点是架构简单，部署成本低，因为每当需要加入新功能时，直接在原有项目或模块中实现即可；其缺点是耦合度高，一旦代码量变高，不可避免地出现耦合，会影响debug、代码更改、功能添加等难度。

### 分布式架构

在业务项目的体量日趋庞大的今天，为了解决单体架构的问题，分布式架构应运而生。它将一个项目中不同的业务功能**拆分成彼此独立的小项目**，称为**服务**。例如淘宝中的支付功能可以拆分出来单独成为支付服务，用户相关功能可以拆分出来单独成为用户服务。其优点是降低了耦合，各服务间的环境、代码都是独立的，单个服务内进行功能添加、版本升级等不会影响其他服务，因此这也有利于服务升级扩展。

分布式架构需要考虑的问题：

- 服务拆分粒度？
- 服务集群地址如何维护？其他集群如何找到自己？
- 服务之间如何完成远程调用？
- 服务健康状况如何感知？
- ... ...

### 微服务

微服务是一种经过良好架构设计的**分布式架构方案**，是目前最佳实践方案。其架构特征：

- **单一职责**：微服务拆分粒度尽可能小，每一个服务对应唯一的业务能力，避免服务间的重复开发
- **面向服务**：微服务对外暴露业务接口
- **自治**：团队独立、技术独立、数据独立、部署独立
- **隔离性强**：服务做好隔离、容错和降级，避免出现级联故障 

### 微服务结构

微服务作为一种方案是需要技术框架来落地的，目前最火热的微服务落地技术就是Spring Cloud和Dubbo。它们也都包含微服务必需的部分：用于提供业务服务的服务集群、统一管理服务及其地址信息的注册中心、统一管理配置的配置中心、用于接受并路由用户请求的服务网关等。

常见的微服务落地技术对比：

![](assets/SpringCloud.assets/微服务技术对比-17017806184121.png)

其中Spring Cloud是目前全球使用最广泛的微服务框架，而Spring Cloud Alibaba可以看作其组件。

# 基础篇

## 1 - 远程服务调用

### RestTemplate

RestTemplate是Spring框架提供的用于远程调用的类，将其注入Bean即可直接使用。

*示例*

```java
/**
	本Service类位于Order服务中，利用RestTemplate向User服务远程访问，
	之后的例子将常常会重复涉及到Order服务向User服务调用请求
*/
@Service
public class OrderService {
    @Autowired
    private OrderMapper orderMapper;

    @Autowired
    RestTemplate restTemplate;

    public Order queryOrderById(Long orderId) {
        // 先查询订单
        Order order = orderMapper.findById(orderId);
		// 调用restTemplate的方法即可远程调用，第一个参数是访问的url，第二个参数是希望返回的类型
        // 根据订单中的userId查询用户
        User user = restTemplate.getForObject("http://localhost:8081/user/"+order.getUserId(), User.class);
		// 把用户封装到订单类中
        order.setUser(user);

        return order;
    }
}
```

**注意**：本机测试时（一般会开启2个以上的服务，这个示例也是一样，只不过为了简便只展示了远程调用的Service）请注意几个**服务的端口号应当不同**，各服务的**数据库配置也是独立的**。

### 提供者与消费者

服务提供者：一次业务中，提供接口被其他微服务调用的服务。

服务消费者：一次业务中，调用其他微服务的业务。

提供者与消费者需要放在一次业务的语境中，一个服务可能同时为提供者或消费者。

## 2 - Eureka注册中心

![](assets/SpringCloud.assets/Eureka关系图.png)

在Eureka架构中，有两种角色：

- Eureka Server：服务器/注册中心
  - 记录服务信息
  - 通过心跳信息监控服务的存活状态
- Eureka Client：客户端/服务集群
  - 服务提供者：
    - 每次上线时向服务器注册自己的信息
    - 每隔30秒向服务器发送心跳
  - 服务消费者：
    - 每次调用服务时根据服务名称从服务器拉取服务列表
    - 基于服务列表做负载均衡，选择其中一个服务URL发起远程调用

### 搭建Eureka服务器

首先需要引入Eureka相关的依赖，例如可以引入Spring Boot Starter依赖，方便地直接完成依赖引入与自动配置。

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

*在application.yml添加配置*

```yml
# 本服务所在端口
server:
  port: 10086
# 本服务名称
spring:
  application:
    name: eurekaserver
# eureka服务器上线时也需要在eureka注册中心注册，
# 目的是让eureka集群都知道自己的存在，因此要在client配置中添加
eureka:
  client:
    service-url:
      defaultZone: http://localhost:10086/eureka
```

*在启动类添加启用eureka的注解*

```java
// 启用eureka注册中心
@EnableEurekaServer
@SpringBootApplication
public class App {
    public static void main( String[] args ) {SpringApplication.run(App.class, args); }
}
```

启动该服务如果没有出问题，则会看到Spring预设好的Eureka监控页面。

### 注册服务

想要将服务注册到eureka中心，首先也需要导入依赖。前面搭建服务器时我们引入的是spring-cloud-starter-netflix-eureka-server，而普通服务则需要导入spring-cloud-starter-netflix-eureka-client。

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

*然后在application.yml增加配置即可。*

```yaml
server:
  port: 8080
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/cloud_order?useSSL=false
    username: yourUserName
    password: yourPassWord
    driver-class-name: com.mysql.jdbc.Driver
  # 在这里指定自己的服务名称
  application:
    name: orderservice # 我们的user服务则起名为userservice
mybatis:
  type-aliases-package: cn.itcast.user.pojo
  configuration:
    map-underscore-to-camel-case: true
logging:
  level:
    cn.itcast: debug
  pattern:
    dateformat: MM-dd HH:mm:ss:SSS
# 在这里将自己注册到eureka服务中心
eureka:
  client:
    service-url:
      defaultZone: http://localhost:10086/eureka
```

将服务和eureka服务器启动后，在监控页面即可看到包括普通服务和eureka服务器在内的所有服务。

在IDEA中，如果想要启动一个服务的多个示例，可以右键services窗口的服务选择Copy Configuration创建一个副本，注意需要在VM options添加-Dserver.port=xxxx，用于在不同的端口运行该实例。

### 使用eureka服务发现

在第一章远程服务调用的基础上使用eureka服务发现，使其解耦合 。

```java
@MapperScan("cn.itcast.order.mapper")
@SpringBootApplication
public class OrderApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderApplication.class, args);
    }
    // 在启动类创建RestTemplate Bean时，用@LoadBalanced注解
    // 声明启用负载均衡
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

*业务方法中将硬编码的url改为其在注册中心的注册名字*

```java
@Service
public class OrderService {
    @Autowired
    private OrderMapper orderMapper;

    @Autowired
    RestTemplate restTemplate;

    public Order queryOrderById(Long orderId) {
        Order order = orderMapper.findById(orderId);
		// 调用restTemplate的方法即可远程调用，第一个参数是访问的url，第二个参数是希望返回的类型
        // 原本的ip和端口号改为了要访问的服务名称userservice
        User user = restTemplate.getForObject("http://userservice/user/"+order.getUserId(), User.class);

        order.setUser(user);

        return order;
    }
}
```

如果启动了多个userservice实例，并且用orderservice访问多次则会发现，负载均衡确实会将请求分发给不同的实例。这种一个服务多个实例在项目中是常态。

## 3 - Ribbon负载均衡

### 负载均衡流程

1. 当一个业务通过服务发现调用另一个业务时，其本质上是发出了一个请求。

2. 发出的请求被LoadBalancerInterceptor拦截器拦截进行处理。

   ```java
   // LoadBalancerInterceptor源码
   public class LoadBalancerInterceptor implements ClientHttpRequestInterceptor {
       private LoadBalancerClient loadBalancer;
       private LoadBalancerRequestFactory requestFactory;
   
       public LoadBalancerInterceptor(LoadBalancerClient loadBalancer, LoadBalancerRequestFactory requestFactory) {
           this.loadBalancer = loadBalancer;
           this.requestFactory = requestFactory;
       }
   
       public LoadBalancerInterceptor(LoadBalancerClient loadBalancer) {
           this(loadBalancer, new LoadBalancerRequestFactory(loadBalancer));
       }
   
       // 正是这个方法进行拦截并处理
       public ClientHttpResponse intercept(final HttpRequest request, final byte[] body, final ClientHttpRequestExecution execution) throws IOException {
           URI originalUri = request.getURI();
           String serviceName = originalUri.getHost();
           Assert.state(serviceName != null, "Request URI does not contain a valid hostname: " + originalUri);
           return (ClientHttpResponse)this.loadBalancer.execute(serviceName, this.requestFactory.createRequest(request, body, execution));
       }
   }
   ```

3. LoadBalancerInterceptor获取url的服务名，例如userservice，而后调用LoadBalancerClient的execute方法处理。顺带一提，这个接口有两个实现类，一个是RibbonLoadBalancerClient，另一个是BlockingLoadBalancerClient。

4. 这里以RibbonLoadBalancerClient的execute方法为例，它会以服务名为参数，调用getLoadBalancer获取一个ILoadBalancer，它有一个实现类是DynamicServerListLoadBalancer，在这里实际上就已经完成了访问eureka注册中心并取回了服务列表的步骤，并封装在了该对象内。

   ```java
   // RibbonLoadBalancerClient有几个execute方法，这是首先将执行的execute方法，它会调用下面一个execute方法
   public <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException {
       return this.execute(serviceId, (LoadBalancerRequest)request, (Object)null);
   }
   
   public <T> T execute(String serviceId, LoadBalancerRequest<T> request, Object hint) throws IOException {
       ILoadBalancer loadBalancer = this.getLoadBalancer(serviceId);
       Server server = this.getServer(loadBalancer, hint);
       if (server == null) {
           throw new IllegalStateException("No instances available for " + serviceId);
       } else {
           RibbonServer ribbonServer = new RibbonServer(serviceId, server, this.isSecure(server, serviceId), this.serverIntrospector(serviceId).getMetadata(server));
           return this.execute(serviceId, (ServiceInstance)ribbonServer, (LoadBalancerRequest)request);
       }
   }
   ```

5. 随后调用自身的getServer，进入了ILoadBalancer的chooseServer方法。

   ```java
   // RibbonLoadBalancerClient的getServer方法
   protected Server getServer(ILoadBalancer loadBalancer, Object hint) {
       return loadBalancer == null ? null : loadBalancer.chooseServer(hint != null ? hint : "default");
   }
   ```

6. chooseServer根据自身的rule对象选择服务，这里的rule对象是IRule接口类型，它有RoundRobinRule、RandomRule几个实现类，从类名可以看出，不同的实现类实现了轮询、随机等不同的选择服务的策略。而最终他们都会依据hint从服务列表中返回一个具体的服务。

   ```java
   // ILoadBalancer的chooseServer方法
   public Server chooseServer(Object key) {
       if (this.counter == null) {
           this.counter = this.createCounter();
       }
   
       this.counter.increment();
       if (this.rule == null) {
           return null;
       } else {
           try {
               return this.rule.choose(key);
           } catch (Exception var3) {
               logger.warn("LoadBalancer [{}]:  Error choosing server for key {}", new Object[]{this.name, key, var3});
               return null;
           }
       }
   }
   ```

7. 接下来函数层层返回，由LoadBalancerInterceptor将处理完的url发送给被选中的服务。

![](assets/SpringCloud.assets/负载均衡流程.png)

### 负载均衡策略

从以上的流程中我们可以看到Ribbon负载均衡的策略是由IRule的运行类型决定的。

*IRule的继承关系图*

![](assets/SpringCloud.assets/IRule继承关系.png)

**默认的实现是使用了ZoneAvoidanceRule类。**

*各负载均衡策略的描述*

![](assets/SpringCloud.assets/负载均衡策略.png)

### 自定义负载均衡策略

通过以上介绍，很自然地能够想到只要我们也编写实现一个IRule接口的类，就可以自定义负载均衡策略。

#### 代码方式

*在业务消费者的启动类或任意配置类，自定义一个IRule接口类型的Bean*

```java
@Bean
public IRule randomRule() {
    return new RandomRule();
}
```

这里编写了一个RandomRule类型的IRule Bean，因此自定义负载均衡策略是随机。

**注意：代码方式自定义会作用于对任何业务提供者的远程调用。**

#### 配置文件方式

*在业务消费者的配置文件中，例如application.yml里添加配置*

```yaml
# 这是你想访问的业务提供者
userservice:
  # Ribbon负载均衡体系
  ribbon:
    # 选择负载均衡策略
    NFLoadBalancerRuleClassName: com.netflex.loadbalancer.RandomRule
```

这里同样也是配置成了随机负载均衡。

**注意：配置文件方式粒度更细，可以针对不同的业务提供者选择不同负载均衡策略。**

### 加载策略

#### 懒加载

Ribbon默认是懒加载，也就是在访问某个具体服务提供者时才会创建相应的ILoadBalancer，即拉取服务列表。这会导致第一次访问服务提供者时耗时较长。

#### 饥饿加载

可以更改Ribbon的相关配置启用饥饿加载，也就是提前为特定的服务提供者创建ILoadBalancer，拉取服务列表。可以显著降低第一次访问的时长。

*在application.yml更改配置*

```yaml
ribbon:
  eager-load:
    enabled:  true # 默认为false，即懒加载
    clients: # 这是一个列表类型的配置
      - userservice # 指定对这个服务预先拉取服务列表
```

## 4 - Nacos注册中心

Nacos是阿里巴巴开发的注册中心产品，现在是SpringCloud中的一个组件。相比Eureka功能更加丰富，例如支持分布式配置。

在github上即可下载Nacos，与Eureka相比，运行Nacos服务器为我们准备了更清爽的监控界面、更多的监控细节以及更多的功能。

### 服务注册与发现

Nacos注册中心中的注册步骤与Eureka一致，并且因为Spring Cloud Commons定义了注册中心的相关接口，更换注册中心并不需要更改大量代码，这是很方便的。

*为了使用阿里巴巴的Nacos，先引入Spring Cloud Alibaba相关依赖*

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-alibaba-dependencies</artifactId>
    <version>2.2.5.RELEASE</version>
    <type>pom</type>
    <scope>import</scope>
</dependency>
```

*在具体的服务项目的pom.xml中引入Nacos*

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```

*在项目的application.yml文件中添加Nacos配置，但记得删除/注释掉Eureka相关配置*

```yaml
spring:
  cloud:
    nacos:
      server-addr: localhost:8848 # Nacos注册中心地址，不写则默认8848端口
```

启动服务即可在Nacos监控页面出现该服务。

### Nacos服务分级存储模型

Nacos服务分级一般从高到低分为服务、集群、实例，将服务实例分开存储能够提供容灾降级等诸多好处。

![](assets/SpringCloud.assets/Nacos服务分级.png)

#### 把服务部署到集群

*服务部署到集群只需要更改application.yml中的cluster-name信息*

```yaml
spring:
  cloud:
    nacos:
      server-addr: localhost:8848
      discovery:
        # 该处设置
        cluster-name: Manchester
```

#### 消费者优先访问同集群提供者

仅仅在User服务与Order服务中配置以上信息，只能实现普通的服务发现功能。因为默认情况下负载均衡策略是ZoneAvoidanceRule，其将在所有的User服务中挑选服务，所以哪怕User有位于Manchester的实例、有位于Shanghai的实例，Orde服务r将轮询所有User服务而不管其在哪个集群。为了使服务消费者Order服务优先访问同集群的服务，我们要在Order的配置文件中配置如下信息。

```yaml
userservice:
  ribbon:
  	# 指定负载均衡策略为NacosRule
    NFLoadBalancerRuleClassName: com.alibaba.cloud.nacos.ribbon.NacosRule
```

NacosRule策略将优先选择同集群服务，若同集群没有健康服务，才会访问外地集群并向管理员提醒该情况。在同集群中NacosRule执行随机选择服务的策略。

#### 根据权重负载平衡

Nacos界面为我们提供了很方便的权重调整功能，权重越低被选中访问的频率越少，权重为0则不会被选中。权重值范围是0 ~ 1。

这种功能可以为很多场景提供便利，例如服务集群升级时，可以先将一部分服务集群权重调整为0，在不会被用户访问的情况下升级，等升级完毕后逐步调高权重，测试升级后服务能否应对原有的流量，以此做到平滑升级。

### 环境隔离

Nacos使用namespace完成环境隔离的功能，Nacos中的服务存储和数据存储的最外层都是namespace，用于最外层隔离。

![](assets/SpringCloud.assets/Nacos环境隔离.png)

namespace的创建可以通过Nacos的监控页面方便地完成，需要注意的是，namespace的id很重要，当不手动设置id时，Nacos会通过UUID唯一的生成，因此手动设置时也应该保证namespace的id唯一。

*为服务指定所处的namespace还是需要在配置文件里设置*

```yaml
spring:
  cloud:
    nacos:
      server-addr: localhost:8848
      discovery:
        cluster-name: Manchester
        # 该处设置
        namespace: b16a2650-75b6-453e-8f14-c1da95f06afc
```

**不同namespace的服务相互不可访问。**

### Nacos注册中心细节

![](assets/SpringCloud.assets/Nacos注册中心细节.png)

Eureka和Nacos都会在消费者端使用服务列表缓存，以减轻频繁拉取服务列表的开销，但是Eureka只会定时拉取服务，而Nacos会在服务提供者的状态更改时主动推送消息给消费者，服务列表的更新更及时。

Eureka中统一采用心跳监测的方式来确认实例的健康状态，而在Nacos实例细分为临时实例和非临时实例——在配置文件中可以用ephemeral来配置。

```yml
spring:
  cloud:
    nacos:
      discovery:
        ephemeral: true
```

临时实例会主动向注册中心发起心跳，一但临时实例被检测为挂机，注册中心直接将其删除，而非临时实例则由注册中心周期性主动询问健康状况，一旦挂机并不会被删除，依然留存在注册中心的列表中等待其再次上线，但非临时实例会占用更多的服务器开销。

Eureka采用AP方式；而Nacos默认是AP方式，当集群中有非临时实例时则切换为CP方式。

## 5 - Nacos配置管理

在微服务架构中，为了管理众多微服务的配置，存在配置管理中心的角色，而Nacos也拥有配置管理的功能，在Nacos的监控页面就可以看到配置管理相关的条目，我们可以利用可视化页面来进行便捷地配置管理，实现统一配置管理、配置热更新等效果。

### 创建配置

在Nacos监控页面创建配置即可，注意Data Id是一个配置文件的唯一标识，可以采用[服务名]-[环境].[后缀]的形式命名。推荐同样使用yaml配置文件。

### 配置获取的步骤

![](assets/SpringCloud.assets/配置获取步骤.png)

服务启动时，需要先读取Nacos配置文件，再读取本地配置文件进行合并。但是Nacos之前同样需要配置文件来指引服务去哪寻找Nacos以及读取Nacos哪个配置文件。而bootstrap配置文件是一个优先级高于本地配置文件的配置文件，因此应当创建bootstrap配置文件存放Nacos服务的地址以及想要读取的Nacos配置文件的名字。

### 服务读取Nacos配置文件

*要使用Nacos配置管理的服务，则当前服务需要引入相关依赖。*

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
```

*接着创建bootstrap.yml配置文件*

```yaml
spring:
  application:
    name: userservice # 服务名
  profiles:
    active: dev # 环境名
  cloud:
    nacos:
      server-addr: localhost:8848 # Nacos服务的地址
      config:
        file-extension: yaml # 配置文件的后缀
```

其中我们配置的属性记得在application.yml**不要重复配置**。

这里的服务名、环境名与后缀拼接起来就形成了Nacos配置文件的名字。有了bootstrap.yml，服务就知道该去哪个Nacos地址读取哪个Nacos配置文件了。

### 配置热更新

配置的热更新就是指在服务不需要重启的情况下，更改配置文件并且能让服务读取到新的配置信息并应用。

#### @Value注解实现配置热更新

*Nacos中的userservice-dev.yaml配置文件*

```yaml
pattern:
    dataformat: yyyy-MM-dd HH:mm:ss
```

*示例*

```java
@Slf4j
@RestController
@RequestMapping("/user")
// 这个注解实现了当更改Nacos配置文件时，发送消息给服务让其更新配置
@RefreshScope
public class UserController {
    // 在Nacos配置文件配置了pattern.dataformat = yyyy-MM-dd HH:mm:ss
    // 用@Value注解读取了配置信息
    @Value("${pattern.dataformat}")
    private String dataformat;
	// 访问该地址返回一个日期字符串，应该符合配置信息的日期格式
    @GetMapping("now")
    public String now() {
        return LocalDateTime.now().format(DateTimeFormatter.ofPattern(dataformat));
    }
}
```

#### @ConfigurationProperties注解直接实现配置热更新

*用另外一个配置类，使用@ConfigurationProperties获取配置属性*

```java
// 需要提供setter给Spring以注入配置信息
@Data
@Configuration
// 该类会寻找这个前缀的配置信息并试图注入到类的属性
@ConfigurationProperties("pattern")
public class NacosConfig {
    private String dataformat;
}
```

*Controller中改成相应的代码*

```java
@Autowired
private NacosConfig nacosConfig;

@GetMapping("now")
public String now() {
    return LocalDateTime.now().format(DateTimeFormatter.ofPattern(nacosConfig.getDataformat()));
}
```

这种方式获取配置属性，**自动带有热更新的功能**。因此相比前一种需要使用@RefreshScope配合的方式，更推荐这种。

**提醒：**不是所有配置都适合放在配置中心，会导致维护困难。建议将一些关键参数或运行时需要调整的参数放在Nacos配置中心，一般都是自定义功能的配置。

### 多环境配置共享

前面我们在Nacos配置中心创建了针对单个环境的在线配置，配置文件一般取名为：[服务名]-[环境名].yaml，当我们想要创建一份所有的环境配置都通用的配置的时候，我们可以将这种通用配置放在[服务名].yaml中。当一个服务启动时，它其实会同时读取[服务名].yaml与[服务名]-[环境名].yaml配置文件，这两个配置文件当然也是需要像先前一样在服务中的bootstrap配置文件中指出的。

需要注意的是，当[服务名].yaml、[服务名]-[环境名].yaml以及本地配置文件有配置信息冲突的时候，存在读取优先级：

**[服务名].yaml > [服务名]-[环境名].yaml > 本地配置文件**

### Nacos集群

要想实现Nacos注册中心的高可用，那么也得将其集群化部署。但注意这里的Nacos集群是指注册中心从一个变成了多个，不要跟服务集群混淆。

可以使用Nginx代理多个Nacos节点，构成一个Nacos集群：

![](assets/SpringCloud.assets/Nacos集群.png)

这里Nginx的作用是接收外来的请求并作负载均衡处理，再送达某一台Nacos节点。不要跟Nacos为送达服务集群的请求作负载均衡混淆。而MySQL集群是用来存放Nacos所需要的数据，例如我们在Nacos监控页面创建的配置文件等。

## 6 - Feign

### 远程调用

Feign客户端是RestTemplate的替代，相比于需要进行地址拼接的RestTemplate，Feign支持声明式的开发。我们只需要提供接口与方法，就能由Feign创建实例并且供我们调用。

使用Feign需要以下maven依赖：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

*编写一个接口及其方法，规定访问的地址、参数、返回值等信息*

```java
// 核心注解，告诉Feign要远程调用的服务名，它会去本地注册中心查找
@FeignClient("userservice")
public interface UserClient {
    // 访问地址，会拼接在服务地址之后，可以使用Spring Boot中的路径变量
    @GetMapping("/user/{id}")
    public User remoteGetUser(@PathVariable("id") Long id);
}
```

*想要使用Feign，务必在启动类加上注解@EnableFeignClients来启用*

```java
@EnableFeignClients
@MapperScan("cn.itcast.order.mapper")
@SpringBootApplication
public class OrderApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderApplication.class, args);
    }
}
```

*在业务类中注入被Feign创建好的接口实例，业务方法中直接调用方法即可*

```java
@Autowired
UserClient userClient;

public Order queryOrderById(Long orderId) {
    Order order = orderMapper.findById(orderId);
    User user = userClient.remoteGetUser(order.getUserId());
    order.setUser(user);
    return order;
}
```

除了远程调用的基本功能，Feign本身集成了Ribbon，并且**默认就能启用负载均衡**，而不需要RestTemplate中的@LoadBalanced注解。

### 自定义Feign配置

可以在配置文件中给出自定义配置，Feign将用其来覆盖默认配置，可以修改的一些重要配置如下：

![](assets/SpringCloud.assets/Feign配置.png)

#### 修改日志级别

以上配置中，修改日志级别可能在开发调试环节最常用，因此介绍修改日志级别的两种方法。

**配置文件方式**

*全局配置*

```yaml
feign:
  client:
    config:
      # 这里使用default，代表访问任何服务都将输出FULL级别的日志
      default:
        loggerLevel: FULL
```

*局部配置*

```yaml
feign:
  client:
    config:
      # 这里使用具体服务名，代表访问该服务会输出BASIC级别的日志
      userservice:
        loggerLevel: BASIC
```

**代码方式**

*首先需要定义一个Bean，代表输出日志级别*

```java
@Bean
public Logger.Level feignLogerLevel() {
    return Logger.Level.HEADERS;
}
```

同样我们有全局配置和局部配置两种方式：

*全局配置*

```java
// 这是启动类上的注解，传入类对象即可读入配置，并配置到全局，使得对所有访问的服务都应用该行为
@EnableFeignClients(defaultConfiguration = FeignClientConfiguration.class)
```

局部配置

```java
// 这是远程调用接口上的注解，传入类对象读入配置，并只对该服务应用配置
@FeignClient(value = "userservice", configuration = {FeignClientConfiguration.class})
```

**注意**，推荐仅在有调试需求时开启FULL日志级别，因为当日志越详细，系统的开销越大，性能越低。因此将日志级别调低也是性能调优的一种方式。

### Feign客户端底层实现

Feign客户端底层可以使用三个实现：

- URLConnection：默认实现，不支持连接池，性能较低
- Apache HttpClient：支持连接池，性能较高
- OKHttp：支持连接池，性能较高

#### 更换底层实现

以Apache HttpClient为例演示更换底层实现。

首先是引入相关依赖：

```xml
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-httpclient</artifactId>
</dependency>
```

*在配置文件加入配置*

```yaml
feign:
  httpclient:
    enabled: true # 默认开启，也就是引入HttpClient依赖会自动更换实现
    max-connections: 200 # 最大连接数
    max-connections-per-route: 50 # 每个路径的最大连接数
```

与其他的组件和配置也一样，Feign提供了许多能够影响程序性能的配置，为了让项目在使用环境中尽可能高效，使用JMeter等测试工具调试并且慢慢将各种配置更改为最佳配置是很常见的。因此学会根据不同的项目，配置调整到最佳是很珍贵的技能。

### Feign最佳实践

#### 继承方式

这种方式给消费者和提供者先定义一个统一的接口为标准——这是因为Feign远程调用代码与服务Controller代码很相似因此能够实现，然后由消费者和提供者分别继承和实现。但这种方法会导致服务紧耦合。尽管Spring官方不推荐这种做法，并且Spring MVC不为其提供舒适的支持——形参上的注解不会被继承，但是这种实践方式也是有在使用的。

![](assets/SpringCloud.assets/Feign最佳实践-继承.png)

#### “抽取”思想

这种方式让服务提供者将FeignClient抽取为独立的模块/jar包，并且把有关的pojo、默认的Feign配置放在里面，提供给所有消费者使用。能够显著降低服务消费者的编码成本。并且没有紧耦合的缺点。但是它提供的模块往往是全面的，如果消费者只想使用api中的一部分功能，则也必须将模块全部引入，可能导致体积变大。因此两种实践方式各有优劣，实际项目中还是得根据需求来选择。

![](assets/SpringCloud.assets/Feign最佳实践-抽取.png)

## 7 - 统一网关Gateway

### 网关的作用

![](assets/SpringCloud.assets/Gateway网关概述.png)

### 网关的技术实现

在Spring Cloud中网关有两种实现：gateway和zuul。

Zuul是早期的基于Servlet的网关实现，属于阻塞式编程。而gateway基于新版Spring提供的WebFlux实现，属于响应式编程，因此具有更好的性能。

### 搭建网关服务

*引入相关依赖*

```xml
<!-- gateway自动配置依赖 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
<!-- gateway也需要注册到Nacos，因此要引入nacos服务发现依赖 -->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```

*注册到Nacos并且配置路由*

```yaml
server:
  port: 10010
spring:
  application:
    name: gateway
  cloud:
    nacos:
      server-addr: localhost:8848
    gateway:
      routes:
        - id: user-service # 路由标识，必须唯一
          uri: lb://userservice # 路由的目标地址
          predicates: # 路由断言，判断请求是否符合规则，可以看成是if语句，必须满足所有断言才可以路由
            - Path=/user/** # 这个路径只是判断是否符合规则并路由，并不是在服务路径额外再加该前缀
```

这里的URI可以写成HTTP的固定地址格式，也可以以lb开头（意为LoadBalancer）跟服务名的格式；路由断言是用来判断路由是否符合某种规则的配置，如果符合则路由。

最后启动gateway服务，就可以通过只访问gateway的地址来访问所有服务。

### 路由断言工厂Route Predicate Factory

我们在配置文件的predicates属性下所写的断言规则只是字符串，这些字符串在运行过程中会被路由断言工厂所读取，解析成真正的路由判断条件。

![](assets/SpringCloud.assets/基本断言工厂.png)

### 路由过滤器

GatewayFilter是网关中提供的一种过滤器，可以对进入网关的请求和微服务返回的相应做处理。如下图所示，外来的请求会先经过路由获得确切访问地址，然后穿过过滤器链，被每个过滤器处理，最后到达服务。

![](assets/SpringCloud.assets/路由过滤器流程.png)

#### 过滤器工厂

路由过滤器与路由断言一样，也是通过过滤器工厂来实现的。

![](assets/SpringCloud.assets/过滤器工厂.png)

路由过滤器种类很多，但是我们可以在要使用的时候再去官网查阅使用方法。配置路由过滤器的方法并不难，只需要在配置文件仿照官网的示例加上配置即可。接下来以AddRequestHeader为例演示：

*在gateway服务中添加filters配置*

```yaml
server:
  port: 10010
spring:
  application:
    name: gateway
  cloud:
    nacos:
      server-addr: localhost:8848
    gateway:
      routes:
        - id: user-service # 路由标识，必须唯一
          uri: lb://userservice # 路由的目标地址
          predicates: # 路由断言，判断请求是否符合规则
            - Path=/user/**
        - id: order-service # 路由标识，必须唯一
          uri: lb://orderservice # 路由的目标地址
          predicates: # 路由断言，判断请求是否符合规则
            - Path=/order/**
            
          filters: # AddRequestHeader指定了过滤器，
                   # statement, Rybin write here!表示请求头中加入了statement=Rybin write here!
                   # 注意我们是在order-service下添加的配置，因此只对访问order-service的路由生效
            - AddRequestHeader=statement, Rybin write here!
```

*order服务的controller中可以通过@RequestHeader注解拿到请求头中某个值*

```java
@GetMapping("{orderId}")
public Order queryOrderByUserId(@PathVariable("orderId") Long orderId, @RequestHeader("statement") String statement) {
    System.out.println("statement: " + statement);
    // 根据id查询订单并返回
    return orderService.queryOrderById(orderId);
}
```

*如果想对所有请求应用过滤器，而不是过滤对于某个服务的请求，可以配置default-filters*

```yaml
gateway:
  routes:
    - id: user-service # 路由标识，必须唯一
      uri: lb://userservice # 路由的目标地址
      predicates: # 路由断言，判断请求是否符合规则
        - Path=/user/**
    - id: order-service # 路由标识，必须唯一
      uri: lb://orderservice # 路由的目标地址
      predicates: # 路由断言，判断请求是否符合规则
        - Path=/order/**
      filters:
        - AddRequestHeader=statement, Rybin write here!
  # 对访问任意服务的请求都应用该过滤器
  default-filters:
    - AddRequestHeader=statement, Rybin send default!
```

### 全局过滤器GlobalFilter

全局过滤器的作用也是处理一切外来请求和微服务响应，与路由过滤器一样。区别在于路由过滤器的种类是固定的，只有31种官方定义好的处理逻辑。但是GlobalFilter可以让我们自定义处理逻辑，有点类似于Servlet中的Filter。实现方式是自定义类实现GlobalFilter。

*在gateway服务中创建类继承GlobalFilter*

```java
// 与Servlet的Filter类似，order属性控制过滤器的执行先后顺序，数值越小越先执行
@Order(1)
// 把过滤器注册为Bean
@Component
/*
    这个过滤器实现了：当请求访问任意服务时，判断其是否携带authorization并且值为admin，
    如果是，则放行；反之，设置状态码为401，并且拒绝请求。
*/
public class AuthorityFilter implements GlobalFilter {
    @Override
    // exchange参数存放请求的上下文，chain主要用于执行下一个过滤器
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        MultiValueMap<String, String> queryParams = request.getQueryParams();
        String authorization = queryParams.getFirst("authorization");

        if("admin".equals(authorization)) {
            return chain.filter(exchange);
        } else {
            ServerHttpResponse response = exchange.getResponse();
            response.setStatusCode(HttpStatus.UNAUTHORIZED);
            // 使用该方法过滤器就会拒绝该请求
            return response.setComplete();
        }
    }
}
```

除了使用@Order注解控制执行顺序，也可以实现Ordered接口来控制执行顺序：

```java
@Component
public class AuthorityFilter implements GlobalFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        MultiValueMap<String, String> queryParams = request.getQueryParams();
        String authorization = queryParams.getFirst("authorization");

        if("admin".equals(authorization)) {
            return chain.filter(exchange);
        } else {
            ServerHttpResponse response = exchange.getResponse();
            response.setStatusCode(HttpStatus.UNAUTHORIZED);
            return response.setComplete();
        }
    }

    // 简单的函数，直接返回order值，与@Order功能一模一样
    @Override
    public int getOrder() {
        return 1;
    }
}
```

### 过滤器执行顺序

请求在到达网关被路由后，之前所述的3个过滤器——Default过滤器、当前服务的路由过滤器和Global过滤器会被合并到一个集合中排序后作为过滤器链来顺序执行。而这三者能放在一个集合中，说明必然存在一个公共接口被他们所继承：

Default和服务路由过滤器在配置文件中配置，虽然位置不一样，但是配置的名字是一样的，因此它们当然会被各自的过滤器工厂所读取，并解析成相应的过滤器。

*AddRequestHeaderGatewayFilterFactory过滤器工厂源码*

```java
public class AddRequestHeaderGatewayFilterFactory extends AbstractNameValueGatewayFilterFactory {
    public AddRequestHeaderGatewayFilterFactory() {
    }

    public GatewayFilter apply(final AbstractNameValueGatewayFilterFactory.NameValueConfig config) {
        return new GatewayFilter() {
            public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
                String value = ServerWebExchangeUtils.expand(exchange, config.getValue());
                ServerHttpRequest request = exchange.getRequest().mutate().header(config.getName(), new String[]{value}).build();
                return chain.filter(exchange.mutate().request(request).build());
            }

            public String toString() {
                return GatewayToStringStyler.filterToStringCreator(AddRequestHeaderGatewayFilterFactory.this).append(config.getName(), config.getValue()).toString();
            }
        };
    }
}
```

可以看到这个过滤器工厂执行apply函数后，会统一地返回GatewayFilter的实现类，因此Default过滤器和路由过滤器最后都会变成一致的GatewayFilter。而对于GlobalFilter，它会被传入一个适配器：

```java
private static class GatewayFilterAdapter implements GatewayFilter {
    private final GlobalFilter delegate;

    GatewayFilterAdapter(GlobalFilter delegate) {
        this.delegate = delegate;
    }

    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        return this.delegate.filter(exchange, chain);
    }

    public String toString() {
        StringBuilder sb = new StringBuilder("GatewayFilterAdapter{");
        sb.append("delegate=").append(this.delegate);
        sb.append('}');
        return sb.toString();
    }
}
```

可以看到，这个适配器有一个GlobalFilter属性，并且实现了GatewayFilter接口。可想而知，如果GlobalFilter被适配器包装了以后，那类型也就变成了GatewayFilter。因此以上提到的这三者都是GatewayFilter接口的，也自然可以放在同一个集合中。

**过滤器执行顺序**

每一个过滤器都具有一个int类型的order，按照order越小越先执行；之前我们自定义过GlobalFilter，了解了可以通过@Order注解或实现@Ordered接口两种方式来指定order，而对于路由过滤器以及default过滤器，order是由spring指定的，默认是从1开始自增长；当多个过滤器order值一样时，会按照default过滤器 > 路由过滤器 > GlobalFilter的优先级执行。

### 跨域问题

跨域：域名不一致就是跨域。主要包括域名不同，例如www.baidu.com和www.jd.com；以及域名相同但是端口不同，例如localhost:8080和localhost:8081。

跨域问题：**浏览器禁止请求的发起者与服务端发生跨域ajax请求**，导致请求被拦截。由这个定义我们可以得知，尽管之前我们编写的几个微服务运行在不同端口，但它们没有发生跨域问题的原因是：微服务之间的访问通过注册中心而非浏览器，因此不会被浏览器拒绝，就无从谈起发生跨域问题。

解决方案：CORS跨域资源共享

#### Gateway解决跨越问题

与Servlet中处理方式相同，Gateway网关处理跨域请求采用的也是CORS方案，并且只需要在配置文件中添加一些配置即可实现。

```yaml
spring:
  cloud:
    gateway:
      globalcors: # 全局跨域处理
        add-to-simple-url-handler-mapping: true # 解决浏览器的OPTIONS被拦截的问题
        cors-configurations:
          '[/**]': # 对任何服务的请求都允许进行跨域处理
            allowedOrigins:
              - "http://www.baidu.com"
            allowedMethods:
              - "GET"
              - "POST"
              - "PUT"
              - "OPTIONS"
            allowedHeaders: "*" # 允许在请求中携带什么头信息
            allowCredentials: true # 是否携带cookie
            maxAge: 360000 # 每次请求都进行跨域处理会对服务器的性能造成影响，
                           # 因此设置跨域检测的有效期可以降低跨域处理的频率
```

## 8 - Docker

Docker是一个快速交付应用、运行应用的技术：

- Docker允许将开发中的应用、依赖、函数库、配置一起打包形成可移植镜像，并且其运行在容器中，使用沙箱机制彼此隔离；
- Docker镜像中含有完整运行环境，包括系统函数库，使得其不依赖于具体的Linux版本（因为具体的Linux版本，例如Ubuntu、centOS都是基于Linux内核，实现了不同的系统函数库，因此普通的应用无法从其中一个Linux版本直接迁移到另一个版本而不做改动），仅依赖Linux内核，因此可以在任意Linux操作系统下运行；
- 用命令完成操作，方便快捷。

### Docker与虚拟机

虚拟机是在本机的操作系统中使用HyperVisor技术模拟了硬件环境，在这种硬件环境下安装的一个操作系统，它本质上是一台独立的计算机。其系统调用先通过自身的操作系统，再来到外层操作系统进行处理，因此性能较差，启动速度慢，并且虚拟机安装往往需要GB级别的空间的光盘映像，磁盘占用大。

而Docker本身封装的是系统函数库，其本质上是一个程序。开启Docker后它执行的系统调用直接调用本机操作系统内核，性能好，启动速度快，并且其作为程序，磁盘占用自然远比虚拟机小。

### 镜像和容器

镜像Image：Docker将应用程序及其所需要的依赖、函数库、环境、配置等文件打包在一起，称为镜像。

容器Container：镜像中的应用程序形成后的进程就叫做容器，并且Docker会对容器作隔离使其对外不可见。

#### DockerHub

DockerHub是一个Docker镜像托管平台，类似于GitHub。这样的平台称为DockerRegistry。国内也有类似于DockerHub的公开服务，例如网易云镜像服务、阿里云镜像库等。之后当我们需要Redis、Nginx等应用程序的镜像时就可以使用公开DockerRegistry直接用命令拉取这些镜像。

#### Docker架构

![](assets/SpringCloud.assets/Docker架构.png)

### 镜像基本操作

镜像名称由两部分构成：[repository]:[tag]，例如mysql:5.7，当不添加冒号时，默认最新版本，等同于mysql:latest。

*基础命令一图流*

![](assets/SpringCloud.assets/docker基础操作.png)

**注意**，如果docker命令显示服务端拒绝，可能是防火墙拦截了请求，或者是用户权限不够，如果问题是后者可以开启超级用户权限，前者可以视情况关闭防火墙，但如果是云服务器要注意防火墙关闭会带来一定的安全风险。

### 容器相关命令

![](assets/SpringCloud.assets/容器相关命令.png)

#### 创建运行一个Docker容器

从镜像创建容器的指令是docker run，该指令有很多参数，可以在需要时在官网的镜像网站查看，例如Redis、Nginx等镜像网站，官方会提供详尽的帮助文档，说明docker run的参数及其作用。

最简单的docker run命令格式：docker run --name [容器名] -p [本机端口]:[容器端口] -d [镜像名]

以创建Nginx为例：

```shell
docker run --name ns -p 10355:80 -d nginx
```

- --name 为运行的容器起一个唯一标识它的名字；

- -p 为容器指定端口映射，因为docker容器是对外不可见的，因此用户直接访问docker容器的对应端口是不可行的，所以需要将docker容器的端口映射到本机端口，提供给外部访问入口。冒号左边是本机端口80，右边是docker容器端口80，这意味着用户访问本机的10355端口，将会转发到docker容器的80端口，从而访问到服务。docker容器的端口一般是固定的，而映射到本机的端口可以由程序员指定尚没有被占用的端口；

- -d 启动后台运行；

- 最后跟的是镜像名，正如前面所述，带冒号表示指定版本，不带冒号表示最新版本，这个参数表明我们想要启动的是哪个镜像。

启动容器后可以用docker logs [容器名]查看某个容器的日志，-f指令可以持续跟踪其日志信息。

#### 进入容器

进入容器的命令为：

```shell
docker exec -it ns bash
```

- docker exec 进入容器并执行一条命令；
- -it 给当前容器创建一个标准输入输出终端，允许我们与容器交互；
- ns 为我们想进入的容器的名称；
- bash 是进入后想要执行的命令，bash是Linux终端交互命令，其中包含了例如ps、pwd的很多命令；
- 当然其他还有很多命令参数，这里不一一列举。

替换某个文件的文本命令：

```shell
sed -i 's#Welcome to nginx#There is Rybin#g' index.html
```

这句命令把index.html文件中的"Welcome to nginx"替换成了"Tehre is Rybin"，在docker中如果想要更改文本内容，但是没有vim等文本编辑器的话，可以尝试该命令，尽管一般不推荐更改容器的内容。

### 数据卷

在基础的容器使用中存在容器与数据耦合的问题，即每一个容器有自己的数据，多容器无法共享数据，并且当容器因为升级、更新等原因需要删除重启时会导致数据丢失。

![](assets/SpringCloud.assets/容器数据耦合问题.png)

数据卷技术将容器的存储目录映射到宿主主机的某个目录，当容器写入数据时，更改会反应在宿主主机的目录中，因此数据不再和容器耦合，数据同时存储在了宿主主机中；同样地，在宿主主机更改数据也可以直接影响挂载在数据卷下的容器，能够解决我们上面遇到的问题。

![](assets/SpringCloud.assets/数据卷.png)

#### 数据卷操作

数据卷操作是一个二级命令，其基本命令格式：docker volume [命令]，数据卷有如下命令：

- create：创建一个volume；
- inspect：显示一个或多个volume的信息；
- ls：列出所有volume；
- prune：删除未使用的volume；
- rm：删除一个或多个指定的volume。

#### 数据卷挂载到容器

想要将容器的目录和数据卷搭建起映射，需要在容器的创建阶段使用-v参数完成：

```shell
docker run --name ns -v html:/usr/share/nginx/html -p 80:80 -d nginx
```

-v参数的左侧指定了数据卷的名称，如果没有该数据卷docker将会自动创建，右侧指定了容器内被映射的目录。可以通过docker volume inspect html命令查看html数据卷的信息，特别是其挂载点的位置。

当我们修改宿主主机的挂载点位置的文件时，会影响到容器内的目录，这就实现了数据和容器的解耦合。

#### 宿主目录直接挂载到容器

宿主目录直接挂载到容器的命令与数据卷类似：-v [宿主机目录]:[容器内目录]或-v [宿主机文件]:[容器内文件]

数据卷挂载耦合度低，虚拟目录可以帮助我们管理，但是目录结构较深，难定位；目录挂载耦合度高，需要自己管理目录，但是目录位置自己创建，容易定位。

### Docker自定义镜像

Docker镜像的构建是分层的，其中EntryPoint和BaseImage尤为重要，前者提供运行镜像的入口，后者需要提供应用依赖的系统函数库、环境、配置等。另外，这种分层结构使得镜像需要更新升级时，可以不用从头开始构建，只需把过时的Layer剔除更新即可，一定程度上得到了复用性。

![](assets/SpringCloud.assets/镜像结构.png)

#### Dockerfile

Dockerfile是一个用于构建镜像的文本文件，其中包含一个个指令，用指令来说明要执行什么操作，每一个指令都会形成一层Layer。

一些常用指令如下：

![](assets/SpringCloud.assets/dockerfile常用指令.png)

更详尽的指令可以参考官方文档。

*Dockerfile示例*

```dockerfile
# Dockerfile的第一行必须是FROM指令指定基础镜像，
# 可以是基础的操作系统，也可以是别人进行过一定配置的基础镜像，例如java:8-alphine
# 利用配置过的基础镜像，可以复用重复指令，减少重复劳动
FROM ubuntu:22.04
# 配置环境变量，JDK的安装目录
ENV JAVA_DIR=/usr/local

# 拷贝当前文件夹的文件，此处是jdk和java项目的包
COPY ./jdk8.tar.gz $JAVA_DIR/
COPY ./docker-demo.jar /tmp/app.jar

# 安装JDK
RUN cd $JAVA_DIR \
 && tar -xf ./jdk8.tar.gz \
 && mv ./jdk1.8.0_144 ./java8

# 配置环境变量
ENV JAVA_HOME=$JAVA_DIR/java8
ENV PATH=$PATH:$JAVA_HOME/bin

# 暴露端口
EXPOSE 8090
# 入口，java项目的启动命令
ENTRYPOINT java -jar /tmp/app.jar

```

写好了Dockerfile文件后，利用Docker build命令即可构建镜像：

```shell
docker build -t javaweb:1.0 .
```

其中-t参数指定镜像名，最后的点号表示Dockerfile在当前目录下。如果镜像构建成功使用docker images即可查看到。

#### DockerCompose

DockerCompose可以基于Compose文件帮助快速部署分布式应用，而无需逐个创建和运行容器。Compose文件是一个yaml格式文本文件，通过指令定义集群中每个容器如何运行。

一个简单的DockerCompose：

```yaml
version: "3.2"

services:
  nacos:
    image: nacos/nacos-server
    environment:
      MODE: standalone
    ports:
      - "8848:8848"
  mysql:
    image: mysql:5.7.25
    environment:
      MYSQL_ROOT_PASSWORD: 123
    volumes:
      - "$PWD/mysql/data:/var/lib/mysql"
      - "$PWD/mysql/conf:/etc/mysql/conf.d/"
  userservice:
    build: ./user-service
  orderservice:
    build: ./order-service
  gateway:
    build: ./gateway
    ports:
      - "10010:10010"
```

在docker-compose文件所在目录下使用docker-compose相关命令即可一键部署分布式服务。

### 搭建私有Docker镜像仓库

搭建镜像仓库可以基于Docker官方提供的DockerRegistry镜像来实现。

官网地址：https://hub.docker.com/_/registry

#### 简化版镜像仓库

直接使用Docker Registry创建的是简化版的镜像仓库，具备仓库管理的完整功能只是缺乏图形化界面。

搭建命令如下：

```shell
docker run -d \
    --restart=always \
    --name registry	\
    -p 5000:5000 \
    -v registry-data:/var/lib/registry \
    registry
```

命令中挂载了一个数据卷registry-data到容器内的/var/lib/registry目录，这是私有镜像库存放数据的目录。

#### 带有图形化界面的镜像仓库

带有图形化界面的镜像仓库只是借助了一个ui镜像封装了Docker registry，因此我们需要使用DockerCompose同时启动Docker Registry和ui镜像。使用DockerCompose部署带有图象界面的DockerRegistry，命令如下：

```yaml
version: '3.0'
services:
  registry:
    image: registry
    volumes:
      - ./registry-data:/var/lib/registry
  ui:
    image: joxit/docker-registry-ui:static
    ports:
      - 8080:80
    environment:
      - REGISTRY_TITLE=任意私有仓库名称
      - REGISTRY_URL=http://registry:5000
    depends_on: # 先创建registry镜像
      - registry
```

创建的私服使用http协议，因此默认不被Docker信任，需要添加一个配置才可使用：

```shell
# 打开要修改的文件
vi /etc/docker/daemon.json
# 添加内容：
"insecure-registries":["http://192.168.150.101:8080"]
# 重加载
systemctl daemon-reload
# 重启docker
systemctl restart docker
```

#### 在私有仓库推送或拉取镜像

使用push命令推送镜像到私有仓库前需要用tag命令将镜像重命名为：仓库url/镜像名称

```shell
docker tag nginx:latest http://192.168.150.101:8080/nginx:1.0
```

然后使用push命令就可以将其推送到指定url的仓库中：

```sh
docker push http://192.168.150.101:8080/nginx:1.0
```

拉取镜像也要使用全名，这样才能得知是从哪个仓库拉取的：

```shell
docker pull http://192.168.150.101:8080/nginx:1.0
```

## 9 - MQ

### 同步通信

我们之前使用的Feign属于同步通信，同步通信时效性好，但是因为其调用会阻塞，会带来很多问题。

![](assets/SpringCloud.assets/同步调用问题.png)

### 异步通信

异步调用方案最常用的是事件驱动模式：服务调用者并不直接调用服务，而是借助于Broker，服务调用者发通知给Broker后便继续处理自己的业务，不理会服务提供者的执行；服务提供者从Broker接收通知，接收到通知便执行自己的业务。

![](assets/SpringCloud.assets/异步通信优势.png)

这种方式的优点：

- 耦合度低；（支付服务不直接调用任何下游服务）
- 吞吐量上升；（支付服务反馈速度提升，使得吞吐量提升）
- 故障隔离；（下游任何一个服务挂掉都不会导致支付服务的停摆）
- 流量削峰。（Broker可以积攒事件，起到流量缓冲作用，使得下游服务可以按照自己的处理速度从Broker拿取事件进行处理）

缺点：

- 严重依赖Broker的可靠性和处理能力；
- 架构变得复杂，业务没有明显的流程线，不便于管理。

综上所述，同步通信和异步通信各有优劣，因此在生产情景下也得根据实际使用。例如如果需要先调用服务，利用从其获取的数据进行下一步处理，则应当使用同步通信；如果对于服务调用的返回没有需求，并且希望服务解耦、处理高并发场景，则可以考虑异步通信。

### MQ消息队列

MQ(Message queue)，即消息队列，也就是前文中事件驱动模式中的Broker。

常见的MQ有以下几个：

![](assets/SpringCloud.assets/各种消息队列.png)

### RabbitMQ

使用docker拉取rabbitmq:3-management镜像，使用docker run指令即可运行RabbitMQ Server：

```sh
docker run -e RABBITMQ_DEFAULT_USER=rybin -e RABBITMQ_DEFAULT_PASS=123 --name mq \
-p 15672:15672 -p 5672:5672 -d rabbitmq:3-management
```

这里的2个-e参数配置了默认管理员的用户名以及密码，2个-p参数中15672是图形化的网页管理界面，浏览器访问该端口即可在图形化界面监管，5672是服务消息通信的端口。其他参数则都在前文使用过。

#### RabbitMQ的结构

![](assets/SpringCloud.assets/RabbitMQ.png)

channel是操作MQ的工具；queue缓存消息，负责提供消息给具体服务；exchange负责将收到的消息路由到queue中；VirtualHost是对queue、exchange的逻辑分组，完成了不同用户间的隔离功能，在图形化管理界面中一般一个用户拥有一个唯一的VirtualHost来管理自己的逻辑分组。**注意**，RabbitMQ中消息是**阅后即焚**的，即一个queue中的消息被读取一次之后就被销毁了。

#### 基本消息队列BasicQueue

基础消息队列包含三种角色：

- publisher：消息发布者，将消息发布到queue
  - queue：消息队列，负责接收消息并缓存

- consumer：订阅者，处理队列中的消息

![](assets/SpringCloud.assets/基础消息模型.png)

##### 消息发送/接收流程

*消息发送者*

```java
public class PublisherTest {
    @Test
    public void testSendMessage() throws IOException, TimeoutException {
        // 1.建立连接
        ConnectionFactory factory = new ConnectionFactory();
        // 1.1.设置连接参数，分别是：主机名、端口号、vhost、用户名、密码
        factory.setHost("192.168.1.105");
        factory.setPort(5672);
        factory.setVirtualHost("/");
        factory.setUsername("rybbin");
        factory.setPassword("123");
        // 1.2.建立连接
        Connection connection = factory.newConnection();

        // 2.创建通道Channel
        Channel channel = connection.createChannel();

        // 3.创建队列
        String queueName = "simple.queue";
        channel.queueDeclare(queueName, false, false, false, null);

        // 4.发送消息
        String message = "hello, rabbitmq!";
        channel.basicPublish("", queueName, null, message.getBytes());
        System.out.println("发送消息成功：【" + message + "】");

        // 5.关闭通道和连接
        channel.close();
        connection.close();

    }
}
```

*消息接收者*

```java
public class ConsumerTest {

    public static void main(String[] args) throws IOException, TimeoutException {
        // 1.建立连接
        ConnectionFactory factory = new ConnectionFactory();
        // 1.1.设置连接参数，分别是：主机名、端口号、vhost、用户名、密码
        factory.setHost("192.168.1.105");
        factory.setPort(5672);
        factory.setVirtualHost("/");
        factory.setUsername("rybbin");
        factory.setPassword("123");
        // 1.2.建立连接
        Connection connection = factory.newConnection();

        // 2.创建通道Channel
        Channel channel = connection.createChannel();

        // 3.创建队列
        String queueName = "simple.queue";
        channel.queueDeclare(queueName, false, false, false, null);

        // 4.订阅消息
        channel.basicConsume(queueName, true, new DefaultConsumer(channel){
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope,
                                       AMQP.BasicProperties properties, byte[] body) throws IOException {
                // 5.处理消息
                String message = new String(body);
                System.out.println("接收到消息：【" + message + "】");
            }
        });
        System.out.println("等待接收消息。。。。");
    }
}
```

### Spring AMQP

Advanced Message Queue Protocol高级消息队列协议是用于在应用程序或服务之间传递业务消息的开放标准。该协议与平台或语言无关，更符合微服务中独立性的要求。

Spring AMQP是基于AMQP协议定义的一套API规范，提供了模板来发送和接收信息，包含两部分，其中spring-amqp是基础抽象，spring-rabbit是底层的默认实现。可以看到之前的基本消息队列的代码编写十分麻烦，因此我们可以借助Spring AMQP来简化操作步骤。

#### 使用Spring AMQP发送消息

要使用Spring AMQP同样要引入依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

将连接相关的配置写入配置文件：

```yaml
spring:
  rabbitmq:
    host: 192.168.1.105
    port: 5672
    virtual-host: /
    username: yourName
    password: yourPass
```

*消息发送者*

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class AMQPtest {
    @Resource
    RabbitTemplate rabbitTemplate;
    @Test
    public void BasicMQ()  {
        String queue = "simple.queue";
        String message = "hello, AMQP!";
        rabbitTemplate.convertAndSend(queue, message);
    }
}
```

可以发现利用Spring AMQP，我们可以把需要重复使用的主机名、端口号、用户名等配置写入配置文件由Spring帮助我们读取并注入到RabbitTemplate，达到复用的效果。在Java代码中，就可以免除建立连接、建立Channel等步骤，直接往指定的queue写入数据。与前文的基本消息队列代码相比，方便了不少。

#### 使用Spring AMQP发送消息

发送消息只需要同样地引入依赖、配置连接相关配置，最后编写一个Listener类及其用于接收消息的方法即可：

```java
@Component
public class AMQPListener {
    @RabbitListener(queues = "simple.queue")
    public void getSimpleQueue(String message) {
        System.out.println("接受到消息：" + message);
    }
}
```

类必须用@Component注解放入Spring容器；@RabbitListener注解可以指定多个queue接收它们的消息，被标注的方法的形参用于接收消息，每当接收到消息时，Spring AMQP就会将消息传入形参中执行该方法。**注意**，消息是**阅后即焚**的，一旦消费就会从队列中删除并且RabbitMQ**没有消息回溯功能**。

#### 工作队列WorkQueue

工作队列类似于基本消息队列，只是一个queue下同时连接了多个消费者，当发送消息很多时，多个消费者的模式可以提高消息处理速度，避免队列中消息堆积。

![](assets/SpringCloud.assets/工作队列.png)

工作队列的代码与之前Spring AMQP实现的基本消息队列代码基本一致，只需要在监听者类写多个处理方法，并且监听同一个队列即可。

**但要注意：**RabbitMQ有消息预取机制，即Channel不理会消费者的处理速度，会从queue预取多条消息存放在Channel等待着交给消费者，默认是250条，这样可能会导致性能差的消费者可能获得的消息与性能突出的消费者一样多，导致前者消息堆积，而后者很空闲。可以在配置中更改prefetch配置，来声明Channel可以预取的消息条数。

```yaml
spring:
  rabbitmq:
    listener:
      direct:
        prefetch: 1
```

当prefetch为1时，Channel在消费者处理完一个才会拿取下一条消息。

#### 发布（publish）订阅（subscribe）模型

发布订阅模型与之前的两个队列模型的区别在于允许将同一消息发送给多个消费者，通过加入了exchange来实现。

![](assets/SpringCloud.assets/发布订阅模型.png)

**注意：**exchange只能路由消息，不会保存消息，消息一旦路由失败就会丢失。

##### Fanout Exchange

Fanout Exchange会将接收到的消息路由到每一个跟其绑定的queue。

![](assets/SpringCloud.assets/fanout.png)

![](assets/SpringCloud.assets/Exchange API.png)

接下来以一个publisher，2个queue分别对应1个consumer为例：

*consumer下创建一个配置类，用于创建Exchange和queue，再进行绑定*

```java
@Configuration
public class FanoutConfig {
    // 创建exchange
    @Bean
    public FanoutExchange fanoutEx() {
        return new FanoutExchange("fanout1");
    }
	// 创建第一个queue
    @Bean
    public Queue fanoutQueue1() {
        return new Queue("fanout.queue1");
    }
    // 创建第二个queue
    @Bean
    public Queue fanoutQueue2() {
        return new Queue("fanout.queue2");
    }

    /*
    将queue分别绑定到同一个fanout exchange，注意形参的名字与前面的queue一致。
    并且返回一个Binding对象。
    */
    @Bean
    public Binding bindFQueue1(Queue fanoutQueue1, FanoutExchange fanoutEx) {
        return BindingBuilder.bind(fanoutQueue1).to(fanoutEx);
    }

    @Bean
    public Binding bindFQueue2(Queue fanoutQueue2, FanoutExchange fanoutEx) {
        return BindingBuilder.bind(fanoutQueue2).to(fanoutEx);
    }
}
```

*在consumer的监听器类配置监听函数，用@RabbitListener注解指定监听的queue*

```java
@Component
public class AMQPListener {
    @RabbitListener(queues = "fanout.queue1")
    public void getFQ1msg(String message) {
        System.out.println("FQ1接受：" + message);
    }

    @RabbitListener(queues = "fanout.queue2")
    public void getFQ2msg(String message) {
        System.err.println("FQ2接受：" + message);
    }
}
```

*publisher的消息发送方法，往exchange发送消息，注意convertAndSend的形参是exchange、routingKey和message*

```java
public void FanoutMQ()  {
    String exchange = "fanout1";
    String message = "hello, AMQP!";
    // 此处第二个参数routingKey在这没有作用，后文在direct exchange会提到它的作用
    rabbitTemplate.convertAndSend(exchange, "", message);
}
```

先启动Consumer创建交换机和消息队列，否则Publisher将找不到发送的交换机（**值得一提的是**，RabbitTemplate并不会主动创建不存在的Exchange和Queue），再启动Publisher发送一条消息，执行成功的话则会发现两个Consumer的监听器都收到了消息。

##### Direct Exchange

Direct Exchangge会将接收到的消息根据规则路由到指定的queue，因此被称为路由模式。

![](assets/SpringCloud.assets/direct exchange.png)

同样以该图为模板编写Java示例：

*Consumer的监听器类*

```java
@Component
public class AMQPListener {
    /**
    在Fanout Exchange模型中，我们是专门用一个配置类来创建queue和exchange，并为它们进行绑定，
    这是java代码方式；我们也可以利用@RabbitListener注解式地完成同样的功能。
    这里我们在@RabbitListener注解中用bindings属性指定了绑定关系，我们不仅在里面创建了queue，
    创建了类型为Direct的Exchange并绑定在了一起，并且提供了多个bindingKey。
    */
    @RabbitListener(bindings = @QueueBinding(
            value = @Queue("direct.queue1"),
            exchange = @Exchange(value = "direct1", type = ExchangeTypes.DIRECT),
            key = {"red", "blue"}
    ))
    public void getDQ1msg(String message) {
        System.out.println("DQ1接受：" + message);
    }

    @RabbitListener(bindings = @QueueBinding(
            value = @Queue("direct.queue2"),
            exchange = @Exchange(value = "direct1", type = ExchangeTypes.DIRECT),
            key = {"red", "yellow"}
    ))
    public void getDQ2msg(String message) {
        System.err.println("DQ1接受：" + message);
    }
}
```

*Publisher的消息发送方法*

```java
@Test
public void DirectMQ() {
    String exchange = "direct1";
    String message = "hello, yellow!";
    rabbitTemplate.convertAndSend(exchange, "yellow", message);
}
```

这里我们实际使用的convertAndSend方法与之前是同一个，只不过第二个参数我们填入了routingKey，因此当我们发送消息到exchange，它会根据routingKey将消息发送到拥有匹配的bindingKey的queue中。结果是该消息应该只被direct.queue2接收到。

**注意**，queue和exchange的绑定可以有**多个**bindingKey，另外使用@RabbitListener注解指定这些元素时，若它们不存在则会自动被创建。而使用rabbitTemplate发送数据到不存在的exchange或queue时，则**并不会自动创建**。

##### Topic Exchange

Topic Exchange和Direct Exchange类似，区别在于routingKey必须是多个单词的列表，以 . 分割。

![](assets/SpringCloud.assets/Topic exchange.png)

以该图为模板编写Java示例：

*Consumer的监听器类*

```java
@Component
public class AMQPListener {
    @RabbitListener(bindings = @QueueBinding(
            value = @Queue("topic.queue1"),
            exchange = @Exchange(value = "topic1", type = ExchangeTypes.TOPIC),
            key = {"china.#", "japan.#"}
    ))
    public void getTQ1msg(String message) {
        System.out.println("TQ1接受：" + message);
    }

    @RabbitListener(bindings = @QueueBinding(
            value = @Queue("topic.queue2"),
            exchange = @Exchange(value = "topic1", type = ExchangeTypes.TOPIC),
            key = {"*.news", "*.weather"}
    ))
    public void getTQ2msg(String message) {
        System.err.println("TQ2接受：" + message);
    }
}
```

Topic Exchange的监听器类与Direct Exchange十分相似，区别只在于exchange需要指定TOPIC类型，并且bindingKey可以带有通配符。

*Publisher的消息发送方法*

```java
@Test
public void DirectMQ() {
    String exchange = "topic1";
    String message = "hello, china!";
    rabbitTemplate.convertAndSend(exchange, "china.news", message);
}
```

这里的routingKey是china.news，因此这个消息可以同时被bindingKey含有china.#的topic.queue1和含有*.news的topic.queue2接收到。

### 消息转换器

在Spring AMQP的发送方法中，发送消息形参的类型是Object，也就是说我们可以发送任意类型的对象，而Spring AMQP其实会帮我们将其序列化后进行发送。

Spring对消息对象的序列化是由org.springframework.amqp.support.converter.MessageConverter类完成的，它的默认实现是SimpleMessageConverter，这个类基于JDK的ObjectOutPutStream实现序列化。但是这个序列化方式效率很低，它会将原本体积不大的对象转化成大体积的序列化串，并且转换后的序列化串就是一串字符串，没有可读性。好在Spring能够让我们自定义更好实现方式替换这个默认实现，常用的序列化就是Jackson库的Json序列化，它将对象序列化为Json串，既不会过大增加体积，也能保证序列化后的可读性，因此我们可以用Jackson提供的MessageConverter来代替默认实现。

要使用Jackson，首先也是引入依赖：

```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
</dependency>
```

在Publisher和Consumer处都加上JacksonConverter的Bean：

```java
@Bean
public MessageConverter jacksonConverter() {
    return new Jackson2JsonMessageConverter();
}
```

其他的代码并不需要做特殊处理，我们已经将Jackson的Converter交由Spring管理，则其底层的MessageConverter自然会采用Jackson的转换方案，隐式地发挥作用。尝试Publisher发送一些复杂类型的对象，其驻留在rabbitMQ期间可以在监控页面查看，发现消息内容确实是Json串。

在接收消息时，当我们在Consumer端也做了MessageConverter底层实现的更换，我们可能不一定接收的是String类型，根据发送端发送的消息类型，形参改为对应的类型，Converter即可智能地将Json格式的消息转换成形参类型的对象传递给Consumer的监听器方法。**注意**，发送端和接收端应采用同一种MessageConverter，否则接收的消息可能不会如程序员所期望的那样。

## 10 - Elastic Search

[ElasticSearch笔记](./ElasticSearch.md)

Elastic Search是一个强大的开源搜索引擎，可以从海量的数据中心快速找到需要的内容。

以Elastic Search为核心，结合Kibana、Logstash和Beats构成了Elastic Stack（ELK），广泛地应用于日志数据分析、实时监控等领域。

![](assets/SpringCloud.assets/ELK.png)

Elastic Search是基于Apache的Lucene开发的，Lucene是一个易扩展、高性能（基于倒排索引）的搜索引擎类库，尽管它只支持Java语言，并且学习路线陡峭，但是很多定制搜索引擎都需要围绕它来创建。而基于其开发的ES则有着这些优势：支持分布式，可水平扩展，提供Restful接口，可以被任何语言调用。

### 倒排索引

传统数据库例如MySQL采用正向索引。正向索引为每条数据的主键id建立索引，这种索引在根据id查找时效率很高，但是对于模糊查询则失效，模糊查询会顺序查找每一条数据，跟数据内容进行比对观察是否和模糊查询字段匹配，因此效率极低。

Elastic Search则采用倒排索引。其中每一条数据被称为文档document，文档会根据语义划分为多个词语，称为词条term。倒排索引在维护文档时，会为文档分词出多个term，为每个term建立索引，每一个term可能会对应多个文档id，文档id能够查询到对应的文档，其中term应该是唯一的。

![](assets/SpringCloud.assets/倒排索引.png)

倒排索引在模糊查询时，会根据模糊查询字段去搜索包含字段的term，找出所有符合的文档。

<img src="assets/SpringCloud.assets/倒排索引查询.png"  />

### 相关名词和概念

#### 文档document

Elastic Search是面向文档存储的，相当于数据库中的一条数据。

不同的是文档数据是被序列化成Json格式存储在ES中的。

#### 索引index

相同类型的文档的集合，类似于数据库中的表。

#### 映射Mapping

索引中文档的字段约束信息，类似于数据库中的表结构。

![](assets/SpringCloud.assets/ES概念.png)

MySQL擅长于事务操作，可以确保数据的一致性和安全性；而ES不具备事务，但是其擅长海量数据的搜索、分析和计算。在实际情况下两者往往需要配合使用，而不是只使用其中一个并试图完全代替另一个的功能。

![](assets/SpringCloud.assets/MySQL&ES.png)

### 部署单点ES

如果ES需要连接Kibana容器等中间件，需要创建一个网络来让它们互连。

```sh
docker network create es-net
```

直接运行docker pull拉取ES即可，然后执行docker run运行：

```shell
docker run -d --name es -e "ES_JAVA_OPTS=-Xms1024m -Xmx1024m" -e "discovery.type=single-node" \
-v es-data:/usr/share/elasticsearch/data \
-v es-plugins:/usr/share/elasticsearch/plugins \
--privileged --network es-net \
-p 9200:9200 -p 9300:9300 \
elasticsearch:7.12.1
```

- 其中环境变量ES_JAVA_OPTS指定了分配给ES的内存的最小和最大值；
- 环境变量discovery.type指定了ES为单点部署；
- -v参数指定数据卷挂载，es容器内的data目录存放文档，而plugins目录存放插件；
- --privileged用于提升容器的权限，使其能够执行一些特殊的系统任务，例如访问主机的内核参数；
- --network参数指定加入的网络；
- -p指定了两个端口，其中容器的9200端口用于外部访问，而9300端口用于ES集群内部访问，因此这里实际用不到。

成功运行容器后，访问9200端口应当能看到一串描述容器的Json串。

### 部署Kibana

Kibana可以提供一个ES的可视化界面，便于进行DSL语句的操作等等。

docker pull拉取kibana镜像以后，执行以下docker run运行：

```sh
docker run -d --name kibana \
-e ELASTICSEARCH_HOSTS=http://es:9200 \
--network es-net -p 5601:5601 kibana:7.12.1
```

其中环境变量指定了ES的端口位于9200，当Kibana和ES位于同一网络中时可以用服务名代替主机名。

### IK分词器

ES在建立倒排索引时需要对文档进行分词，在搜索时也会对用户的输入进行分词。默认的分词规则对于英文语句的切分很有效，但是对中文不够友好，基本上只能将中文语句划分成一个一个单字。因此我们需要根据使用的语言，更换不同的分词器来更好地切分语句。对于中文，IK分词器十分地智能且常用。

如果ES容器内没有IK分词器，可以利用docker exec命令进入容器，执行以下安装语句：

```shell
./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.12.1/elasticsearch-analysis-ik-7.12.1.zip
```

本质上是从该GitHub网址下载分词器并安装到plugins目录下，因此利用es-plugins数据卷，也可以直接在GitHub下载压缩包，解压到数据卷挂载点即可。

安装完毕后可以在Kibana的Dev tools利用以下语句测试：

```json
POST _analyze
{
  "analyzer": "ik_smart",
  "text" : "我今天买了一双鞋"
}
```

POST指定发送的是POST请求；_analyze是一个帮助分析语句如何分词的API；analyzer属性可以指定分词器，例如standard标准分词器——它可以对英语智能分词，但是对中文效果较差。

当我们完成了IK分词器的安装后，有以下两种analyzer：

- ik_smart：粗粒度的分词，它从长语句开始分词，不能分词则查看其中的短语句，当长语句是一个term时，便不再继续查看更短的语句，例如”程序员“一词在这种分词规则下只会被分为”程序员“；
- ik_max_word：细粒度的分词，这种分词规则即便将长语句分成了一个term，也会继续查看其中的短语句，例如“程序员”会被分为“程序员”、“程序”和“员”三个词。

#### 拓展词库

IK分词器底层拥有自己的字典，分词的原理就是将语句中的词语与字典进行比对。但是分词器字典往往不会包含所有我们想要得到的词，也常常会放行我们不想要的屏蔽词。因此IK分词器提供了一种功能，可以让我们在其字典库里加入词，或者停用词。

在config中找到IKAnalyzer.cfg.xml文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
<properties>
        <comment>IK Analyzer 扩展配置</comment>
        <!--用户可以在这里配置自己的扩展字典-->
        <entry key="ext_dict">ext.dic</entry>
         <!--用户可以在这里配置自己的扩展停止词字典  *** 添加停用词词典-->
        <entry key="ext_stopwords">stopword.dic</entry>
</properties>
```

**注意**，其中的ext.dic和stopword.dic是同目录下的文件名，没有则自己创建即可。在其中书写扩展词/屏蔽词后重启ES即可生效，每一行写一个扩展词/屏蔽词。

#### 索引库操作

##### mapping属性

mapping是对索引库中文档的约束，类似于MySQL中的Schema表结构。常见的mapping属性有：

- type：字段数据类型，常见的简单类型有：
  - 字符串：text（可分词的文本）、keyword（不需要分词的精确值）
  - 数值：long、integer、short、byte、float、double
  - 布尔：boolean
  - 日期：date
  - 对象：object
- index：是否创建倒排索引，默认为true
- analyzer：使用的分词器
- properties：某个字段的子字段
- ... ...

##### 创建索引库

ES中通过Restful请求操作索引库和文档，请求内容用DSL语句表示。创建索引库及其mapping的DSL语句如下：

```json
PUT /rybin // PUT创建索引库，斜杠后是索引库名
{
  "mappings": { // 创建索引库的mapping
    "properties": { // mapping内有哪些字段
      "info": {
        "type": "text",
        "analyzer": "ik_smart"
      },
      "email": {
        "type": "keyword",
        "index": false
      },
      "name": {
        "type": "object",  // 创建对象类型的字段
        "properties": { // 定义子字段
          "firstName": {
            "type": "keyword"
          },
          "lastName": {
            "type": "keyword"
          }
        }
      }
    }
  }
}
```

##### 查询索引库

```json
GET /rybin // GET后跟索引库名即可
```

##### 修改索引库

索引库能支持添加字段或其他信息，但不允许修改已有信息（因为修改已有信息会破坏已存在的倒排索引）：

```json
PUT /rybin/_mapping // 为索引库的mapping添加了一个字段
{
  "properties": {
    "age": {
      "type": "integer"
    }
  }
}
```

但如果PUT中是修改已有信息，则会返回一个错误。

##### 删除索引库

```json
DELETE /rybin // DELETE后跟索引库名即可
```

#### 文档操作

##### 添加文档

```json
POST /rybin/_doc/1 // 添加文档的格式：POST /索引库名/_doc/文档id
{
  "info": "Java程序员",
  "email": "KDB@123.com",
  "name": {
    "firstName": "Kevin",
    "lastName": "De bruyne"
  }
}
```

##### 查询文档

```json
GET /rybin/_doc/1
```

##### 删除文档

```json
DELETE /rybin/_doc/1
```

##### 修改文档

**1、全量修改**

除了请求方式，与增加文档的语句基本一致，其会先查找文档id，如果有则删除原有文档，然后进行添加。

```json
PUT /rybin/_doc/1
{
  "info": "Java程序员",
  "email": "KDB@123.com",
  "name": {
    "firstName": "Kevin",
    "lastName": "De bruyne"
  }
}
```

**2、增量修改**

查找文档id，只修改部分字段。

```json
POST /rybin/_update/1
{
  "doc": {
    "email": "Kevin@123.com"
  }
}
```

### RestClient

ES官方提供了各种不同语言的客户端，用来操作ES。这些客户端本质上就是组装DSL语句，通过http请求发送给ES。因此Java当然也拥有用于操作ES的RestClient。

以下示例将要使用的sql表：

```sql
CREATE TABLE `tb_hotel`  (
  `id` bigint(20) NOT NULL COMMENT '酒店id',
  `name` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL COMMENT '酒店名称',
  `address` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL COMMENT '酒店地址',
  `price` int(10) NOT NULL COMMENT '酒店价格',
  `score` int(2) NOT NULL COMMENT '酒店评分',
  `brand` varchar(32) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL COMMENT '酒店品牌',
  `city` varchar(32) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL COMMENT '所在城市',
  `star_name` varchar(16) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '酒店星级，1星到5星，1钻到5钻',
  `business` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '商圈',
  `latitude` varchar(32) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL COMMENT '纬度',
  `longitude` varchar(32) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL COMMENT '经度',
  `pic` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '酒店图片',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Compact;
```

对应表结构，在ES建立索引库：

```json
PUT /hotel
{
  "mappings": {
    "properties": {
      "id": {
        "type": "keyword"
      },
      "name": {
        "type": "text",
        "analyzer": "ik_max_word",
        "copy_to": "all"
      },
      "address": {
        "type": "keyword",
        "index": false
      },
      "price": {
        "type": "integer"
      },
      "score": {
        "type": "integer"
      },
      "brand": {
        "type": "keyword",
        "copy_to": "all"
      },
      "city": {
        "type": "keyword"
      },
      "starName": {
        "type": "keyword"
      },
      "business": {
        "type": "keyword",
        "copy_to": "all"
      },
      "location": {
        "type": "geo_point"
      },
      "pic": {
        "type": "keyword",
        "index": false
      },
      "all": {
        "type": "text",
        "analyzer": "ik_max_word"
      }
    }
  }
}
```

其中location字段的type是geo_point，geo_point是一个组合经度和纬度在一个字符串的专用于表达地理坐标点的类型。

在查询中可能常常需要组合多个字段进行查询，但多字段查询会使得效率降低，因此ES中可以创建一个索引专用的字段，这里取名为all，并且支持分词，在其他字段中使用"copy_to"指向这个字段，就可以使得针对这些字段的多字段查询转变为单字段查询，ES为这种查询进行了性能的优化。

#### RestClient基本使用

在POM文件引入依赖

```xml
<dependency>
    <groupId>org.elasticsearch.client</groupId>
    <artifactId>elasticsearch-rest-high-level-client</artifactId>
</dependency>
```

如果想要使用的RestClient版本与Spring Boot预设的版本不一致，可以使用以下配置覆盖预设的版本

```xml
<properties>
    <elasticsearch.version>7.12.1</elasticsearch.version>
</properties>
```

想要使用RestClient，先要创建RestHighLevelClient

```java
// HttpHost的create方法中传入的是ES的外部访问端口
client = new RestHighLevelClient(RestClient.builder(HttpHost.create("http://192.168.50.118:9200")));
```

#### 创建索引库

```java
@Test
// 该测试往ES中创建了一个名为hotel的索引库，仔细观察会发现这个流程实际上与Kibana
// 发送DSL语句的方式很相似：请求方法 + 索引名 + DSL语句
void createIndex() throws IOException {
    // 创建一个用于创建索引的CreateIndexRequest，索引名为hotel
    CreateIndexRequest request = new CreateIndexRequest("hotel");
    // MAPPING_TEMPLATE是自己编写的包含DSL语句的字符串，此处用到的DSL语句即先前说明的hotel创建语句，
    // 第二个参数指明内容为JSON格式
    request.source(MAPPING_TEMPLATE, XContentType.JSON);
    // indices方法返回索引操作集，调用create方法即可发送索引创建命令，
    // 第二个参数可以设置请求头，这里使用默认
    client.indices().create(request, RequestOptions.DEFAULT);
}
```

#### 查询索引库是否存在

```java
@Test
void indexExist() throws IOException {
    GetIndexRequest request = new GetIndexRequest("hotel");

    Boolean exist = client.indices().exists(request, RequestOptions.DEFAULT);

    System.out.println(exist ? "索引存在" : "索引不存在");
}
```

#### 删除索引库

```java
@Test
void deleteIndex() throws IOException {
    DeleteIndexRequest request = new DeleteIndexRequest("hotel");

    client.indices().delete(request, RequestOptions.DEFAULT);
}
```

#### 新增文档

```java
@Test
void insertDoc() throws IOException {
    // Hotel类是用于数据库存储的数据类型
    Hotel hotel = hotelService.getById(61083);
    // HotelDoc与Hotel的数据成员有区别，用于ES存储
    HotelDoc hotelDoc = new HotelDoc(hotel);

    IndexRequest request = new IndexRequest("hotel").id(hotelDoc.getId().toString());
	// 使用FastJson可以直接将HotelDoc转成符合索引库映射的Json格式
    request.source(JSON.toJSONString(hotelDoc), XContentType.JSON);

    client.index(request, RequestOptions.DEFAULT);
}
```

#### 查询文档

```java
@Test
void lookupDoc() throws IOException {
    GetRequest request = new GetRequest("hotel", "61083");

    GetResponse document = client.get(request, RequestOptions.DEFAULT);
	// 这里获取到的是source的JSON字符串格式
    String source = document.getSourceAsString();
	// 使用FastJson转换为HotelDoc
    HotelDoc hotel = JSON.parseObject(source, HotelDoc.class);

    System.out.println(hotel);
}
```

#### 修改文档

*增量修改*

```java
@Test
void modifyDoc() throws IOException {
    UpdateRequest request = new UpdateRequest("hotel", "61083");

    // doc方法是一个接收不定参数的方法，它的格式是一个key后接一个value，
    // 意为将该key的值改为value
    request.doc(
            "price", 100,
            "starName", "四钻"
    );

    client.update(request, RequestOptions.DEFAULT);
}
```

#### 删除文档

```java
@Test
void deleteDoc() throws IOException {
    DeleteRequest request = new DeleteRequest("hotel", "61083");

    client.delete(request, RequestOptions.DEFAULT);
}
```

#### 批量新增文档

```java
@Test
void bulkInsert() throws IOException {
    // MyBatis查询全部Hotel
    List<Hotel> hotels = hotelService.list();
	// 创建批处理Request
    BulkRequest bulkRequest = new BulkRequest();

    hotels.forEach(hotel -> {
        // 记得将数据库存储格式的Hotel转换成ES存储格式的HotelDoc
        HotelDoc hotelDoc = new HotelDoc(hotel);
        // 对于每个hotelDoc创建一个IndexRequest并放入BulkRequest，
        // 最后直接执行BulkRequest即可
        bulkRequest.add(new IndexRequest("hotel")
                .id(hotelDoc.getId().toString())
                .source(JSON.toJSONString(hotelDoc), XContentType.JSON)
        );
    });

    client.bulk(bulkRequest,  RequestOptions.DEFAULT);
}
```

### DSL查询语法

ES提供了基于JSON的DSL(Domain Specific Language)语句来定义查询。我们在前面也使用了一些DSL基本语句。

常见的查询类型：

- 查询所有文档，例如match_all
- Full text全文检索查询：利用分词器对用户输入内容分词，然后进行倒排索引匹配。例如match、multi_match
- 精确查询：根据精确词条值查询文档，一般查找keyword、数值、日期等类型字段。例如ids、range、term
- 地理查询：根据经纬度查询，例如geo_distance、geo_bounding_box
- compound复合查询：复合查询可以将上述查询组合起来合并查询条件

查询的基本语法如下：

```json
GET /indexName/_search
{
  "query": {
    "查询类型": {
      "查询条件": "条件值"
    }
  }
}
```

#### 查询所有文档

```json
# 一般查询所有文档时，如果发现返回的文档并不全，这是因为显示的文档数目有限制
GET /hotel/_search
{
  "query": {
    "match_all": {}
  }
}
```

#### 全文检索查询

##### match查询

可以对用户的输入进行分词，在倒排索引库检索。**但只支持单字段查询。**

```json
GET /hotel/_search
{
  "query": {
    "match": {
      "city": "上海"
    }
  }
}
```

##### multi_match查询

与match查询功能类似，但支持同一内容的多字段查询。

```json
GET /hotel/_search
{
  "query": {
    "multi_match": {
      "query": "外滩", 
      "fields": ["name", "business"]
    }
  }
}
```

#### 精确查询

精确查询一般用于查找不被分词的类型，它是一种不会对输入内容进行分词的查询。

##### term查询

term根据词条值精确查询文档

```json
GET /hotel/_search
{
  "query": {
    "term": {
      "city": {
        "value": "上海"
      }
    }
  }
}
```

##### range查询

range查询能够查询到词条值在给定范围内的文档，可以查询数值、日期等类型

```json
GET /hotel/_search
{
  "query": {
    "range": {
      "price": {
        # 此处gte代表大于等于，lte代表小于等于
        "gte": 100,
        "lte": 200
      }
    }
  }
}
```

#### 地理查询

根据经纬度进行查询。

##### geo_bounding_box查询

查询某个字段的经纬度落在给定矩形范围内的文档

```json
GET /hotel/_search
{
  "query": {
    "geo_bounding_box": {
      "location": {
        "top_left": {
          "lat": "31.1",
          "lon": "121.5"
        },
        "bottom_right": {
          "lat": "30.9",
          "lon": "121.7"
        }
      }
    }
  }
}
```

##### geo_distance查询

查询某个字段离给定地理坐标小于一定距离的文档

```json
GET /hotel/_search
{
  "query": {
    "geo_distance": {
      "distance": "15km",
      "location": "31.21, 121.5"
    }
  }
}
```

#### 复合查询

复合(compound)查询可以将多个简单查询组合起来实现更复杂的查询逻辑，例如function score算分函数查询，可以通过计算文档的相关性得分来为查询结果排名。

##### 相关性算分

当使用match查询时，文档结果会根据搜索词条的关联度打分，返回结果按分数降序排列。

![](assets/SpringCloud.assets/词条频率算法.png)

TF-IDF和BM25都是用来计算文档得分的算法。其中ES5前TF-IDF算法使用较多，而新版本中则更多采用BM25算法。这是因为TF-IDF算法的得分会随词频增高而无限增大，而BM25算法改进了这一缺陷，使得词频对于分数的贡献有上界，并且得分增长更加平滑。

**function_score查询**

使用function_score查询可以改变文档的相关性算分得到新的排序。

function_score的三要素：过滤条件、算分函数、加权模式

![](assets/SpringCloud.assets/function_score.png)

##### Boolean查询

布尔查询是一个或多个查询语句的组合。子查询的组合方式有：

- must：必须匹配每个子查询，参与算分，类似“与”
- should：选择性匹配子查询，参与算分，类似“或”
- must_not：必须不匹配，类似“非”
- filter：必须匹配，但不参与算分

可以理解为其用布尔运算将子查询组合在了一起，与MySQL的复杂查询SELECT ... WHERE ... AND ... OR ...有点类似。

*示例*

```json
GET /hotel/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "name": "如家" # 查询关键字“如家”
          }
        }
      ],
      "must_not": [
        {
          "range": {
            "price": {
              "gte": 700 # 必须不大于等于700，即小于700
            }
          }
        }
      ],
      "filter": [ # 必须符合这个条件，但不参与算分
        {
          "geo_distance": {
            "distance": "10km",
            "location": {
              "lat": "31.21",
              "lon": "121.5"
            }
          }
        }
      ]
    }
  }
}
```

如果将filter中的子查询放在must则会发现结果的得分改变了，这说明当子查询放在filter中时**不会参与算分**。

**注意，算分操作会消耗es性能，因此为了提高性能，应尽量把不需要参与算分的查询放在filter和must_not中。**

#### 搜索结果处理

##### 排序

ES支持对搜索结果排序，默认按照相关度算分_score排序。可以排序的字段类型有：keyword类型、数值类型、地理坐标类型、日期类型等。

*示例*

```json
GET /hotel/_search
{
  "query": { # 查询语句
    "match_all": {}
  },
  "sort": [ # 查询结果排序，二者是并列关系
    {
      "score": { # 可以简写成"score": "desc"
        "order": "desc" # 升序asc，降序desc
      }
    },
    {
      "_geo_distance": {
        "location": { # 可以简写成 "location": "31.1, 121.5"
          "lat": 31.1,
          "lon": 121.5
        },
        "order": "asc",
        "unit": "km" # 距离的显示单位
      }
    }
  ]
}
```

##### 分页

ES默认只返回排序后前10条文档，如果需要查询更多文档则需要使用分页。ES通过修改from、size参数指定从第几条开始，返回几条文档。

*示例*

```json
GET /hotel/_search
{
  "query": {
    "match_all": {}
  },
  "from": 20, # 从第20条文档开始返回
  "size": 10, # 返回10条文档
  "sort": [
    {
      "price": { # 按价格降序排序后再返回
        "order": "desc"
      }
    }
  ]
}
```

然而，因为ES支持分布式，当查询排序结果时需要从多个服务器获取文档合并再排序，才能获取到真实的排序。当获取文档数较大时对于性能的损耗也是巨大的，ES因此规定了查询结果总数的上限，即from+size不能超过10000，由此产生了**深度分页问题**。

针对深度分页问题，ES提供了2种解决方案：

- search after：根据前一页文档的排序值来查询下一页文档，因此只能向后查询，不支持随机翻页，这种方案可以应用于例如手机向下滚动翻页。
- scroll：将排序结果形成快照存储在内存中，这种方案内存消耗大并且结果非实时，官方已不推荐使用。

##### 高亮

在搜索结果中把搜索关键字突出显示。其实是搜索结果中的关键字用<em>标签修饰，并且在CSS中指定<em>的样式，就可以使得在前端显示的搜索结果的关键字呈现高亮。而ES可以帮助我们在返回搜索结果时就用标签将搜索关键字修饰。

*示例*

```json
GET /hotel/_search
{
  "query": {
    "match": {
      "all": "如家"
    }
  },
  "highlight": {
    "fields": {
      "name": {
        "require_field_match": "false",
        "pre_tags": "<em>",
        "post_tags": "</em>"
      }
    }
  }
}
```

ES默认情况下，高亮字段必须与搜索字段一致，否则不能添加标签，改变require_field_match参数为false就可以忽略不一致。而pre_tag和post_tag参数分别指定在关键字的前后增加什么标签，不写则默认被<em></em>修饰。

**注意，包含高亮的查询的返回结果并不直接就是带有标签的值，而是source字段中存储原始文档，highlight字段中才存储了关键字被标签包围的文本值。**

### RestCilent查询文档

*RestCilent进行match_all查询*

```java
@Test
void searchMatchAll() throws IOException {
    // 创建SearchRequest对象，并指定索引名
    SearchRequest request = new SearchRequest("hotel");
    // source返回的是SearchSourceBuilder，用于完成DSL语句的功能，可以进行链式编程，
    // 例如指定查询条件、分页、排序等
    request.source().query(QueryBuilders.matchAllQuery());
    // search方法发送搜索请求，并返回一个SearchResponse，
    // result拥有搜索相关的信息，比如命中结果、总条数、分页信息等
    SearchResponse result = client.search(request, RequestOptions.DEFAULT);
    // getHits返回Hits字段，里面又包含_index、total、搜索结果等信息
    SearchHits searchHits = result.getHits();
    // Total是查询结果的数目
    long hitsNum = searchHits.getTotalHits().value;
    System.out.printf("搜索到%d条数据\n", hitsNum);
    // SearchHit数组存放的是真正的文档结果
    SearchHit[] hitsArr = searchHits.getHits();
    for (SearchHit hit : hitsArr) {
        String jsonSource = hit.getSourceAsString();

        HotelDoc hotelDoc = JSON.parseObject(jsonSource, HotelDoc.class);

        System.out.println(hotelDoc);
    }
}
```

其对应ES的DSL语句为：

```json
GET /hotel/_search
{
  "query": {
    "match_all": {}
  }
}
```

#### 全文检索查询

编写查询的基本步骤与之前的示例类似，而只需要更改根据DSL语句创建不同的QueryBuilder并赋给request.source()即可。

*示例*

```java
@Test
void searchMatchAll() throws IOException {
    SearchRequest request = new SearchRequest("hotel");
    // 只修改了这部分，利用match查询搜索city字段包括上海的文档
    request.source().query(QueryBuilders.matchQuery("city", "上海"));

    SearchResponse result = client.search(request, RequestOptions.DEFAULT);

    SearchHits searchHits = result.getHits();

    long hitsNum = searchHits.getTotalHits().value;
    System.out.printf("搜索到%d条数据\n", hitsNum);
    
    SearchHit[] hitsArr = searchHits.getHits();
    for (SearchHit hit : hitsArr) {
        String jsonSource = hit.getSourceAsString();

        HotelDoc hotelDoc = JSON.parseObject(jsonSource, HotelDoc.class);

        System.out.println(hotelDoc);
    }
}
```

#### 精确查询

*示例*

```java
// 以下只展示变动的代码
// 利用term查询搜索city字段的精确值为上海的文档
request.source().query(QueryBuilders.termQuery("city", "上海"));
```

#### 复合查询

*示例*

```java
/*
之所以能链式编程，是因为这些能够指定DSL语句的函数都返回QueryBuilder类型的对象
*/
request.source().query(QueryBuilders.boolQuery() // 创建Boolean查询
        .must(QueryBuilders.termQuery("city", "上海")) // must字段要求city精确匹配“上海”
        .filter(QueryBuilders.rangeQuery("price") // filter字段使用range查询
                .gte(200) // 要求price >= 200
                .lte(1000))); // 要求price <= 1000
```

其对应的ES的DSL语句如下：

```json
GET /hotel/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "term": {
            "city": "上海"
          }
        }
      ],
      "filter": [
        {
          "range": {
            "price": {
              "gte": 200,
              "lte": 1000
            }
          }
        }
      ]
    }
  }
}
```

#### 搜索结果处理

##### 排序和分页

*示例*

```java
@Test
void sortAndHighlight() throws IOException {
    SearchRequest request = new SearchRequest("hotel");
	// 指定查询方式，这里是match_all
    request.source().query(QueryBuilders.matchAllQuery());
	// 指定排序方式
    request.source().sort("price", SortOrder.ASC);
	// 指定分页
    int page = 3, num = 5;
    request.source().from((page - 1) * num).size(num);
	
    // 以下结果处理代码与前文一致
    SearchResponse result = client.search(request, RequestOptions.DEFAULT);

    SearchHits searchHits = result.getHits();

    long hitsNum = searchHits.getTotalHits().value;
    System.out.printf("搜索到%d条数据\n", hitsNum);

    SearchHit[] hitsArr = searchHits.getHits();
    for (SearchHit hit : hitsArr) {
        String jsonSource = hit.getSourceAsString();

        HotelDoc hotelDoc = JSON.parseObject(jsonSource, HotelDoc.class);

        System.out.println(hotelDoc);
    }
}
```

这里query、sort、from、size函数均返回SearchSourceBuilder，尽管示例中将它们分别调用，不过也可以写成链式调用的形式。

##### 高亮

在ES中，包含高亮的查询的结果将原始文档和被标签修饰的字段值分别存储，因此想要获得被标签修饰的文档不仅需要在RestClient中构建DSL语句，还需要对返回的结果进行解析处理。

```java
@Test
void sortAndHighlight() throws IOException {
    SearchRequest request = new SearchRequest("hotel");
	// match查询all字段包含“如家”的文档
    request.source().query(QueryBuilders.matchQuery("all", "如家"));
    // 排序
    request.source().sort("price", SortOrder.ASC);
    // 分页
    int page = 3, num = 5;
    request.source().from((page - 1) * num).size(num);
    // 高亮name字段，并且置require_filed_match参数为false
    request.source().highlighter(new HighlightBuilder()
            .field("name")
            .requireFieldMatch(false));
	
    SearchResponse result = client.search(request, RequestOptions.DEFAULT);

    SearchHits searchHits = result.getHits();

    long hitsNum = searchHits.getTotalHits().value;
    System.out.printf("搜索到%d条数据\n", hitsNum);

    SearchHit[] hitsArr = searchHits.getHits();
    for (SearchHit hit : hitsArr) {
        String jsonSource = hit.getSourceAsString();

        HotelDoc hotelDoc = JSON.parseObject(jsonSource, HotelDoc.class);
		// 结果解析只有该处有变化
        // 先从hit中获取highlight字段，这是一个Map对象，key为指定高亮的字段名
        Map<String, HighlightField> highlightFields = hit.getHighlightFields();
		// 从Map中获取到的HighlightFiled可以调用getFragments获取高亮文本的数组，
        // 数组中每个对象都是Text类型，调用string函数来将其转换为String类型。
        String highlightedField = highlightFields.get("name").getFragments()[0].string();
		// 最后用高亮处理过的字段来替换掉hotelDoc中的原始字段即可
        hotelDoc.setName(highlightedField);

        System.out.println(hotelDoc);
    }
}
```

### 聚合

聚合aggregation是对文档的统计、分析和运算。类似于MySQL中聚合函数的概念。

- bucket桶聚合：用于对文档进行分组
  - Term Aggregation：按照文档字段值分组，不能对可分词字段分组
  - Date Aggregation：按照日期阶梯分组，例如同一周的文档为一组
- metrics度量聚合：对字段值进行计算，例如计算平均值、最大值、最小值等
  - Avg求平均值
  - Max求最大值
  - Min求最小值
  - Stats同时求Max、Min、Sum等
- pipeline管道聚合：将其他聚合的结果为基础做聚合

被聚合的字段可以是keyword、数值、日期、布尔等类型，不能是可分词字段。

#### Bucket聚合

**聚合的三要素是聚合名称、聚合类型和聚合字段。**

*按照brand字段聚合*

```json
GET /hotel/_search
{
  "size": 0,
  "aggs": { # 指定一种或多种聚合
    "brandArg": { # 聚合的名字
      "terms": { # 聚合类型，这里根据字段进行聚合
        "field": "brand", # 根据brand字段进行聚合
        "size": 10, # 显示聚合完成后的前10条数据
        "order": {
          "_count": "asc"
        }
      }
    }
  }
}
```

外层的size指定查询结果的返回数量，而聚合内的size是指定聚合操作后的返回数据的数量。默认情况下bucket聚合会计算一个桶内的 _count并将结果按 _count排序，可以通过order参数指定排序方式。

如果想要限定想要聚合的文档范围，利用query查询即可指定范围。

*示例*

```json
GET /hotel/_search
{
  "query": {
    "range": {
      "price": {
        "gte": 0,
        "lte": 300
      }
    }
  }, 
  "size": 0,
  "aggs": {
    "brandAgg": {
      "terms": {
        "field": "brand",
        "size": 10,
        "order": {
          "_count": "asc"
        }
      }
    }
  }
}
```

这里先使用query进行查询，再将结果进行聚合。效果上就是限定了聚合的范围。

#### Metrics聚合

*先进行桶聚合，再对每个桶进行度量聚合*

```json
GET /hotel/_search
{
  "size": 0,
  "aggs": {
    "brandAgg": {
      "terms": {
        "field": "brand",
        "size": 10,
        "order": {
          "scoreAgg.avg": "desc" # 根据每个桶的分数平均值降序排序，可以发现这里使用.来描述子字段
        }
      },
      "aggs": {
        "scoreAgg": {
          "stats": {
            "field": "score" # 对score进行stats聚合
          }
        }
      }
    }
  }
}
```

这里是将metrics聚合的aggs嵌套在了bucket聚合中，目的是对于每个桶进行聚合计算。当然也可以将aggs写在最外层，只是这样做将聚合计算所有文档，具体如何编写依业务场景与需求而定。

### RestClient实现聚合

#### Bucket聚合

```java
@Test
void bucketAggregation() throws IOException {
    SearchRequest request = new SearchRequest("hotel");

    // 调用aggregation函数构建DSL语句
    request.source().aggregation(AggregationBuilders // 创建TermsAggregationBuilder并且可以链式调用
                    .terms("brandAgg") // term函数表明聚合类型，参数指定聚合名称
                    .field("brand")	// 指定聚合字段
                    .size(10)); // 指定显示的条数

    SearchResponse response = client.search(request, RequestOptions.DEFAULT);
	// 调用response的getAggregations函数返回aggregations，再调用get函数获取
    // 想要的聚合名称对应的结果，此处需要用Terms来接收，否则无法调用getBuckets函数
    Terms scoreAgg = response.getAggregations().get("brandAgg");
	// getBuckets函数获取bucket的数组
    List<? extends Terms.Bucket> buckets = scoreAgg.getBuckets();

    for (Terms.Bucket bucket : buckets) {
        // key是品牌名，docCount是该桶结果的数量
        String key = bucket.getKeyAsString();
        long docCount = bucket.getDocCount();
        System.out.printf("%s: %d\n", key, docCount);
    }
}
```

### 自动补全

#### 拼音分词器

拼音分词器可以将中文语句分词成拼音，借助拼音分词器能够实现对于拼音的自动补全。analyzer-pinyin就是一个拼音分词器，它与ik分词器由同一个作者编写，可以在GitHub上找到。

在ES容器中可以按照与安装IK分词器同样的流程安装pinyin分词器：

*先用exec命令进入容器*

```bash
docker exec -it es bash
```

这里的es是先前创建es时自定义的容器名。

*在容器内使用如下命令安装pinyin分词器*

```bash
./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-pinyin/releases/download/v7.12.1/elasticsearch-analysis-pinyin-7.12.1.zip 
```

**版本应该和es匹配，否则可能出现错误。**

最后用docker restart命令重启es容器即可。

等待es服务重启完成后，利用kibana的开发工具测试如下DSL语句：

```json
GET /_analyze
{
  "text": ["足球比赛很精彩"],
  "analyzer": "pinyin"
}
```

如果返回的分词是拼音，则说明pinyin分词器安装成功。

#### 自定义分词器

ES中的分词器(analyzer)分成三部分：

- character filters：在tokenizer前对文本进行处理。例如删除字符、替换字符。
- tokenizer：将文本按照一定规则切割成词条(term)。例如ik_smart、keyword。
- tokenizer filters：将tokenizer输出的词条进一步处理。例如大小写转换、同义词转换、拼音转换等。

![](assets/SpringCloud.assets/分词器三部分.png)

在创建索引库时，可以通过settings属性配置自定义analyzer：

```json
PUT /test
{
  "settings": {
    "analysis": { // 自定义analyzer、filter
      "analyzer": { // 自定义分词器
        "my_analyzer": { // 分词器名称
          "tokenizer": "ik_smart",
          "filter": "py"
        }
      },
      "filter": {
        "py": { // 自定义tokenizer filter
          "type": "pinyin", // 指定过滤器类型
          "keep_full_pinyin": false, // 这个属性会把一个词的每个字拆开翻译成拼音
          "keep_joined_full_pinyin": true, // 这个属性会把一个词整体翻译成拼音
          "keep_original": true, // 会保留原始输入
          "remove_duplicated_term": true, // 词条去重
          "non_chinese_pinyin_tokenize": true // 可以把连在一起的拼音分成词条，例如liudehua->liu de hua
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "my_analyzer", // 插入文档时使用自定义分词器
        "search_analyzer": "ik_smart" // 搜索时使用该分词器
      }
    }
  }
}
```

之所以搜索时不使用自定义分词器（内含拼音分词器）是因为这样做的话用户的中文输入会搜索到可能不相关的同音词，例如搜索“狮子”可能也会搜索到“柿子”。而ik分词器并不会将中文输入分词成拼音的词条去查询倒排索引，也就自然不会搜索到同音词，而且ik分词器对于拼音输入也有效。

#### Completion Suggester自动补全查询

ES提供了Completion Suggester来帮助完成自动查询功能。该查询会匹配以用户输入为开头的词条并返回。为了提高补全查询的效率，需要补全查询的字段必须是completion类型。

*创建一个简单的索引库，只有一个类型为completion的字段title*

```json
PUT /test2
{
  "mappings": {
    "properties": {
      "title": {
        "type":"completion"
      }
    }
  }
}
```

*插入文档*

```json
POST /test2/_doc
{
  "title": ["Sony", "WH-1000BX3"]
}
```

这里用多词条的数组的形式存储，是为了让自动补全能够更灵活的匹配，对于这个文档，用"S"或"W"都能匹配到该文档，而如果不用数组改为"SonyWH-1000BX3"，则只能用"S"匹配到该文档。

*查询语句*

```json
GET /test2/_search
{
  "suggest": { // 定义查询
    "titleSuggest": { // 自定义查询的名称
      "text": "s", // 要自动补全的文本
      "completion": { // 指定查询类型
        "field": "title",
        "size": 10
      }
    }
  }
}
```

### RestClient实现自动补全

要使用自动补全，索引库需要将相关字段设置成suggestion属性，并且配置合适的分词器，因此我们使用如下DSL语句创建一个用于测试自动补全的索引库。

```json
PUT /hotel
{
  "settings": {
    "analysis": {
      "analyzer": {
        "text_anlyzer": {
          "tokenizer": "ik_max_word",
          "filter": "py"
        },
        "completion_analyzer": {
          "tokenizer": "keyword", // 该分词器不进行分词
          "filter": "py" // 该分词器允许转拼音
        }
      },
      "filter": {
        "py": {
          "type": "pinyin",
          "keep_full_pinyin": false,
          "keep_joined_full_pinyin": true,
          "keep_original": true,
          "limit_first_letter_length": 16,
          "remove_duplicated_term": true,
          "none_chinese_pinyin_tokenize": false
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "id":{
        "type": "keyword"
      },
      "name":{
        "type": "text",
        "analyzer": "text_anlyzer",
        "search_analyzer": "ik_smart",
        "copy_to": "all"
      },
      "address":{
        "type": "keyword",
        "index": false
      },
      "price":{
        "type": "integer"
      },
      "score":{
        "type": "integer"
      },
      "brand":{
        "type": "keyword",
        "copy_to": "all"
      },
      "city":{
        "type": "keyword"
      },
      "starName":{
        "type": "keyword"
      },
      "business":{
        "type": "keyword",
        "copy_to": "all"
      },
      "location":{
        "type": "geo_point"
      },
      "pic":{
        "type": "keyword",
        "index": false
      },
      "all":{
        "type": "text",
        "analyzer": "text_anlyzer",
        "search_analyzer": "ik_smart"
      },
      "suggestion":{
          "type": "completion",
          "analyzer": "completion_analyzer" // 用于实现补全功能的字段，不需要对其分词
      }
    }
  }
}
```

想要使用RestClient实现的DSL语句：

```json
GET /hotel/_search
{
  "suggest": {
    "sgg": {
      "text": "h",
      "completion": {
        "field": "suggestion",
        "skipDuplicates": true,
        "size": 10
      }
    }
  }
}
```

该DSL语句返回的查询结果：

```json
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 0,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  },
  "suggest" : {
    "sgg" : [
      {
        "text" : "上",
        "offset" : 0,
        "length" : 1,
        "options" : [
          {
            "text" : "三里屯",
            "_index" : "hotel",
            "_type" : "_doc",
            "_id" : "396189",
            "_score" : 1.0,
            "_source" : {
              "address" : "三丰北里3号",
              "brand" : "皇冠假日",
              "business" : "三里屯/工体/东直门地区",
              "city" : "北京",
              "id" : 396189,
              "location" : "39.92129, 116.43847",
              "name" : "北京朝阳悠唐皇冠假日酒店",
              "pic" : "https://m.tuniucdn.com/fb3/s1/2n9c/tT6ipLain1ZovR5gnQ7tJ4KKym5_w200_h200_c1_t0.jpg",
              "price" : 944,
              "score" : 46,
              "starName" : "五钻",
              "suggestion" : [
                "皇冠假日",
                "三里屯",
                "工体",
                "东直门地区"
              ]
            }
          }
          // 省略其他结果，足够展示返回结果的文本结构
        ]
      }
    ]
  }
}
```

*RestClient测试代码*

```java
@Test
void testSuggest() throws IOException {
    SearchRequest request = new SearchRequest("hotel");
	// 对应DSL语句的"suggest"
    request.source().suggest(new SuggestBuilder()
            //对应DSL语句中：声明suggestion名称，以及声明补全字段为suggestion
            .addSuggestion("sgg", SuggestBuilders.completionSuggestion("suggestion")
                    .prefix("h") // 对应指定text
                    .skipDuplicates(true)
                    .size(10)));

    SearchResponse response = client.search(request, RequestOptions.DEFAULT);

    // 获取了名为sgg的对象，对应返回结果中suggest下的sgg
    CompletionSuggestion sgg = response.getSuggest().getSuggestion("sgg");
	// 获取sgg下的options，options包含了所有结果文档
    List<CompletionSuggestion.Entry.Option> options = sgg.getOptions();

    options.forEach(option -> {
        // 对于每个option，获取text值，即自动补全功能匹配到的值
        String text = option.getText().toString();
        System.out.println(text);
    });
}
```

**总之，使用RestClient编写Java代码查询ES，一个很好的方法就是对照DSL语句进行编写，RestClient所给的API与DSL语句的诸多关键字基本匹配。**

### 数据同步

一般的项目中，ES中的数据来源于MySQL，因此MySQL数据变更时ES必须跟着进行变化，这就是ES和数据库之间的**数据同步**。

在单体架构中，可以在插入数据的过程中先写入MySQL再写入ES，即可实现数据同步。但是微服务的情景下，不同服务间相互不可访问数据库，因此需要使用其他方法完成数据同步。

#### 数据同步问题的解决方案

![](assets/SpringCloud.assets/1-17017656391662-17017806184132.png)

同步调用实现简单，但是耦合度高，若一个环节出错则会影响到别的环节。

![](assets/SpringCloud.assets/1-17017656058041-17017806184143.png)

使用MQ中间件来异步通知ES服务完成数据同步，能够解除一定耦合，提高服务执行速度也能避免连环故障；但是需要依赖MQ的可靠性，MQ故障则会导致数据的不一致等问题。

![](assets/SpringCloud.assets/1-17017657416333-17017806184144.png)

使用MySQL自带的binlog当MySQL写数据时binlog新增日志，只需要监听binlog即可在数据新增时自动为ES同步数据。这种方案完全解耦，但是binlog的开启会为MySQL服务器带来压力。

### ES集群

#### ES集群结构

单机的ES做数据存储会遇到两个问题：海量数据存储问题和单点故障问题。通过多台服务器构成ES集群能够解决这两个问题。

