## Ribbon 负载均衡

Ribbon的核心作用就是进行请求的负载均衡。客户端集成Ribbon这个组件，Ribbon中会针对已经配置的服务提供者地址列表进行负载均衡的计算，
得到一个目标地址之后，再发起请求。

Ribbon 的原理主要是靠拦截器 LoadBalancerInterceptor 拦截住有 @LoadBalancer 注解的 RestTemplate，将服务名转化为具体的 IP 地址。
由 ILoadBalancer 和 IRule 实现负载均衡机制。也可以自定义负载均衡机制。

可以在 yaml 配置文件中指定负载均衡策略（可以在 spring 官网查询的到）。
配置 key 为 "要访问的服务名.ribbon.NFLoadBalancerRuleClassName"，value 是负载均衡类的完整包路径名。
例：
```properties
PRODUCT.ribbon.NFLoadBalancerRuleClassName=com.netflix.loadbalancer.RandomRule
```

### @Qualifier 注解的作用

@LoadBalancer 注解解析过程之前，先了解下 @Qualifier 注解的作用。

我们平时在使用注解去注入一个Bean时，都是采用@Autowired。并且大家应该知道@Autowired是可以注入一个List或者Map的。
给大家举个例子（在一个springboot应用中）：

定义一个TestClass：
```java
@AllArgsConstructor
@Data
public class TestClass {
    private String name;
}
```

声明一个配置类，并注入TestClass：
```java
@Configuration
public class TestConfig {
    @Bean("testClass1")
    TestClass testClass(){
        return new TestClass("testClass1");
    }

    @Bean("testClass2")
    TestClass testClass2(){
        return new TestClass("testClass2");
    }
}
```

定义一个Controller，用于测试， 注意，此时我们使用的是@Autowired来注入一个List集合：
```java
@RestController
public class TestController {

    @Autowired(required = false)
    List<TestClass> testClasses= Collections.emptyList();

    @GetMapping("/test")
    public Object test(){
        return testClasses;
    }
}
```

此时访问：http://localhost:8080/test , 得到的结果是：
```json
[
    {
        "name": "testClass1"
    },
    {
        "name": "testClass2"
    }
]
```

可以看到，在没使用 @Qualifier 注解的情况下，声明的两个 Bean 都获取到了。

再来看一下使用 @Qualifier 注解的情况：

修改TestConfig和TestController：
```java
@Configuration
public class TestConfig {

    @Bean("testClass1")
    @Qualifier
    TestClass testClass(){
        return new TestClass("testClass1");
    }

    @Bean("testClass2")
    TestClass testClass2(){
        return new TestClass("testClass2");
    }
}
```
```java
@RestController
public class TestController {

    @Autowired(required = false)
    @Qualifier
    List<TestClass> testClasses= Collections.emptyList();

    @GetMapping("/test")
    public Object test(){
        return testClasses;
    }
}
```

再次访问：http://localhost:8080/test , 得到的结果是：
```json
[
    {
        "name": "testClass1"
    }
]
```

可以看到，在声明和引用的地方都使用了 @Qualifier 注解，获取到的就是使用 @Qualifier 注解的 Bean。

### @LoadBalancer 注解解析过程分析

在使用RestTemplate的时候，我们加了一个@LoadBalance注解，就能让这个RestTemplate在请求时，就拥有客户端负载均衡的能力。
```java
@Component
public class JnRestTemplateConfig {
    @Bean
    @LoadBalanced
    public RestTemplate jnRestTemplate() {
        return new RestTemplate();
    }
}
```

然后，我们打开 @LoadBalanced 这个注解，可以发现该注解仅仅是声明了一个@qualifier注解。
```java
@Target({ ElementType.FIELD, ElementType.PARAMETER, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Qualifier
public @interface LoadBalanced {

}
```

综合前面提到的 @Qualifier 注解的作用，再回到 @LoadBalancer 注解上，就不难理解了。

