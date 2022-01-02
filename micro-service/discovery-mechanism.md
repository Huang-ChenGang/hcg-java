### 为什么要使用服务发现

代码在调用RestApi来完成一次请求的时候，需要知道服务实例的网络位置，也就是IP和端口。
在传统应用中，服务的IP和端口相对固定，代码只需要从配置文件中去读取就行了。
但是在微服务中，服务实例的网络位置(IP和端口)，是动态分配的。
由于扩展、失败和升级，服务实例会经常的动态改变，因此要使用服务发现机制。

服务发现有两大模式：客户端发现模式和服务端发现模式。

### 客户端发现模式

客户端发现模式，客户端决定相应服务实例的网络位置，并且对清酒实现负载均衡。
客户端查询服务注册表，服务注册表可以理解为是一个可用服务实例的数据库。
客户端使用负载均衡算法从中选出一个实例，并发出请求。

服务实例的网络位置在服务启动时被记录到注册表，等实例终止时被删除。
服务实例的注册信息通常使用心跳机制来定期刷新。

eureka是客户端发现的绝佳范例。eureka服务器提供一个注册表，为服务实例注册和查询提供了RestApi接口。
ribbon在eureka客户端实现对请求的负载均衡。

优点：客户端知晓所有可用的服务实例，可以实现负载均衡。

缺点：要在每个客户端都实现服务发现逻辑。

### 服务端发现模式

客户端通过负载均衡器向某个服务提出请求，负载均衡器查询服务注册表，并将请求转发到可用的服务实例。
nginx可作为负载均衡器，zookeeper可作为服务注册表。

优点：客户端无需关心服务发现的细节，只需要简单地向负载均衡器发送请求。

缺点：负载均衡器可能会成为一个需要配置和管理的高可用系统组件。