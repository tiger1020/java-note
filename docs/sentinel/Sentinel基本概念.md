# 1.基本概念

## Resource

`StringResourceWrapper` 使用string来标识一个资源

`MethodResouceWrapper` 使用一个函数签名来标识一个资源

## Node

<img src="images/Sentinel 基本概念.png">

`DefaultNode`和`ClusterNode`区别

`DefaultNode`：保存着某个resource在某个context中的实时指标，每个DefaultNode都指向一个ClusterNode

`ClusterNode`：保存着某个resource在所有的context中实时指标的总和，同样的resource会共享同一个ClusterNode，不管他在哪个context中

## Context

上下文是用来保存当前调用的元数据，存储在ThreadLocal中，它包含了几个信息：

1. EntranceNode 整个调用树的根节点，即入口
2. Entry 当前的调用点
3. Node 关联到当前调用点的统计信息
4. Origin 通常用来标识调用方，这在我们需要按照调用方来区分流控策略的时候会非常有用

每当我们调用SphU.entry()或者 SphO.entry()获取访问资源许可的时候都需要当前线程处在某个context中，如果我们没有显式调用ContextUtil.enter()，默认会使用Default context。如果我们在一个上下文中多次调用SphU.entry()来获取多个资源，一个调用树就会被创建出来

## EntryType

EntryType 说的是这次请求的流量类型，共有两种类型：IN 和 OUT 。

* IN：是指进入我们系统的入口流量，比如 http 请求或者是其他的 rpc 之类的请求。

* OUT：是指我们系统调用其他第三方服务的出口流量。

 入口、出口流量只有在配置了系统规则时才有效。

 设置Type 为 IN 是为了统计整个系统的流量水平，防止系统被打垮，用以自我保护的一种方式。

## Slot

- `NodeSelectorSlot` 负责收集资源的路径，并将这些资源的调用路径，以树状结构存储起来，用于根据调用路径来限流降级；
- `ClusterBuilderSlot` 则用于存储资源的统计信息以及调用者信息，例如该资源的 RT, QPS, thread count 等等，这些信息将用作为多维度限流，降级的依据；
- `StatisticsSlot` 则用于记录，统计不同维度的 runtime 信息；
- `SystemSlot` 则通过系统的状态，例如 load1 等，来控制总的入口流量；
- `AuthoritySlot` 则根据黑白名单，来做黑白名单控制；
- `FlowSlot` 则用于根据预设的限流规则，以及前面 slot 统计的状态，来进行限流；
- `DegradeSlot` 则通过统计信息，以及预设的规则，来做熔断降级；