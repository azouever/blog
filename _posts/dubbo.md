---

title: Dubbo学习和对于RPC的理解

date: 2020-09-18 15:42:17

---

##### 什么是RPC

> Remote Procedure Call 远程过程调用，就是一种通过网络从远程计算机程序上请求服务，而不需要了解底层网络技术的协议。满足分布式系统架构中不同的系统之间的远程通信和相互调用

- 从通信协议来看可以分为：
基于HTTP协议：Rest（Json）、Hession（binary）；
基于TCP协议（以Netty高性能网络框架，然后使用自己实现的高效率应用协议）。

- 从语言和平台理解：
 仅支持特定语言： Java的RMI；
 可以跨平台：HTTP Rest、Thrift。

- 从调用过程来看
 同步调用;
 异步调用。

- 不同的序列化协议
 序列化：将对象转成二进制流;
 反序列化：将二进制流转换成对象。
 使用不同的序列化框架对RPC的性能有影响,所以要选择高效率的序列化框架，考虑速度，空间，能否跨语言

- 网络协议，网络模型，线程模型

- RPC安全服务接口的鉴权和访问控制

- 常见的RPC框架：
> 
> Dubbo alibaba Java
> 
> Motan weibo Java
> 
> Tars tencent C++
> 
> gRPC google multi language
> 
> Thrift facebook multi language
> 
> Spring Cloud 全套的微服务下的服务治理方案
> 

##### 架构

- Provider
- Consumer 从提供者地址列表中，基于软负载均衡算法，选一台提供者进行调用，如果调用失败，再选另一台调用。
- Registry 服务注册与发现的注册中心，注册中心返回服务提供者地址列表给消费者，如果有变更，注册中心将基于长连接推送变更数据给消费者；负责服务地址的注册与查找，相当于目录服务，服务提供者和消费者只在启动时与注册中心交互，注册中心不转发请求，压力较小。
- Monitor 统计服务的调用次数和调用时间的监控中心，异步
- Container

> 连通性 注册中心，服务提供者，服务消费者三者之间均为长连接，监控中心除外
> 
> 健壮性  注册中心和监控中心全部宕机，不影响已运行的提供者和消费者，消费者在本地缓存了提供者列表
> 注册中心和监控中心都是可选的，服务消费者可以直连服务提供者
> 
> 伸缩性
> 注册中心为对等集群，可动态增加机器部署实例，所有客户端将自动发现新的注册中心
服务提供者无状态，可动态增加机器部署实例，注册中心将推送新的服务提供者信息给消费者

##### 源码结构模块

> 写在前面
> 
> dubbo的基本设计原则是采用URL作为配置信息的统一格式，所有拓展点都通过传递URL携带配置信息。
> 以URL为总线，运行过程中所有的状态数据信息都可以通过URL来获取，比如当前系统采用什么序列化，采用什么通信，采用什么负载均衡等信息，都是通过URL的参数来呈现的，所以在框架运行过程中，运行到某个阶段需要相应的数据，都可以通过对应的Key从URL的参数列表中获取

###### 注册中心模块

> dubbo的注册中心实现有Multicast、Zookeeper、Redis、Nacos、Consul、Etcd、Eureka、Sofa、Simple，这个模块就是封装了dubbo所支持的注册中心的实现。官方推荐Zookeeper。

- Zookeeper
  1. Provider向zookeeper注册自身的url，生成一个临时的znode
  
  2. Provider从Dubbo容器中退出，停止提供RPC调用。也就是移除zookeeper内自身url对应的znode
  
  3. Consumer订阅 " /dubbo/…Service/providers" 目录的子节点，生成ChildListener
  
  4. Consumer从Dubbo容器中退出，移除之前创建的ChildListener
  
  5. 当Dubbo里的zookeeper Client和Server重新连接上时，将之前注册的的URL添加入这几个失败集合中，然后重新注册和订阅。
  
  6. 当重新连接时( 重新连接意味着之前连接断开了 )，将已经注册和订阅的URL添加到失败集合中，定时重试，也就是重新注册和订阅
  
  7. zookeeper Server宕机了，Dubbo里的Client并没有对此事件做什么响应，当然其内部的zkClient会不停地尝试连接Server。当Zookeeper Server宕机了不影响Dubbo里已注册的组件的RPC调用，因为已经通过URL生成了Invoker对象，这些对象还在Dubbo容器内。当然因为注册中心宕机了，肯定不能感知到新的Provider。同时因为在之前订阅获得的Provider信息已经持久化到本地文件，当Dubbo应用重启时，如果zookeeper注册中心不可用，会加载缓存在文件内的Provider信息，还是能保证服务的高可用。
  
     Consumer会一直维持着对Provider的ChildListener，监听Provider的实时数据信息。当Providers节点的子节点发生变化时，实时通知Dubbo，更新URL，同时更新Dubbo容器内的Consumer Invoker对象，只要是订阅成功均会实时同步Provider，更新Invoker对象，无论是第一次订阅还是断线重连后的订阅。