看一下 LoadBalancerAutoConfiguration 这个类：
```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(RestTemplate.class)
@ConditionalOnBean(LoadBalancerClient.class)
@EnableConfigurationProperties(LoadBalancerRetryProperties.class)
public class LoadBalancerAutoConfiguration {
    
    /*
     * 因为我们需要扫描到增加了 @LoadBalancer 注解的 RestTemplate 实例，所以，@LoadBalancer可以完成这个动作
     * 会使用 @Qualifier 同样的方式，把配置了 @LoadBalanced 注解的 RestTemplate 注入到 restTemplates 集合中。
     */
    @LoadBalanced
    @Autowired(required = false)
    private List<RestTemplate> restTemplates = Collections.emptyList();

    @Bean
    @ConditionalOnMissingBean
    public LoadBalancerRequestFactory loadBalancerRequestFactory(
            LoadBalancerClient loadBalancerClient) {
        return new LoadBalancerRequestFactory(loadBalancerClient, this.transformers);
    }

    @Configuration(proxyBeanMethods = false)
    @ConditionalOnMissingClass("org.springframework.retry.support.RetryTemplate")
    static class LoadBalancerInterceptorConfig {

        // 装载一个 LoadBalancerInterceptor 的实例到 IOC 容器
        @Bean
        public LoadBalancerInterceptor loadBalancerInterceptor(
                LoadBalancerClient loadBalancerClient,
                LoadBalancerRequestFactory requestFactory) {
            return new LoadBalancerInterceptor(loadBalancerClient, requestFactory);
        }

        // 遍历所有加了 @LoadBalanced 注解的 RestTemplate，在原有的拦截器之上，再增加了一个 LoadBalancerInterceptor
        @Bean
        @ConditionalOnMissingBean
        public RestTemplateCustomizer restTemplateCustomizer(
                final LoadBalancerInterceptor loadBalancerInterceptor) {
            return restTemplate -> {
                List<ClientHttpRequestInterceptor> list = new ArrayList<>(
                        restTemplate.getInterceptors());
                list.add(loadBalancerInterceptor);
                restTemplate.setInterceptors(list);
            };
        }

    }
}
```

LoadBalancerAutoConfiguration 的配置到这里先放一放，先去看一下 RestTemplate 的调用过程。

使用 RestTemplate.getForObject 发起远程请求的过程如下：
- RestTemplate.getForObject()
- AbstractClientHttpRequest.execute()
- AbstractBufferingClientHttpRequest.executeInternal()
- InterceptingClientHttpRequest.executeInternal()
- InterceptingRequestExecution.execute()

InterceptingRequestExecution 是 InterceptingClientHttpRequest 的私有内部类，具体代码如下：
```java
private class InterceptingRequestExecution implements ClientHttpRequestExecution {
    
    private final Iterator<ClientHttpRequestInterceptor> iterator;
    public InterceptingRequestExecution() {
        this.iterator = interceptors.iterator();
    }

    @Override
    public ClientHttpResponse execute(HttpRequest request, byte[] body) throws IOException {
        // 遍历所有的拦截器，通过拦截器进行逐个处理
        if (this.iterator.hasNext()) {
            ClientHttpRequestInterceptor nextInterceptor = this.iterator.next();
            // 这里可以看到，拦截器执行完直接就返回响应了
            return nextInterceptor.intercept(request, body, this);
        }
        else {
            HttpMethod method = request.getMethod();
            Assert.state(method != null, "No standard HTTP method");
            ClientHttpRequest delegate = requestFactory.createRequest(request.getURI(), method);
            request.getHeaders().forEach((key, value) -> delegate.getHeaders().addAll(key, value));
            if (body.length > 0) {
                if (delegate instanceof StreamingHttpOutputMessage) {
                    StreamingHttpOutputMessage streamingOutputMessage = (StreamingHttpOutputMessage) delegate;
                    streamingOutputMessage.setBody(outputStream -> StreamUtils.copy(body, outputStream));
                }
                else {
                    StreamUtils.copy(body, delegate.getBody());
                }
            }
            return delegate.execute();
        }
    }
}
```

