# 1.整体流程

<img src="images/Sentinel 核心流程.png">

# 2.Slot详解

## 2.1.NodeSelectorSlot

### 2.1.1.Context不同,只考虑Resource相同情况

1. 第一种场景：Context 

   > name = "entrance1"
   >
   > origin = "appA"

```java
ContextUtil.enter("entrance1", "appA");
Entry nodeA = SphU.entry("nodeA");
if (nodeA != null) {
  nodeA.exit();
}
ContextUtil.exit();
```

转换为

```java
              machine-root
                  /
                 /
           EntranceNode1
               /
              /
        DefaultNode(nodeA)
						 /
            /
     ClusterNode(nodeA);
```

2. 第二种场景：Context

   > name = "entrance1" origin = "appA"

   > name = "entrance2" origin = "appA"

```java
ContextUtil.enter("entrance1", "appA");
Entry nodeA = SphU.entry("nodeA");
if (nodeA != null) {
  nodeA.exit();
}
ContextUtil.exit();

ContextUtil.enter("entrance2", "appA");
nodeA = SphU.entry("nodeA");
if (nodeA != null) {
  nodeA.exit();
}
ContextUtil.exit();
```

转换为

```xml
                   machine-root
                   /         \
                  /           \
          EntranceNode1   EntranceNode2
                /               \
               /                 \
       DefaultNode(nodeA)   DefaultNode(nodeA)
              \                  /
               \                /  
                ClusterNode(nodeA)
```

### 3.1.2.Context相同,Resource相同和不同

* Resource相同情况

```java
Entry nodeA = SphU.entry("nodeA");
Entry nodeA = SphU.entry("nodeA");
```

* Resource不相同情况

```java
Entry nodeA = SphU.entry("nodeA");
Entry nodeB = SphU.entry("nodeB");
```

<img src="images/Sentinel Resource场景.png">

## 2.2.FlowSlot

### 2.2.1.限流阀值类型grade

* 线程数限制
* QPS限制

### 2.2.2.流控调用来源limitApp

* 默认限制来源 default (不区分调用来源)
* 其他限制来源 other (不等于default和orgin的来源)
* 指定限制来源 orgin

### 2.2.3.流控模式strategy

* 直接限流 direct (当前节点的数据)
* 关联限流 relate (对应关联节点的数据)
* 链路限流 chain (根据链路入口节点信息)

### 2.2.4.流控效果controlBehavior

* 直接拒绝 default

* 冷启动 WarmUp

  > grade = QPS

* 匀速器RateLimiter

  >grade = QPS

## 2.3.DegradSlot

### 2.3.1.降级模式grade

* 根据RT降级
* 根据异常比例降级
* 根据异常总数降级

### 2.3.2.降级时间

* 中断时间timeWindow