###### 集群模块
> 将多个服务提供方伪装为一个提供方，包括：负载均衡, 容错，路由等，集群的地址列表可以是静态配置的，也可以是由注册中心下发。
> 
> 为了避免单点故障，现在的应用通常至少会部署在两台服务器上。对于一些负载比较高的服务，会部署更多的服务器。这样，在同一环境下的服务提供者数量会大于1。对于服务消费者来说，同一环境下出现了多个服务提供者。这时会出现一个问题，服务消费者需要决定选择哪个服务提供者进行调用。另外服务调用失败时的处理措施也是需要考虑的，是重试呢，还是抛出异常，亦或是只打印异常等。为了处理这些问题，Dubbo 定义了集群接口 Cluster 以及 Cluster Invoker。集群 Cluster 用途是将多个服务提供者合并为一个 Cluster Invoker，并将这个 Invoker 暴露给服务消费者。这样一来，服务消费者只需通过这个 Invoker 进行远程调用即可，至于具体调用哪个服务提供者，以及调用失败后如何处理等问题，现在都交给集群模块去处理。集群模块是服务提供者和服务消费者的中间层，为服务消费者屏蔽了服务提供者的情况，这样服务消费者就可以专心处理远程调用相关事宜。比如发请求，接受服务提供者返回的数据等。这就是集群的作用。

###### 公共逻辑模块
> 包括 Util 类和通用模型
> 

###### 配置模块
> Dubbo对外的 API，用户通过 Config 使用Dubbo，隐藏 Dubbo 所有细节。用户都是使用配置来使用dubbo，dubbo也提供了四种配置方式，包括XML配置、属性配置、API配置、注解配置，配置模块就是实现了这四种配置的功能。
> 

###### 远程调用模块
> 抽象各种协议，以及动态代理，只包含一对一的调用，不关心集群的管理。远程调用，最主要的肯定是协议，dubbo提供了许许多多的协议实现，不过官方推荐时使用dubbo自己的协议，还给出了一份性能测试报告。这个模块依赖于dubbo-remoting模块，抽象了各类的协议
> 

###### 远程通信模块
> 相当于 Dubbo 协议的实现，如果 RPC 用 RMI协议则不需要使用此包。提供了多种客户端和服务端通信功能，比如基于Grizzly、Netty、Etcd3、Http、Mina等等，RPC用除了RMI的协议都要用到此模块。
> 
> 默认使用 Netty 框架。

###### 容器模块
>  因为后台服务不需要Tomcat/JBoss 等 Web 容器的功能，不需要用这些厚实的容器去加载服务提供方，既资源浪费，又增加复杂度。服务容器只是一个简单的Main方法，加载一些内置的容器，也支持扩展容器。
> 

###### 监控模块
>  统计服务调用次数，调用时间的，调用链跟踪的服务。这个模块很清楚，就是对服务的监控。
> 

###### 过滤器模块
> 内置的一些过滤器：提供缓存过滤器。提供参数验证过滤器。
> 

###### 插件模块
>  内置的插件：提供了在线运维的命令。
> 

###### 序列化模块
> 封装了各类序列化框架的支持实现。序列化也支持扩展。
> 默认使用Hessian序列化。还有其他序列化框架：gson，protobuf，hessian2，jdk，avro，kryo等

