## 服务注册与发现-eureka

这篇分析是建立在SpringBoot版本为2.3.7.RELEASE、SpringCloud版本为Hoxton.SR9之上，eureka版本为2.2.6

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
// 添加EurekaDashboard和示例注册的一些配置
@EnableConfigurationProperties({ EurekaDashboardProperties.class,
		InstanceRegistryProperties.class })
// 加载自定义配置
@PropertySource("classpath:/eureka/server.properties")
/*
 * 实现WebMvcConfigurer：以JavaBean的形式提供SpringMVC的配置
 */
public class EurekaServerAutoConfiguration implements WebMvcConfigurer {
    
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