## 服务注册与发现-eureka

这篇分析是建立在SpringBoot版本为2.3.7.RELEASE、SpringCloud版本为Hoxton.SR9之上，eureka版本为2.2.6。
一下所有代码部分并没有贴出全部源码，仅供分析eureka使用。

eureka位于SpringCloudNetflix模块下，该模块提供了服务注册与发现功能：
1. 可以使用声明性 Java 配置创建嵌入式 Eureka 服务器。
2. 可以注册 Eureka 实例，客户端可以使用 Spring 管理的 bean 发现实例。

### eureka服务端分析

创建一个eureka-server，启动类如下：
```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

// 声明当前应用为一个eurekaServer
@EnableEurekaServer
// 声明当前应用为一个SpringBoot应用
@SpringBootApplication
public class EurekaApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaApplication.class, args);
    }

}
```

我们看到，只要一个@EnableEurekaServer注解就可以创建一个eureka服务器。
再看一下EnableEurekaServer：
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(EurekaServerMarkerConfiguration.class)
public @interface EnableEurekaServer {

}
```

EnableEurekaServer注解中引入了一个EurekaServerMarkerConfiguration，接着看一下：
```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * 负责添加标记以激活EurekaServerAutoConfiguration
 *
 * Configuration注解：
 * 官方文档：表示一个类声明了一个或多个@Bean 方法，并且可能会被 Spring 容器处理以在运行时为这些 bean 生成 bean 定义和服务请求。
 * 可以理解为一个添加Configuration注解的类，是用来声明可以被Spring加载的Bean或服务的
 * proxyBeanMethods为true：声明的Bean允许动态代理拦截
 * proxyBeanMethods为false：声明的Bean不允许动态代理拦截
 */
@Configuration(proxyBeanMethods = false)
public class EurekaServerMarkerConfiguration {

	@Bean
	public Marker eurekaServerMarkerBean() {
		return new Marker();
	}

	class Marker {

	}

}
```

既然是用来激活EurekaServerAutoConfiguration，再来看一下EurekaServerAutoConfiguration：
```java
// 在当前类声明Bean，并且声明的Bean不允许动态代理拦截
@Configuration(proxyBeanMethods = false)
// 引入EurekaServerInitializerConfiguration，下面有分析说明
@Import(EurekaServerInitializerConfiguration.class)
// 对应EurekaServerMarkerConfiguration中的Marker，因为Marker已经存在，所以Spring会实例化EurekaServerAutoConfiguration
@ConditionalOnBean(EurekaServerMarkerConfiguration.Marker.class)
// 添加EurekaDashboard和实例注册的一些配置
@EnableConfigurationProperties({ EurekaDashboardProperties.class,
		InstanceRegistryProperties.class })
// 加载自定义配置
@PropertySource("classpath:/eureka/server.properties")
/*
 * 实现WebMvcConfigurer：以JavaBean的形式提供SpringMVC的配置
 */
public class EurekaServerAutoConfiguration implements WebMvcConfigurer {

    /**
     * 初始化 Eureka Server 注册所需信息并被其他组件发现的类
     */
    @Autowired
    private ApplicationInfoManager applicationInfoManager;

    /*
     * eureka服务器运行所需的配置信息
     * 包括弹性IP、自我保护机制、从eureka接待同步注册列表等一些机制相关的配置
     * 细节先不看，先知道这里是eureka服务器运行所需的配置信息
     */
    @Autowired
    private EurekaServerConfig eurekaServerConfig;

    // eureka 客户端向 Eureka 服务器注册实例所需的配置信息
    @Autowired
    private EurekaClientConfig eurekaClientConfig;

    // eurekaClient
    @Autowired
    private EurekaClient eurekaClient;

    // 注册中心配置
    @Autowired
    private InstanceRegistryProperties instanceRegistryProperties;

    /**
     * 一个用于json和xml的编解码器
     * 会从eurekaServerConfig中读取jsonCodecName和xmlCodecName
     */
    @Bean
    public ServerCodecs serverCodecs() {
        return new CloudServerCodecs(this.eurekaServerConfig);
    }