###### SPI机制
> Jdk的SPI，Dubbo的SPI，SpringBoot的SPI
>
>SPI的全名为Service Provider Interface，面向对象的设计里面，模块之间推荐基于接口编程，而不是对实现类进行硬编码，这样做也是为了模块设计的可拔插原则。
>为了在模块装配的时候不在程序里指明是哪个实现，就需要一种服务发现的机制。
>
>jdk的spi就是为某个接口寻找服务实现。jdk提供了服务实现查找的工具类：java.util.ServiceLoader，它会去加载META-INF/service/目录下的配置文件。
>JDK标准的SPI只能通过遍历来查找扩展点和实例化，有可能导致一次性加载所有的扩展点，如果不是所有的扩展点都被用到，就会导致资源的浪费。
>
>Dubbo把SPI配置文件中扩展实现的格式修改，例如META-INF/dubbo/com.xxx.Protocol里的com.foo.XxxProtocol格式改为了xxx = com.foo.XxxProtocol这种以键值对的形式，这样做的目的是为了让我们更容易的定位到问题，
>
>
>dubbo的SPI机制增加了对IOC、AOP的支持，一个扩展点可以直接通过setter注入到其他扩展点。
>
>
>
>
1. @SPI，在某个接口上加上@SPI注解后，表明该接口为可扩展接口。如下:

```java
//在Protocol上有@SPI("dubbo")注解。
// 而这个protocol属性值或者默认值会被当作该接口的实现类中的一个key，
// dubbo会去META-INF/dubbo/internal/com.alibaba.dubbo.rpc.Protocol文件中找该key对应的value
@SPI("dubbo")
public interface Protocol {

}
```

2. @Adaptive，该注解为了保证dubbo在内部调用具体实现的时候不是硬编码来指定引用哪个实现，也就是为了适配一个接口的多种实现。
这样做符合模块接口设计的可插拔原则，也增加了整个框架的灵活性，该注解也实现了扩展点自动装配的特性。
- 实现类上面加上@Adaptive注解，表明该实现类是该接口的适配器。
dubbo中的ExtensionFactory接口就有一个实现类AdaptiveExtensionFactory，加了@Adaptive注解，AdaptiveExtensionFactory就不提供具体业务支持，用来适配ExtensionFactory的SpiExtensionFactory和SpringExtensionFactory这两种实现。AdaptiveExtensionFactory会根据在运行时的一些状态来选择具体调用ExtensionFactory的哪个实现，
- 在接口方法上加@Adaptive注解，dubbo会动态生成适配器类。有该注解的方法的参数必须包含URL，ExtensionLoader会通过createAdaptiveExtensionClassCode方法动态生成一个Transporter$Adaptive类。
```java
//@Adaptive注解中有一些key值，比如connect方法的注解中有两个key，分别为“client”和“transporter”，
// URL会首先去取client对应的value来作为注解@SPI中写到的key值，
// 如果为空，则去取transporter对应的value，
// 如果还是为空，则会根据SPI默认的key，也就是netty去调用扩展的实现类，
// 如果@SPI没有设定默认值，则会抛出IllegalStateException异常。
@SPI("netty")
public interface Transporter {
    /**
     * Connect to a server.
     */
    @Adaptive({Constants.CLIENT_KEY, Constants.TRANSPORTER_KEY})
    Client connect(URL url, ChannelHandler handler) throws RemotingException;
```

3. @Activate，扩展点自动激活加载的注解，
就是用条件来控制该扩展点实现是否被自动激活加载，在扩展实现类上面使用，实现了扩展点自动激活的特性，它可以设置两个参数，分别是group和value。

```java
//META-INF/dubbo/com.alibaba.dubbo.rpc.Filter，内容为：
//clearXbootContext=com.xboot.core.config.dubbo.ClearXbootContextFilter
//当配置了xxx参数，并且参数为有效值时激活，比如配了cache="lru"，自动激活CacheFilter。
@Activate(group = "provider", value = "xxx", order = -10000) // 只对提供方激活，group可选"provider"或"consumer"
public class ClearXbootContextFilter implements Filter {
    // ...
}
```
###### 不同的应用层协议
> 推荐使用dubbo协议。
> 
> Dubbo 的设计目的是为了满足高并发小数据量的 rpc 调用，在大数据量下的性能表现并不好，建议使用 rmi 或 http 协议。
>
> 如对性能有更高要求可以使用 dubbo 序列化，由其是在处理复杂对象时，在大数据量下能获得 50% 的提升（但此时已不建议使用 Dubbo 协议）
> 
> 不同服务在性能上适用不同协议进行传输，比如大数据用短连接协议，小数据大并发用长连接协议 

###### 不同的容错方案

> 调用一个服务节点失败后如何处理

