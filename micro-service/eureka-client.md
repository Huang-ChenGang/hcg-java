### eureka客户端分析

这篇分析是建立在SpringBoot版本为2.3.7.RELEASE、SpringCloud版本为Hoxton.SR9之上，eureka版本为2.2.6。
一下所有代码部分并没有贴出全部源码，仅供分析eureka使用。

eureka位于SpringCloudNetflix模块下，该模块提供了服务注册与发现功能：
1. 可以使用声明性 Java 配置创建嵌入式 Eureka 服务器。
2. 可以注册 Eureka 实例，客户端可以使用 Spring 管理的 bean 发现实例。

作为 Eureka 客户端，首先要引入 Eureka 客户端依赖：
```gitignore
implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-client'
```

在[SpringBoot自动装配](/spring/springBoot-autoConfiguration.md)中介绍了 
SpringBoot 会自动加载 META-INF/spring.factories 文件中 org.springframework.boot.autoconfigure.EnableAutoConfiguration
指定的配置类，看一下 spring-cloud-starter-netflix-eureka-client 中的 spring.factories 文件：
```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.netflix.eureka.config.EurekaClientConfigServerAutoConfiguration,\
org.springframework.cloud.netflix.eureka.config.DiscoveryClientOptionalArgsConfiguration,\
org.springframework.cloud.netflix.eureka.EurekaClientAutoConfiguration,\
org.springframework.cloud.netflix.ribbon.eureka.RibbonEurekaAutoConfiguration,\
org.springframework.cloud.netflix.eureka.EurekaDiscoveryClientConfiguration,\
org.springframework.cloud.netflix.eureka.reactive.EurekaReactiveDiscoveryClientConfiguration,\
org.springframework.cloud.netflix.eureka.loadbalancer.LoadBalancerEurekaAutoConfiguration
```

看到 SpringBoot 启动时会自动加载 EurekaClientAutoConfiguration 类：
```java
public class EurekaClientAutoConfiguration {
    /*
     * 声明 EurekaAutoServiceRegistration
     * Eureka 自动注册类
     */
    @Bean
    @ConditionalOnBean(AutoServiceRegistrationProperties.class)
    @ConditionalOnProperty(
            value = "spring.cloud.service-registry.auto-registration.enabled",
            matchIfMissing = true)
    public EurekaAutoServiceRegistration eurekaAutoServiceRegistration(
            ApplicationContext context, EurekaServiceRegistry registry,
            EurekaRegistration registration) {
        return new EurekaAutoServiceRegistration(context, registry, registration);
    }

    @Configuration(proxyBeanMethods = false)
    @ConditionalOnMissingRefreshScope
    protected static class EurekaClientConfiguration {

        @Autowired
        private ApplicationContext context;

        @Autowired
        private AbstractDiscoveryClientOptionalArgs<?> optionalArgs;

        /*
         * 声明 EurekaClient，返回的是 CloudEurekaClient 实例
         * 负责和 EurekaServer 进行交互的客户端实现类
         */
        @Bean(destroyMethod = "shutdown")
        @ConditionalOnMissingBean(value = EurekaClient.class,
                search = SearchStrategy.CURRENT)
        public EurekaClient eurekaClient(ApplicationInfoManager manager,
                EurekaClientConfig config) {
            return new CloudEurekaClient(manager, config, this.optionalArgs,
                    this.context);
        }

        /*
         * 声明 ApplicationInfoManager
         */
        @Bean
        @ConditionalOnMissingBean(value = ApplicationInfoManager.class,
                search = SearchStrategy.CURRENT)
        public ApplicationInfoManager eurekaApplicationInfoManager(
                EurekaInstanceConfig config) {
            InstanceInfo instanceInfo = new InstanceInfoFactory().create(config);
            // 构造一个 ApplicationInfoManager 实例
            return new ApplicationInfoManager(config, instanceInfo);
        }

        /*
         * 声明 EurekaRegistration
         * 由 ApplicationInfoManager.listeners 追踪到这里
         */
        @Bean
        @ConditionalOnBean(AutoServiceRegistrationProperties.class)
        @ConditionalOnProperty(
                value = "spring.cloud.service-registry.auto-registration.enabled",
                matchIfMissing = true)
        public EurekaRegistration eurekaRegistration(EurekaClient eurekaClient,
                CloudEurekaInstanceConfig instanceConfig,
                ApplicationInfoManager applicationInfoManager, @Autowired(
                        required = false) ObjectProvider<HealthCheckHandler> healthCheckHandler) {
            return EurekaRegistration.builder(instanceConfig).with(applicationInfoManager)
                    .with(eurekaClient).with(healthCheckHandler).build();
        }

    }
}
```

