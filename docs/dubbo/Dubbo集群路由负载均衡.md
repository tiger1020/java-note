# 1.集群的容错模式

## Failover Cluster

这是dubbo中默认的集群容错模式

- 失败自动切换，当出现失败，重试其它服务器。
- 通常用于读操作，但重试会带来更长延迟。
- 可通过retries=”2”来设置重试次数(不含第一次)。

## Failfast Cluster

- 快速失败，只发起一次调用，失败立即报错。
- 通常用于非幂等性的写操作，比如新增记录。

## Failsafe Cluster

- 失败安全，出现异常时，直接忽略。
- 通常用于写入审计日志等操作。

## Failback Cluster

- 失败自动恢复，后台记录失败请求，定时重发。
- 通常用于消息通知操作。

## Forking Cluster

- 并行调用多个服务器，只要一个成功即返回。
- 通常用于实时性要求较高的读操作，但需要浪费更多服务资源。
- 可通过forks=”2”来设置最大并行数。

## Broadcast Cluster

- 广播调用所有提供者，逐个调用，任意一台报错则报错。(2.1.0开始支持)
- 通常用于通知所有提供者更新缓存或日志等本地资源信息。

# 2. 负载均衡

dubbo默认的负载均衡策略是random，随机调用。

## Random LoadBalance

- 随机，按权重设置随机概率。
- 在一个截面上碰撞的概率高，但调用量越大分布越均匀，而且按概率使用权重后也比较均匀，有利于动态调整提供者权重。

## RoundRobin LoadBalance

- 轮循，按公约后的权重设置轮循比率。
- 存在慢的提供者累积请求问题，比如：第二台机器很慢，但没挂，当请求调到第二台时就卡在那，久而久之，所有请求都卡在调到第二台上。

## LeastActive LoadBalance

- 最少活跃调用数，相同活跃数的随机，活跃数指调用前后计数差。
- 使慢的提供者收到更少请求，因为越慢的提供者的调用前后计数差会越大。

## ConsistentHash LoadBalance

- 一致性Hash，相同参数的请求总是发到同一提供者。
- 当某一台提供者挂时，原本发往该提供者的请求，基于虚拟节点，平摊到其它提供者，不会引起剧烈变动。
- 缺省只对第一个参数Hash。
- 缺省用160份虚拟节点。

# 3.源码解析

## 3.1.调用方法入口

`com.alibaba.dubbo.registry.integration.RegistryProtocol#doRefer`

```java
private <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url) {
  RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);
  directory.setRegistry(registry);
  directory.setProtocol(protocol);

	//默认FailoverCluster
  Invoker invoker = cluster.join(directory);
  ProviderConsumerRegTable.registerConsumer(invoker, url, subscribeUrl, directory);
  return invoker;
}
```

返回的Invoker是一个MockClusterInvoker，MockClusterInvoker内部包含一个Directory和一个FailoverClusterInvoker。

```java
public class FailoverCluster implements Cluster {
  public final static String NAME = "failover";
  @Override
  public <T> Invoker<T> join(Directory<T> directory) throws RpcException {
    return new FailoverClusterInvoker<T>(directory);
  }
}
```

Invoker都封装好了之后，就是创建代理，然后使用代理调用我们的要调用的方法。

## 3.2.调用方法时集群的处理

在进行具体方法调用的时候，代理中会`invoker.invoke()`，这里Invoker就是我们上面封装好的MockClusterInvoker，所以首先进入MockClusterInvoker的invoke方法：

```java
public Result invoke(Invocation invocation) throws RpcException {
  Result result = null;
  //我们没配置mock，所以这里为false
  //Mock通常用于服务降级
  String value = directory.getUrl().getMethodParameter(invocation.getMethodName(), Constants.MOCK_KEY, Boolean.FALSE.toString()).trim(); 
  //没有使用mock
  if (value.length() == 0 || value.equalsIgnoreCase("false")){
    //这里的invoker是FailoverClusterInvoker
    result = this.invoker.invoke(invocation);
  } else if (value.startsWith("force")) {
    //mock=force:return+null
    //表示消费方对方法的调用都直接返回null，不发起远程调用
    //可用于屏蔽不重要服务不可用的时候，对调用方的影响
    //force:direct mock
    result = doMockInvoke(invocation, null);
  } else {
    //mock=fail:return+null
    //表示消费方对该服务的方法调用失败后，再返回null，不抛异常
    //可用于对不重要服务不稳定的时候，忽略对调用方的影响
    //fail-mock
    try {
      result = this.invoker.invoke(invocation);
    }catch (RpcException e) {
      if (e.isBiz()) {
        throw e;
      } else {
        result = doMockInvoke(invocation, e);
      }
    }
  }
  return result;
}
```