    /**
     * 声明一个注册中心，这个暂时先不看，我们先把代码走下去
     * 粗略看了下，eureka的注册和同步应该都是在PeerAwareInstanceRegistry中进行的
     * 可以看出返回的是InstanceRegistry，InstanceRegistry又继承自PeerAwareInstanceRegistryImpl
     * PeerAwareInstanceRegistryImpl实现了PeerAwareInstanceRegistry
     * 所以注册中心基本的处理逻辑在InstanceRegistry和PeerAwareInstanceRegistryImpl
     */
    @Bean
    public PeerAwareInstanceRegistry peerAwareInstanceRegistry(
            ServerCodecs serverCodecs) {
        this.eurekaClient.getApplications(); // force initialization
        return new InstanceRegistry(this.eurekaServerConfig, this.eurekaClientConfig,
                serverCodecs, this.eurekaClient,
                this.instanceRegistryProperties.getExpectedNumberOfClientsSendingRenews(),
                this.instanceRegistryProperties.getDefaultOpenForTrafficCount());
    }

    /**
     * 由LinkedHashSet存储的一组过滤器
     */
    @Bean
    @ConditionalOnMissingBean
    public ReplicationClientAdditionalFilters replicationClientAdditionalFilters() {
        return new ReplicationClientAdditionalFilters(Collections.emptySet());
    }

    /**
     * 管理PeerEurekaNode的助手类
     */
    @Bean
    @ConditionalOnMissingBean
    public PeerEurekaNodes peerEurekaNodes(PeerAwareInstanceRegistry registry,
            ServerCodecs serverCodecs,
            ReplicationClientAdditionalFilters replicationClientAdditionalFilters) {
        return new RefreshablePeerEurekaNodes(registry, this.eurekaServerConfig,
                this.eurekaClientConfig, serverCodecs, this.applicationInfoManager,
                replicationClientAdditionalFilters);
    }

    /**
     * 声明 eurekaServerContext，传进去了四个参数，先看一下这四个参数都是什么，再总结eurekaServerContext都有什么
     * serverCodecs：一个用于json和xml的编解码器
     * registry：对等节点注册中心，无主从
     * peerEurekaNodes：管理eureka节点的助手类
     * applicationInfoManager：初始化 Eureka Server 注册所需信息并被其他组件发现的类
     *
     * DefaultEurekaServerContext：表示本地服务器上下文并将 getter 暴露给本地服务器的组件，例如注册表。
     */
    @Bean
    @ConditionalOnMissingBean // 当eurekaServerContext不存在时才会进行实例化
    public EurekaServerContext eurekaServerContext(ServerCodecs serverCodecs,
            PeerAwareInstanceRegistry registry, PeerEurekaNodes peerEurekaNodes) {
        return new DefaultEurekaServerContext(this.eurekaServerConfig, serverCodecs,
                registry, peerEurekaNodes, this.applicationInfoManager);
    }
    
    /**
     * 声明eurekaServerBootstrap，用于初始化并启动了eureka服务器
     */
    @Bean
    public EurekaServerBootstrap eurekaServerBootstrap(PeerAwareInstanceRegistry registry,
            EurekaServerContext serverContext) {
        return new EurekaServerBootstrap(this.applicationInfoManager,
                this.eurekaClientConfig, this.eurekaServerConfig, registry,
                serverContext);
    }

}
```

看到EurekaServerAutoConfiguration中引入了EurekaServerInitializerConfiguration，看一下：
```java
// 在当前类声明Bean，并且声明的Bean不允许动态代理拦截
@Configuration(proxyBeanMethods = false)
/*
 * 看名字意思是：eureka服务器初始化配置
 *
 * 实现ServletContextAware：
 * void setServletContext(ServletContext servletContext)：在本类中获取到servletContext
 *
 * SmartLifecycle：Lifecycle接口的扩展，
 * 用于需要以特定顺序在ApplicationContext刷新和关闭时启动和关闭自定义组件，这里是eurekaServer。
 * void start()：开启组件
 * boolean isAutoStartup()：用来设置当前组件的生命周期函数是否被自动回调
 * void stop()：关闭组件
 *
 * Ordered：表明当前组件可以被排序，可以理解为优先级，数值越小，优先级越高
 */