可以看到在 EurekaClientAutoConfiguration 类中声明了 EurekaAutoServiceRegistration 类：
```java
/*
 * Eureka 自动服务注册
 *
 * SmartLifecycle：Lifecycle接口的扩展，
 * 用于需要以特定顺序在ApplicationContext刷新和关闭时启动和关闭自定义组件，这里是eurekaServer。
 * void start()：开启组件
 * boolean isAutoStartup()：用来设置当前组件的生命周期函数是否被自动回调
 * void stop()：关闭组件
 */
public class EurekaAutoServiceRegistration implements AutoServiceRegistration,
		SmartLifecycle, Ordered, SmartApplicationListener {

    /**
     * 1. 主要在该方法中启动任务或者其他异步服务，比如开启MQ接收消息
     * 2. 当上下文被刷新（所有对象已被实例化和初始化之后）时，将调用该方法
     * 默认生命周期处理器将检查每个SmartLifecycle对象的isAutoStartup()方法返回的布尔值。
     * 如果为“true”，则该方法会被调用，而不是等待显式调用自己的start()方法。
     */
    @Override
    public void start() {
        // only set the port if the nonSecurePort or securePort is 0 and this.port != 0
        if (this.port.get() != 0) {
            if (this.registration.getNonSecurePort() == 0) {
                this.registration.setNonSecurePort(this.port.get());
            }

            if (this.registration.getSecurePort() == 0 && this.registration.isSecure()) {
                this.registration.setSecurePort(this.port.get());
            }
        }

        // only initialize if nonSecurePort is greater than 0 and it isn't already running
        // because of containerPortInitializer below
        if (!this.running.get() && this.registration.getNonSecurePort() > 0) {
            // 实现服务注册
            this.serviceRegistry.register(this.registration);
            // 发布事件
            this.context.publishEvent(new InstanceRegisteredEvent<>(this,
                    this.registration.getInstanceConfig()));
            this.running.set(true);
        }
    }

    /**
     * 如果该`Lifecycle`类所在的上下文在调用`refresh`时,希望能够自己自动进行回调，则返回`true`,
     * false的值表明组件打算通过显式的start()调用来启动，类似于普通的Lifecycle实现。
     */
    @Override
    public boolean isAutoStartup() {
        return true;
    }
}
```

实现服务注册实际调用的是 EurekaServiceRegistry.register 方法：
```java
public class EurekaServiceRegistry implements ServiceRegistry<EurekaRegistration> {
    @Override
    public void register(EurekaRegistration reg) {
        maybeInitializeClient(reg);

        if (log.isInfoEnabled()) {
            log.info("Registering application "
                    + reg.getApplicationInfoManager().getInfo().getAppName()
                    + " with eureka with status "
                    + reg.getInstanceConfig().getInitialStatus());
        }

        // 设置当前实例的状态，一旦这个实例的状态发生变化，只要状态不是DOWN，那么就会被监听器监听并且执行服务注册
        reg.getApplicationInfoManager()
                .setInstanceStatus(reg.getInstanceConfig().getInitialStatus());

        // 设置健康检查的处理
        reg.getHealthCheckHandler().ifAvailable(healthCheckHandler -> reg
                .getEurekaClient().registerHealthCheck(healthCheckHandler));
    }
}
```

