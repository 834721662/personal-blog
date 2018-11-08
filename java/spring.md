[TOP]

---
#Spring

###Spring获取bean的方式
参考:https://www.cnblogs.com/yjbjingcha/p/6752265.html
```java
public class SpringContextHolder implements ApplicationContextAware {

    private static ApplicationContext applicationContext;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        SpringContextHolder.applicationContext = applicationContext;
    }
    
    public static <T> T getBean(Class<T> requiredType) {
        return applicationContext.getBean(requiredType);
    }

    public static <T> T getBean(String name) {
        return (T) applicationContext.getBean(name);
    }

    public static <T> T getBean(String name, Class<T> requiredType) {
        return applicationContext.getBean(name, requiredType);
    }

    public static <T> T getBean(String name, Object... args) {
        return (T) applicationContext.getBean(name, args);
    }

    public static <T> T getBean(Class<T> requiredType, Object... var2) throws BeansException {
        return applicationContext.getBean(requiredType, var2);
    }

    public static void setContext(ApplicationContext applicationContext) {
        SpringContextHolder.applicationContext = applicationContext;
    }
}
```