public class EurekaServerInitializerConfiguration
		implements ServletContextAware, SmartLifecycle, Ordered {

	private static final Log log = LogFactory
			.getLog(EurekaServerInitializerConfiguration.class);

	@Autowired
	private EurekaServerConfig eurekaServerConfig;

	private ServletContext servletContext;

	@Autowired
	private ApplicationContext applicationContext;

	@Autowired
	private EurekaServerBootstrap eurekaServerBootstrap;

	private boolean running;

	private int order = 1;

    /**
     * 实现ServletContextAware
     * 在本类中获取到servletContext
     */
	@Override
	public void setServletContext(ServletContext servletContext) {
		this.servletContext = servletContext;
	}

    /**
     * 1. 主要在该方法中启动任务或者其他异步服务，比如开启MQ接收消息
     * 2. 当上下文被刷新（所有对象已被实例化和初始化之后）时，将调用该方法
     * 默认生命周期处理器将检查每个SmartLifecycle对象的isAutoStartup()方法返回的布尔值。
     * 如果为“true”，则该方法会被调用，而不是等待显式调用自己的start()方法。
     */
	@Override
	public void start() {
        // 新启一个线程
		new Thread(() -> {
			try {
                /*
                 * 初始化并启动了eureka服务器
                 * eurekaServerBootstrap实例是在EurekaServerAutoConfiguration中声明的，可以回到上边看一下
                 */
				eurekaServerBootstrap.contextInitialized(
						EurekaServerInitializerConfiguration.this.servletContext);
				log.info("Started Eureka Server");
        
				// 发布注册中心启动事件，这里并没有具体的监听器，可以理解为SpringCloud提供回调接口让开发者实现自己的业务逻辑
				publish(new EurekaRegistryAvailableEvent(getEurekaServerConfig()));
                // 设置eureka服务器状态为已启动
				EurekaServerInitializerConfiguration.this.running = true;
                // 发布Eureka服务器启动事件，这里并没有具体的监听器，可以理解为SpringCloud提供回调接口让开发者实现自己的业务逻辑
				publish(new EurekaServerStartedEvent(getEurekaServerConfig()));
			}
			catch (Exception ex) {
				// Help!
				log.error("Could not initialize Eureka servlet context", ex);
			}
		}).start();
	}

	private EurekaServerConfig getEurekaServerConfig() {
		return this.eurekaServerConfig;
	}

	private void publish(ApplicationEvent event) {
		this.applicationContext.publishEvent(event);
	}

    /**
     * 停止自定义的组件，在这里是停止eurekaServer
     */
	@Override
	public void stop() {
		this.running = false;
		eurekaServerBootstrap.contextDestroyed(this.servletContext);
	}

    /**
     * 返回自定义组件的运行状态
     */
	@Override
	public boolean isRunning() {
		return this.running;
	}

    /**
     * 返回自定义组件的执行顺序
     * 当有多个实现SmartLifecycle的组件时，phase越小越先start
     * stop时相反，phase越大越先stop
     */
	@Override
	public int getPhase() {
		return 0;
	}

    /**
     * 如果该`Lifecycle`类所在的上下文在调用`refresh`时,希望能够自己自动进行回调，则返回`true`,
     * false的值表明组件打算通过显式的start()调用来启动，类似于普通的Lifecycle实现。
     */
	@Override
	public boolean isAutoStartup() {
		return true;
	}

    /**
     * 有回调函数的停止组件
     */
	@Override
	public void stop(Runnable callback) {
		callback.run();
	}

    /**
     * 返回当前组件的优先级，数值越小，优先级越高
     */
	@Override
	public int getOrder() {
		return this.order;
	}

}
```

eureka服务器是在EurekaServerBootstrap中初始化并启动了的，我们看一下：
```java
/**
 * eureka服务器引导程序
 */
public class EurekaServerBootstrap {

    protected EurekaServerConfig eurekaServerConfig;
    
    protected ApplicationInfoManager applicationInfoManager;

    protected EurekaClientConfig eurekaClientConfig;

    protected PeerAwareInstanceRegistry registry;

    protected volatile EurekaServerContext serverContext;