我们这里么有配置mock属性。首先进入的是AbstractClusterInvoker的incoke方法：

```java
public Result invoke(final Invocation invocation) throws RpcException {
  //可以看到这里该处理负载均衡的问题了
  LoadBalance loadbalance;
  //根据invocation中的信息从Directory中获取Invoker列表
  //这一步中会进行路由的处理
  List<Invoker<T>> invokers = list(invocation);
  if (invokers != null && invokers.size() > 0) {
    //使用扩展机制，加载LoadBalance的实现类，默认使用的是random
    //我们这里得到的就是RandomLoadBalance
    loadbalance = ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(invokers.get(0).getUrl()
                                                                                     .getMethodParameter(invocation.getMethodName(),Constants.LOADBALANCE_KEY, Constants.DEFAULT_LOADBALANCE));
  } 
  
  return doInvoke(invocation, invokers, loadbalance);
}
```

看下FailoverClusterInvoker#doInvoke方法：

```java
public Result doInvoke(Invocation invocation, final List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
  //Invoker列表
  List<Invoker<T>> copyinvokers = invokers;
  //确认下Invoker列表不为空
  checkInvokers(copyinvokers, invocation);
  //重试次数,默认3次
  int len = getUrl().getMethodParameter(invocation.getMethodName(), Constants.RETRIES_KEY, Constants.DEFAULT_RETRIES) + 1;
  
  // retry loop.
  RpcException le = null; // last exception.
  List<Invoker<T>> invoked = new ArrayList<Invoker<T>>(copyinvokers.size()); // invoked invokers.
  Set<String> providers = new HashSet<String>(len);
  for (int i = 0; i < len; i++) {
    //重试时，进行重新选择，避免重试时invoker列表已发生变化.
    //注意：如果列表发生了变化，那么invoked判断会失效，因为invoker示例已经改变
    if (i > 0) {
      checkWheatherDestoried();
      copyinvokers = list(invocation);
      //重新检查一下
      checkInvokers(copyinvokers, invocation);
    }
    //使用loadBalance选择一个Invoker返回
    Invoker<T> invoker = select(loadbalance, invocation, copyinvokers, invoked);
    invoked.add(invoker);
    RpcContext.getContext().setInvokers((List)invoked);

      //使用选择的结果Invoker进行调用，返回结果
      Result result = invoker.invoke(invocation);
      return result;
   
  }
}
```

先看下使用loadbalance选择invoker的select方法：

```java
protected Invoker<T> select(LoadBalance loadbalance, Invocation invocation, List<Invoker<T>> invokers, List<Invoker<T>> selected) throws RpcException {
  if (invokers == null || invokers.isEmpty())
    return null;
  String methodName = invocation == null ? "" : invocation.getMethodName();

  boolean sticky = invokers.get(0).getUrl().getMethodParameter(methodName, Constants.CLUSTER_STICKY_KEY, Constants.DEFAULT_CLUSTER_STICKY);
  {
    //ignore overloaded method
    if (stickyInvoker != null && !invokers.contains(stickyInvoker)) {
      stickyInvoker = null;
    }
    //ignore concurrency problem
    if (sticky && stickyInvoker != null && (selected == null || !selected.contains(stickyInvoker))) {
      if (availablecheck && stickyInvoker.isAvailable()) {
        return stickyInvoker;
      }
    }
  }
  Invoker<T> invoker = doSelect(loadbalance, invocation, invokers, selected);

  if (sticky) {
    stickyInvoker = invoker;
  }
  return invoker;
}
```

doselect方法：

