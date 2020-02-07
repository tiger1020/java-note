# 1.解决什么问题

线程同步机制是多个线程共享同一个变量，而ThreadLocal是为每个线程创建一个单独的变量副本，每个线程都可以改变自己的变量副本而不影响其它线程所对应的副本。

Synchronized用于线程间的数据共享，以时间换空间思想；

ThreadLocal用于线程间的数据隔离，以空间换时间思想

# 2.如何使用ThreadLocal

```java
public class ThreadLocalTest {

  private static final ThreadLocal<ThreadLocalTest> LOCAL = new ThreadLocal<ThreadLocalTest>();
  private String name;
  private ThreadLocalTest(String name) {
    this.name = name;
  }
  public static void main(String[] args) {
    //线程1
    new Thread(()->{
      LOCAL.set(new ThreadLocalTest("aa"));
      System.out.println(LOCAL.get());
    }).start();
    //线程2
    new Thread(()->{
      LOCAL.set(new ThreadLocalTest("bb"));
      System.out.println(LOCAL.get());
    }).start();
    //线程3 main线程
    LOCAL.set(new ThreadLocalTest("cc"));
    System.out.println(LOCAL.get());
  }

  @Override
  public String toString() {
    return this.name;
  }
}
//输出
aa
cc
bb
```

# 3.ThreadLocal实现原理

<img src="/images/ThreadLocal.png" alt="ThreadLocal" style="zoom:50%;" />

```java
public
class Thread implements Runnable {
  ...
  //每个线程都自带一个threadLocals变量
  ThreadLocal.ThreadLocalMap threadLocals = null;
  ...
}
```

```java
public class ThreadLocal<T> {
  
  -------------------ThreadLocal对外暴露的方法，内部操作还是基于ThreadLocalMap-------------------
  //对外代理ThreadLocalMap来操作set方法
  public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    //判断该线程是否已经创建了ThreadLocalMap
    if (map != null)
      map.set(this, value);
    else
      createMap(t, value);
  }
  //对外代理ThreadLocalMap来操作remove方法
  public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    //判断该线程是否已经创建了ThreadLocalMap
    if (m != null)
      m.remove(this);
  }
  //对外代理ThreadLocalMap来操作get方法
  public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    //判断该线程是否已经创建了ThreadLocalMap
    if (map != null) {
      ThreadLocalMap.Entry e = map.getEntry(this);
      ///能够Entry则返回e.value
      if (e != null) {
        @SuppressWarnings("unchecked")
        T result = (T)e.value;
        return result;
      }
    }
    //如果不能获取设置默认值
    return setInitialValue();
  }
  
---------------------查询和创建ThreadLocalMap类------------------------------------
  //返回该线程对应的【ThreadLocalMap】
  ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
  }
  //创建该线程对应的【ThreadLocalMap】
  void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
  }
  
-----------------------ThreadLocalMap静态内部类----------------------------------
  static class ThreadLocalMap {

    static class Entry extends WeakReference<ThreadLocal<?>> {
      Object value;
      Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
      }
    }
    //设置
    private void set(ThreadLocal<?> key, Object value) {
      ...
    }
    //删除
    private void remove(ThreadLocal<?> key) {
      ...
    }
    //查询
    private Entry getEntry(ThreadLocal<?> key) {
      ...
    }
  }
  ---------------------------------------------------------
}
```

**每个线程的本地变量不是存放到ThreadLocal实例里面的，而是存放到调用线程的threadLocals变量里面**

**ThreadLocal就是一个工具壳，它通过set方法把value值放入调用线程的threadLocals里面存放起来，当调用线程调用它的get方法时再从当前线程的threadLocals变量里面拿出来使用。**

# 4.使用ThreadLocal的案例

在dubbo中会先使用请求参数判断当前线程是否刚刚发起过同样参数的调用——这个调用会使ThreadLocalCache保存起来

```java
public class ThreadLocalCache implements Cache {
  private final ThreadLocal<Map<Object, Object>> store;
  public ThreadLocalCache(URL url) {
    this.store = new ThreadLocal<Map<Object, Object>>() {
      @Override
      protected Map<Object, Object> initialValue() {
        return new HashMap<Object, Object>();
      }
    };
  }
  @Override
  public void put(Object key, Object value) {
    store.get().put(key, value);
  }
  @Override
  public Object get(Object key) {
    return store.get().get(key);
  }
}
```

```java
public abstract class AbstractCacheFactory implements CacheFactory {
	//全局缓存,通过key=Cache【 String key = url.toFullString();】
  //在Cache缓存中存储有 key = 同一个请求的返回值
  //【String key = StringUtils.toArgumentString(invocation.getArguments());】
  private final ConcurrentMap<String, Cache> caches = new ConcurrentHashMap<String, Cache>();

  @Override
  public Cache getCache(URL url, Invocation invocation) {
    url = url.addParameter(Constants.METHOD_KEY, invocation.getMethodName());
    String key = url.toFullString();
    Cache cache = caches.get(key);
    if (cache == null) {
      caches.put(key, createCache(url));
      cache = caches.get(key);
    }
    return cache;
  }
  protected abstract Cache createCache(URL url);

}
```

**最常见的ThreadLocal使用场景为 ：用来解决 数据库连接、Session管理等。**

# 5.ThreadLocal导致的内存泄漏

ThreadLocalMap的key为ThreadLocal实例，他是一个弱引用，我们知道弱引用有利于GC的回收，当key == null时，GC就会回收这部分空间，但value不一定能被回收，因为他和Current Thread之间还存在一个强引用的关系。由于这个强引用的关系，会导致value无法回收，如果线程对象不消除这个强引用的关系，就可能会出现OOM。有些时候，我们调用ThreadLocalMap的remove()方法进行显式处理。

# 6.ThreadLocalRandom应用案例

java.util.Random是应用广泛的随机数生成工具类。根据JDK介绍，是线程安全的。

**单线程下的Random实现：**

> while循环中的CAS操作会保证只有一个线程可以更新老的种子为新的，失败的线程会通过循环重新获取更新后的种子作为当前种子去计算老的种子，保证了随机数的随机性

```java
protected int next(int bits) {
  long oldseed, nextseed;
  AtomicLong seed = this.seed;
  do {
    oldseed = seed.get();
    nextseed = (oldseed * multiplier + addend) & mask;
  } while (!seed.compareAndSet(oldseed, nextseed));
  return (int)(nextseed >>> (48 - bits));
}
```

**多线程下使用单个Random实例生成随机数，多个线程同时计算随机数计算新种子的时候，它们会竞争同一个原子变量的更新操作，因为原子变量的更新是CAS操作，同时只有一个线程会成功，所以会造成大量线程进行自旋重试，这是会降低并发性能**，于是ThreadLocalRandom应运而生。

```java
public class ThreadLocalRandom extends Random {
  ...
	private static final ThreadLocal<Double> nextLocalGaussian =
        new ThreadLocal<Double>();
  ...
}
```