在 LoadBalancerAutoConfiguration 看到声明了 LoadBalancerInterceptor 拦截器，
在 InterceptingRequestExecution 中也看到了用拦截器进行拦截。

看一下 LoadBalancerInterceptor 的拦截方法：
```java
public class LoadBalancerInterceptor implements ClientHttpRequestInterceptor {

	private LoadBalancerClient loadBalancer;

	private LoadBalancerRequestFactory requestFactory;

	public LoadBalancerInterceptor(LoadBalancerClient loadBalancer,
			LoadBalancerRequestFactory requestFactory) {
		this.loadBalancer = loadBalancer;
		this.requestFactory = requestFactory;
	}

	public LoadBalancerInterceptor(LoadBalancerClient loadBalancer) {
		// for backwards compatibility
		this(loadBalancer, new LoadBalancerRequestFactory(loadBalancer));
	}

    /*
     * LoadBalancerInterceptor 是一个拦截器，当一个被 @LoadBalanced 注解修饰的 RestTemplate 对象发起 HTTP 请求时，
     * 会被 LoadBalancerInterceptor 的 intercept 方法拦截
     */
	@Override
	public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
			final ClientHttpRequestExecution execution) throws IOException {
		final URI originalUri = request.getURI();
    
        /*
         * 通过 getHost 方法就可以获取到服务名，因为我们在使用 RestTemplate 调用服务的时候，使用的是服务名而不是域名，
         * 所以这里可以通过 getHost 直接拿到服务名
         */
		String serviceName = originalUri.getHost();
		Assert.state(serviceName != null,
				"Request URI does not contain a valid hostname: " + originalUri);

        // 然后去调用 LoadBalancerClient.execute 方法发起请求
		return this.loadBalancer.execute(serviceName,
				this.requestFactory.createRequest(request, body, execution));
	}

}
```
```java
public class LoadBalancerRequestFactory {

	private LoadBalancerClient loadBalancer;

	private List<LoadBalancerRequestTransformer> transformers;

	public LoadBalancerRequestFactory(LoadBalancerClient loadBalancer,
			List<LoadBalancerRequestTransformer> transformers) {
		this.loadBalancer = loadBalancer;
		this.transformers = transformers;
	}

	public LoadBalancerRequestFactory(LoadBalancerClient loadBalancer) {
		this.loadBalancer = loadBalancer;
	}

    /*
     * 用lambda表达式实现的匿名内部类。在该内部类中，创建了一个ServiceRequestWrapper，
     * 这个ServiceRequestWrapper实际上就是HttpRequestWrapper的一个子类，
     * ServiceRequestWrapper重写了HttpRequestWrapper的getURI()方法，
     * 重写的URI实际上就是通过调用LoadBalancerClient接口的reconstructURI函数来重新构建一个URI进行访问
     */
	public LoadBalancerRequest<ClientHttpResponse> createRequest(
			final HttpRequest request, final byte[] body,
			final ClientHttpRequestExecution execution) {
		return instance -> {
			HttpRequest serviceRequest = new ServiceRequestWrapper(request, instance,
					this.loadBalancer);
			if (this.transformers != null) {
				for (LoadBalancerRequestTransformer transformer : this.transformers) {
					serviceRequest = transformer.transformRequest(serviceRequest,
							instance);
				}
			}
            // 会进入到InterceptingClientHttpRequest.execute方法中
			return execution.execute(serviceRequest, body);
		};
	}

}
```
```java
class InterceptingClientHttpRequest extends AbstractBufferingClientHttpRequest {
    // request对象的实例是HttpRequestWrapper
    public ClientHttpResponse execute(HttpRequest request, byte[] body) throws IOException {
        if (this.iterator.hasNext()) {
            ClientHttpRequestInterceptor nextInterceptor = this.iterator.next();
            return nextInterceptor.intercept(request, body, this);
        }
        else {
            HttpMethod method = request.getMethod();
            Assert.state(method != null, "No standard HTTP method");
            
            /*
             * 当调用request.getURI()获取目标地址创建http请求时，会调用 ServiceRequestWrapper 中的 getURI() 方法，
             * ServiceRequestWrapper 中的 getURI() 方法调用 RibbonLoadBalancerClient 实例中的 reconstructURI 方法
             * 根据 service-id 生成目标服务地址。
             */
            ClientHttpRequest delegate = requestFactory.createRequest(request.getURI(), method);
            
            request.getHeaders().forEach((key, value) -> delegate.getHeaders().addAll(key, value));
            if (body.length > 0) {
                if (delegate instanceof StreamingHttpOutputMessage) {
                    StreamingHttpOutputMessage streamingOutputMessage = (StreamingHttpOutputMessage) delegate;
                    streamingOutputMessage.setBody(outputStream -> StreamUtils.copy(body, outputStream));
                }
                else {
                    StreamUtils.copy(body, delegate.getBody());
                }
            }
            return delegate.execute();
        }
    }
}
```

