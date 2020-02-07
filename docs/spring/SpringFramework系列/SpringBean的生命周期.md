# 1.整体流程图

<img src="/images/Bean生命周期.png">

Spring Bean对象的可扩展性主要就是依靠InstantiationAwareBeanPostProcessor和BeanPostProcessor来实现的，其中重要的2个接口:

- InstantiationAwareBeanPostProcessor 主要是作用于实例化阶段。
- BeanPostProcessor 主要作用与初始化阶段。

```java
public void refresh() throws BeansException, IllegalStateException {
  // Invoke factory processors registered as beans in the context.
  invokeBeanFactoryPostProcessors(beanFactory);  
  // Register bean processors that intercept bean creation.
  registerBeanPostProcessors(beanFactory);
  // Instantiate all remaining (non-lazy-init) singletons.
  finishBeanFactoryInitialization(beanFactory);
}
```

# 2.简化版invokeBeanFactoryPostProcessors过程

```java
if (beanFactory instanceof BeanDefinitionRegistry) {
 for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
  if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
    BeanDefinitionRegistryPostProcessor registryProcessor =
      (BeanDefinitionRegistryPostProcessor) postProcessor;
    // 🚩1. 首先执行系统已经存在的，并且实现BeanDefinitionRegistryPostProcessor接口的类
    registryProcessor.postProcessBeanDefinitionRegistry(registry); 
    registryProcessors.add(registryProcessor); 
  }
  else {
    regularPostProcessors.add(postProcessor);
  }
 }
  //🚩其次从注册的beanDefinitionMap中查找，实现BeanDefinitionRegistryPostProcessor接口的类
  String[] postProcessorNames=beanFactory.getBeanNamesForType
    (BeanDefinitionRegistryPostProcessor.class, true, false);
  
  //🚩 分别按照优先级 [PriorityOrdered->Ordered->无序] 来执行
  invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
  
  //⭐️ 这里执行实现BeanFactoryPostProcessor 接口的类
  invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
  invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
}

//🚩2. 其次从注册的beanDefinitionMap中查找，实现BeanFactoryPostProcessor接口的类
String[] postProcessorNames=beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);
//🚩 分别优先级执行：PriorityOrdered
invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);
//🚩 分别优先级执行：Ordered
invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);
//🚩 分别优先级执行：无序
invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);
```

```java
//备注：BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor 
//简单理解：首先处理 BeanDefinitionRegistryPostProcessor 接口

List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();

class MyBean1 implements BeanDefinitionRegistryPostProcessor {
  System.out.println("系统自带的最先执行：BeanDefinitionRegistryPostProcessor");
  //添加
  registryProcessors.add(MyBean1);
}
class MyBean2 implements BeanDefinitionRegistryPostProcessor,PriorityOrdered {
  System.out.println("我是第二个执行：BeanDefinitionRegistryPostProcessor");
  //添加
  registryProcessors.add(MyBean2);
}
class MyBean3 implements BeanDefinitionRegistryPostProcessor,Ordered {
  System.out.println("我是第三个执行：BeanDefinitionRegistryPostProcessor");
  //添加
  registryProcessors.add(MyBean3);
}
class MyBean4 implements BeanDefinitionRegistryPostProcessor {
  System.out.println("我是第四个执行：BeanDefinitionRegistryPostProcessor");
  //添加
  registryProcessors.add(MyBean4);
}
----------------------------------
//其次处理 BeanFactoryPostProcessor接口
invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
----------------------------------

//再次处理 BeanFactoryPostProcessor接口 ,从注册的beanDefinitionMap中查找
class MyBean5 implements BeanFactoryPostProcessor ,PriorityOrdered{
  System.out.println("执行：BeanFactoryPostProcessor");
}
class MyBean6 implements BeanFactoryPostProcessor ,Ordered{
  System.out.println("执行：BeanFactoryPostProcessor");
}
class MyBean7 implements BeanFactoryPostProcessor {
  System.out.println("执行：BeanFactoryPostProcessor");
}
```

最终执行的结果在BeanFactory如下：

* BeanFactoryPostProcessor接口

