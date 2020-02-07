# 1.Spring Boot事件

| 序号 | 监听方法              | Spring Boot事件                     | 发生阶段说明                                                 |
| ---- | --------------------- | ----------------------------------- | ------------------------------------------------------------ |
| 1    | starting()            | ApplicationStartingEvent            | Spring 应用刚启动                                            |
| 2    | environmentPrepared() | ApplicationEnvironmentPreparedEvent | ConfigurableEnvironment 准备妥当，允许将其调整               |
| 3    | contextPrepared()     | ApplicationContextInitializedEvent  | ConfigurableApplicationContext 准备妥当，允许将其调整        |
| 4    | contextLoaded()       | ApplicationPreparedEvent            | ConfigurableApplicationContext 已装载，但仍未启动            |
| 5    | started()             | ApplicationStartedEvent             | ConfigurableApplicationContext 已启动，此时 Spring Bean 已初始化完成 |
| 6    | running()             | ApplicationReadyEvent               | Spring 应用正在运行                                          |
| 7    | failed()              | ApplicationFailedEvent              | Spring 应用运行失败                                          |

# 2.Spring Context事件

| 序号 | spring context事件    | 发生阶段说明            |
| ---- | --------------------- | ----------------------- |
| 1    | ContextStartedEvent   | Lifecycle.start()方法   |
| 2    | ContextRefreshedEvent | Spring Bean已初始化完成 |
| 3    | ContextClosedEvent    | Spring 容器关闭         |
| 4    | ContextStoppedEvent   | Lifecycle.stop()方法    |

