>核心思想
>
>1. 用原子操作来同步状态
>2. 加锁线程变量=null，通过（阻塞或者唤醒一个线程）来修改加锁线程标记
>3. 内部应该维护一个队列

<img src="images/AQS.png" style="zoom: 67%;" >

**流程**

<img src="images/AQS流程.png">