> 1. SharedMetadataReaderFactoryContextInitializer$CachingMetadataReaderFactoryPostProcessor
> 2. ConfigurationWarningsApplicationContextInitializer$ConfigurationWarningsPostProcessor
> 3. ConfigurationClassPostProcessor
> 4. ConfigFileApplicationListener
> 5. PropertySourcesPlaceholderConfigurer
> 6. ConfigurationBeanFactoryMetadata
> 7. EventListenerMethodProcessor 

* BeanDefinitionRegistryPostProcessor接口

>1. SharedMetadataReaderFactoryContextInitializer$CachingMetadataReaderFactoryPostProcessor
>2. ConfigurationWarningsApplicationContextInitializer$ConfigurationWarningsPostProcessor
>3. ConfigurationClassPostProcessor

# 3.简化版的registerBeanPostProcessors过程

```java
//从注册的beanDefinitionMap中查找
String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

for (String ppName : postProcessorNames) {
  // 优先级：PriorityOrdered
  if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
    BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
    priorityOrderedPostProcessors.add(pp);
    if (pp instanceof MergedBeanDefinitionPostProcessor) {
      internalPostProcessors.add(pp);
    }
  }
  // 优先级：Ordered
  else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
    BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
    orderedPostProcessors.add(pp);
    if (pp instanceof MergedBeanDefinitionPostProcessor) {
      internalPostProcessors.add(pp);
    }
  }
  // 优先级：无
  else {
    BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
    nonOrderedPostProcessors.add(pp);
    if (pp instanceof MergedBeanDefinitionPostProcessor) {
      internalPostProcessors.add(pp);
    }
  }
}
//按顺序添加
registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);
registerBeanPostProcessors(beanFactory, orderedPostProcessors);
registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);
registerBeanPostProcessors(beanFactory, internalPostProcessors);
```

最终执行的结果在BeanFactory如下：

* BeanPostProcessor接口  【⭐️】

> 1. ApplicationContextAwareProcessor
> 2. ConfigurationClassPostProcessor$ImportAwareBeanPostProcessor
> 3. PostProcessorRegistrationDelegate$BeanPostProcessorChecker
> 4. ConfigurationPropertiesBindingPostProcessor
> 5. CommonAnnotationBeanPostProcessor
> 6. AutowiredAnnotationBeanPostProcessor
> 7. ApplicationListenerDetector

* MergedBeanDefinitionPostProcessor接口

> 1. CommonAnnotationBeanPostProcessor
> 2. AutowiredAnnotationBeanPostProcessor
> 3. ApplicationListenerDetector

* InstantiationAwareBeanPostProcessor接口  【⭐️】

> 1. ConfigurationClassPostProcessor$ImportAwareBeanPostProcessor
> 2. CommonAnnotationBeanPostProcessor
> 3. AutowiredAnnotationBeanPostProcessor

# 4.注册Bean的过程

## AbstractBeanFactory#doGetBean

```java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {
  //缓存  
  Object sharedInstance = getSingleton(beanName);
  if (sharedInstance != null && args == null) {
    bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
  }
  else {
    // 判断循环引用,抛异常
    if (isPrototypeCurrentlyInCreation(beanName)) {
      throw new BeanCurrentlyInCreationException(beanName);
    }  
  }
  //-----------------
  // 获取bean的依赖，实例化bean前先实例化依赖。
  String[] dependsOn = mbd.getDependsOn();
  if (dependsOn != null) {
    for (String dep : dependsOn) {
      registerDependentBean(dep, beanName);
      try {
        getBean(dep);
      }
    }
  }
   // Create bean instance
   // 创建 bean 实例
   // 单例模式
   if (mbd.isSingleton()) {
     sharedInstance = getSingleton(beanName, () -> {
       return createBean(beanName, mbd, args);
     });
     bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
   }
   // 原型模式
   else if (mbd.isPrototype()) {
     // It's a prototype -> create a new instance.
    Object prototypeInstance = null;
    try {
      beforePrototypeCreation(beanName);
      prototypeInstance = createBean(beanName, mbd, args);
    }
    finally {
      afterPrototypeCreation(beanName);
    }
    bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
  }
}
```

* 先getSingleton()从缓存中获取Bean，如果没有则创建。
* 创建过程先检查有无循环依赖，有则抛出异常。
* 实例化bean前先实例化所依赖的对象。