- Failover Cluster - 失败自动切换，自动重试其它服务器。
- Failfast Cluster - 快速失败，立即报错，只发起一次调用。
- Failsafe Cluster - 失败安全，出现异常时，直接忽略。
- Failback Cluster - 失败自动恢复，记录失败请求，定时重发。
- Forking Cluster - 并行调用多个服务提供者，并行调用多个服务器，只要一个成功即返回 。
- Broadcast Cluster | 广播逐个调用所有提供者，任意一个报错则报错

###### 不同的负载均衡策略

> LoadBalance 中文意思为负载均衡，它的职责是将网络请求，或者其他形式的负载“均摊”到不同的机器上。避免集群中部分服务器压力过大，而另一些服务器比较空闲的情况。通过负载均衡，可以让每台服务器获取到适合自己处理能力的负载。在为高负载服务器分流的同时，还可以避免资源浪费，一举两得。负载均衡可分为软件负载均衡和硬件负载均衡。在我们日常开发中，一般很难接触到硬件负载均衡。但软件负载均衡还是可以接触到的，比如 Nginx。在 Dubbo 中，也有负载均衡的概念和相应的实现。Dubbo 需要对服务消费者的调用请求进行分配，避免少数服务提供者负载过大。服务提供者负载过大，会导致部分请求超时。因此将负载均衡到每个服务提供者上，是非常必要的。Dubbo 提供了4种负载均衡实现，分别是基于权重随机算法的 RandomLoadBalance、基于最少活跃调用数算法的 LeastActiveLoadBalance、基于 hash 一致性的 ConsistentHashLoadBalance，以及基于加权轮询算法的 RoundRobinLoadBalance。这几个负载均衡算法代码不是很长，但是想看懂也不是很容易，需要大家对这几个算法的原理有一定了解才行。如果不是很了解，也没不用太担心。我们会在分析每个算法的源码之前，对算法原理进行简单的讲解，帮助大家建立初步的印象。

> RandomLoadBalance 是加权随机算法的具体实现，它的算法思想很简单。假设我们有一组服务器 servers = [A, B, C]，他们对应的权重为 weights = [5, 3, 2]，权重总和为10。现在把这些权重值平铺在一维坐标值上，[0, 5) 区间属于服务器 A，[5, 8) 区间属于服务器 B，[8, 10) 区间属于服务器 C。接下来通过随机数生成器生成一个范围在 [0, 10) 之间的随机数，然后计算这个随机数会落到哪个区间上。比如数字3会落到服务器 A 对应的区间上，此时返回服务器 A 即可。权重越大的机器，在坐标轴上对应的区间范围就越大，因此随机数生成器生成的数字就会有更大的概率落到此区间内。只要随机数生成器产生的随机数分布性很好，在经过多次选择后，每个服务器被选中的次数比例接近其权重比例。比如，经过一万次选择后，服务器 A 被选中的次数大约为5000次，服务器 B 被选中的次数约为3000次，服务器 C 被选中的次数约为2000次。
> 

> LeastActiveLoadBalance 翻译过来是最小活跃数负载均衡。活跃调用数越小，表明该服务提供者效率越高，单位时间内可处理更多的请求。此时应优先将请求分配给该服务提供者。在具体实现中，每个服务提供者对应一个活跃数 active。初始情况下，所有服务提供者活跃数均为0。每收到一个请求，活跃数加1，完成请求后则将活跃数减1。在服务运行一段时间后，性能好的服务提供者处理请求的速度更快，因此活跃数下降的也越快，此时这样的服务提供者能够优先获取到新的服务请求、这就是最小活跃数负载均衡算法的基本思想。除了最小活跃数，LeastActiveLoadBalance 在实现上还引入了权重值。所以准确的来说，LeastActiveLoadBalance 是基于加权最小活跃数算法实现的。举个例子说明一下，在一个服务提供者集群中，有两个性能优异的服务提供者。某一时刻它们的活跃数相同，此时 Dubbo 会根据它们的权重去分配请求，权重越大，获取到新请求的概率就越大。如果两个服务提供者权重相同，此时随机选择一个即可。
> 