    public EurekaServerBootstrap(ApplicationInfoManager applicationInfoManager,
            EurekaClientConfig eurekaClientConfig, EurekaServerConfig eurekaServerConfig,
            PeerAwareInstanceRegistry registry, EurekaServerContext serverContext) {
        this.applicationInfoManager = applicationInfoManager;
        this.eurekaClientConfig = eurekaClientConfig;
        this.eurekaServerConfig = eurekaServerConfig;
        this.registry = registry;
        this.serverContext = serverContext;
    }

    /**
     * 初始化并启动了eureka服务器
     */
    public void contextInitialized(ServletContext context) {
        try {
            initEurekaEnvironment();
            initEurekaServerContext();

            context.setAttribute(EurekaServerContext.class.getName(), this.serverContext);
        }
        catch (Throwable e) {
            log.error("Cannot bootstrap eureka server :", e);
            throw new RuntimeException("Cannot bootstrap eureka server :", e);
        }
    }

    protected void initEurekaEnvironment() throws Exception {
        log.info("Setting the eureka configuration..");
    
        // ConfigurationManager.getConfigInstance()可以理解为是一个获取和设置系统配置的实例
        String dataCenter = ConfigurationManager.getConfigInstance()
                .getString(EUREKA_DATACENTER);
        if (dataCenter == null) {
            // 通过eurekaServer的启动日志看到这句有输出，所以配置项 ARCHAIUS_DEPLOYMENT_DATACENTER 设置为了 DEFAULT
            log.info(
                    "Eureka data center value eureka.datacenter is not set, defaulting to default");
            ConfigurationManager.getConfigInstance()
                    .setProperty(ARCHAIUS_DEPLOYMENT_DATACENTER, DEFAULT);
        }
        else {
            ConfigurationManager.getConfigInstance()
                    .setProperty(ARCHAIUS_DEPLOYMENT_DATACENTER, dataCenter);
        }
        String environment = ConfigurationManager.getConfigInstance()
                .getString(EUREKA_ENVIRONMENT);
        if (environment == null) {
            ConfigurationManager.getConfigInstance()
                    .setProperty(ARCHAIUS_DEPLOYMENT_ENVIRONMENT, TEST);
            // 通过eurekaServer的启动日志看到这句有输出，所以配置项 ARCHAIUS_DEPLOYMENT_ENVIRONMENT 设置为了 TEST
            log.info(
                    "Eureka environment value eureka.environment is not set, defaulting to test");
        }
        else {
            ConfigurationManager.getConfigInstance()
                    .setProperty(ARCHAIUS_DEPLOYMENT_ENVIRONMENT, environment);
        }
    }

    protected void initEurekaServerContext() throws Exception {
        // 设置json和xml序列化转换器和优先级，源码中注释是为了向后兼容，可以先不看
        JsonXStream.getInstance().registerConverter(new V1AwareInstanceInfoConverter(),
                XStream.PRIORITY_VERY_HIGH);
        XmlXStream.getInstance().registerConverter(new V1AwareInstanceInfoConverter(),
                XStream.PRIORITY_VERY_HIGH);
    
        // 应该是为了支持AWS，启动日志里有"isAws returned false"，这里可以先跳过
        if (isAws(this.applicationInfoManager.getInfo())) {
            this.awsBinder = new AwsBinderDelegate(this.eurekaServerConfig,
                    this.eurekaClientConfig, this.registry, this.applicationInfoManager);
            this.awsBinder.start();
        }

        /*
         * 初始化eurekaServer上下文
         * serverContext是在EurekaServerAutoConfiguration声明并传进来的
         * 这里看起来只是把EurekaServerContext交给EurekaServerContextHolder
         */
        EurekaServerContextHolder.initialize(this.serverContext);

        log.info("Initialized server context");

        // 从其他eureka节点同步注册表信息，具体执行方法在PeerAwareInstanceRegistryImpl类里
        int registryCount = this.registry.syncUp();
        this.registry.openForTraffic(this.applicationInfoManager, registryCount);

        // Register all monitoring statistics.
        EurekaMonitors.registerAllStats();
    }
}
```

注册中心实现代码PeerAwareInstanceRegistryImpl：
```java
/*
 * 将所有操作复制到 AbstractInstanceRegistry 到对等 Eureka 节点，以保持它们同步
 * 复制的主要操作是注册、续订、取消、到期和状态更改
 *
 * 当 eureka 服务器启动时，它会尝试从对等 eureka 节点获取所有注册表信息。
 * 如果由于某种原因此操作失败，则服务器不允许用户在指定的时间段内获取注册表信息
 * 指定时间段：EurekaServerConfig#getWaitTimeInMsWhenSyncEmpty()
 * 配置项：eureka.waitTimeInMsWhenSyncEmpty，默认为5分钟(1000 * 60 * 5)
 *
 * 关于续订需要注意的一件重要事情。 
 * 如果在 {EurekaServerConfig#getRenewalThresholdUpdateIntervalMs()} 的时间内
 * 续订下降超过 {EurekaServerConfig#getRenewalPercentThreshold()} 中指定的阈值，
 * eureka 会感知到这一点 作为危险并停止过期实例。
 * getRenewalPercentThreshold配置项：eureka.renewalPercentThreshold，默认85%(0.85)
 * getRenewalThresholdUpdateIntervalMs配置项：eureka.renewalThresholdUpdateIntervalMs，默认15分钟(15 * 60 * 1000)
 */