从上述代码来看，注册方法中并没有真正调用Eureka的方法去执行注册，而是仅仅设置了一个状态以及设置健康检查处理器。
我们继续看一下 reg.getApplicationInfoManager().setInstanceStatus 方法：
```java
@Singleton
public class ApplicationInfoManager {
    protected final Map<String, StatusChangeListener> listeners;

    /*
     * 通过追溯代码可以看到，listeners 是 ApplicationInfoManager 的属性，
     * ApplicationInfoManager 又是 EurekaServiceRegistry 中 EurekaRegistration 的属性，
     * EurekaRegistration 和 ApplicationInfoManager 都是在 EurekaClientAutoConfiguration 中声明的。
     * 可以看到声明 ApplicationInfoManager 时，实际调用的是 ApplicationInfoManager 的构造方法。
     */
    @Inject
    public ApplicationInfoManager(EurekaInstanceConfig config, InstanceInfo instanceInfo, OptionalArgs optionalArgs) {
        this.config = config;
        this.instanceInfo = instanceInfo;

        // 初始化了一个listeners对象，它是一个ConcurrentHashMap集合，但是初始化的时候，这个集合并没有赋值
        this.listeners = new ConcurrentHashMap<String, StatusChangeListener>();

        if (optionalArgs != null) {
            this.instanceStatusMapper = optionalArgs.getInstanceStatusMapper();
        } else {
            this.instanceStatusMapper = NO_OP_MAPPER;
        }

        // Hack to allow for getInstance() to use the DI'd ApplicationInfoManager
        instance = this;
    }

    public synchronized void setInstanceStatus(InstanceStatus status) {
        InstanceStatus next = instanceStatusMapper.map(status);
        if (next == null) {
            return;
        }

        InstanceStatus prev = instanceInfo.setStatus(next);
        if (prev != null) {
            for (StatusChangeListener listener : listeners.values()) {
                try {
                    /*
                     * 通过监听器来发布一个状态变更事件。ok，此时listener的实例是StatusChangeListener，
                     * 也就是调用StatusChangeListener的notify()方法。这个事件是触发一个服务状态变更，应该是有地方会监听这个事件，
                     * 然后基于这个事件去执行具体的注册操作。
                     */
                    listener.notify(new StatusChangeEvent(prev, next));
                } catch (Exception e) {
                    logger.warn("failed to notify listener: {}", listener.getId(), e);
                }
            }
        }
    }

    /*
     * 对 listeners 进行赋值的地方
     * 通过查看被调用的地方，发现只有 DiscoveryClient.initScheduledTasks 方法
     * 在 EurekaClientAutoConfiguration 这个自动配置类的静态内部类 EurekaClientConfiguration 中，
     * 通过 @Bean 注入了一个 CloudEurekaClient 实例，可以往上看 EurekaClientAutoConfiguration
     * CloudEurekaClient 的父类是 DiscoveryClient
     */
    public void registerStatusChangeListener(StatusChangeListener listener) {
        listeners.put(listener.getId(), listener);
    }
}
```