>一致性 hash 算法由麻省理工学院的 Karger 及其合作者于1997年提出的，算法提出之初是用于大规模缓存系统的负载均衡。它的工作过程是这样的，首先根据 ip 或者其他的信息为缓存节点生成一个 hash，并将这个 hash 投射到 [0, 2^32 - 1] 的圆环上。当有查询或写入请求时，则为缓存项的 key 生成一个 hash 值。然后查找第一个大于或等于该 hash 值的缓存节点，并到这个节点中查询或写入缓存项。如果当前节点挂了，则在下一次查询或写入缓存时，为缓存项查找另一个大于其 hash 值的缓存节点即可。
> 引入虚拟节点，让 Invoker 在圆环上分散开来，避免数据倾斜问题。所谓数据倾斜是指，由于节点不够分散，导致大量请求落到了同一个节点上，而其他节点只会接收到了少量请求的情况

>  RoundRobinLoadBalance。这里从最简单的轮询开始讲起，所谓轮询是指将请求轮流分配给每台服务器。举个例子，我们有三台服务器 A、B、C。我们将第一个请求分配给服务器 A，第二个请求分配给服务器 B，第三个请求分配给服务器 C，第四个请求再次分配给服务器 A。这个过程就叫做轮询。轮询是一种无状态负载均衡算法，实现简单，适用于每台服务器性能相近的场景下。但现实情况下，我们并不能保证每台服务器性能均相近。如果我们将等量的请求分配给性能较差的服务器，这显然是不合理的。因此，这个时候我们需要对轮询过程进行加权，以调控每台服务器的负载。经过加权后，每台服务器能够得到的请求数比例，接近或等于他们的权重比。比如服务器 A、B、C 权重比为 5:2:1。那么在8次请求中，服务器 A 将收到其中的5次请求，服务器 B 会收到其中的2次请求，服务器 C 则收到其中的1次请求。
>  

###### 不同粒度配置的覆盖关系

> 方法级优先，接口级次之，全局配置再次之。
如果级别一样，则消费方优先，提供方次之。

###### 服务提供方的异步执行和服务消费方的异步调用

> 服务消费方的异步调用
> 
> 基于 NIO 的非阻塞实现并行调用，客户端不需要启动多线程即可完成并行调用多个远程服务，相对多线程开销较小。
> 

``` Java
// 调用直接返回CompletableFuture
CompletableFuture<String> future = asyncService.sayHello("async call request");
// 增加回调
future.whenComplete((v, t) -> {
    if (t != null) {
        t.printStackTrace();
    } else {
        System.out.println("Response: " + v);
    }
});
// 早于结果输出
System.out.println("Executed before response return.");
```
> 服务提供方的异步执行
> 
> Provider端异步执行将阻塞的业务从Dubbo内部线程池切换到业务自定义线程，避免Dubbo线程池的过度占用，有助于避免不同服务间的互相影响。异步执行无益于节省资源或提升RPC响应性能，因为如果业务执行需要阻塞，则始终还是要有线程来负责执行。
> 

``` Java
// 业务执行已从Dubbo线程切换到业务线程，避免了对Dubbo线程池的阻塞。
public class AsyncServiceImpl implements AsyncService {
    @Override
    public CompletableFuture<String> sayHello(String name) {
        RpcContext savedContext = RpcContext.getContext();
        // 建议为supplyAsync提供自定义线程池，避免使用JDK公用线程池
        return CompletableFuture.supplyAsync(() -> {
            System.out.println(savedContext.getAttachment("consumer-key1"));
            try {
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return "async response from provider.";
        });
    }
}
```
###### 路由规则和配置规则

> 路由规则在发起一次RPC调用前起到过滤目标服务器地址的作用，过滤后的地址列表，将作为消费端最终发起RPC调用的备选地址。

条件路由。支持以服务或Consumer应用为粒度配置路由规则。

标签路由。以Provider应用为粒度配置路由规则。通过将某一个或多个服务的提供者划分到同一个分组，约束流量只在指定分组中流转，从而实现流量隔离的目的，可以作为蓝绿发布、灰度发布等场景的能力基础。

---

> 配置覆盖规则是Dubbo设计的在无需重启应用的情况下，动态调整RPC调用行为的一种能力。2.7.0版本开始，支持从服务和应用两个粒度来调整动态配置。
> 


##### 好玩的

```lua
-- 配合各种Escape Sequence
string.rep('hahaha\t', 10)
```
##### 相关介绍


##### 参考
- https://dubbo.apache.org/zh-cn/docs/user/quick-start.html
- https://segmentfault.com/a/1190000016741532