```java
private Invoker<T> doselect(LoadBalance loadbalance, Invocation invocation, List<Invoker<T>> invokers, List<Invoker<T>> selected) throws RpcException {
  if (invokers == null || invokers.size() == 0)
    return null;
  //只有一个invoker，直接返回，不需要处理
  if (invokers.size() == 1)
    return invokers.get(0);
  // 如果只有两个invoker，退化成轮循
  if (invokers.size() == 2 && selected != null && selected.size() > 0) {
    return selected.get(0) == invokers.get(0) ? invokers.get(1) : invokers.get(0);
  }
  //使用loadBalance进行选择
  Invoker<T> invoker = loadbalance.select(invokers, getUrl(), invocation);

  //如果 selected中包含（优先判断） 或者 不可用&&availablecheck=true 则重试.
  if( (selected != null && selected.contains(invoker))
     ||(!invoker.isAvailable() && getUrl()!=null && availablecheck)){
    try{
      //重新选择
      Invoker<T> rinvoker = reselect(loadbalance, invocation, invokers, selected, availablecheck);
      if(rinvoker != null){
        invoker =  rinvoker;
      }else{
        //看下第一次选的位置，如果不是最后，选+1位置.
        int index = invokers.indexOf(invoker);
        try{
          //最后在避免碰撞
          invoker = index <invokers.size()-1?invokers.get(index+1) :invoker;
        }catch (Exception e) {。。。 }
      }
    }catch (Throwable t){。。。}
  }
  return invoker;
}
```

接着看使用loadBalance进行选择，首先进入AbstractLoadBalance的select方法：

```java
public <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) {
  if (invokers == null || invokers.size() == 0)
    return null;
  if (invokers.size() == 1)
    return invokers.get(0);
  //	进行选择，具体的子类实现，我们这里是RandomLoadBalance
  return doSelect(invokers, url, invocation);
}
```

接着去RandomLoadBalance中查看：

```java
protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
  int length = invokers.size(); // 总个数
  int totalWeight = 0; // 总权重
  boolean sameWeight = true; // 权重是否都一样
  for (int i = 0; i < length; i++) {
    int weight = getWeight(invokers.get(i), invocation);
    totalWeight += weight; // 累计总权重
    if (sameWeight && i > 0
        && weight != getWeight(invokers.get(i - 1), invocation)) {
      sameWeight = false; // 计算所有权重是否一样
    }
  }
  if (totalWeight > 0 && ! sameWeight) {
    // 如果权重不相同且权重大于0则按总权重数随机
    int offset = random.nextInt(totalWeight);
    // 并确定随机值落在哪个片断上
    for (int i = 0; i < length; i++) {
      offset -= getWeight(invokers.get(i), invocation);
      if (offset < 0) {
        return invokers.get(i);
      }
    }
  }
  // 如果权重相同或权重为0则均等随机
  return invokers.get(random.nextInt(length));
}
```

上面根据权重之类的来进行选择一个Invoker返回。接下来reselect的方法不在说明，是先从非selected的列表中选择，没有在从selected列表中选择。

选择好了Invoker之后，就回去FailoverClusterInvoker的doInvoke方法，接着就是根据选中的Invoker调用invoke方法进行返回结果，接着就是到具体的Invoker进行调用的过程了。这部分的解析在消费者和提供者请求响应过程已经解析过了，不再重复。

## 3.3.路由策略

回到AbstractClusterInvoker的invoke方法中，这里有一步是`List<Invoker<T>> invokers = list(invocation);`获取Invoker列表，这里同时也进行了路由的操作，看下list方法：

```java
protected  List<Invoker<T>> list(Invocation invocation) throws RpcException {
  List<Invoker<T>> invokers = directory.list(invocation);
  return invokers;
}
```

接着看AbstractDirectory的list方法：

```java
public List<Invoker<T>> list(Invocation invocation) throws RpcException {
  if (destroyed){
    throw new RpcException("Directory already destroyed .url: "+ getUrl());
  }
  //RegistryDirectory中的doList实现
  List<Invoker<T>> invokers = doList(invocation);
  List<Router> localRouters = this.routers; // local reference
  if (localRouters != null && localRouters.size() > 0) {
    for (Router router: localRouters){
      try {
        if (router.getUrl() == null || router.getUrl().getParameter(Constants.RUNTIME_KEY, true)) {
          //路由选择
          //MockInvokersSelector中
          invokers = router.route(invokers, getConsumerUrl(), invocation);
        }
      } catch (Throwable t) {。。。}
    }
  }
  return invokers;
}
```