DiscoveryClient 类：
```java
@Singleton
public class DiscoveryClient implements EurekaClient {

    /*
     * EurekaClientAutoConfiguration 中声明 CloudEurekaClient 实例，最终可以调到这个方法
     */
    @Inject
    DiscoveryClient(ApplicationInfoManager applicationInfoManager, EurekaClientConfig config, AbstractDiscoveryClientOptionalArgs args,
                    Provider<BackupRegistry> backupRegistryProvider, EndpointRandomizer endpointRandomizer) {
        if (args != null) {
            this.healthCheckHandlerProvider = args.healthCheckHandlerProvider;
            this.healthCheckCallbackProvider = args.healthCheckCallbackProvider;
            this.eventListeners.addAll(args.getEventListeners());
            this.preRegistrationHandler = args.preRegistrationHandler;
        } else {
            this.healthCheckCallbackProvider = null;
            this.healthCheckHandlerProvider = null;
            this.preRegistrationHandler = null;
        }
        
        this.applicationInfoManager = applicationInfoManager;
        InstanceInfo myInfo = applicationInfoManager.getInfo();

        clientConfig = config;
        staticClientConfig = clientConfig;
        transportConfig = config.getTransportConfig();
        instanceInfo = myInfo;
        if (myInfo != null) {
            appPathIdentifier = instanceInfo.getAppName() + "/" + instanceInfo.getId();
        } else {
            logger.warn("Setting instanceInfo to a passed in null value");
        }

        this.backupRegistryProvider = backupRegistryProvider;
        this.endpointRandomizer = endpointRandomizer;
        this.urlRandomizer = new EndpointUtils.InstanceInfoBasedUrlRandomizer(instanceInfo);
        localRegionApps.set(new Applications());

        fetchRegistryGeneration = new AtomicLong(0);

        remoteRegionsToFetch = new AtomicReference<String>(clientConfig.fetchRegistryForRemoteRegions());
        remoteRegionsRef = new AtomicReference<>(remoteRegionsToFetch.get() == null ? null : remoteRegionsToFetch.get().split(","));
    
        // 是否要从 eureka server 上获取服务地址信息
        if (config.shouldFetchRegistry()) {
            this.registryStalenessMonitor = new ThresholdLevelsMetric(this, METRIC_REGISTRY_PREFIX + "lastUpdateSec_", new long[]{15L, 30L, 60L, 120L, 240L, 480L});
        } else {
            this.registryStalenessMonitor = ThresholdLevelsMetric.NO_OP_METRIC;
        }

        // 是否要注册到 eureka server 上
        if (config.shouldRegisterWithEureka()) {
            this.heartbeatStalenessMonitor = new ThresholdLevelsMetric(this, METRIC_REGISTRATION_PREFIX + "lastHeartbeatSec_", new long[]{15L, 30L, 60L, 120L, 240L, 480L});
        } else {
            this.heartbeatStalenessMonitor = ThresholdLevelsMetric.NO_OP_METRIC;
        }

        logger.info("Initializing Eureka in region {}", clientConfig.getRegion());

        // 如果不需要注册并且不需要更新服务地址
        if (!config.shouldRegisterWithEureka() && !config.shouldFetchRegistry()) {
            logger.info("Client configured to neither register nor query for data.");
            scheduler = null;
            heartbeatExecutor = null;
            cacheRefreshExecutor = null;
            eurekaTransport = null;
            instanceRegionChecker = new InstanceRegionChecker(new PropertyBasedAzToRegionMapper(config), clientConfig.getRegion());

            // This is a bit of hack to allow for existing code using DiscoveryManager.getInstance()
            // to work with DI'd DiscoveryClient
            DiscoveryManager.getInstance().setDiscoveryClient(this);
            DiscoveryManager.getInstance().setEurekaClientConfig(config);

            initTimestampMs = System.currentTimeMillis();
            initRegistrySize = this.getApplications().size();
            registrySize = initRegistrySize;
            logger.info("Discovery Client initialized at timestamp {} with initial instances count: {}",
                    initTimestampMs, initRegistrySize);

            return;  // no need to setup up an network tasks and we are done
        }

        try {
            // default size of 2 - 1 each for heartbeat and cacheRefresh
            // 构建一个延期执行的线程池
            scheduler = Executors.newScheduledThreadPool(2,
                    new ThreadFactoryBuilder()
                            .setNameFormat("DiscoveryClient-%d")
                            .setDaemon(true)
                            .build());

            // 处理心跳的线程池
            heartbeatExecutor = new ThreadPoolExecutor(
                    1, clientConfig.getHeartbeatExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
                    new SynchronousQueue<Runnable>(),
                    new ThreadFactoryBuilder()
                            .setNameFormat("DiscoveryClient-HeartbeatExecutor-%d")
                            .setDaemon(true)
                            .build()
            );  // use direct handoff

            // 处理缓存刷新的线程池
            cacheRefreshExecutor = new ThreadPoolExecutor(
                    1, clientConfig.getCacheRefreshExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
                    new SynchronousQueue<Runnable>(),
                    new ThreadFactoryBuilder()
                            .setNameFormat("DiscoveryClient-CacheRefreshExecutor-%d")
                            .setDaemon(true)
                            .build()
            );  // use direct handoff

            eurekaTransport = new EurekaTransport();
            scheduleServerEndpointTask(eurekaTransport, args);

            AzToRegionMapper azToRegionMapper;
            if (clientConfig.shouldUseDnsForFetchingServiceUrls()) {
                azToRegionMapper = new DNSBasedAzToRegionMapper(clientConfig);
            } else {
                azToRegionMapper = new PropertyBasedAzToRegionMapper(clientConfig);
            }
            if (null != remoteRegionsToFetch.get()) {
                azToRegionMapper.setRegionsToFetch(remoteRegionsToFetch.get().split(","));
            }
            instanceRegionChecker = new InstanceRegionChecker(azToRegionMapper, clientConfig.getRegion());
        } catch (Throwable e) {
            throw new RuntimeException("Failed to initialize DiscoveryClient!", e);
        }

        // 如果需要注册到Eureka server并且是开启了初始化的时候强制注册，则调用register()发起服务注册
        if (clientConfig.shouldFetchRegistry()) {
            try {
                // 从Eureka-Server中拉去注册地址信息
                boolean primaryFetchRegistryResult = fetchRegistry(false);
                if (!primaryFetchRegistryResult) {
                    logger.info("Initial registry fetch from primary servers failed");
                }
                
                // 从备用地址拉去服务注册信息
                boolean backupFetchRegistryResult = true;
                if (!primaryFetchRegistryResult && !fetchRegistryFromBackup()) {
                    backupFetchRegistryResult = false;
                    logger.info("Initial registry fetch from backup servers failed");
                }
    
                // 如果还是没有拉取到，并且配置了强制拉取注册表的话，就会抛异常
                if (!primaryFetchRegistryResult && !backupFetchRegistryResult && clientConfig.shouldEnforceFetchRegistryAtInit()) {
                    throw new IllegalStateException("Fetch registry error at startup. Initial fetch failed.");
                }
            } catch (Throwable th) {
                logger.error("Fetch registry error at startup: {}", th.getMessage());
                throw new IllegalStateException(th);
            }
        }

        // call and execute the pre registration handler before all background tasks (inc registration) is started
        // 这里是判断一下有没有预注册处理器，有的话就执行一下
        if (this.preRegistrationHandler != null) {
            this.preRegistrationHandler.beforeRegistration();
        }

        // 如果需要注册到Eureka server并且是开启了初始化的时候强制注册，则调用register()发起服务注册(默认情况下，shouldEnforceRegistrationAtInit为false)
        if (clientConfig.shouldRegisterWithEureka() && clientConfig.shouldEnforceRegistrationAtInit()) {
            try {
                if (!register() ) {
                    throw new IllegalStateException("Registration error at startup. Invalid server response.");
                }
            } catch (Throwable th) {
                logger.error("Registration error at startup: {}", th.getMessage());
                throw new IllegalStateException(th);
            }
        }

        // finally, init the schedule tasks (e.g. cluster resolvers, heartbeat, instanceInfo replicator, fetch
        // 初始化一个定时任务，负责心跳、实例数据更新
        initScheduledTasks();

        try {
            Monitors.registerObject(this);
        } catch (Throwable e) {
            logger.warn("Cannot register timers", e);
        }

        // This is a bit of hack to allow for existing code using DiscoveryManager.getInstance()
        // to work with DI'd DiscoveryClient
        DiscoveryManager.getInstance().setDiscoveryClient(this);
        DiscoveryManager.getInstance().setEurekaClientConfig(config);

        initTimestampMs = System.currentTimeMillis();
        initRegistrySize = this.getApplications().size();
        registrySize = initRegistrySize;
        logger.info("Discovery Client initialized at timestamp {} with initial instances count: {}",
                initTimestampMs, initRegistrySize);
    }

    private void initScheduledTasks() {
        // 如果配置了开启从注册中心刷新服务列表，则会开启cacheRefreshExecutor这个定时任务
        if (clientConfig.shouldFetchRegistry()) {
            // registry cache refresh timer
            int registryFetchIntervalSeconds = clientConfig.getRegistryFetchIntervalSeconds();
            int expBackOffBound = clientConfig.getCacheRefreshExecutorExponentialBackOffBound();
            cacheRefreshTask = new TimedSupervisorTask(
                    "cacheRefresh",
                    scheduler,
                    cacheRefreshExecutor,
                    registryFetchIntervalSeconds,
                    TimeUnit.SECONDS,
                    expBackOffBound,
                    new CacheRefreshThread()
            );
            scheduler.schedule(
                    cacheRefreshTask,
                    registryFetchIntervalSeconds, TimeUnit.SECONDS);
        }

        // 如果开启了服务注册到Eureka
        if (clientConfig.shouldRegisterWithEureka()) {
            int renewalIntervalInSecs = instanceInfo.getLeaseInfo().getRenewalIntervalInSecs();
            int expBackOffBound = clientConfig.getHeartbeatExecutorExponentialBackOffBound();
            logger.info("Starting heartbeat executor: " + "renew interval is: {}", renewalIntervalInSecs);

            // 开启一个心跳任务
            heartbeatTask = new TimedSupervisorTask(
                    "heartbeat",
                    scheduler,
                    heartbeatExecutor,
                    renewalIntervalInSecs,
                    TimeUnit.SECONDS,
                    expBackOffBound,
                    new HeartbeatThread()
            );
            scheduler.schedule(
                    heartbeatTask,
                    renewalIntervalInSecs, TimeUnit.SECONDS);

            // 创建一个instanceInfoReplicator实例信息复制器
            instanceInfoReplicator = new InstanceInfoReplicator(
                    this,
                    instanceInfo,
                    clientConfig.getInstanceInfoReplicationIntervalSeconds(),
                    2); // burstSize

            // 初始化一个状态变更监听器
            statusChangeListener = new ApplicationInfoManager.StatusChangeListener() {
                @Override
                public String getId() {
                    return "statusChangeListener";
                }

                @Override
                public void notify(StatusChangeEvent statusChangeEvent) {
                    if (statusChangeEvent.getStatus() == InstanceStatus.DOWN) {
                        logger.error("Saw local status change event {}", statusChangeEvent);
                    } else {
                        logger.info("Saw local status change event {}", statusChangeEvent);
                    }
                    // 当服务实例状态发生变化时，实际执行的是这里
                    instanceInfoReplicator.onDemandUpdate();
                }
            };

            // 注册实例状态变化的监听
            if (clientConfig.shouldOnDemandUpdateStatusChange()) {
                /*
                 * 注册一个StatusChangeListener，保存到ApplicationInfoManager中的listener集合中。
                 * 通过之前的代码分析，我们知道当服务器启动或者停止时，会调用ApplicationInfoManager.listener，
                 * 逐个遍历调用listener.notify方法，而这个listener集合中的对象是在这里，也就是 DiscoveryClient 初始化的时候完成的。
                 */
                applicationInfoManager.registerStatusChangeListener(statusChangeListener);
            }

            // 启动一个实例信息复制器，主要就是为了开启一个定时线程，每40秒判断实例信息是否变更，如果变更了则重新注册
            instanceInfoReplicator.start(clientConfig.getInitialInstanceInfoReplicationIntervalSeconds());
        } else {
            logger.info("Not registering with Eureka server per configuration");
        }
    }

    /*
     * 实际进行服务注册的地方，又调用了 AbstractJerseyEurekaHttpClient.register 方法，这里就不再贴代码了
     * 很显然，这里是发起了一次http请求，访问Eureka-Server的apps/${APP_NAME}接口，将当前服务实例的信息发送到Eureka Server进行保存
     */
    boolean register() throws Throwable {
        logger.info(PREFIX + "{}: registering service...", appPathIdentifier);
        EurekaHttpResponse<Void> httpResponse;
        try {
            httpResponse = eurekaTransport.registrationClient.register(instanceInfo);
        } catch (Exception e) {
            logger.warn(PREFIX + "{} - registration failed {}", appPathIdentifier, e.getMessage(), e);
            throw e;
        }
        if (logger.isInfoEnabled()) {
            logger.info(PREFIX + "{} - registration status: {}", appPathIdentifier, httpResponse.getStatusCode());
        }
        return httpResponse.getStatusCode() == Status.NO_CONTENT.getStatusCode();
    }
}
```

