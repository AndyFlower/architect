## Static 静态变量 不能直接使用 @autowired标签的问题

### 1.问题原因
被static修饰变量，是不属于任何实例化的对象拥有，spring的依赖注入只能在对象层级上进行依赖注入，所以不能直接使用@autowired标签进行注入。

### 2.解决方案

#### 2.1 在静态方法中使自定义的工具类，该工具类实现ApplicationContextAware ，在该工具类中通过applicationContext.getBean 来湖区想要的bean类。

```java
public class SpringContextUtil implements ApplicationContextAware {
    private static ApplicationContext applicationContext = null;
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        SpringContextUtil.applicationContext = applicationContext;
    }
    public static ApplicationContext getApplicationContext() {
        return applicationContext;
    }
    public static Object getBean(String beanName) {

        return applicationContext.getBean(beanName);
    }
    public static Object getBeanByNull(String beanName) {

        try {
            return applicationContext.getBean(beanName);
        }
        catch (NullPointerException e)
        {
            return null;
        }
    }
    public static boolean containsBean(String beanName) {
        return applicationContext.containsBean(beanName);
    }
}

```

#### 2.2 使用@autowired 标签进行set方法注入。

具体为，将需要视同bean申明为静态的全局变量， 在通过set方法注入，在静态方法中就能直接使用该变量。

```java
@Component
public class TestClass{
    private static FirstClass firstClass;
    
    @Autowired
    public void setFirstClass(FirstClass firstClass){
        TestClass.firstClass = firstClass;
    }
    public static void print(){
        firstClass.print();
    }
}
```

#### 2.3 使用@PostConstruct注解

```java
@Component
public class TestClass{
    @Autowired
    private FirstClass firstClass;;
    
    private static FirstClass firstClass2;
    @PostConstruct
    public void init(){
        firstClass2=firstClass;
    }
    public static void print(){
        firstClass2.print();
    }
}
```

被@PostConstruct修饰的方法会在服务器加载Servlet的时候运行，并且只会被服务器调用一次，类似于Servlet的init()方法。被@PostConstruct修饰的方法会在构造函数之后，init()方法之前运行。