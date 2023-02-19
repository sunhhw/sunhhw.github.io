## ThreadLocal父子间通信的四种解决方案

ThreadLocal 是存储在线程栈帧中的一块数据存储区域，其可以做到线程与线程之间的读写隔离。

但是在我们的日常场景中，经常会出现父线程需要向子线程中传递消息，而 ThreadLocal 仅能在当前线程上进行数据缓存，这里就介绍4种父子间通信问题;

- 在子线程中手动设置父线程的值
- ThreadPoolTaskExecutor + TaskDecorator
- InheritableThreadLocal
- TransmittableThreadLocal

### 1.在子线程中手动设置父线程的值

``` java
ThreadLocal<String> threadLocal = new ThreadLocal<>();

@BeforeEach
public void init() {
  threadLocal.set("thread-01");
}

@Test
public void test4() {
  String s = threadLocal.get();
  new Thread(() -> {
    threadLocal.set(s);
    System.out.println(threadLocal.get());
  }).start();
  threadLocal.remove();
}
```

在子线程里手动的设置变量,@BeforeEach是junit5的写法,对应junit4的Before

```java
输出结果: thread-01
```

### 2.ThreadPoolTaskExecutor + TaskDecorator

使用ThreadPoolTaskExecutor线程池的时候,可自定义一个TaskDecorator包装类,这个类的作用就是在执行子线程之前手动的设置父线程的变量,跟第一种方法类似;

- 储存线程用户信息

```java
public class UserContextUtils {

    private static final ThreadLocal<String> userThreadLocal = new ThreadLocal<>();

    public static void set(String username) {
        userThreadLocal.set(username);
    }

    public static String get() {
        return userThreadLocal.get();
    }

    public static void clear() {
        userThreadLocal.remove();
    }

}
```

- 这是一个执行回调方法的装饰器，主要应用于传递上下文，或者提供任务的监控/统计信息

```java
public class ContextTaskDecorator implements TaskDecorator {
    @Override
    public Runnable decorate(Runnable runnable) {
        String username = UserContextUtils.get();
        return () -> {
            try {
                // 将主线程的请求信息，设置到子线程中
                UserContextUtils.set(username);
                // 执行子线程，这一步不要忘了
                runnable.run();
            } finally {
                // 线程结束，清空这些信息，否则可能造成内存泄漏
                UserContextUtils.clear();
            }
        };
    }
}
```

- 初始化线程池

```java
@Bean
public Executor getAsyncExecutor() {
  ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
  executor.setCorePoolSize(4);
  executor.setMaxPoolSize(8);
  executor.setQueueCapacity(100);
  executor.setAllowCoreThreadTimeOut(false);
  executor.setKeepAliveSeconds(0);
  executor.setThreadNamePrefix("DefaultAsync-");
  executor.setTaskDecorator(new ContextTaskDecorator());
  executor.setWaitForTasksToCompleteOnShutdown(true);
  executor.initialize();
  System.out.println("初始化Async的线程池");
  return executor;
}
```

- 测试

```java
@BeforeEach
public void init() {
  threadLocal.set("thread-01");
  UserContextUtils.set("taskExecutor");
}

@Resource
private ThreadPoolTaskExecutor getAsyncExecutor;

@Test
public void test3() {
  getAsyncExecutor.execute(()->{
    System.out.println(UserContextUtils.get());
  });
}
```

使用ThreadPoolTaskExecutor线程池的时候，使用构造器在子线程写入主线程参数,但是使用ThreadPoolExecutor就不能这么做了,建议使用第四种方式TTL;

### 3.InheritableThreadLocal

inheritableThreadLocal是ThreadLocal中自带的一种方法,只要替换原来的ThreadLocal就行了,但是这种方法有缺陷,会存在核心线程旧值的重复使用,不建议使用;

这里我设置一个线程池,核心线程数为2个,核心线程数重复使用的时候不会重新拿新值,而是用原来的旧值