@Singleton
public class PeerAwareInstanceRegistryImpl extends AbstractInstanceRegistry implements PeerAwareInstanceRegistry {
    
    // 从其他eureka节点同步注册表信息
    public int syncUp() {
        // 同步到的实例数量
        int count = 0;

        /*
         * 同步次数小于配置中的启动时尝试同步次数，并且没有同步过
         * 启动时尝试同步次数配置项：eureka.numberRegistrySyncRetries，默认5次
         */
        for (int i = 0; ((i < serverConfig.getRegistrySyncRetries()) && (count == 0)); i++) {
            if (i > 0) {
                try {
                    /*
                     * 每次同步之间间隔一定时间
                     * 同步间隔时间配置项：eureka.registrySyncRetryWaitMs，默认30秒(30 * 1000)
                     */
                    Thread.sleep(serverConfig.getRegistrySyncRetryWaitMs());
                } catch (InterruptedException e) {
                    logger.warn("Interrupted during registry transfer..");
                    break;
                }
            }
        
            // 循环获取eurekaClient中的应用实例
            Applications apps = eurekaClient.getApplications();
            for (Application app : apps.getRegisteredApplications()) {
                for (InstanceInfo instance : app.getInstances()) {
                    try {
                        // 是否可注册该实例 非AWS的全部可注册
                        if (isRegisterable(instance)) {
                            // register方法在AbstractInstanceRegistry类里
                            register(instance, instance.getLeaseInfo().getDurationInSecs(), true);
                            count++;
                        }
                    } catch (Throwable t) {
                        logger.error("During DS init copy", t);
                    }
                }
            }
        }
        return count;
    }

    public void openForTraffic(ApplicationInfoManager applicationInfoManager, int count) {
        // 更新客户端发送续订的数量
        this.expectedNumberOfClientsSendingRenews = count;
        // 更新续订百分比最小临界值
        updateRenewsPerMinThreshold();
        logger.info("Got {} instances from neighboring DS node", count);
        logger.info("Renew threshold is: {}", numberOfRenewsPerMinThreshold);
        this.startupTime = System.currentTimeMillis();
        if (count > 0) {
            this.peerInstancesTransferEmptyOnStartup = false;
        }

        // 兼容AWS
        DataCenterInfo.Name selfName = applicationInfoManager.getInfo().getDataCenterInfo().getName();
        boolean isAws = Name.Amazon == selfName;
        if (isAws && serverConfig.shouldPrimeAwsReplicaConnections()) {
            logger.info("Priming AWS connections for all replicas..");
            primeAwsReplicas(applicationInfoManager);
        }
        logger.info("Changing status to UP");
        applicationInfoManager.setInstanceStatus(InstanceStatus.UP);

        // 启动清除定时任务
        super.postInit();
    }

