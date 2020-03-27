# 前言

## 微服务模块基本公式

1.  建 Module
2.  改 pom
3.  写 yml
4.  主启动
5.  业务类

## 学习技术的4个维度

1. 是什么
2. 能干什么
3. 去哪下
4. 怎么玩

## 免责声明

​		以下内容均为本人学习[尚硅谷的spring cloud视频](https://www.bilibili.com/video/BV18E411x7eT)所学内容，整理了自己觉得重要的部分。仅供本人学习使用，侵删,联系方式1641074950@qq.com



# 一、Eureka（停更）

## 1、Eureka工作示意图



![Eureka工作示意图](E:\工作外的软件\image\Eureka示意图.png)

*Eureka Server功能 :*

* **服务注册**： 将服务信息注册进注册中心
* **服务发现**：从注册中心获取服务信息
* **实质**： 存key  服务名 ， 取value 调用地址



## 2、微服务RPC[^1]远程服务调用的核心

​		***高可用***  ：如果注册中心只有一个，他出故障后会导致整个微服务环境不可用。所以需要搭建Eureka注册中心集群，实现负载均衡加故障容错。

[^1]:全称Remote Procedure Call，:远程过程调用,它是一种通过网络从远程计算机程序上请求服务,而不需要了解底层网络技术的思想。



## 3、Eureka集群的注册原理

​		互相注册，相互守望

![注册原理示意图](E:\工作外的软件\image\Eureka注册原理.png)

## 4、 Eureka工作流程（以项目demo举例）

1. 先启动Eureka注册中心
2. 启动服务提供者payment支付服务
3. 支付服务启动后会把自身信息（比如服务地址）以别名的方式t注册进Eureka
4. 消费者order服务在需要调用接口时，使用服务别名取注册中心获取实际的RPC远程调用地址
5. 消费者获得调用地址后，底层实际利用HttpClient技术实现远程调用
6. 消费者获得服务地址后，会缓存在本地jvm内存中，默认每间隔30秒更新一次服务调用地址



## 5、Eureka实操关键内容

### 	Ⅰ Eureka注册中心

#### 1）pom

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>		<!-- 可视化管理需要actuator ,一般web+actuator组合,但也有例外，比如含有webflux的依赖就不能导入web,以下都一样 就不重复写这个两个依赖了-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
```

####  			2）yml

```yaml
server:
  port: 7001

eureka:
  instance:
    hostname: eureka7001.com  #需要在C:\Windows\System32\driver\etc\hosts文件中做映射 127.0.0.1 eureka7001.com
    
  client:
    fetch-registry: false #false 表示自己端就是注册中心，职责就是维护服务实例，并不需要去检索服务
    register-with-eureka: false # false表示不向注册中心注册自己
    service-url:
      defaultZone: http://eureka7002.com:7002/eureka/     #设置与Eureka server交互的地址查询服务和注册服务都需要依赖这个地址(集群，若想再加入一个eureka，用逗号连接)

#  #配置eureka是否进入绝情模式(接受不到服务心跳时，立即删除)
#  server:
#    enable-self-preservation: false #禁用自我保护机制（自我保护:微服务掉线或网络堵塞时，也不会删除该服务信息（好死不如赖活着））
#    eviction-interval-timer-in-ms: 2000
```

#### 			3）主启动

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaMain7001 {
    public static void main(String[] args) {
        SpringApplication.run(EurekaMain7001.class, args);
    }
}
```

-----

### 	Ⅱ Provider提供服务方

#### 			1）pom

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
```

#### 			2）yml

```yaml
server:
  port: 8001
spring:
  application:
    name: cloud-payment-service
eureka:
  client:
    #表示是否将自己注册进EurekaServer默认为true
    register-with-eureka: true
    #是否从EurekaServer抓取已有的注册信息，默认为true。单节点无所谓，集群必须设置为true才能配合ribbon使用负载均衡
    fetch-registry: true
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka/,http://eureka7002.com:7002/eureka/ #集群版
      #defaultZone: http://localhost:7001/eureka #单机版（单个eureka）
  instance:
    instance-id: payment8001 #服务名称
    prefer-ip-address: true #在eureka界面，鼠标悬停在该服务的status时，页面左下角显示ip
#    #eureka客户端向服务端发送心跳的时间间隔，单位为秒（默认30秒）
#    lease-renewal-interval-in-seconds: 1
#    #eureka服务端在收到最后一次心跳后等待时间上限，单位为秒（默认是90秒），超时将剔除服务
#    lease-expiration-duration-in-seconds: 2
```

#### 			3）主启动

```java
@SpringBootApplication
@EnableEurekaClient //@EnableEurekaClient 为Eureka注册中心
@EnableDiscoveryClient //@EnableDiscoveryClient 可以是其他注册中心
public class PaymentMain8001 {
    public static void main(String[] args) {
        SpringApplication.run(PaymentMain8001.class, args);
    }
}
```

#### 			4）业务类

```java
@RestController
@Slf4j
public class PaymentController {
    @Resource
    private PaymentService paymentService;

    @Value("${server.port}")
    private String servicePort;

    @PostMapping(value = "/payment/create")
    public CommonResult create(@RequestBody Payment payment) {
        int result = paymentService.create(payment);
        if (result > 0) {
            return new CommonResult(200, "插入数据库成功,端口号为：" + servicePort, result);
        } else {
            return new CommonResult(444, "插入数据库失败", null);
        }
    }

    @GetMapping(value = "/payment/get/{id}")
    public CommonResult getPaymentById(@PathVariable("id") Long id) {
        Payment payment = paymentService.getPaymentById(id);
        if (payment != null) {
            return new CommonResult(200, "查询成功,端口号为：" + servicePort, payment);
        } else {
            return new CommonResult(444, "没有对应记录，查询ID：" + id, null);
        }
    }
}
```

------

### Ⅲ Consumer客户端

#### 1）pom

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>		
```

#### 2）yml

```yaml
server:
  port: 80
eureka:
  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka/,http://eureka7002.com:7002/eureka/ 
```

#### 3）主启动

```java
@SpringBootApplication
@EnableEurekaClient
public class OrderMain80 {
    public static void main(String[] args) {
        SpringApplication.run(OrderMain80.class, args);
    }
}
```

#### 4）业务类

* controller

  ```java
  @RestController
  @Slf4j
  public class OrderController {
      /**
       * 需要请求的服务的服务名
       */
      public static final String PAYMENT_URL = "http://CLOUD-PAYMENT-SERVICE";
  
      /**
       * 基于Rest规范提供Http请求的工具，须在configure里配置
       */
      @Resource
      private RestTemplate restTemplate;
  
      @GetMapping("consumer/payment/create")
      public CommonResult<Payment> create(Payment payment){
          return restTemplate.postForObject(PAYMENT_URL+"/payment/create",payment,CommonResult.class);
      }
  
      @GetMapping("consumer/payment/get/{id}")
      public CommonResult<Payment> getPaymentById(@PathVariable("id")Long id){
          return restTemplate.getForObject(PAYMENT_URL+"/payment/get/"+id,CommonResult.class);
      }
  
      /**
       * 需要更多详细的信息时使用getForEntity
       *
       * @param id
       * @return
       */
      @GetMapping("/consumer/payment/getForEntity/{id}")
      public CommonResult<Payment> getPayment2(@PathVariable("id")Long id){
          ResponseEntity<CommonResult> entity = restTemplate.getForEntity(PAYMENT_URL+"/payment/get/"+id,CommonResult.class);
          if(entity.getStatusCode().is2xxSuccessful()){
              return entity.getBody();
          }else{
              return new CommonResult<>(444,"操作失败");
          }
      }
  }
  ```

* configure

  ```java
  @Configuration
  public class ApplicationContextConfig {
  
      @Bean
      @LoadBalanced
      public RestTemplate getRestTemplate(){
          //@LoadBalanced 负载均衡机制(默认轮询的负载机制)
          return new RestTemplate();
      }
  }
  ```

  

## 6、Eureka停更后的几个替换方案

### Ⅰ Zookeeper

#### 1）provider提供服务方

##### ① pom

```xml
<!--        自带了一个3.5.3版本的，但安装的是3.4.9，所以此处需要排除自带的zookeeper-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-zookeeper-discovery</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>org.apache.zookeeper</groupId>
                    <artifactId>zookeeper</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
<!--        此处添加3.4.9版本-->
        <dependency>
            <groupId>org.apache.zookeeper</groupId>
            <artifactId>zookeeper</artifactId>
            <version>3.4.9</version>
        </dependency>
    </dependencies>
```

##### ② yml

```yaml
server:
  port: 8004
spring:
  application:
    name: cloud-provider-payment
  cloud:
    zookeeper:
      connect-string: localhost:2181 # zookeeper所在地址 ip:端口(默认2181)
```

##### ③ 主启动

```java
@SpringBootApplication
@EnableDiscoveryClient
public class PaymentMain8004 {
    public static void main(String[] args) {
        SpringApplication.run(PaymentMain8004.class, args);
    }
}
```

##### ④ 业务类

```java
@RestController
@Slf4j
public class PaymentController {
    @Value("${server.port}")
    private String serverPort;

    @GetMapping(value = "/payment/zk")
    public String paymentZk(){
        return "SpringCloud with Zookeeper:" +serverPort+"\t"+ UUID.randomUUID().toString();
    }
}
```

#### 2）consumer客户端

​	业务类与eureka类似，pom、yml与、主启动与zookeeper的服务端一致，需要修改的为yml中的端口号与容器名

#### 3）附

* 在docker中安装的zookeeper，要启动zookeeper客户端， 用如下命令 ***docker exec -it 容器id zkCli.sh***
* zookeeper注册中心时单独运行的,暂不用配置（集群还没有配过）

------

### Ⅱ Consul

### 1）provider提供服务方

#### ①pom

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-consul-discovery</artifactId>
        </dependency>
```

#### ②yml

```yaml
server:
  port: 8006
spring:
  application:
    name: consul-provider-payment
  cloud:
    consul:
      host: localhost
      port: 8500 #默认8500端口
      discovery:
        service-name: ${spring.application.name}
```

#### ③主启动

```java
@SpringBootApplication
@EnableDiscoveryClient
public class PaymentMain8006 {
    public static void main(String[] args) {
        SpringApplication.run(PaymentMain8006.class,args);
    }
}
```

#### ④业务类

```java
@RestController
@Slf4j
public class PaymentController {
    @Value("${server.port}")
    private String serverPort;

    @GetMapping(value = "/payment/consul")
    public String paymentConsul(){
        return "springcloud with consul:"+serverPort+"\t"+ UUID.randomUUID().toString();
    }
}
```

### 2）Comsumer客户端

​	业务类与eureka类似，pom、yml与、主启动与consul的服务端一致，需要修改的为yml中的端口号与容器名

### 3）附

* consul安装在windows操作系统中，在官网下载后需要配置环境变量。下载的文件解压后只有一个文件，在此目录下打开cmd，输入consul agent -dev运行
* 通过<http://localhost:8500>访问客户端

### Ⅲ Nacos

----



# 二、Ribbon（维护模式） 负载均衡服务调用

## 1、概念

​		基于Netfix Ribbon实现的一套 ***客户端*** 负载均衡的工具。主要功能是提供客户端的软件负载均衡算法和服务调用。

​		主要是RestTemplate[^2]+负载均衡

[^2]:① getForObject: 返回对象为响应体中数据转化成的对象，基本上可理解为json ②getForEntity: 返回对象为ResponseEntity对象，包含了响应中的一些重要信息，比如响应头，响应状态码，响应体等

## 2、Ribbon 七大实现

*  com.netflix.loadbalance.***RoundRobinRule***   轮询
*  com.netflix.loadbalance.***RandomRule***  随机
*  com.netflix.loadbalance.***RetryRule***  先按随机，若获取失败则在指定时间内重试
* ***WeightedResponseTimeRule***  对轮询的扩展，响应速度越快的实例选择权重越大，越容易选择
* ***BestAvailableRule*** 会先过滤掉由于多次访问故障而处于断路器跳闸状态的服务，然后选择一个并发量最小的服务
* ***AraillabilityFilteringRule***  先过滤掉故障实例，再选择并发量小的实例
* ***ZoneAvoidanceRule*** 默认规则，符合判断server所在区域的性能和server的可用性选择服务器

## 3、Ribbon实操关键内容

### Ⅰ provider服务提供方

#### 1）pom

```xml
<!-- 含有ribbon的相关依赖 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

#### 2）yml

​		不需要特别加入配置，默认eureka的配置即可

#### 3）主启动

```java
@SpringBootApplication
@EnableEurekaClient
@EnableDiscoveryClient
public class PaymentMain8001 {
    public static void main(String[] args) {
        SpringApplication.run(PaymentMain8001.class, args);
    }
}
```

#### 4）业务类

```java
/**
 * 验证负载均衡
 */
@GetMapping(value = "/payment/lb")
public String getPaymentLb(){
    return servicePort;
}
```

-----

### Ⅱ consumer客户端

#### 1）pom

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

#### 2）yml

​		不需要特别加入配置，默认eureka的配置即可

#### 3）主启动

```java
@SpringBootApplication
@EnableEurekaClient
//此处为替换默认的负载均衡策略
//@RibbonClient(name = "CLOUD-PAYMENT-SERVICE",configuration = MyselfRule.class)
public class OrderMain80 {
    public static void main(String[] args) {
        SpringApplication.run(OrderMain80.class, args);
    }
}
```

#### 4）业务类

* 替换自带的负载均衡策略，***一定注意，想用其它自带的负载均衡策略时，不能将MyselfRule这个文件所在的包放在@SpringBootApplication或@ComponentScan的包或下面的包，必须跳出取。并且注释掉config里的@loadBalance***

  ```java
  @Configuration
  public class MyselfRule {
      @Bean
      public IRule myRule() {
          return new RandomRule();
      }
  }
  ```

* 自定义负载均衡策略(放在平时的位置即可，不需要跳出扫描包)

  ```java
  public interface LoadBalancer {
      ServiceInstance instances(List<ServiceInstance> serviceInstances);
  }
  ```

  ```java
  @Component
  public class MyLb implements LoadBalancer {
  
      /**
       * 用于记录调用次数（AtomicInteger 原子操作类，防止高并发下i++出错）
       */
      private AtomicInteger atomicInteger = new AtomicInteger(0);
  
      /**
       * 获取rest接口第几次请求数并加一（每次服务重启后rest接口计数从1开始）
       *
       * @return
       */
      public final int getAndIncrement() {
          int current;
          int next;
          //判断当前时间的请求次数与期望的值是否相同，不同则被修改过，需要重新获取当前值。直至相同，才允许修改（自旋，防止多线程时自增出错。）
          do {
              //获取当前的值
              current = this.atomicInteger.get();
              //防止超过最大Integer的值
              next = current >= Integer.MAX_VALUE ? 0 : current + 1;
  
          } while (!this.atomicInteger.compareAndSet(current, next));
          System.out.println("******第几次访问，次数next:" + next);
          return next;
      }
  
      @Override
      public ServiceInstance instances(List<ServiceInstance> serviceInstances) {
          //rest接口第几次请求数 % 服务器集群总数 = 实际调用服务器位置下标。
          int index = getAndIncrement() % serviceInstances.size();
          return serviceInstances.get(index);
      }
  }
  ```

* controller层

  ```java
  @GetMapping(value = "/consumer/payment/lb")
  public String getPaymentUrlLB(){
      //获取注册中心里服务名为CLOUD-PAYMENT-SERVICE的所有服务信息
      List<ServiceInstance> instances = discoveryClient.getInstances("CLOUD-PAYMENT-SERVICE");
      if (CollectionUtils.isEmpty(instances)){
          return null;
      }
      //通过自定义算法获取提供服务的信息
      ServiceInstance serviceInstance = loadBalancer.instances(instances);
      URI uri = serviceInstance.getUri();
      return restTemplate.getForObject(uri+"/payment/lb",String.class);
  }
  ```



# 三、 Feign（实际为openFeign） 声明式WebService客户端

## 1、客户端关键内容

### Ⅰ pom

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

### Ⅱ yml

```yml
server:
  port: 80
eureka:
  client:
    register-with-eureka: false
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka/,http://eureka7002.com:7002/eureka/
#设置feign客户端超时时间(openFeign默认支持ribbon)
ribbon:
#指的是建立连接所用的时间，适用于网络状况正常的情况下，两端连接所用的时间
  ReadTimeout: 5000
#指的是建立连接后从服务器读取到可用资源所用到的时间
  ConnectTimeout: 5000

logging:
  level:
    # feign日志以什么级别监管哪个接口
    com.guiyang.springcloud.service.PaymentFeignService: debug
```

### Ⅲ 主启动

```java
@SpringBootApplication
@EnableFeignClients
public class OrderFeignMain80 {
    public static void main(String[] args) {
        SpringApplication.run(OrderFeignMain80.class,args);
    }
}
```

### Ⅳ 业务类

* service层内容与需要调用服务的服务端controller层相似

  ```java
  @Component
  @FeignClient("cloud-payment-service") //对应所要调用的服务名
  public interface PaymentFeignService {
  
      @GetMapping(value = "/payment/get/{id}")
      public CommonResult getPaymentById(@PathVariable("id") Long id);
  
      @GetMapping(value = "/payment/feign/timeout")
      public String paymentFeignTimeout();
  }
  ```

* controller

  ```java
  @RestController
  @Slf4j
  public class OrderFeignController {
      @Resource
      private PaymentFeignService paymentFeignService;
  
      @GetMapping(value = "/consumer/payment/get/{id}")
      public CommonResult<Payment> getPaymentById(@PathVariable("id") Long id) {
          return paymentFeignService.getPaymentById(id);
      }
  
      @GetMapping(value = "/consumer/payment/feign/timeout")
      public String paymentFeignTimeout(){
          //默认响应时间超过一秒报错，不用担心，在配置文件中设置了5秒
          return paymentFeignService.paymentFeignTimeout();
      }
  }
  ```

* config

  ```java
  @Configuration
  public class FeignConfig {
      @Bean
      Logger.Level feignLoggerLevel(){
          return Logger.Level.FULL;
      }
  }
  ```

# 四、Hystrix 断路器

## 1、概念

​			用于处理分布式系统的 ***延迟*** 和 ***容错*** 的开源库。能保证在一个依赖出问题的情况下，<u>不会导致整个服务失败，避免级联故障，以提高分布式系统的弹性</u>



## 2、作用

* 服务降级 ： 友好提示
* 服务熔断 ： 直接拒绝访问并友好提示
* 接近事实的监控
* 限流
* ......

流程 :  服务降级 -----> 熔断 ------> **恢复调用链路**

## 3、服务降级

### Ⅰ provider服务端

#### 1）pom

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```

#### 2）yml

```yaml
#为了服务监控而配置 访问监控服务地址时，需要加actuator，eg.localhost:8001/actuator/hystrix.stream
management:
  endpoints:
    web:
      exposure:
        include: hystrix.stream
```

#### 3）主启动

```java
@SpringBootApplication
@EnableEurekaClient
@EnableCircuitBreaker
public class PaymentHystrixMain8001 {
    public static void main(String[] args) {
        SpringApplication.run(PaymentHystrixMain8001.class,args);
    }
}
```

#### 4）业务类

```java
@Service
public class PaymentService {
    /**
     * 正常访问，肯定ok
     *
     * @param id
     * @return
     */
    public String paymentInfo_OK(Integer id) {
        return "线程池： " + Thread.currentThread().getName() + "   paymentInfo_OK,id:  " + id + "\t";
    }

    /**
     * 延迟访问(fallbackMethod = "paymentInfo_TimeOutHandler"超时和出错都会降级处理 )
     *
     * @param id
     * @return
     */
    @HystrixCommand(fallbackMethod = "paymentInfo_TimeOutHandler", commandProperties = {
            @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "5000")
    })
    public String paymentInfo_TimeOut(Integer id) {
//        int error = 10 / 0;
        try { TimeUnit.MILLISECONDS.sleep(3000); } catch (InterruptedException e) { e.printStackTrace(); }
        return "线程池： " + Thread.currentThread().getName() + "  id:  " + id + "\t 耗时";
    }

    public String paymentInfo_TimeOutHandler(Integer id) {

        return "线程池： " + Thread.currentThread().getName() + "   系统繁忙或运行报错，请稍后再试,id:  " + id;
    }
}
```

>  	@HystrixCommand(fallbackMethod = "方法名",commandProperties={@HystrixProperty(....),@HystrixProperty(....)}) : 服务方法失败并抛出错误信息后，会自动调用@HystrixCommand标注好的fallbackMethod调用类中指定的方法。

----

### Ⅱ consumer客户端

#### 1） pom

* 与openFeign整合

```xml
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
    </dependency>