路由来过滤之后，进行负载均衡的处理。

# 4.Dubbo一致性hash实现   

## 4.1.一致性哈希负载均衡配置

```xml
<!-- 配置接口 -->
<dubbo:service interface="..." loadbalance="consistenthash" />
<dubbo:reference interface="..." loadbalance="consistenthash" />

<!-- 配置方法 -->
<dubbo:service interface="...">
    <dubbo:method name="..." loadbalance="consistenthash"/>
</dubbo:service>
<dubbo:reference interface="...">
    <dubbo:method name="..." loadbalance="consistenthash"/>
</dubbo:reference>
```

## 4.2.主要配置参数

一致性Hash负载均衡涉及到两个主要的配置参数为

> 1. hash.arguments 
>
>     当进行调用时候根据调用方法的哪几个参数生成key，并根据key来通过一致性hash算法来选择调用结点。例如调用方法invoke(String s1,String s2); 若hash.arguments为1(默认值)，则仅取invoke的参数1（s1）来生成hashCode
>
> 2. hash.nodes
>
>    为结点的副本数

## 4.3.ConsistentHashSelector详解

```java
private static final class ConsistentHashSelector<T> {
  //虚拟节点
  private final TreeMap<Long, Invoker<T>> virtualInvokers;
  //副本数，默认160
  private final int replicaNumber;
  //hashcode
  private final int identityHashCode;

  private final int[] argumentIndex;

  ConsistentHashSelector(List<Invoker<T>> invokers, String methodName, int identityHashCode) {
    //创建TreeMap来保持结点
    this.virtualInvokers = new TreeMap<Long, Invoker<T>>();
    this.identityHashCode = identityHashCode;
    URL url = invokers.get(0).getUrl();
    // 获取所配置的结点数，如没有设置则使用默认值160
    this.replicaNumber = url.getMethodParameter(methodName, "hash.nodes", 160);
    //获取需要进行hash的参数数组索引，默认对第一个参数进行hash
    String[] index = Constants.COMMA_SPLIT_PATTERN.split(url.getMethodParameter(methodName, "hash.arguments", "0"));
    argumentIndex = new int[index.length];
    for (int i = 0; i < index.length; i++) {
      argumentIndex[i] = Integer.parseInt(index[i]);
    }
    // 创建虚拟结点
    // 对每个invoker生成replicaNumber个虚拟结点，并存放于TreeMap中
    for (Invoker<T> invoker : invokers) {
      String address = invoker.getUrl().getAddress();
      for (int i = 0; i < replicaNumber / 4; i++) {
        // 根据md5算法为每4个结点生成一个消息摘要，摘要长为16字节128位。
        byte[] digest = md5(address + i);
        // 随后将128位分为4部分，0-31,32-63,64-95,95-128，并生成4个32位数，存于long中，long的高32位都为0
        // 并作为虚拟结点的key。
        for (int h = 0; h < 4; h++) {
          long m = hash(digest, h);
          virtualInvokers.put(m, invoker);
        }
      }
    }
  }

  // 选择结点
  public Invoker<T> select(Invocation invocation) {
    // 根据调用参数来生成Key
    String key = toKey(invocation.getArguments());
    // 根据这个参数生成消息摘要
    byte[] digest = md5(key);
    //调用hash(digest, 0)，将消息摘要转换为hashCode，这里仅取0-31位来生成HashCode
    //调用sekectForKey方法选择结点。
    Invoker<T> invoker = sekectForKey(hash(digest, 0));
    return invoker;
  }

  private String toKey(Object[] args) {
    StringBuilder buf = new StringBuilder();
    //由于hash.arguments没有进行配置，因为只取方法的第1个参数作为key
    for (int i : argumentIndex) {
      if (i >= 0 && i < args.length) {
        buf.append(args[i]);
      }
    }
    return buf.toString();
  }

  private Invoker<T> selectForKey(long hash) {
    Map.Entry<Long, Invoker<T>> entry = virtualInvokers.tailMap(hash, true).firstEntry();
    if (entry == null) {
      entry = virtualInvokers.firstEntry();
    }
    return entry.getValue();
  }
}
```