LoadBalancerClient 是个接口，这里是通过它的实现类 RibbonLoadBalancerClient 的 execute 方法进行调用的：
```java
public class RibbonLoadBalancerClient implements LoadBalancerClient {

    // 根据service-id生成目标服务地址
    public URI reconstructURI(ServiceInstance instance, URI original) {
        Assert.notNull(instance, "instance can not be null");
        // 获取实例id，也就是服务名称
        String serviceId = instance.getServiceId();
        // 获取RibbonLoadBalancerContext上下文，这个是从spring容器中获取的对象实例
        RibbonLoadBalancerContext context = this.clientFactory
                .getLoadBalancerContext(serviceId);

        URI uri;
        Server server;
        if (instance instanceof RibbonServer) {
            RibbonServer ribbonServer = (RibbonServer) instance;
            server = ribbonServer.getServer();
            uri = updateToSecureConnectionIfNeeded(original, ribbonServer);
        }
        else {
            server = new Server(instance.getScheme(), instance.getHost(),
                    instance.getPort());
            IClientConfig clientConfig = clientFactory.getClientConfig(serviceId);
            ServerIntrospector serverIntrospector = serverIntrospector(serviceId);
            uri = updateToSecureConnectionIfNeeded(original, clientConfig,
                    serverIntrospector, server);
        }
        // 调用这个方法拼接成一个真实的目标服务器地址
        return context.reconstructURIWithServer(server, uri);
    }

    @Override
    public <T> T execute(String serviceId, LoadBalancerRequest<T> request)
            throws IOException {
        return execute(serviceId, request, null);
    }

    public <T> T execute(String serviceId, LoadBalancerRequest<T> request, Object hint)
            throws IOException {
        // 根据serviceId获得一个ILoadBalancer，实例为：ZoneAwareLoadBalancer
        ILoadBalancer loadBalancer = getLoadBalancer(serviceId);

        // 调用getServer方法去获取一个服务实例，返回一个具体的目标服务器
        Server server = getServer(loadBalancer, hint);
        // 判断Server的值是否为空。这里的Server实际上就是传统的一个服务节点，这个对象存储了服务节点的一些元数据，比如host、port等
        if (server == null) {
            throw new IllegalStateException("No instances available for " + serviceId);
        }
        // 在调用execute方法之前，会包装一个RibbonServer对象传递下去，它的主要作用是用来记录请求的负载信息
        RibbonServer ribbonServer = new RibbonServer(serviceId, server,
                isSecure(server, serviceId),
                serverIntrospector(serviceId).getMetadata(server));

        return execute(serviceId, ribbonServer, request);
    }

    public <T> T execute(String serviceId, ServiceInstance serviceInstance,
            LoadBalancerRequest<T> request) throws IOException {
        Server server = null;
        if (serviceInstance instanceof RibbonServer) {
            server = ((RibbonServer) serviceInstance).getServer();
        }
        if (server == null) {
            throw new IllegalStateException("No instances available for " + serviceId);
        }

        RibbonLoadBalancerContext context = this.clientFactory
                .getLoadBalancerContext(serviceId);
        RibbonStatsRecorder statsRecorder = new RibbonStatsRecorder(context, server);

        try {
            /*
             * request是LoadBalancerRequest接口，它里面提供了一个apply方法
             * request对象是从LoadBalancerInterceptor的intercept方法中传递过来的
             */
            T returnVal = request.apply(serviceInstance);
            // 记录请求状态
            statsRecorder.recordStats(returnVal);
            return returnVal;
        }
        // catch IOException and rethrow so RestTemplate behaves correctly
        catch (IOException ex) {
            // 记录请求状态
            statsRecorder.recordStats(ex);
            throw ex;
        }
        catch (Exception ex) {
            // 记录请求状态
            statsRecorder.recordStats(ex);
            ReflectionUtils.rethrowRuntimeException(ex);
        }
        return null;
    }

    protected ILoadBalancer getLoadBalancer(String serviceId) {
        return this.clientFactory.getLoadBalancer(serviceId);
    }

    // 获得一个具体的服务节点
    protected Server getServer(ILoadBalancer loadBalancer, Object hint) {
        if (loadBalancer == null) {
            return null;
        }
        // 实际调用了 ILoadBalancer.chooseServer 方法
        return loadBalancer.chooseServer(hint != null ? hint : "default");
    }
}
```

