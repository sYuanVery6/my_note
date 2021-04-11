### 微服务列表
* 客户端消费者：80
* 微服务提供者：8001
* Commons：公共entity类



### 构建微服务模块：
1. 建module
2. 改POM
3. 写yml
4. 主启动
5. 业务类
    - 5.1 建sql
    - 5.2 entity
    - 5.3 dao
    - 5.4 service
    - 5.5 controller



### 端口：
    1. 80端口为http的默认访问端口，在浏览器访问80端口时可以不加端口号。
    2. 443端口为https的默认访问端口，在浏览器访问443端口时可以不加端口号。



### RestTemplate服务调用

##### restTemplate相当于对httpClient进行了一次封装，实现了微服务之间的请求调用。
1. 使用restTemplate发送POST请求时，接收端的参数要加@RequestBody注解。
2. getForObject与getForEntity区别：
   - 2.1 getForObject返回的是pojo对象
   - 2.2 getForEntity返回的是ResponseEntity，如果需要转换成pojo对象，则需要使用JSON工具类。



## 服务注册与发现

### Eureka（官方已停更）

##### 微服务RPC远程服务调用最核心的是什么？

> 高可用，试想你的注册中心只有一个，他出现故障了，会导致整个微服务环境不可用
>
> 解决办法就是搭建Eureka注册中心集群，实现负载均衡+故障容错

##### Eureka集群的实现原理：互相注册，相互守望。

> 假设由A、B、C三个服务器组成注册中心的集群。则在三个服务的配置文件中修改配置，使得每个EurekaServer都注册在其他的Eureka中。

##### Eureka微服务节点的负载均衡

> 若有一个服务在集群中部署了多台服务器，并且每个机器上的服务在Eureka上注册的服务名相同，那么在其他服务访问这个服务时，Eureka会对这个集群进行负载均衡，选择集群中最优的服务节点调用。

```java
/**
* 调用端如果使用RestTemplate进行接口调用，需要在注册Bean的配置类上加@LoadBalanced注解，给RestTemplate赋予负载均衡的功能
*/
@Configuration
public class ApplicationContextConfig {

    /**
     * LoadBalanced注解可以使restTemplate有负载均衡的能力
     * @return
     */
    @Bean
    @LoadBalanced
    public RestTemplate getRestTemplate(){
        return new RestTemplate();
    }

}
```



##### 服务发现Discovery

拿到eureka上面注册成功的服务信息

```java
/**
 * @author sYuan
 */
@RestController
@Slf4j
@RequestMapping("payment")
public class PaymentController {
    
    /**
     * 服务发现，将基础信息注册到eureka
     */
    @Autowired
    private DiscoveryClient discoveryClient;

    @GetMapping("discovery")
    public Object discovery(){

        List<String> list = discoveryClient.getServices();

        for (String ele:list){
            log.info("element == "+ele);
        }
        return list;
    }
}

```



##### 自我保护机制

> 某时刻某一个微服务不可用了，Eureka不会立刻清理，依旧会对该微服务的信息进行保存，可禁用。

宁可保留错误的服务注册信息，也不会盲目注销任何可能健康的服务实例。

##### 停更说明

> 停更不停用

### Zookeeper实现注册中心

##### Zookeeper环境安装

