# Dubbo与分布式

## Dubbo

![image-20201227154340064](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\image-20201227154340064.png)

Provider 启动时会向注册中心把自己的元数据注册上去（比如服务IP和端口等），Consumer 在启动时从注册中心订阅（第一次订阅会拉取全量数据）服务提供方的元数据，注册中心中发生数据变更会推送给订阅的Consumer.在获取服务元数据后，Consumer可以发起RPC调用，在RPC调用前后向监控中心上报统计信息。（比如并发数和调用的接口）

- @Service 注解，由Dubbo服务将这个实现类提升为Spring容器的Bean，并且负责配置初始化和服务暴露。
- @Reference 注解，消费服务，该注解适用于对象字段和方法中。



### 服务暴露的实现原理

Dubbo 框架做服务暴露分为两大部分：第一步将持有的服务实例通过代理转换成Invoker，第二步会把Invoker通过具体的协议转换成Exporter。

框架真正进行服务暴露的入口点在 ServiceConfig#doExport 中，无论XML还是注解，都会转换成ServiceBean，继承自ServiceConfig；Dubbo 支持多注册中心同时写，Dubbo 也支持相同服务暴露多个协议，框架内部会依次对使用的协议都做一次服务暴露，每个协议注册元数据都会写入多个注册中心。

**远程服务的暴露机制**：

1. 通过反射获取配置对象并放到map中用于后续构造URL参数，并且获取全局配置(属性前面有default前缀)；
2. 动态代理的方式创建Invoker对象，将服务实例转换成Invoker；在服务端生成的是`AbstractProxyInvoker`实例，所有真实的方法调用都会委托给代理，然后代理转发给服务ref调用；目前框架实现两种代理：JavassistProxyFactory 和 JdkProxyFactory。
3. 先触发服务暴露(端口打开等)，然后进行服务元数据注册（比如服务IP和端口等）；
4. 将服务实例ref转换成Invoker之后，如果有注册中心时，注册中心在做服务暴露会依次做以下事情：
   1. 委托具体协议(Dubbo)进行服务暴露，创建NettyServer监听端口和保存服实例；
   2. 创建注册中心对象，与注册中心创建TCP连接；
   3. 注册服务元数据到注册中心；
   4. 订阅configurators节点，监听服务动态属性变更事件；
   5. 服务销毁收尾工作，比如关闭端口、反注册服务信息等。

拦截器的添加流程：

1. 在触发Dubbo协议暴露前先对服务Invoker做了一层拦截器构建(构造了拦截器链)，在加载所有拦截器时会过滤只对provider生效的数据；
2. 获取真实服务ref对应的Invoker并挂载到整个拦截器链尾部，然后逐层包裹其他拦截器，保证了真实服务调用是最后触发的；
3. 逐层转发拦截器服务调用，是否调用 下一个拦截器由具体拦截器实现。



**本地服务的暴露机制**：

1. 本地服务暴露时，Dubbo指定用 `injvm` 协议暴露服务，这个协议不会做端口打开操作，仅仅把服务保存在内存中而已；
2. 在 InjvmProtocal 类中存储服务实例信息；



### 服务消费的实现原理

从整体上看，Dubbo 框架做服务消费也分为两大部分，第一步通过持有远程服务实例生成Invoker，这个Invoker在客户端是核心的远程代理对象。第二步会把Invoker通过动态代理转换成用户接口的动态代理引用。

框架真正进行服务引用的入口点在ReferenceBean#getObject，转换成ReferenceBean它继承自ReferenceConfig。

Dubbo 支持多注册中心同时消费，如果配置了服务同时注册多个注册中心，则会在 ReferenceConfig#createProxy中合并成一个Invoker。

**单注册中心消费原理**：

1. 优先判断是否在同一个JVM中包含要消费的服务；
2. 注册中心地址后添加refer存储服务消费元数据信息，配置好了消费者元数据消息；
3. 处理只有一个注册中心站的场景，这种场景在客户端是最常见的，客户端启动拉取服务元信息，订阅provider、路由、配置变更。
4. 处理多注册中心的场景；



**核心方法：refer**：

当经过注册中心消费时，通过 refer 方法触发数据拉取、订阅和服务Invoker转换等操作；

1. 根据用户指定的注册中心进行协议替换；
2. 创建注册中心实例，真实消费方的元数据信息是放在refer属性中存储的，`registryFactory.getRegistry(url)`，url 已经配置好了消费者元数据信息；
3. 提取消费方refer中保存的元数据信息；
4. 触发真正的订阅和Invoker转换；
5. 创建RegistryDirectory对象，该类实现了NotifyListener接口，服务变更会触发这个类回调notify方法用于重新引用服务；
6. 把消费者元数据信息**注册**到注册中心，比如消费方应用名、IP和端口号等；
7. 订阅服务提供者、路由和动态配置，第一次发起订阅时会拉取类的全量数据；把Invoker转换成接口代理；

Dubbo 协议在返回 DubboInvoker 对象之前会先初始化客户端连接对象，建立TCP连接；Dubbo 支持客户端是否立即和远程服务建立TCP连接。



**多注册中心消费原理**：按照配置注册中心的顺序决定的，配置靠前优先消费(判断注册中心是否有服务可用后消费);



## 分布式

### CAP理论

Consistency: 一致性；

Availability：可用性；

Partition tolerance：分区容错性；

