看一下 ILoadBalancer 接口：
```java
public interface ILoadBalancer {
    /* 向负载均衡器中维护的实例列表增加服务实例 */
    void addServers(List<Server> var1);

    /* 过某种策略，从负载均衡服务器中挑选出一个具体的服务实例 */
    Server chooseServer(Object var1);

    /* 用来通知和标识负载均衡器中某个具体实例已经停止服务，否则负载均衡器在下一次获取服务实例清单前都会认为这个服务实例是正常工作的 */
    void markServerDown(Server var1);

    /* 获取当前正常工作的服务实例列表 */
    List<Server> getReachableServers();

    /* 获取所有的服务实例列表，包括正常的服务和停止工作的服务 */
    List<Server> getAllServers();
}
```

通过分析 ILoadBalancer 的实现类及其子类可以得出：
- AbstractLoadBalancer 实现了 ILoadBalancer 接口，它定义了服务分组的枚举类、chooseServer（用来选取一个服务实例）、
getServerList（获取某一个分组中的所有服务实例）、getLoadBalancerStats用来获得一个LoadBalancerStats对象，
这个对象保存了每一个服务的状态信息。
- BaseLoadBalancer，它实现了作为负载均衡器的基本功能，比如服务列表维护、服务存活状态监测、负载均衡算法选择 Server 等。
但是它只是完成基本功能，在有些复杂场景中还无法实现，比如动态服务列表、Server过滤、Zone区域意识
（服务之间的调用希望尽可能是在同一个区域内进行，减少延迟）。
- DynamicServerListLoadBalancer 是 BaseLoadbalancer 的一个子类，它对基础负载均衡提供了扩展，
从名字上可以看出，它提供了动态服务列表的特性。
- ZoneAwareLoadBalancer 它是在 DynamicServerListLoadBalancer 的基础上，增加了以 Zone 的形式来配置多个 LoadBalancer 的功能。

那在 RibbonLoadBalancerClient.getServer 方法中，loadBalancer.chooseServer 具体的实现类是哪一个呢？
我们找到 RibbonClientConfiguration 这个类：
```java
@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties
@Import({ HttpClientConfiguration.class, OkHttpRibbonConfiguration.class,
		RestClientRibbonConfiguration.class, HttpClientRibbonConfiguration.class })
public class RibbonClientConfiguration {

    /*
     * 如果没有自定义 ILoadBalancer，则直接返回一个 ZoneAwareLoadBalancer
     */
    @Bean
    @ConditionalOnMissingBean
    public ILoadBalancer ribbonLoadBalancer(IClientConfig config,
            ServerList<Server> serverList, ServerListFilter<Server> serverListFilter,
            IRule rule, IPing ping, ServerListUpdater serverListUpdater) {
        if (this.propertiesFactory.isSet(ILoadBalancer.class, name)) {
            return this.propertiesFactory.get(ILoadBalancer.class, config, name);
        }
        return new ZoneAwareLoadBalancer<>(config, rule, ping, serverList,
                serverListFilter, serverListUpdater);
    }

    /* 声明 IRule 的实现为 ZoneAvoidanceRule */
    @Bean
    @ConditionalOnMissingBean
    public IRule ribbonRule(IClientConfig config) {
        if (this.propertiesFactory.isSet(IRule.class, name)) {
            return this.propertiesFactory.get(IRule.class, config, name);
        }
        ZoneAvoidanceRule rule = new ZoneAvoidanceRule();
        rule.initWithNiwsConfig(config);
        return rule;
    }
}
```