## AbstractAutowireCapableBeanFactory#createBean 

```java
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) throws BeanCreationException {
  try {
    // 🚩实例化之前动作 处理InstantiationAwareBeanPostProcessor
    Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
    if (bean != null) {
      return bean;
    }
  }
  ...
  try {
    // 🚩 创建实例
    Object beanInstance = doCreateBean(beanName, mbdToUse, args);
    return beanInstance;
  }
  
}
```

* resolveBeforeInstantiation

```java
protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
		Object bean = null;
  if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
    // Make sure bean class is actually resolved at this point.
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
      Class<?> targetType = determineTargetType(beanName, mbd);
      if (targetType != null) {
        //调用InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation()
        bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
        if (bean != null) {
          bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
        }
      }
    }
    mbd.beforeInstantiationResolved = (bean != null);
  }
  return bean;
}
```

* applyBeanPostProcessorsBeforeInstantiation

```java
protected Object applyBeanPostProcessorsBeforeInstantiation(Class<?> beanClass, String beanName) {
  for (BeanPostProcessor bp : getBeanPostProcessors()) {
    if (bp instanceof InstantiationAwareBeanPostProcessor) {
      InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
      // 执行所有InstantiationAwareBeanPostProcessor的postProcessBeforeInstantiation
      Object result = ibp.postProcessBeforeInstantiation(beanClass, beanName);
      if (result != null) {
        return result;
      }
    }
  }
  return null;
}
```

## AbstractAutowireCapableBeanFactory#doCreateBean

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args) throws BeanCreationException {
	if (instanceWrapper == null) {
    //创建bean实例
    instanceWrapper = createBeanInstance(beanName, mbd, args);
  }
  synchronized (mbd.postProcessingLock) {
    if (!mbd.postProcessed) {
      try {
        //
        applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
      }
      mbd.postProcessed = true;
    }
  }
  // Initialize the bean instance. 初始化 bean
  Object exposedObject = bean;
  try {
    populateBean(beanName, mbd, instanceWrapper);
    exposedObject = initializeBean(beanName, exposedObject, mbd);
  }
  
 // Register bean as disposable. 注册bean销毁方法
  try {
    registerDisposableBeanIfNecessary(beanName, bean, mbd);
  }
  return exposedObject;
}
```

* applyMergedBeanDefinitionPostProcessors

```java
protected void applyMergedBeanDefinitionPostProcessors(RootBeanDefinition mbd, Class<?> beanType, String beanName) {
  for (BeanPostProcessor bp : getBeanPostProcessors()) {
    if (bp instanceof MergedBeanDefinitionPostProcessor) {
      MergedBeanDefinitionPostProcessor bdp = (MergedBeanDefinitionPostProcessor) bp;
      bdp.postProcessMergedBeanDefinition(mbd, beanType, beanName);
    }
  }
}
```

## AbstractAutowireCapableBeanFactory#populateBean

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
 if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
  for (BeanPostProcessor bp : getBeanPostProcessors()) {
    if (bp instanceof InstantiationAwareBeanPostProcessor) {
      InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
      // 执行实例化后方法
      if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
        continueWithPropertyPopulation = false;
        break;
      }
    }
  }
 }  
 if (pvs != null) {
  //其他属性填充
   applyPropertyValues(beanName, mbd, bw, pvs);
 }
}
```

## AbstractAutowireCapableBeanFactory#initializeBean

```java
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
  //Aware接口方法执行
	invokeAwareMethods(beanName, bean);
  
  Object wrappedBean = bean;
  if (mbd == null || !mbd.isSynthetic()) {
    //初始化之前
    wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
  }
  try {
    //初始化
    invokeInitMethods(beanName, wrappedBean, mbd);
  }
  if (mbd == null || !mbd.isSynthetic()) {
    //初始化之后
    wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
  }
  return wrappedBean;
}
```

* invokeAwareMethods

