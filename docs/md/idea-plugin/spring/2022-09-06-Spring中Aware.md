## 一、什么是Aware

```java
public interface Aware {
}
```

**Aware接口是标记接口,用于标记Bean有资格通过回调拦截方法由Spring容器通知特定对象,例如Bean实现了ApplicationContextAware接口,Spring将向其中注入一个ApplicationContext对象.**

常见的还有BeanFactoryAware,ApplicationContextAware

## 二、常见用法

```java
/**
 * 以静态变量保存Spring ApplicationContext, 可在任何代码任何地方任何时候取出ApplicationContext.
 */
@Component
@Lazy(false)
public class SpringContextHolder implements ApplicationContextAware, DisposableBean {

    private static ApplicationContext applicationContext = null;

    private static Logger logger = LoggerFactory.getLogger(SpringContextHolder.class);

    /**
     * 取得存储在静态变量中的ApplicationContext.
     */
    public static ApplicationContext getApplicationContext() {
        assertContextInjected();
        return applicationContext;
    }

    /**
     * 从静态变量applicationContext中取得Bean, 自动转型为所赋值对象的类型.
     */
    @SuppressWarnings("unchecked")
    public static <T> T getBean(String name) {
        assertContextInjected();
        return (T) applicationContext.getBean(name);
    }

    /**
     * 从静态变量applicationContext中取得Bean, 自动转型为所赋值对象的类型.
     */
    public static <T> T getBean(Class<T> requiredType) {
        assertContextInjected();
        return applicationContext.getBean(requiredType);
    }

    /**
     * 清除SpringContextHolder中的ApplicationContext为Null.
     */
    public static void clearHolder() {
        if (logger.isDebugEnabled()) {
            logger.debug("清除SpringContextHolder中的ApplicationContext:" + applicationContext);
        }
        applicationContext = null;
    }

    /**
     * 实现ApplicationContextAware接口, 注入Context到静态变量中.
     */
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) {
        SpringContextHolder.applicationContext = applicationContext;
    }

    /**
     * 实现DisposableBean接口, 在Context关闭时清理静态变量.
     */
    @Override
    public void destroy() throws Exception {
        SpringContextHolder.clearHolder();
    }

    /**
     * 检查ApplicationContext不为空.
     */
    private static void assertContextInjected() {
        Validate.validState(applicationContext != null,
                "applicationContext属性未注入, 请在applicationContext.xml中定义SpringContextHolder.");
    }

    /**
     * 获取aop代理对象
     *
     * @param invoker
     * @return
     */
    public static <T> T getAopProxy(T invoker)
    {
        return (T) AopContext.currentProxy();
    }
}
```

```java
RedisTemplate redisTemplate = SpringContextHolder.getBean(StringRedisTemplate.class);
```

