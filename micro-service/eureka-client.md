### eureka客户端分析

这篇分析是建立在SpringBoot版本为2.3.7.RELEASE、SpringCloud版本为Hoxton.SR9之上，eureka版本为2.2.6。
一下所有代码部分并没有贴出全部源码，仅供分析eureka使用。

eureka位于SpringCloudNetflix模块下，该模块提供了服务注册与发现功能：
1. 可以使用声明性 Java 配置创建嵌入式 Eureka 服务器。
2. 可以注册 Eureka 实例，客户端可以使用 Spring 管理的 bean 发现实例。