    /**
     * 可以看出来这个方法只是为了AWS用的，非AWS的全部可注册
     */
    public boolean isRegisterable(InstanceInfo instanceInfo) {
        DataCenterInfo datacenterInfo = instanceInfo.getDataCenterInfo();
        String serverRegion = clientConfig.getRegion();
        if (AmazonInfo.class.isInstance(datacenterInfo)) {
            AmazonInfo info = AmazonInfo.class.cast(instanceInfo.getDataCenterInfo());
            String availabilityZone = info.get(MetaDataKey.availabilityZone);
            // Can be null for dev environments in non-AWS data center
            if (availabilityZone == null && US_EAST_1.equalsIgnoreCase(serverRegion)) {
                return true;
            } else if ((availabilityZone != null) && (availabilityZone.contains(serverRegion))) {
                // If in the same region as server, then consider it registerable
                return true;
            }
        }
        return true; // Everything non-amazon is registrable.
    }

}
```

AbstractInstanceRegistry：
```java
/**
 * 处理来自尤里卡客户端的所有注册请求
 * 执行的主要操作是注册、更新、取消、到期和状态更改
 */
public abstract class AbstractInstanceRegistry implements InstanceRegistry {
    // 注册表，可以看出注册表是通过ConcurrentHashMap实现的，key是应用的名称appName
    private final ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>> registry
                = new ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>>();

    // 环形队列，在这里仅用做调试和统计
    private final CircularQueue<Pair<Long, String>> recentRegisteredQueue;
    private final CircularQueue<Pair<Long, String>> recentCanceledQueue;
    private ConcurrentLinkedQueue<RecentlyChangedItem> recentlyChangedQueue = new ConcurrentLinkedQueue<RecentlyChangedItem>();

    private final ReentrantReadWriteLock readWriteLock = new ReentrantReadWriteLock();
    private final Lock read = readWriteLock.readLock();
    private final Lock write = readWriteLock.writeLock();
    protected final Object lock = new Object();

    // 预期的客户端发送更新的数量
    protected volatile int expectedNumberOfClientsSendingRenews;