ZoneAwareLoadBalancer 类：

Zone 表示区域的意思，区域指的就是地理区域的概念，一般较大规模的互联网公司，都会做跨区域部署，这样做有几个好处，
第一个是为不同地域的用户提供最近的访问节点减少访问延迟，其次是为了保证高可用，做容灾处理。

而 ZoneAwareLoadBalancer 就是提供了具备区域意识的负载均衡器，它的主要作用是对 Zone 进行了感知，
保证每个 Zone 里面的负载均衡策略都是隔离的，它并不保证 A 区域过来的请求一定会发动到 A 区域对应的 Server 内。
真正实现这个需求的是 ZonePreferenceServerListFilter/ZoneAffinityServerListFilter。

ZoneAwareLoadBalancer 的核心功能是：
- 若开启了区域意识，且 zone 的个数 > 1，就继续区域选择逻辑。
- 根据 ZoneAvoidanceRule.getAvailableZones() 方法拿到可用区们（会剔除掉完全不可用的区域们，以及可用但是负载最高的一个区域）。
- 从可用区 zone 们中，通过 ZoneAvoidanceRule.randomChooseZone 随机选一个 zone 出来 
（该随机遵从权重规则：谁的 zone 里面 Server 数量最多，被选中的概率越大）。
- 在选中的 zone 里面的所有 Server 中，采用该 zone 对对应的 Rule，进行 choose。

```java
public class ZoneAwareLoadBalancer<T extends Server> extends DynamicServerListLoadBalancer<T> {
    @Override
    public Server chooseServer(Object key) {
        //ENABLED，表示是否用区域意识的choose选择Server，默认是true，
        //如果禁用了区域、或者只有一个zone，就直接按照父类的逻辑来进行处理，父类默认采用轮询算法
        if (!ENABLED.get() || getLoadBalancerStats().getAvailableZones().size() <= 1) {
            logger.debug("Zone aware logic disabled or there is only one zone");
            return super.chooseServer(key);
        }
        Server server = null;
        try {
            LoadBalancerStats lbStats = getLoadBalancerStats();
            Map<String, ZoneSnapshot> zoneSnapshot = ZoneAvoidanceRule.createSnapshot(lbStats);
            logger.debug("Zone snapshots: {}", zoneSnapshot);
            if (triggeringLoad == null) {
                triggeringLoad = DynamicPropertyFactory.getInstance().getDoubleProperty(
                    "ZoneAwareNIWSDiscoveryLoadBalancer." + this.getName() + ".triggeringLoadPerServerThreshold", 0.2d);
            }
    
            if (triggeringBlackoutPercentage == null) {
                triggeringBlackoutPercentage = DynamicPropertyFactory.getInstance().getDoubleProperty(
                    "ZoneAwareNIWSDiscoveryLoadBalancer." + this.getName() + ".avoidZoneWithBlackoutPercetage", 0.99999d);
            }
            //根据相关阈值计算可用区域
            Set<String> availableZones = ZoneAvoidanceRule.getAvailableZones(zoneSnapshot, triggeringLoad.get(), triggeringBlackoutPercentage.get());
            logger.debug("Available zones: {}", availableZones);
            if (availableZones != null &&  availableZones.size() < zoneSnapshot.keySet().size()) {
                //从可用区域中随机选择一个区域，zone里面的服务器节点越多，被选中的概率越大
                String zone = ZoneAvoidanceRule.randomChooseZone(zoneSnapshot, availableZones);
                logger.debug("Zone chosen: {}", zone);
                if (zone != null) {
                    //根据zone获得该zone中的LB，然后根据该Zone的负载均衡算法选择一个server
                    BaseLoadBalancer zoneLoadBalancer = getLoadBalancer(zone);
                    server = zoneLoadBalancer.chooseServer(key);
                }
            }
        } catch (Exception e) {
            logger.error("Error choosing server using zone aware logic for load balancer={}", name, e);
        }
        if (server != null) {
            return server;
        } else {
            logger.debug("Zone avoidance logic is not invoked.");
            return super.chooseServer(key);
        }
    }
}
```