有以上分析可以看出，当服务实例状态发生变化时，实际执行的是 InstanceInfoReplicator.onDemandUpdate 方法
```java
class InstanceInfoReplicator implements Runnable {
    public boolean onDemandUpdate() {
        // 限流判断
        if (rateLimiter.acquire(burstSize, allowedRatePerMinute)) {
            if (!scheduler.isShutdown()) {
                // 提交一个任务
                scheduler.submit(new Runnable() {
                    @Override
                    public void run() {
                        logger.debug("Executing on-demand update of local InstanceInfo");
    
                        // 取出之前已经提交的任务，也就是在start方法中提交的更新任务，如果任务还没有执行完成，则取消之前的任务。
                        Future latestPeriodic = scheduledPeriodicRef.get();
                        if (latestPeriodic != null && !latestPeriodic.isDone()) {
                            logger.debug("Canceling the latest scheduled update, it will be rescheduled at the end of on demand update");
                            // 如果此任务未完成，就立即取消
                            latestPeriodic.cancel(false);
                        }
    
                        // 通过调用run方法，令任务在延时后执行，相当于周期性任务中的一次
                        InstanceInfoReplicator.this.run();
                    }
                });
                return true;
            } else {
                logger.warn("Ignoring onDemand update due to stopped scheduler");
                return false;
            }
        } else {
            logger.warn("Ignoring onDemand update due to rate limiter");
            return false;
        }
    }

    public void run() {
        try {
            // 刷新实例信息
            discoveryClient.refreshInstanceInfo();
    
            // 是否有状态更新过了，有的话获取更新的时间
            Long dirtyTimestamp = instanceInfo.isDirtyWithTime();
            if (dirtyTimestamp != null) {
                // 有脏数据，要重新注册，可以看到，这里调用了 DiscoveryClient.register 方法，往上看
                discoveryClient.register();
                instanceInfo.unsetIsDirty(dirtyTimestamp);
            }
        } catch (Throwable t) {
            logger.warn("There was a problem with the instance info replicator", t);
        } finally {
            // 每隔30s，执行一次当前的`run`方法
            Future next = scheduler.schedule(this, replicationIntervalSeconds, TimeUnit.SECONDS);
            scheduledPeriodicRef.set(next);
        }
    }
}
```

### 服务注册总结

1. DiscoveryClient 这个对象，在初始化时，调用 initScheduledTask() 方法，构建一个 StatusChangeListener 监听。
2. Spring Cloud 应用在启动时，基于 SmartLifeCycle 接口回调，触发 StatusChangeListener 事件通知。
3. 在 StatusChangeListener 的回调方法中，通过调用 onDemandUpdate 方法，去更新客户端的地址信息，从而完成服务注册。