>在进行选择时候若HashCode，有三种情况：
>
>1. 直接与某个虚拟结点的key一样，则直接返回该结点，例如hashCode落在某个结点上(圆圈所表示)。
>2. 若不相等，则通过`virtualInvokers.tailMap(hash).firstKey()`找到大于该hash值的集合，并通过`firstKey()`方法找到第一个值返回
>3. 若返回值为null，则说明该hash值已经在最大值区间了，则直接返回第一个虚拟节点`virtualInvokers.firstEntry()`

## 4.4.测试一致性hash代码

```java
public class ConsistentHashHasVirtualNode {
  // 待添加入Hash环的服务器列表
  private static String[] servers = { "192.168.1.0:111", "192.168.1.1:111", "192.168.1.2:111", "192.168.1.3:111","192.168.1.4:111" };

  // 真实结点列表,考虑到服务器上线、下线的场景，即添加、删除的场景会比较频繁，这里使用LinkedList会更好
  private static List<String> realNodes = new LinkedList<String>();

  // 虚拟节点，key表示虚拟节点的hash值，value表示虚拟节点的名称
  private static SortedMap<Integer, String> virtualNodes = new TreeMap<Integer, String>();

  // 虚拟节点的数目，这里写死，为了演示需要，一个真实结点对应5个虚拟节点
  private static final int VIRTUAL_NODES = 5;

  static {
    // 先把原始的服务器添加到真实结点列表中
    for (int i = 0; i < servers.length; i++)
      realNodes.add(servers[i]);

    // 再添加虚拟节点，遍历LinkedList使用foreach循环效率会比较高
    for (String str : realNodes) {
      for (int i = 0; i < VIRTUAL_NODES; i++) {
        String virtualNodeName = str + "&&VN" + String.valueOf(i);
        int hash = getHash(virtualNodeName);
        System.out.println("虚拟节点[" + virtualNodeName + "]被添加, hash值为" + hash);
        virtualNodes.put(hash, virtualNodeName);
      }
    }
    System.out.println();
  }

  // 使用FNV1_32_HASH算法计算服务器的Hash值,这里不使用重写hashCode的方法，最终效果没区别
  private static int getHash(String str) {
    final int p = 16777619;
    int hash = (int) 2166136261L;
    for (int i = 0; i < str.length(); i++)
      hash = (hash ^ str.charAt(i)) * p;
    hash += hash << 13;
    hash ^= hash >> 7;
    hash += hash << 3;
    hash ^= hash >> 17;
    hash += hash << 5;

    // 如果算出来的值为负数则取其绝对值
    if (hash < 0)
      hash = Math.abs(hash);
    return hash;
  }

  // 得到应当路由到的结点
  private static String getServer(String key) {
    // 得到该key的hash值
    int hash = getHash(key);
    // 得到大于该Hash值的所有Map
    SortedMap<Integer, String> subMap = virtualNodes.tailMap(hash);
    String virtualNode;
    if (subMap.isEmpty()) {
      // 如果没有比该key的hash值大的，则从第一个node开始
      Integer i = virtualNodes.firstKey();
      // 返回对应的服务器
      virtualNode = virtualNodes.get(i);
    } else {
      // 第一个Key就是顺时针过去离node最近的那个结点
      Integer i = subMap.firstKey();
      // 返回对应的服务器
      virtualNode = subMap.get(i);
    }
    // virtualNode虚拟节点名称要截取一下
    if (virtualNode != null && virtualNode != "") {
      return virtualNode.substring(0, virtualNode.indexOf("&&"));
    }
    return null;
  }

  public static void main(String[] args) {
    String[] keys = { "太阳", "月亮", "星星", "白云", "蓝天" };
    for (int i = 0; i < keys.length; i++)
      System.out.println("[" + keys[i] + "]的hash值为" + getHash(keys[i]) + ", 被路由到结点[" + getServer(keys[i]) + "]");
  }
}
```

