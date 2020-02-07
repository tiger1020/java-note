# 1.整体流程

<img src="/images/Sentinel 削峰填谷.png">

# 2.源码分析

RateLimiterController

```java
@Override
public boolean canPass(Node node, int acquireCount) {

  // 按照斜率来计算计划中应该什么时候通过
  long currentTime = TimeUtil.currentTimeMillis();

  long costTime = Math.round(1.0 * (acquireCount) / count * 1000);

  //期待时间 
  long expectedTime = costTime + latestPassedTime.get();

  if (expectedTime <= currentTime) {
    //这里会有冲突,然而冲突就冲突吧.
    latestPassedTime.set(currentTime);
    return true;
  } else {
    // 计算自己需要的等待时间
    long waitTime = costTime + latestPassedTime.get() - TimeUtil.currentTimeMillis();
    //大于最大等待时间
    if (waitTime >= maxQueueingTimeMs) {
      return false;
    } else {
      long oldTime = latestPassedTime.addAndGet(costTime);
      try {
        waitTime = oldTime - TimeUtil.currentTimeMillis();
        //二次判断 大于 最大等待时间
        if (waitTime >= maxQueueingTimeMs) {
          latestPassedTime.addAndGet(-costTime);
          return false;
        }
        //线程休眠
        Thread.sleep(waitTime);
        return true;
      } catch (InterruptedException e) {
      }
    }
  }

  return false;
}
```