```java
@Test
public void test1() {
  //1.创建一个自己定义的线程池
  ExecutorService executorService = new ThreadPoolExecutor(2, 3, 0, TimeUnit.MILLISECONDS, new SynchronousQueue<>());
  InheritableThreadLocal<String> threadLocal = new InheritableThreadLocal<>();
  
  threadLocal.set("thread-01");
  executorService.execute(() -> {
    String s = threadLocal.get();
    System.out.println(s);
  });

  threadLocal.set("thread-02");
  executorService.execute(() -> {
    String s = threadLocal.get();
    System.out.println(s);
  });

  threadLocal.set("thread-03");
  executorService.execute(() -> {
    String s = threadLocal.get();
    System.out.println(s);
  });

  threadLocal.set("thread-04");
  executorService.execute(() -> {
    String s = threadLocal.get();
    System.out.println(s);
  });
}
```

测试结果: 因为线程2被重复使用

```
thread-01
thread-02
thread-02
thread-02
```

### 4.TransmittableThreadLocal

**TransmittableThreadLocal** 是Alibaba开源的、用于解决 **“在使用线程池等会缓存线程的组件情况下传递ThreadLocal”** 问题的 InheritableThreadLocal 扩展。若希望 TransmittableThreadLocal 在线程池与主线程间传递，需配合 *TtlExecutors.getTtlExecutorService*,*TtlRunnable* 和 *TtlCallable* 使用。

- 引入TTL的jar包

```java
 <dependency>
   <groupId>com.alibaba</groupId>
   <artifactId>transmittable-thread-local</artifactId>
   <version>2.12.6</version>
 </dependency>
```

- **TtlExecutors.getTtlExecutorService()**

使用Ttl提供的TtlExecutors.getTtlExecutorService来对原来线程池进行包装,但是此时变量需要使用`TransmittableThreadLocal`,建议使用这种方式;

```java
@Test
public void test2() {
  // 1. 创建一个自己定义的线程池
  ExecutorService executorService = new ThreadPoolExecutor(2, 3, 0, TimeUnit.MILLISECONDS, new SynchronousQueue<>());
  // 2. 使用TransmittableThreadLocal修饰变量
  TransmittableThreadLocal<String> threadLocal1 = new TransmittableThreadLocal<>();
  // 3. 使用TtlExecutors.getTtlExecutorService包装线程池
  ExecutorService ttlExecutorService = TtlExecutors.getTtlExecutorService(executorService);

  threadLocal1.set("thread-01");
  ttlExecutorService.execute(() -> {
    String s = threadLocal1.get();
    System.out.println(s);
  });

  threadLocal1.set("thread-02");
  ttlExecutorService.execute(() -> {
    String s = threadLocal1.get();
    System.out.println(s);
  });

  threadLocal1.set("thread-03");
  ttlExecutorService.execute(() -> {
    String s = threadLocal1.get();
    System.out.println(s);
  });

  threadLocal1.set("thread-04");
  ttlExecutorService.execute(() -> {
    String s = threadLocal1.get();
    System.out.println(s);
  });

}
```

```java
thread-01
thread-02
thread-03
thread-04
```

- **TtlRunnable.get()**

```java
@Test
public void test5() {
  // 1. 创建一个自己定义的线程池
  ExecutorService executorService = new ThreadPoolExecutor(2, 3, 2, TimeUnit.MILLISECONDS, new SynchronousQueue<>());
  // 2. 使用TransmittableThreadLocal修饰变量
  TransmittableThreadLocal<String> threadLocal1 = new TransmittableThreadLocal<>();

  threadLocal1.set("thread-01");
  executorService.execute(TtlRunnable.get(() -> {
    String s = threadLocal1.get();
    System.out.println(s);
  }));

  threadLocal1.set("thread-02");
  executorService.execute(TtlRunnable.get(() -> {
    String s = threadLocal1.get();
    System.out.println(s);
  }));

  threadLocal1.set("thread-03");
  executorService.execute(TtlRunnable.get(() -> {
    String s = threadLocal1.get();
    System.out.println(s);
  }));

  threadLocal1.set("thread-04");
  executorService.execute(TtlRunnable.get(() -> {
    String s = threadLocal1.get();
    System.out.println(s);
  }));

}
```

```java
thread-01
thread-02
thread-03
thread-04
```