[centos安装zookeeper教程](https://www.linuxidc.com/Linux/2018-03/151327.htm)

[zookeeper注册中心原理-csdn](https://blog.csdn.net/weixin_44801979/article/details/90550579)

##### 临时节点

> 服务注册进zookeeper中为临时节点，服务停掉后，重新注册进zookeeper中后，会被重新分配一个序列号

##### 查看服务信息

```bash
# 进入zookeeper的bin目录
cd zookeeper/bin
# 执行zkCli.sh
./zkCli.sh
# 查看当前zookeeper中包含的所有内容
ls /
[services, zookeeper]
# 查看所有服务
[zk: localhost:2181(CONNECTED) 18] ls /services
[cloud-consumer-order, cloud-provider-payment]
# 查看服务信息
[zk: localhost:2181(CONNECTED) 5] ls /services/cloud-provider-payment 
[7c5b918d-4df1-414e-8284-6214f9315247]
[zk: localhost:2181(CONNECTED) 6] ls /services/cloud-provider-payment/7c5b918d-4df1-414e-8284-6214f9315247 
{"name":"cloud-provider-payment","id":"31b91d6e-bb88-4ec0-b9c4-d60edc42dac0","address":"LAPTOP-K1JH64NC","port":8004,"sslPort":null,"payload":{"@class":"org.springframework.cloud.zookeeper.discovery.ZookeeperInstance","id":"application-1","name":"cloud-provider-payment","metadata":{}},"registrationTimeUTC":1614489491379,"serviceType":"DYNAMIC","uriSpec":{"parts":[{"value":"scheme","variable":true},{"value":"://","variable":false},{"value":"address","variable":true},{"value":":","variable":false},{"value":"port","variable":true}]}}

```

### Consul

是一套开源的基于go语言的分布式服务发现和配置的管理系统。

* 服务发现
* 健康监测
* KV存储
* 多数据中心
* 可视化web界面

### 三个服务注册中心异同点

| 组件名    | 语言 | CAP  | 服务健康检查 | 对外暴露接口 | SpringCloud集成 |
| --------- | ---- | ---- | ------------ | ------------ | --------------- |
| Eureka    | java | AP   | 可配支持     | HTTP         | 已集成          |
| Consul    | go   | CP   | 支持         | HTTP/DNS     | 已集成          |
| Zookeeper | java | CP   | 支持         | 客户端       | 已集成          |

##### CAP

> * A：Availability-高可用
> * C：Consistency-强一致
> * P：Partition Tolerance-分布式的分区容错性
>
> ###### 核心理论
>
> ​	一个分布式系统不可能同时很好的满足一致性,可用性和分区容错性这三个需求,因此,根据CAP原理将NoSql数据库分成了满足CA原则、满足CP原则和满足AP原则三大类：
>
> * CA-单点集群，满足一致性，可用性的系统，通常在可扩展性上不太强大
> * CP-满足一致性，分区容错性的系统，通常性能不是特别高
> * AP-满足可用性，分区容错性的系统，通常可能对一致性要求低一些 



## 服务调用—负载均衡

##### 负载均衡

> 简单的说就是将用户请求平摊的分配到多个服务器上，从而达到系统的HA（高可用）
>
> 常见的负载均衡有软件Nginx，LVS，硬件F5等

##### 集中式LB与进程内LB

> * 集中式LB：即在服务的消费方和提供方之间使用独立的LB设施（可以是硬件，如F5，也可以是软件，如nginx），由该设施负责把访问请求通过某种策略转发至服务的提供方
>
> * 进程内LB：将LB逻辑继承到消费方，消费方从服务注册中心获知有哪些地址可用，然后自己再从这些地址中选择出一个合适的服务器。Ribbon就属于进程内LB，它只是一个类库，集成于消费方进程，消费方通过它来获取到服务提供方的地址

### Ribbon(维护)—Spring-Cloud-LoadBalance

##### 与nginx区别

> Nginx是服务器负载均衡，客户端所有请求都会交给nginx，然后由nginx实现转发请求，即负载均衡是由服务端实现的
>
> Ribbon本地负载均衡，在调用微服务接口时候，会在注册中心上获取注册信息服务列表之后缓存到本地JVM本地，从而在本地实现RPC远程服务调用技术

##### Ribbon工作时分成两步

> 1. 先选择EurekaServer，它优先选择在同一个区域内负载较少的server
> 2. 再根据用户指定的策略，在从server取到的服务注册列表中选择一个地址；其中Ribbon提供了很多种策略：比如轮询、随机和根据响应时间加权

##### IRule

> 选择一个有效的负载均衡算法实现负载均衡的接口
>
> * **RoundRobinRule**—轮询
> * **RandomRule**—随机
> * **RetryRule**—先按照RoundRobinRule的策略获取服务，如果获取服务失败则在指定时间内会进行重新获取服务
> * **WeightedResponseTimeRule**—对RoundRobinRule的扩展，响应速度越快的实例选择权重越大，越容易被选择
> * **BestAvailableRule**—会先过滤由于多次访问故障而处于断路器跳闸状态的服务，然后选择一个并发量最小的服务
> * **AvailabilityFIlteringRule**—选过滤掉故障实例，在选择并发较小的实例
> * **ZoneAvoidanceRule**—默认规则，复合判断server所在区域的性能和server的可能性选择服务器

##### 默认负载轮询算法原理

> serviceList.get(次数%服务数)
>
> ###### demo实现：
>
> ```java
> public interface LoadBalance {
> 
>     ServiceInstance instances(List<ServiceInstance> serviceInstances);
> 
> }
> ```
>
> ```java
> @Component
> public class MyLB implements LoadBalance {
> 
>     private AtomicInteger atomicInteger = new AtomicInteger(0);
> 
>     /**
>      * 获取并增加
>      * @return next
>      */
>     public final int getAndIncerement(){
>         int current ;
>         int next;
>         do {
>             current = this.atomicInteger.get();
>             // Integer.MAX_VALUE = 2147483647
>             next = current >= Integer.MAX_VALUE ? 0 :current + 1;
>         }while (!this.atomicInteger.compareAndSet(current,next));
>         System.out.println("*****第"+next+"次访问.*****");
>         return next;
>     }
> 
>     @Override
>     public ServiceInstance instances(List<ServiceInstance> serviceInstances) {
>         int index = getAndIncerement() % serviceInstances.size();
>         return serviceInstances.get(index);
>     }
>     
> }
> ```
>
> ```java
> @RestController
> @RequestMapping("consumer/payment")
> public class OrderController {
> 
>     @Autowired
>     private LoadBalance loadBalance;
> 
>     @Autowired
>     private RestTemplate restTemplate;
> 
>     @Autowired
>     private DiscoveryClient discoveryClient;
> 
>     @GetMapping("lb")
>     public String getPaymentLB(){
>         List<ServiceInstance> instances = discoveryClient.getInstances("CLOUD-PAYMENT-SERVICE");
>         if(instances == null || instances.size()<=0){
>             return null;
>         }
> 
>         ServiceInstance serviceInstance = loadBalance.instances(instances);
>         URI uri = serviceInstance.getUri();
>         return restTemplate.getForObject(uri+"/payment/test",String.class);
>     }
> }
> ```
>

### OpenFeign

Feign是一个声明式的web服务客户端，让编写web服务客户端变得非常容易，只需要创建一个接口

##### Feign和OpenFeign的区别

> **Fegin**：Feign是SpringCloud组件中的一个轻量级RESTFul的HTTP服务客户端，Feign内置了Ribbon，用来做客户端负载均衡，去调用服务注册中心的服务；Feign的使用方式是：使用Feign的注解定义接口，调用这个接口，就可以服务注册中心的服务
>
> **OpenFeign**：OpenFeign是SpringCloud在Feign的基础上支持了SpringMVC的注解，如@RequestMapping等；OpenFeign的@FeignClient可以解析SpringMVC的@RequestMapping注解下的接口，并通过动态代理的方式产生实现类，实现类中做负载均衡并调用其他服务。

##### OpenFeign超时控制

> 当消费者通过OpenFeign调用生产者时，默认的请求等待时间为1秒，如果超过默认等待时间，就会报超时异常。
>
> 修改默认等待时间：
>
> ```yml
> # openFeign内部使用Ribbon进行负载均衡，所以通过设置Ribbon来调整请求超时时间
> ribbon:
> # 指定是建立连接后，从服务器读取到可用资源的所用时间
> 	ReadTimeout: 5000
> # 指的是建立连接所用的时间，适用于网络状况正常的情况下，两端连接所用的时间
> 	ConnectTimeOut: 5000
> ```
>
> 

#####  OpenFeign日志打印功能

对Feign接口的调用情况进行监控和输出

###### 日志级别：

* NONE:默认的，不显示任何日志
* BASIC:仅记录请求方法、URL、响应状态码及执行时间
* HEADERS:除了BASIC中定义的信息之外，还有请求和响应的头信息
* FULL:除了HEADERS中定义的信息之外，还有请求和响应的正文及元数据

###### 设置日志级别

```java
public class FeignConfig{
    @Bean
    public Logger.Level feginLoggerLevel(){
        return Logger.Level.FULL;
    }
}
```

```yml
logging:
	level:
	# feign日志以什么级别监控哪个接口
		com.ayuan.springcloud.service.PaymentService: debug
```



## 服务降级

### Hystrix（已停更）

##### 分布式系统面临的问题

> 复杂分布式体系结构中的应用程序有数十个依赖关系，每个依赖关系在某些时候将不可避免的失败
>
> ###### 服务雪崩
>
> 多个微服务之间调用的时候，假设微服务A调用微服务B和微服务C，微服务B和微服务C又调用其他的微服务，这就是所谓的`“扇出“`,如果扇出的链路上某个微服务的调用响应时间过长或者不可用，对微服务A的调用就会占用越来越多的系统资源，进而引起系统崩溃，所谓的雪崩效应。
>
> 对于高流量的应用来说，单一的后端依赖可能会导致所有服务器上的资源都在几秒钟内饱和。比失败更糟糕的是，这些应用程序还可能导致服务之间的延迟增加，备份队列，线程和其他系统资源紧张，导致整个系统发生更多的级联故障。这些都表示需要对故障和延迟进行隔离和管理，以便单个依赖关系的失败，不能取消整个应用程序或系统。
>
> 所以，通常当你发现一个模块下的某个实例失败后，这时候这个模块依然还会接收流量，然后这个有问题的模块还调用了其他模块，这样就会发生级联故障，或者叫雪崩。

##### 简介

> Hystrix是一个用于处理分布式系统的延迟和容错的开源库，在分布式系统里，许多依赖不可避免的会调用失败，比如超时、异常等，Hystrix能够保证早一个依赖出问题的情况下，不会导致整体服务失败，避免级联故障，以提高分布式系统的弹性。
>
> “断路器”本身是一种开关装置，当某个服务单元发生故障后，通过断路器的故障监控（类似熔断保险丝），向调用方返回一个符合预期的、可处理的备选响应（fallback），而不是长时间的等待或者抛出调用方法处理的异常，这样就保证了服务调用方的线程不会被长时间、不必要的占用，从而避免了故障在分布式系统中的蔓延，乃至雪崩。

##### 功能

>* 服务降级
>* 服务熔断
>* 接近实时的监控

##### 重要概念

> ###### 服务降级(fallback)
>
> 假设对方系统不可用，你需要给我一个兜底的解决方案。
>
> 出现服务降级的场景：
>
> * 程序运行异常
> * 超时
> * 服务熔断触发服务降级
> * 线程池/信号量打满也会导致服务降级
>
> ###### 服务熔断(break)
>
> 类比保险丝达到最大服务访问后，直接拒绝访问，拉闸限电，然后调用服务降级的方法并返回友好提示
>
> 服务降级->进而熔断->恢复调用链路
>
> ###### 服务限流(flowLimit)
>
> 秒杀高并发等操作，严禁一窝蜂的过来拥挤，大家排队，一秒钟N个有序进行
>
> > ##### 压力测试工具
> >
> > Jmeter
>
> ##### 服务降级解决方案
>
> * 从服务端找问题：设置自身调用超时时间的峰值，峰值内可以正常运行，超过了需要有兜底的方法处理，做服务降级fallback
>
>   * 业务类启动:@HystrixCommand调用指定方法fallback配合主启动类的@*EnableCircuitBreaker*
>
>     ```java
>      	@HystrixCommand(fallbackMethod = "paymentInfoTimeoutHandle",commandProperties = {
>                 //峰值上线3秒钟
>                 @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds",value = "3000")
>         })
>         public String paymentInfoTimeout(Integer id){
>             try {
>                 TimeUnit.SECONDS.sleep(5);
>             } catch (InterruptedException e) {
>                 e.printStackTrace();
>             }
>             return "线程池"+Thread.currentThread().getName()+" payment_ok,id:\t"
>                     +" hana"+"   耗时3秒钟";
>         }
>         public String paymentInfoTimeoutHandle(Integer id){
>             return "线程池"+Thread.currentThread().getName()+" payment_ok,id:\t"
>                     +" wuwu"+"   耗时3秒钟";
>         }
>     ```
>
> * 对客户端进行降级保护
>
>   ```java
>    	@GetMapping("/consumer/hystrix/timeout/{id}")
>       @HystrixCommand(fallbackMethod = "paymentTimeOutFallbackMethod",commandProperties = {
>               @HystrixProperty(name="execution.isolation.thread.timeoutInMilliseconds",value="500")
>       })
>       public String paymentInfoTimeout(@PathVariable("id") Integer id){
>           return paymentHystrixService.paymentInfoTimeout(id);
>       };
>   
>       public String paymentTimeOutFallbackMethod(@PathVariable("id") Integer id)
>       {
>           return "我是消费者80,对方支付系统繁忙请10秒钟后再试或者自己运行出错请检查自己,o(╥﹏╥)o";
>       }
>   ```
>
> * Global FallBack
>
>   * 类上加@DefaultProperties(defaultFallback = "")注解实现默认fallback