```java
private void invokeAwareMethods(final String beanName, final Object bean) {
  if (bean instanceof Aware) {
    if (bean instanceof BeanNameAware) {
      ((BeanNameAware) bean).setBeanName(beanName);
    }
    if (bean instanceof BeanClassLoaderAware) {
      ClassLoader bcl = getBeanClassLoader();
      if (bcl != null) {
        ((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
      }
    }
    if (bean instanceof BeanFactoryAware) {
      ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
    }
  }
}
```

* applyBeanPostProcessorsBeforeInitialization

```java
public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName) throws BeansException {
  Object result = existingBean;
  for (BeanPostProcessor processor : getBeanPostProcessors()) {
    Object current = processor.postProcessBeforeInitialization(result, beanName);
    if (current == null) {
      return result;
    }
    result = current;
  }
  return result;
}
```

* applyBeanPostProcessorsAfterInitialization

```java
public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName) throws BeansException {
  Object result = existingBean;
  for (BeanPostProcessor processor : getBeanPostProcessors()) {
    Object current = processor.postProcessAfterInitialization(result, beanName);
    if (current == null) {
      return result;
    }
    result = current;
  }
  return result;
}
```

# 5.销毁bean的过程

## AbstractBeanFactory#registerDisposableBeanIfNecessary

```java
protected void registerDisposableBeanIfNecessary(String beanName, Object bean, RootBeanDefinition mbd) {
  AccessControlContext acc = (System.getSecurityManager() != null ? getAccessControlContext() : null);
  //判断是否满足条件
  if (!mbd.isPrototype() && requiresDestruction(bean, mbd)) {
    if (mbd.isSingleton()) {
      // Register a DisposableBean implementation that performs all destruction
  // work for the given bean: DestructionAwareBeanPostProcessors,
      // DisposableBean interface, custom destroy method.
      registerDisposableBean(beanName,
             new DisposableBeanAdapter(bean, beanName, mbd, getBeanPostProcessors(), acc));
    }
    else {
      // A bean with a custom scope...
      Scope scope = this.scopes.get(mbd.getScope());
      scope.registerDestructionCallback(beanName,
          new DisposableBeanAdapter(bean, beanName, mbd, getBeanPostProcessors(), acc));
    }
  }
}
```

* 判断是否满足条件

> 1.是否存在：对DestructionAwareBeanPostProcessors接口的处理的实现类，目前有CommonAnnotationBeanPostProcessor 可以来处理 @PreDestroy 注解
>
>  2.是否有销毁方法：bean是否有实现 DisposableBean 接口 或者 bean是否存在有销毁的方法名
>
>  3.bean是否是单例模式

## DisposableBeanAdapter#hasApplicableProcessors

```java
public static boolean hasApplicableProcessors(Object bean, List<BeanPostProcessor> postProcessors) {
  if (!CollectionUtils.isEmpty(postProcessors)) {
    for (BeanPostProcessor processor : postProcessors) {
      //DestructionAwareBeanPostProcessor 接口
      if (processor instanceof DestructionAwareBeanPostProcessor) {
        DestructionAwareBeanPostProcessor dabpp = (DestructionAwareBeanPostProcessor) processor;
        if (dabpp.requiresDestruction(bean)) {
          return true;
        }
      }
    }
  }
  return false;
}
```

## DisposableBeanAdapter#destroy 

```java
public void destroy() {
  //首先执行 CommonAnnotationBeanPostProcessor中对 @PreDestroy 注解的处理
  if (!CollectionUtils.isEmpty(this.beanPostProcessors)) {
    for (DestructionAwareBeanPostProcessor processor : this.beanPostProcessors) {
      processor.postProcessBeforeDestruction(this.bean, this.beanName);
    }
  }
  //其次执行 bean实现DisposableBean接口
  if (this.invokeDisposableBean) {
    try {
        ((DisposableBean) this.bean).destroy();
      }
    }
  }
  //最后执行 @Bean(destroyMethod="") 或者 xml中配置的destroy-method="destroyMethod"
  if (this.destroyMethod != null) {
    invokeCustomDestroyMethod(this.destroyMethod);
  }
  else if (this.destroyMethodName != null) {
    Method methodToInvoke = determineDestroyMethod(this.destroyMethodName);
    if (methodToInvoke != null) {
      invokeCustomDestroyMethod(ClassUtils.getInterfaceMethodIfPossible(methodToInvoke));
    }
  }
}
```