    /**
     * 注册一个新实例
     */
    public void register(InstanceInfo registrant, int leaseDuration, boolean isReplication) {
        // 读锁上锁
        read.lock();
        try {
            // 从注册表中取出需要注册的应用的全部实例
            Map<String, Lease<InstanceInfo>> gMap = registry.get(registrant.getAppName());
            // 在eureka统计中，将注册的数量+1
            REGISTER.increment(isReplication);
            // 当前应用从来没有注册过
            if (gMap == null) {
                // 新建一个实例列表，这里可已看出注册表中应用所对应的实例列表也是通过ConcurrentHashMap实现的，key是实例ID
                final ConcurrentHashMap<String, Lease<InstanceInfo>> gNewMap = new ConcurrentHashMap<String, Lease<InstanceInfo>>();
                // 先调用putIfAbsent存一下，应该是为了防止多线程并发注册
                gMap = registry.putIfAbsent(registrant.getAppName(), gNewMap);
                // 当前应该没有被注册过，就把新建的实例列表赋值给gMap
                if (gMap == null) {
                    gMap = gNewMap;
                }
            }
    
            // 根据实例ID检查是否注册过
            Lease<InstanceInfo> existingLease = gMap.get(registrant.getId());
            // 当前实例ID已经注册过
            if (existingLease != null && (existingLease.getHolder() != null)) {
                Long existingLastDirtyTimestamp = existingLease.getHolder().getLastDirtyTimestamp();
                Long registrationLastDirtyTimestamp = registrant.getLastDirtyTimestamp();
                logger.debug("Existing lease found (existing={}, provided={}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);

                // 将本地已注册的实例和将要注册的实例进行比较，保留最后使用时间更新的那一个
                if (existingLastDirtyTimestamp > registrationLastDirtyTimestamp) {
                    logger.warn("There is an existing lease and the existing lease's dirty timestamp {} is greater" +
                            " than the one that is being registered {}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);
                    logger.warn("Using the existing instanceInfo instead of the new instanceInfo as the registrant");
                    registrant = existingLease.getHolder();
                }
            } else {
                // 当前实例ID没有注册过，加锁更新
                synchronized (lock) {
                    if (this.expectedNumberOfClientsSendingRenews > 0) {
                        // 更新客户端发送续订的数量
                        this.expectedNumberOfClientsSendingRenews = this.expectedNumberOfClientsSendingRenews + 1;
                        // 更新续订百分比最小临界值
                        updateRenewsPerMinThreshold();
                    }
                }
                logger.debug("No previous lease information found; it is new registration");
            }

            // 新建待注册实例，并更新实例启动时间
            Lease<InstanceInfo> lease = new Lease<InstanceInfo>(registrant, leaseDuration);
            if (existingLease != null) {
                lease.setServiceUpTimestamp(existingLease.getServiceUpTimestamp());
            }

            /*
             * 新实例注册进registry
             * gMap是从registry中取出的对象引用，对象引用就是堆中对象的存储地址，
             * 所以这里put进gMap是会实际影响到registry的，也就是保存进了registry
             */
            gMap.put(registrant.getId(), lease);

            recentRegisteredQueue.add(new Pair<Long, String>(
                    System.currentTimeMillis(),
                    registrant.getAppName() + "(" + registrant.getId() + ")"));

            // 更新覆盖状态，暂时没看懂什么用，先放一放
            if (!InstanceStatus.UNKNOWN.equals(registrant.getOverriddenStatus())) {
                logger.debug("Found overridden status {} for instance {}. Checking to see if needs to be add to the "
                                + "overrides", registrant.getOverriddenStatus(), registrant.getId());
                if (!overriddenInstanceStatusMap.containsKey(registrant.getId())) {
                    logger.info("Not found overridden id {} and hence adding it", registrant.getId());
                    overriddenInstanceStatusMap.put(registrant.getId(), registrant.getOverriddenStatus());
                }
            }
            InstanceStatus overriddenStatusFromMap = overriddenInstanceStatusMap.get(registrant.getId());
            if (overriddenStatusFromMap != null) {
                logger.info("Storing overridden status {} from map", overriddenStatusFromMap);
                registrant.setOverriddenStatus(overriddenStatusFromMap);
            }

            // Set the status based on the overridden status rules
            InstanceStatus overriddenInstanceStatus = getOverriddenInstanceStatus(registrant, existingLease, isReplication);
            registrant.setStatusWithoutDirty(overriddenInstanceStatus);

            // 如果实例是启动状态，更新实例的启动时间
            if (InstanceStatus.UP.equals(registrant.getStatus())) {
                lease.serviceUp();
            }

            registrant.setActionType(ActionType.ADDED);
            recentlyChangedQueue.add(new RecentlyChangedItem(lease));
            registrant.setLastUpdatedTimestamp();

            // 无效化当前应用的缓存 eureka在读取实际的注册表前设置了两层缓存，所以这里要把之前的缓存无效化
            invalidateCache(registrant.getAppName(), registrant.getVIPAddress(), registrant.getSecureVipAddress());
            logger.info("Registered instance {}/{} with status {} (replication={})",
                    registrant.getAppName(), registrant.getId(), registrant.getStatus(), isReplication);
        } finally {
            // 最后释放读锁
            read.unlock();
        }
    }

    // 更新续订百分比最小临界值
    protected void updateRenewsPerMinThreshold() {
        this.numberOfRenewsPerMinThreshold = (int) (this.expectedNumberOfClientsSendingRenews
                * (60.0 / serverConfig.getExpectedClientRenewalIntervalSeconds())
                * serverConfig.getRenewalPercentThreshold());
    }
}
```

eureka监控的统计信息EurekaMonitors：
```java
/*
 * 封装 Eureka 监控的所有统计信息的枚举
 */
public enum EurekaMonitors {
    // 注册统计枚举
    REGISTER("registerCounter", "Number of total registers seen since startup");

    // 总注册数
    @com.netflix.servo.annotations.Monitor(name = "count", type = DataSourceType.COUNTER)
    private final AtomicLong counter = new AtomicLong();

    // 当前eureka节点注册数
    @com.netflix.servo.annotations.Monitor(name = "count-minus-replication", type = DataSourceType.COUNTER)
    private final AtomicLong myZoneCounter = new AtomicLong();

    /**
     * isReplication表示是否是从其他eureka节点同步来的
     */
    public void increment(boolean isReplication) {
        // 总注册数+1
        counter.incrementAndGet();

        // 如果不是从其他节点同步来的，则当前eureka节点注册数也要+1
        if (!isReplication) {
            myZoneCounter.incrementAndGet();
        }
    }
}
```