假设我们现在没有使用多区域部署，那么负载策略会执行到 BaseLoadBalancer.chooseServer：
```java
public class BaseLoadBalancer extends AbstractLoadBalancer implements PrimeConnectionListener, IClientConfigAware {
    public Server chooseServer(Object key) {
        if (counter == null) {
            counter = createCounter();
        }
        counter.increment();
        if (rule == null) {
            return null;
        } else {
            try {
                return rule.choose(key);
            } catch (Exception e) {
                logger.warn("LoadBalancer [{}]:  Error choosing server for key {}", name, key, e);
                return null;
            }
        }
    }
}
```

在 RibbonClientConfiguration 声明了 IRule 的默认实现为 ZoneAvoidanceRule。
所以，在 BaseLoadBalancer.chooseServer 中调用 rule.choose(key) 实际会进入到 ZoneAvoidanceRule 的 choose 方法：
```java
public class ZoneAvoidanceRule extends PredicateBasedRule {
    @Override
    public Server choose(Object key) {
        // 获取负载均衡器
        ILoadBalancer lb = getLoadBalancer();
        // 通过该方法获取目标服务
        Optional<Server> server = getPredicate().chooseRoundRobinAfterFiltering(lb.getAllServers(), key);
        if (server.isPresent()) {
            return server.get();
        } else {
            return null;
        }       
    }

    /*
     * 从方法名称可以看出来，它是通过对目标服务集群通过过滤算法过滤一遍后，再使用轮询实现负载均衡
     */
    public Optional<Server> chooseRoundRobinAfterFiltering(List<Server> servers, Object loadBalancerKey) {
        List<Server> eligible = getEligibleServers(servers, loadBalancerKey);
        if (eligible.size() == 0) {
            return Optional.absent();
        }
        return Optional.of(eligible.get(incrementAndGetModulo(eligible.size())));
    }

    /*
     * 使用主过滤条件对所有实例过滤并返回过滤后的清单
     */
    @Override
    public List<Server> getEligibleServers(List<Server> servers, Object loadBalancerKey) {
        List<Server> result = super.getEligibleServers(servers, loadBalancerKey);
        
        // 按照fallbacks中存储的过滤器顺序进行过滤（此处就行先ZoneAvoidancePredicate然后AvailabilityPredicate）
        Iterator<AbstractServerPredicate> i = fallbacks.iterator();

        /*
         * 依次使用次过滤条件对主过滤条件的结果进行过滤
         * 不论是主过滤条件还是次过滤条件，都需要判断下面两个条件，//只要有一个条件符合，就不再过滤，将当前结果返回供线性轮询
         *     第1个条件：过滤后的实例总数>=最小过滤实例数（默认为1）
         *     第2个条件：过滤互的实例比例>最小过滤百分比（默认为0）
         */
        while (!(result.size() >= minimalFilteredServers && result.size() > (int) (servers.size() * minimalFilteredPercentage))
               && i.hasNext()) {
            AbstractServerPredicate predicate = i.next();
            result = predicate.getEligibleServers(servers, loadBalancerKey);
        }
        return result;
    }

}
```

之后的筛选代码还有很多，就不再一一列举，总的来说就是删选出可用的服务器，如果有多可就默认采用轮询的策略。
可以回到 RibbonLoadBalancerClient 继续 getServer 之后的逻辑。