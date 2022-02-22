## 熔断降级机制

Hystrix 是利用切面来拦截住我们所发起的一切对外请求，当拦截到请求产生异常的时候，面向于切面的方法就可以感知到。
主要是靠 AOP 的机制来拦截用户的请求。

熔断指的是在多次发生远程服务调用失败的时候，会将当前的服务置为不可用，暂时先屏蔽掉。接下来会在一段时间内调用其他服务，
当其他服务负载比较慢，或者屏蔽时间比较长的时候，会再尝试调用之前屏蔽掉的服务，这就是熔断机制。

降级服务是指在调用一些服务的时候，服务本质上不能再提供我们想要的数据，我们会准备一份降级数据来保底。
比如，我们想要获取一份最新的菜单列表，但是因为服务不可用，可以尝试从本地缓存里提供一份菜单列表。

### Hystrix 及其实现原理

Hystrix 是一个延迟和容错库，旨在隔离对远程系统、服务和第三方库的访问点，停止级联故障，并在不可避免发生故障的复杂分布式系统中实现快速恢复。
主要靠 Spring 的 AOP 实现。

正常情况下，断路器关闭，服务消费者正常请求微服务。
一段时间内，失败率达到一定阈值，断路器将断开，此时不再请求服务提供者，而是快速失败的方法（断路方法）。
断路器打开一段时间，自动进入"半开"状态，此时断路器允许一个请求去请求服务提供者，如果请求调用成功，则关闭断路器，否则继续保持断路器打开。

断路器 Hystrix 保证了局部发生的错误不会扩展到整个系统，从而保证系统即使出现局部问题也不会导致系统雪崩。

### Hystrix 降级使用

1. 首先引入 spring-cloud-starter-hystrix 组件。

2. 在项目启动类上加 @EnableCircuitBreaker 注解。

3. 在服务请求方的请求方法上加 @HystrixCommand 注解。@HystrixCommand 注解中的 fallbackMethod 属性指定降级方法。
在当前类中新建 fallbackMethod 指定的降级方法，返回值要跟正常的请求方法一致。

4. 可以通过 @HystrixCommand 的 commandProperties 属性来指定超时等设置。
commandProperties 是一个数组，每个数组元素是一个 @HystrixProperty，通过 name 和 value 来设置。
可以在 HystrixCommandProperties 类里查看所有配置。

### Hystrix 熔断使用

通过 @HystrixCommand 的 commandProperties 属性来设置。
commandProperties 是一个数组，每个数组元素是一个 @HystrixProperty，通过 name 和 value 来设置。
主要是 circuitBreaker 开头的一些设置：
- circuitBreaker.enable：设置为 true，表示开启断路器设置。
- circuitBreaker.requestVolumeThreshold：设置在滚动时间窗口中，断路器的最小请求数。
设置为 10 就表示请求达到 10 次后才进行错误百分比的计算。
- circuitBreaker.sleepWindowInMilliseconds：设置为 10000，表示断路器打开时间为 10 秒，在休眠时间内，降级逻辑会成为主逻辑。
当休眠时间到期，断路器关闭，释放一次请求到服务提供方，如果请求正常返回，那么断路器将继续关闭。如果这次请求还是有问题，断路器进入打开状态，
继续降级逻辑，休眠时间窗重新计时。
- circuitBreaker.errorThresholdPercentage：设置断路器打开的错误百分比条件。

### Hystrix 配置文件设置

在 @HystrixProperty 注解中设置的配置，都可以在 yaml 中设置（注意：使用配置也要加 @HystrixCommand 注解）：
- 超时的设置为：hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds。

### Feign 中 Hystrix 的实现

Feign 的依赖中已经集成了 Hystrix，需要在 yaml 文件中加个配置：
```properties
feign.hystrix.enabled = true
```
然后直接在 @FeignClient 注解的 fallback 属性中指定目标类即可。
目标类要实现使用 @FeignClient 注解的接口，并实现全部方法，在实现方法中做降级逻辑。

### Hystrix Dashboard

需要在项目里引入 spring-cloud-starter-hystrix-dashboard 依赖，