</dependencies>
```

#### 2） yml

```yaml
feign:
  hystrix:
    enabled: true
```

#### 3） 主启动

```java
@SpringBootApplication
@EnableFeignClients
@EnableHystrix
public class OrderHystrixMain80 {
    public static void main(String[] args){
        SpringApplication.run(OrderHystrixMain80.class,args);
    }
}
```

#### 4） 业务类

* service

  ```java
  @Component
  @FeignClient(value = "CLOUD-PROVIDER-HYSTRIX-PAYMENT",fallback = PaymentFallbackService.class)
  public interface PaymentHystrixService {
      @GetMapping(value = "/payment/hystrix/ok/{id}")
      public String paymentInfo_OK(@PathVariable("id")Integer id);
  
      @GetMapping(value = "/payment/hystrix/timeout/{id}")
      public String paymentInfo_TimeOut(@PathVariable("id")Integer id);
  }
  ```

* impl 当服务端宕机或出现延迟无法得到消息时，返回impl里的内容，这样就能保证两方都要兜底方案。同时可实现解耦

  ```java
  @Component
  public class PaymentFallbackService implements PaymentHystrixService{
      @Override
      public String paymentInfo_OK(Integer id) {
          return "----------PaymentFallbackService fallback paymentInfo_OK";
      }
  
      @Override
      public String paymentInfo_TimeOut(Integer id) {
          return "----------PaymentFallbackService fallback paymentInfo_TimeOut";
      }
  }
  ```

* controller

  ```java
  @RestController
  @Slf4j
  @DefaultProperties(defaultFallback = "paymentGlobalFallbackMethod")//全局兜底方案，可以不用一个方法写一个兜底方案了。只需要指定全局兜底的函数，配合@HystrixCommand（不指定参数出错时调用全局兜底方案）即可完成。也解决了代码膨胀的问题
  public class OrderHystrixController {
      @Resource
      private PaymentHystrixService paymentHystrixService;
  
      @GetMapping(value = "/consumer/payment/hystrix/ok/{id}")
      public String paymentInfo_OK(@PathVariable("id")Integer id){
          return paymentHystrixService.paymentInfo_OK(id);
      }
  
      @GetMapping(value = "/consumer/payment/hystrix/timeout/{id}")
  //    @HystrixCommand(fallbackMethod = "paymentTimeOutFallbackMethod",commandProperties = {
  //            @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds",value = "1500")
  //    })
      @HystrixCommand
      public String paymentInfo_TimeOut(@PathVariable("id")Integer id){
          //此行必定报错，用于测试运行时错误是否会使用兜底
          int error = 10 / 0;
          return paymentHystrixService.paymentInfo_TimeOut(id);
      }
  
      public String paymentTimeOutFallbackMethod(@PathVariable("id")Integer id){
          return "我是消费者80，对方支付系统繁忙，请十秒钟后再试或者自己运行出错请检查自己!";
      }
  
      //下面是全局fallback方法
      public String paymentGlobalFallbackMethod(){
          return "Global异常处理信息，请稍后再试！";
      }
  ```

### Ⅲ 作用范围

​		可在客户端，也可在服务端。一般放在服务端。

----

### Ⅳ 哪些情况触发服务降级

* 程序运行异常

* 超时

* 服务熔断触发服务降级

* 线程池/信号量打满也会导致服务降级

  

## 4、服务熔断

### Ⅰ 概念

* 应对雪崩效应的一种微服务保护机制。当扇出链路的某个微服务出错不可用或者响应时间太长时，会进行服务降级。进而熔断该节点微服务的调用，快速返回错误的响应信息。
* ***当检查到该节点微服务调用响应正常后，恢复调用链路***
* 在spring框架中，熔断机制通过hystrix实现。hystrix会监控微服务间调用的优化，当失败的调用达到一定的阈值（缺省是5秒内20次调用失败），就会启动熔断机制。

-----

### Ⅱ 熔断机制示意图

![熔断示意图](E:\工作外的软件\image\熔断示意图.png)

### Ⅲ 熔断类型

* 打开：请求不再进行当前服务调用，当打开时长达到所设时钟进入半开
* 关闭：不会对服务进行熔断
* 半开：部分请求根据规则调用当前服务，如果请求成功且符合规则，则认为当前服务恢复正常，关闭熔断

----

### Ⅳ provider服务端

```java
/**
 * @HystrixProperty额外的参数:
 * execution.isolation.strategy THREAD 设置隔离策略，THREAD表示线程池 SEMAPHORE:信号池隔离
 * execution.isolation.semaphore.maxConcurrentRequests 10 当隔离策略选择信号池隔离的时候，用来设置信号池的大小(最大并发数)
 * execution.isolation.thread.timeoutInMilliseconds 10 配置命令执行的超时时间
 * execution.timeout.enabled true 是否启用超时时间
 * execution.isolation.thread.interruptOnTimeout true 执行超时的时候是否中断
 * execution.isolation.thread.interruptOnCancel true 执行被取消的时候是否中断
 * fallback.isolation.semaphore.maxConcurrentRequests 10 允许回调方法执行的最大并发数
 * fallback.enabled true 服务降级是否启用，是否执行回调函数
 * circuitBreaker.enabled true 是否启用断路器
 * ..........详情参照官方文档
 * @param id
 * @return
 */
@HystrixCommand(fallbackMethod = "paymentCircuitBreakerFallback",commandProperties = {
        @HystrixProperty(name="circuitBreaker.enabled",value = "true"), //是否开启断路器
        @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold",value = "10"), //请求次数
        @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds",value = "10000"), //时间窗口期（时间范围// ）
        @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage",value = "60") //失败率达到多少后跳闸
})
public String paymentCircuitBreaker(@PathVariable("id")Integer id){
    if(id<0){
        throw new RuntimeException("********id 不能为负数");
    }
    String serialNumber = IdUtil.simpleUUID();
    return Thread.currentThread().getName()+"\t 调用成功，流水号："+ serialNumber;
}

public String paymentCircuitBreakerFallback(@PathVariable("id")Integer id){
    return "id 不能为负数，请稍后再试，id"+id;
}
```
