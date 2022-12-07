

# 第5章 Dubbo的内核解析

## 5.1 JDK的SPI机制

### 5.1.1 简介

SPI：Service Provider Interface，服务提供者接口，是一种服务发现的机制

### 5.1.2 JDK SPI规范

* 接口名：可随意定义
* 实现类名：可随意定义
* 提供者配置文件路径：META-INF/services
* 提供者配置文件名称：接口的全限定类名，没有扩展名
* 提供者配置文件内容：接口**实现类**的**全限定类名**，一个类名占一行

### 5.1.3 代码举例

```java
// 接口
package com.abc.service;

public interface SomeService {
    void doSome();
}
// 实现类
package com.abc.service;

public class OneServiceImpl implements SomeService{
    @Override
    public void doSome() {
        System.out.println("执行OneServiceImpl的doSome方法");
    }
}
// 实现类
package com.abc.service;

public class TwoServiceImpl implements SomeService{
    @Override
    public void doSome() {
        System.out.println("执行TwoServiceImpl的doSome方法");
    }
}
```

```java
public class SPITest {
    public static void main(String[] args) {
        // 加载提供者配置文件，创建提供者类的加载器
        ServiceLoader<SomeService> load = ServiceLoader.load(SomeService.class);
        // ServiceLoader本身是一个迭代器
        Iterator<SomeService> iterator = load.iterator();

        // 迭代加载每一个实现类，并生成一个提供者对象
        while (iterator.hasNext()) {
            SomeService service = iterator.next();
            service.doSome();
        }
    }
}
```

配置文件内容：com.abc.service.SomeService

```
com.abc.service.OneServiceImpl
com.abc.service.TwoServiceImpl
```

### 5.1.4 代码分析

```java
public final class ServiceLoader<S> implements Iterable<S>{
   // 默认加载配置文件路径
   private static final String PREFIX = "META-INF/services/";
   // 接口类
   private final Class<S> service;
   // 类的加载器
   private final ClassLoader loader;
   // 实现类实例缓存
   private LinkedHashMap<String,S> providers = new LinkedHashMap<>();
   // 内部迭代器
   private LazyIterator lookupIterator;
   private class LazyIterator Implements Iterator<S>{
       Class<S> service;
       ClassLoader loader;
       Enumeration<URL> configs = null;
       Iterator<String> pending = null;
       String nextName = null;
   }
}
// 该对象会在iterator.next()方法后获取对象

```

## 5.2 Dubbo 的SPI

### 5.2.1 规范说明

* 接口名：随意定义，必须采用@SPI("dubbo")注解，注解当中value为默认实现key

* 实现类名：在接口名前添加一个用于表示自身功能的 “标识前缀”字符串

* 配置文件路径：依次查找目录为

  META-INF/dubbo/internal

  META-INF/dubbo

  META-INF/services

* 配置文件名称：接口的全限定名，无需扩展名

* 配置文件内容：key=value格式，key可以随意，一般为标识前缀，value为全限定类名

* 提供者加载：ExtensionLoader.getExtensionLoader(Class clazz)

### 5.2.2 Dubbo SPI 示例

SPI 接口：

```java
@SPI("alipay")// 指定默认key
public interface Order{}
```

定义扩展类：

```java
public class AlipayOrder implements Order{
    @Override
    public String way() {
        System.out.println("---- 支付宝way()------");
        return "支付宝支付方式";
    }
}
public class WeChatOrder implements Order{
    @Override
    public String way() {
        System.out.println("---- 微信way()------");
        return "微信支付方式";
    }
}
```

配置文件内容：com.abc.service.Order

```
alipay=com.abc.service.AlipayOrder
wechat=com.abc.service.WeChatOrder
```

调用示例：

```java
public class OrderTest {
    public static void main(String[] args) {
        ExtensionLoader<Order> loader = ExtensionLoader.getExtensionLoader(Order.class);
		Order alipay = loader.getExtension("alipay");
        System.out.println("支付宝：" + alipay.way());
        Order wechat = loader.getExtension("wechat");
        System.out.println("微信：" + wechat.way());
    }
}
```

## 5.3 Adaptive机制

### 5.3.1 简介

Adaptive机制是Dubbo的自适应机制，可以自动适配要调用哪个接口实现类的方法。该机制分为两种类型：

* Adaptive类实现：需要自定义一个Adaptive类，根据传入的参数匹配要创建的实现类
* Adaptive方法实现：根据URL参数创建匹配的实现类，代理类会根据javaasist自动创建

### 5.3.2 @Adaptive注解

#### 1. @Adaptive修饰类

被@Adaptive修饰的SPI接口扩展类称为Adaptive类，表示该SPI扩展类会按照该类中的指定方式获取，即用于固定实现方式，其是装饰者设计模式的实现。

在Dubbbo中只有两个这样的类 AdaptiveCompiler，AdaptiveExtensionFactory

#### 2. @Adaptive修饰方法

被@Adaptive修饰的方法称为Adaptive方法，该注解会用于接口方法中。在SPI扩展类中若没有找到 Adaptive类，但系统却发现了Adaptive方法，就会根据Adaptive方法自动为该SPI接口动态生成Adaptive类，并自动将其编译。

### 5.3.3 Adaptive类示例

Adaptive类

```java
@Adaptive
public class AdaptiveOrder implements Order {
    private String orderName;
    public void setOrderName(String orderName) {
        this.orderName = orderName;
    }
    @Override
    public String way() {
        ExtensionLoader<Order> loader = ExtensionLoader.getExtensionLoader(Order.class);
        String name = orderName;
        Order order;
        if (StringUtils.isEmpty(name)) {
            // 若没有指定参数则获取默认的扩展类
            order = loader.getDefaultExtension();
        } else {
            order = loader.getExtension(name);
        }
        return order.way();
    }
}
```

配置文件：com.abc.service.Order

```
alipay=com.abc.service.AlipayOrder
wechat=com.abc.service.WeChatOrder
adaptive=com.abc.service.AdaptiveOrder
```

### 5.3.4 Adaptive方法示例

Adaptive方法：

```java
@SPI("alipay")
public interface Order {
    String way();

    @Adaptive
    String pay(URL url);
}
```

测试方法：

```java
public class OrderTest {
    @Test
    public void test01(){
        ExtensionLoader<Order> loader = ExtensionLoader.getExtensionLoader(Order.class);
        // 获取自适应扩展类
        Order adaptive = loader.getAdaptiveExtension();
        // 模拟一个URL
        URL url = URL.valueOf("http://localhost/aaa");
        System.out.println(adaptive.pay(url));
    }

    @Test
    public void test02(){
        ExtensionLoader<Order> loader = ExtensionLoader.getExtensionLoader(Order.class);
        // 获取自适应扩展类
        Order adaptive = loader.getAdaptiveExtension();
        // 模拟一个URL
        // 当SPI接口为 GoodsOrder 此时URL为 http://localhost/aaa?goods.order=wechat
        URL url = URL.valueOf("http://localhost/aaa?order=wechat");
        System.out.println(adaptive.pay(url));
    }
}
```

配置文件：

```
alipay=com.abc.service.AlipayOrder
wechat=com.abc.service.WeChatOrder
```



URL中参数解析：

* key会根据SPI接口生成：StringUtils.camelToSplitName(type.getSimpleName(), ".");

产生的动态代理类为：

```java
package com.abc.service;

import org.apache.dubbo.common.extension.ExtensionLoader;

public class Order$Adaptive implements com.abc.service.Order {

    public java.lang.String pay(org.apache.dubbo.common.URL arg0) {
        if (arg0 == null) throw new IllegalArgumentException("url == null");
        org.apache.dubbo.common.URL url = arg0;
        // 这里的order来自于StringUtils.camelToSplitName(type.getSimpleName(), ".");
        String extName = url.getParameter("order", "alipay");
        if (extName == null)
            throw new IllegalStateException("Failed to get extension (com.abc.service.Order) name from url (" + url.toString() + ") use keys([order])");
        com.abc.service.Order extension = (com.abc.service.Order) ExtensionLoader.getExtensionLoader(com.abc.service.Order.class).getExtension(extName);
        return extension.pay(arg0);
    }

    public java.lang.String way() {
        throw new UnsupportedOperationException("The method public abstract java.lang.String com.abc.service.Order.way() of interface com.abc.service.Order is not adaptive method!");
    }
}
```

## 5.4 Wrapper机制

### 5.4.1 Wrapper机制规范

Wrapper机制不是根据注解实现的，而是通过一套公共契约实现的

* 该类要实现SPI接口
* 该类要有SPI接口类的引用
* **该类要有一个包含SPI接口参数的带参构造器**
* 该类名称要以Wrapper结尾

### 5.4.2 代码示例

```
public class OrderWrapper implements Order{
    private Order order;

    public OrderWrapper(Order order) {
        this.order = order;
    }

    @Override
    public String way() {
        System.out.println("before-这是wrapper对way()的增强");
        String way = order.way();
        System.out.println("after-这是wrapper对way()的增强");
        return way;
    }

    @Override
    public String pay(URL url) {
        System.out.println("before-这是wrapper对pay()的增强");
        String pay = order.pay(url);
        System.out.println("after-这是wrapper对pay()的增强");
        return pay;
    }
}

public class OrderWrapper2 implements Order{
    private Order order;

    public OrderWrapper2(Order order) {
        this.order = order;
    }

    @Override
    public String way() {
        System.out.println("before2-这是wrapper对way()的增强");
        String way = order.way();
        System.out.println("after2-这是wrapper对way()的增强");
        return way;
    }

    @Override
    public String pay(URL url) {
        System.out.println("before2-这是wrapper对pay()的增强");
        String pay = order.pay(url);
        System.out.println("after2这是wrapper对pay()的增强");
        return pay;
    }
}
```

配置文件：

```
alipay=com.abc.service.AlipayOrder
wechat=com.abc.service.WeChatOrder
wrapper=com.abc.service.OrderWrapper
wrapper2=com.abc.service.OrderWrapper2
```

测试用例：

```java
public class OrderTest {

    @Test
    public void test01(){
        ExtensionLoader<Order> loader = ExtensionLoader.getExtensionLoader(Order.class);
        // 获取自适应扩展类
        Order adaptive = loader.getAdaptiveExtension();
        // 模拟一个URL
        URL url = URL.valueOf("http://localhost/aaa");
        System.out.println(adaptive.pay(url));
    }

    @Test
    public void test02(){
        ExtensionLoader<Order> loader = ExtensionLoader.getExtensionLoader(Order.class);
        // 获取自适应扩展类
        Order adaptive = loader.getAdaptiveExtension();
        // 模拟一个URL
        URL url = URL.valueOf("http://localhost/aaa?order=wechat");
        System.out.println(adaptive.pay(url));
    }

    @Test
    public void test03(){
        ExtensionLoader<Order> loader = ExtensionLoader.getExtensionLoader(Order.class);
        Order order = loader.getExtension("alipay");
        System.out.println(order.way());
    }
}
```

### 5.4.3 Wrapper原理

```java
private T createExtension(String name) {
        Class<?> clazz = getExtensionClasses().get(name);
        if (clazz == null) {
            throw findException(name);
        }
        try {
        	// 创建扩展类
            T instance = (T) EXTENSION_INSTANCES.get(clazz);
            if (instance == null) {
                EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
                instance = (T) EXTENSION_INSTANCES.get(clazz);
            }
            // 通过IOC注入set属性
            injectExtension(instance);
            Set<Class<?>> wrapperClasses = cachedWrapperClasses;
            if (CollectionUtils.isNotEmpty(wrapperClasses)) {
                for (Class<?> wrapperClass : wrapperClasses) {
                    // *****如果当前loader存在包装类，则进行包装*****
                    instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
                }
            }
            return instance;
        } catch (Throwable t) {
            throw new IllegalStateException("Extension instance (name: " + name + ", class: " +
                    type + ") couldn't be instantiated: " + t.getMessage(), t);
        }
    }
```

## 5.5 Activate 机制

activate机制 用于一次性获取多个扩展类，通过@Activate(group={},value={})

### 5.5.1 @Activate注解

* group：为扩展类指定所属的组类别，是扩展类的一个标识
* value：为扩展类指定key，是当前扩展类的一个标识
* order：指定筛选条件相同时的加载顺序，默认为0，数字越小优先级越高

### 5.5.2 代码示例

```java
@Activate(group = "online")
public class AlipayOrder implements Order{
    @Override
    public String way() {
        System.out.println("---- 支付宝way()------");
        return "支付宝支付方式";
    }
}
@Activate(group = {"online","offline"},order = 3)
public class CardOrder implements Order{
    @Override
    public String way() {
        System.out.println("---- 银行卡way()------");
        return "银行卡支付方式";
    }
}
@Activate(group = "offline" ,order = 4)
public class CashOrder implements Order{
    @Override
    public String way() {
        System.out.println("---- 现金way()------");
        return "现金支付方式";
    }
}
@Activate(group = "offline" ,order = 5)
public class CouponOrder implements Order{
    @Override
    public String way() {
        System.out.println("---- 购物券way()------");
        return "购物券支付方式";
    }
}
```

测试用例：

```java
public class OrderTest {
    @Test
    public void test05() {
        ExtensionLoader<Order> loader = ExtensionLoader.getExtensionLoader(Order.class);
        URL url = URL.valueOf("http://localhost/ooo?order=mobile");

        List<Order> activateExtension = loader.getActivateExtension(url, "order", "online");
        for (Order order : activateExtension) {
            System.out.println(order.way());
        }
    }
}
```

**分析：**

```java
public List<T> getActivateExtension(URL url, String key, String group);
// key：获取匹配扩展类之后，另外增加返回类的 扩展key，该key对应的value会从url中获取
// group：匹配要筛选的group，如果@Activate(group={},value={key1,key2})
// 如果group匹配上后，要根据value中的v1和v2进行url参数匹配
// 例如: url = http://localhost/xxx?order=alipay&key1=value1&a_key=a_value
// 那么会根据key1和 参数中key1进行匹配，如果匹配上且value1不为空，则表示符合条件
```

## 5.6 总结

* 扩展文件中存在四种类：普通扩展类，Adaptive类，Wrapper类，Activate类
* SPI接口中只会存在一个Adaptive类
* Adaptive类和Activate类都是通过注解实现的，Wrapper类是通过公共契约实现的
* 只有普通类和Activate类属于扩展类的范畴

##  5.7 Dubbo的SPI源码解析

### 5.7.1 总思路

![image-20221124212722010](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221124212722010.png)

### 5.7.2 SPI原理分析

![image-20221125112018240](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221125112018240.png)

### 5.7.3 ServiceLoader 属性分析

```java
public class ExtensionLoader<T> {
    // 定义获取配置文件的路径
    private static final String SERVICES_DIRECTORY = "META-INF/services/";
    private static final String DUBBO_DIRECTORY = "META-INF/dubbo/";
    private static final String DUBBO_INTERNAL_DIRECTORY = DUBBO_DIRECTORY + "internal/";
	// 定义名称分隔符号
    private static final Pattern NAME_SEPARATOR = Pattern.compile("\\s*[,]+\\s*");
	// 静态变量缓存ExtensionLoader对象
    private static final ConcurrentMap<Class<?>, ExtensionLoader<?>> EXTENSION_LOADERS = new ConcurrentHashMap<>();
	// 静态变量缓存扩展类对象 这里缓存的是原生对象，并没有经过Wrapper代理增强
    private static final ConcurrentMap<Class<?>, Object> EXTENSION_INSTANCES = new ConcurrentHashMap<>();
    // -----------------------------
    // SPI接口类型
    private final Class<?> type; 
    // 对象工厂，默认为AdaptiveExtensionFactory
    private final ExtensionFactory objectFactory;
	// 缓存扩展类的名称 缓存第一个扩展名与扩展类
    private final ConcurrentMap<Class<?>, String> cachedNames = new ConcurrentHashMap<>();
	// 缓存扩展类的class文件 key为扩展名
    // 扩展名来源1.配置文件中的key 2.Extension注解value值 3.(实例简单名称-SPI接口名称)小写
    private final Holder<Map<String, Class<?>>> cachedClasses = new Holder<>();
	// 缓存activate key为 扩展名，value为 Activate注解对象，这里缓存第一个名称
    // a,b=com.xxx 这里只存储a=com.xxx
    private final Map<String, Object> cachedActivates = new ConcurrentHashMap<>();
    // 缓存扩展类的实例（wrapper增强过得）
    private final ConcurrentMap<String, Holder<Object>> cachedInstances = new ConcurrentHashMap<>();
    // 自适应对象 只会存在一个
    private final Holder<Object> cachedAdaptiveInstance = new Holder<>();
    // 自适应类，只会存在一个
    private volatile Class<?> cachedAdaptiveClass = null;
    // SPI默认实现类
    private String cachedDefaultName;
    // 缓存wrapperclass类
    private Set<Class<?>> cachedWrapperClasses;
}
```

## 5.8 Dubbo的IOC源码分析

### 5.8.1 Dubbo IOC第一个入口（能够执行set方法）

dubbo暴露服务时需要获取ZK的客户端，进行连接ZK

ZookeeperDynamicConfigurationFactory implements  DynamicConfigurationFactory：

其中有一个属性 zookeeperTransporter 是通过IOC注入的，创建该对象的流程为：

```java
ServiceConfig#export()
ServiceConfig#checkAndUpdateSubConfigs()
	AbstractInterfaceConfig#checkRegistry()
    AbstractInterfaceConfig#useRegistryForConfigIfNecessary()
    AbstractInterfaceConfig#startConfigCenter()
    AbstractInterfaceConfig#prepareEnvironment()
    AbstractInterfaceConfig#getDynamicConfiguration()
    	ExtensionLoader#getExtension(String name)
    	ExtensionLoader#createExtension(String name)
    	// 在这里通过反射和ExtensionLoader的objectFactory进行注入
    	ExtensionLoader#injectExtension(T instance)
    	
```

总结：通过SPI获取extension对象时，会进行IOC注入

## 5.9 Dubbo的AOP分析

### 5.9.1分析入口：

```java
ExtensionLoader#getExtension(String name)
   		//在这个方法中进行wrapper类包装
    	ExtensionLoader#createExtension(String name)
            if (CollectionUtils.isNotEmpty(wrapperClasses)) {
                    // 遍历所有wrapper，逐层包装instance
                    for (Class<?> wrapperClass : wrapperClasses) {
                        instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
                    }
                }
```

## 5.10 Dubbo的动态编译

Dubbo通过自适应方法获取自适应类时是通过javassist获取的

### 5.10.1 入口

ServiceConfig<T>

```java
private static Procotol Protocol = ExtensionLoader.getLoader(Protocol.class).getAdaptiveExtension();
ExtensionLoader#createAdaptiveExtension()
ExtensionLoader#getAdaptiveExtensionClass()
// 在内存中生成代码并进行编译加载，默认采用的编译器为 JavassistCompiler
ExtensionLoader#createAdaptiveExtensionClass()
```

# 第6章 Dubbo的源码解析

## 6.1 Dubbo与Spring整合

### 6.1.1 查找解析器

Spring中支持自定义标签的解析：通过定义spring.handlers配置文件，用于定义解析自定义命名空间的标签

META-INF/spring.handlers

```xml-dtd
http\://dubbo.apache.org/schema/dubbo=org.apache.dubbo.config.spring.schema.DubboNamespaceHandler
http\://code.alibabatech.com/schema/dubbo=org.apache.dubbo.config.spring.schema.DubboNamespaceHandler
```

META-INF/spring-schemas

```java
//定义xsd文件路径
http\://dubbo.apache.org/schema/dubbo/dubbo.xsd=META-INF/dubbo.xsd
http\://code.alibabatech.com/schema/dubbo/dubbo.xsd=META-INF/compat/dubbo.xsd
```

### 6.1.2 Dbuubo标签解析

```java
public class DubboNamespaceHandler extends NamespaceHandlerSupport {
    static {
        Version.checkDuplicate(DubboNamespaceHandler.class);
    }
    @Override
    public void init() {
        registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
        registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class, true));
        registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));
        registerBeanDefinitionParser("config-center", new DubboBeanDefinitionParser(ConfigCenterBean.class, true));
        registerBeanDefinitionParser("metadata-report", new DubboBeanDefinitionParser(MetadataReportConfig.class, true));
        registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class, true));
        registerBeanDefinitionParser("metrics", new DubboBeanDefinitionParser(MetricsConfig.class, true));
        registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));
        registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));
        registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));
        registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
        registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, false));
        registerBeanDefinitionParser("annotation", new AnnotationBeanDefinitionParser());
    }
```

解析Dubbo标签的方法：

```java
public class DubboBeanDefinitionParser implements BeanDefinitionParser{
    // 解析Dubbo标签
    private static BeanDefinition parse(Element element, ParserContext parserContext, Class<?> beanClass, boolean required) {
    }
}
```

## 6.2 重要接口

### 6.2.1 Invocation

其中封装了远程调用的具体信息

![image-20221128194012009](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221128194012009.png)

### 6.2.2 Invocker

提供者provider 的代理对象，在代码中就代表提供者。在消费者进行远程调用时通过服务路由，负载均衡，集群容错等机制要查找的就是Invoker，找到了其需要的Invoker实例就可以进行远程调用了。

![image-20221128194444909](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221128194444909.png)

### 6.2.3 Exporter

服务暴露的对象，其中包含一个很重要的方法getInvoker()，用于获取当前服务暴露实例所包含的远程调用实例Invoker，即可以进行远程调用。unexport()方法会使服务不进行暴露

![image-20221128195008851](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221128195008851.png)

### 6.2.4 Directory

Directory中有一个很重要的方法list()，其返回结果为List<Invoker> 可以将Directory理解为一个动态的Invoker列表

![image-20221128195152468](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221128195152468.png)

## 6.3 服务发布

服务发布的流程图为：

![image-20221128222431805](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221128222431805.png)

![image-20221128222502117](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221128222502117.png)

### 6.3.1 服务发布入口

```java
public class ServiceBean{
	// 服务发布入口
	public void export();
	publishExportEvent
}
```

## 6.4 服务订阅

### 6.4.1服务订阅流程

服务订阅的流程就是消费者端创建代理对象的过程

```java
ReferenceConfig#get();
// 初始化过程会创建ref引用对象
ReferenceConfig#init();
/**该方法的处理过程
1.判断是本地调用则创建InJvmInvoker
InJvmInvoker包装类为 ProtocolListenerWrapper#refer() -> ProtocolFilterWrapper#refer() -> InjvmProtocol(AbstractProtocol)#refer()
2.注册中心的调用
如果是dubbo协议 则创建DubboInvoker，包装类为
调用过程为 Cluster$Adaptive#join() -> (服务降级)MockClusterWrapper#join() -> （负载均衡/容错）FailoverCluster#join()
3.直连方式调用
如果是dubbo协议 则创建DubboInvoker(创建该对象时会创建Netty客户端)
4.根据产生的Inovker创建消费者代理对象
**/
ReferenceConfig#createProxy(Map<String,String> map);
  Protocol$Adaptive#ref(URL);
    RegsitryProtocol#refer(Class,URL);
	/**
	1.将cousumer注册到zk
	2.将所有router添加到directory
	3.更新Invoker列表
	**/
	RegsitryProtocol#doRef();
	  /*
	  更新zk上 providers,configurators,routers信息
	  更新invokers
	  */
	  RegistryProtocol#subscribe();
		FailbackRegistry#subscribe();
		  /*
		  创建zk中providers,configurators,routers节点并添加监听主动触发
		  */
		  ZookeeperRegistry#doSubscribe();
			// 更新本地invokers
			AbstractRegistry#notify();
			  RegistryDirectory#refreshInvoker();
				/*
				1.创建DubboInvoker
				*/
				RegistryDirectory#toInvokers();
				
```

## 6.5 远程调用

远程调用的入口：

![image-20221129212918008](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221129212918008.png)

### 6.5.1 远程调用流程

```java
// 入口调用
org.apache.dubbo.rpc.proxy.InvokerInvocationHandler#invoke();
// 服务降级
org.apache.dubbo.rpc.cluster.support.wrapper.MockClusterInvoker#invoke();
// 通过服务路由获取可用的invokers，获取负载均衡策略
org.apache.dubbo.rpc.cluster.support.AbstractClusterInvoker#invoke();
// 负载均衡容错
org.apache.dubbo.rpc.cluster.support.FailoverClusterInvoker#doInvoke(); 
org.apache.dubbo.rpc.protocol.InvokerWrapper#invoke();
org.apache.dubbo.rpc.listener.ListenerInvokerWrapper#invoke();
// 给调用的异步结果添加监听，做非主线的回调操作
org.apache.dubbo.rpc.protocol.ProtocolFilterWrapper#invoke();
org.apache.dubbo.rpc.filter.ConsumerContextFilter#invoke();
org.apache.dubbo.rpc.protocol.ProtocolFilterWrapper#invoke();
org.apache.dubbo.monitor.support.MonitorFilter#invoke();
//异步转同步结果，这里接收到调用结果为 AsyncRpcResult
//如果是同步等待，则会调用该结果的 get方法
org.apache.dubbo.rpc.protocol.AsyncToSyncInvoker#invoke();
org.apache.dubbo.rpc.protocol.AbstractInvoker#invoke();
// 这里会构造一个 AsyncRpcResult，并且订阅Netty返回结果 responseFuture(CompletableFuture类型)
// 如果responseFuture结束 会调用AsyncRpcResult.complete方法
org.apache.dubbo.rpc.protocol.dubbo.DubboInvoker#doInvoke();
org.apache.dubbo.rpc.protocol.dubbo.ReferenceCountExchangeClient#request();
org.apache.dubbo.remoting.exchange.support.header.HeaderExchangeClient#request();
// 这里会构建一个DefaultFuture(CompletableFuture子类)对象，并存储在上下文DefaultFuture.FUTURES
// 当有结果返回时，会从上下文中获取这个对象，并调用setComplete方法
// HeaderExchangeChannel中channel属性是 NettyClient对象
org.apache.dubbo.remoting.exchange.support.header.HeaderExchangeChannel#request();
org.apache.dubbo.remoting.transport.AbstractPeer#send();
// 在这里调用getChannel方法，获取channel后发送请求
org.apache.dubbo.remoting.transport.AbstractClient#send();

```

> 远程调用返回异常时，该异常的抛出是在proxy代理类中 InvokerInvocationHandler#invoke中的recreate方法中即 Result#recreate()方法

## 6.6 提供者处理消费者请求

### 6.6.1 入口

```java
org.apache.dubbo.remoting.transport.netty4.NettyServer$nettyServerHandler
```

流程分析

```java
// 将ByteBuf类型的消息转换为 Request对象类型
io.netty.handler.codec.channelRead();
org.apache.dubbo.remoting.transport.netty4.NettyServerHandler#channelRead();
// 判断channel是否关闭
org.apache.dubbo.remoting.transport.AbstractPeer#received();
org.apache.dubbo.remoting.transport.MultiMessageHandler#received();
// 增加multipart参数类型处理
org.apache.dubbo.remoting.exchange.support.header.HeartbeatHandler#received();
//增加是心跳响应处理类型
org.apache.dubbo.remoting.exchange.support.header.HeartbeatHandler#received();
//增加线程池处理流程
org.apache.dubbo.remoting.transport.dispatcher.all.AllChannelHandler#received();
org.apache.dubbo.remoting.transport.dispatcher.ChannelEventRunnable#run();
// 解码传递的message
org.apache.dubbo.remoting.transport.DecodeHandler#received();
// 可以处理Request和Response
org.apache.dubbo.remoting.exchange.support.header.HeaderExchangeHandler#received();
// 给调用返回值 增加一个whenComplete监听，当有返回值后发送返回结果给客户端
org.apache.dubbo.remoting.exchange.support.header.HeaderExchangeHandler#handleRequest();
// 获取真正的Invoker进行调用
org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol#reply();
org.apache.dubbo.rpc.protocol.ProtocolFilterWrapper#invoke();
org.apache.dubbo.rpc.protocol.ProtocolFilterWrapper#buildInvokerChain#invoke();
org.apache.dubbo.rpc.proxy.AbstractProxyInvoker#invoke();
org.apache.dubbo.rpc.proxy.javassist.getInvoker#doInvoke();
```

## 6.7  消费者处理提供者响应

NettyClient创建入口流程：

```java
org.apache.dubbo.rpc.protocol.AbstractProtocol#refer();
org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol#protocolBindingRefer();
org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol#getCLients();
// 根据URL创建ExchangeClient
org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol#initClient();
// 获取HeaderExchanger对象，并调用其connect方法
org.apache.dubbo.remoting.exchange.Exchangers#connect();
// 创建HeaderExchangeClient[属性为NettyClient]对象
org.apache.dubbo.remoting.exchange.support.header.HeaderExchanger#connect();
//获取Transporter自适应类调用其connect方法
org.apache.dubbo.remoting.Transporters#connect();
// 创建NettyClient对象
org.apache.dubbo.remoting.transport.netty4.NettyTransporter#connect();
```

接收服务端的响应流程：

```java
org.apache.dubbo.remoting.transport.netty4$nettyClientHandler;
org.apache.dubbo.remoting.transport.netty4.NettyClientHandler#channelRead();
org.apache.dubbo.remoting.transport.AbstractPeer#received();
org.apache.dubbo.remoting.transport.MultiMessageHandler#received();
// 增加处理心跳响应情况
org.apache.dubbo.remoting.exchange.support.header.HeartbeatHandler#received();
// 增加线程池去处理
org.apache.dubbo.remoting.transport.dispatcher.all.AllChannelHandler#received();
// 线程任务处理响应结果
org.apache.dubbo.remoting.transport.dispatcher.ChannelEventRunnable#run();
// 增加解码工作
org.apache.dubbo.remoting.transport.DecodeHandler#received();
// 
org.apache.dubbo.remoting.exchange.support.header.HeaderExchangeHandler#received();
// 增加Null值和心跳判断
org.apache.dubbo.remoting.exchange.support.header.HeaderExchangeHandler#handleResponse();
// 获取缓存对象中的future对象，并调用其中方法
org.apache.dubbo.remoting.exchange.support.DefaultFuture#received();
// 调户消费者端 Future对象的complete方法
org.apache.dubbo.remoting.exchange.support.DefaultFuture#doReceived();
```

## 6.8 服务路由

### 6.8.1 什么是服务路由

服务路由包含一条路由规则，路由规则决定了服务消费者的调用目标。

Dubbo目前提供了三种服务路由实现，分别为条件路由ConditionRouter，脚本路由 ScriptRouter和标签路由 TagRouter。其中条件路由是我们最常使用的。

### 6.8.2 路由规则的设置

路由规则是在Dubbo管控平台 Dubbo-Admin中设置的，会上传到Dubbo-Admin的管控平台

#### A 启动管理台

* 更改dubbo-admin-devlop 配置信息(zk信息，启动端口)
* 打包为jar并，进行启动访问

#### B访问 设置路由规则

![image-20221204152317053](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221204152317053.png)



### 6.8.3 路由规则详解

#### (1) 初识路由规则

应用粒度路由规则：

```yaml
scope: application
force: true
runtime: true
enabled: true
key: governance-conditionrouter-consumer
conditions:
# app1的消费者只能消费所有端口为20880的服务提供实例
- application=app1 => address=*:20880
# app2的消费者只能消费所有端口为20881的服务提供实例
- application=app2 => address=*:20881
```



服务粒度路由规则：

```yaml
scope: service
force: true
runtime: true
enabled: true
key: org.apache.dubbo.samples.governance.api.DemoService
conditions:
# DemoService的sayHello方法只能消费所有端口为20880的服务提供者实例
- method=sayHello => address=*:20880
# DemoService的sayHi方法只能消费所有端口为20881的服务提供者实例
- method=sayHi => address=*:20881
```

#### (2) 属性详解

* scope :必填项，表示路由规则的作用粒度，其值会决定Key的取值，作用范围如下：
  * service：表示服务的粒度
  * application：表示应用的粒度
* key：必填项，指定规则提将作用域哪个服务或者应用，其取值决定于scope值
  * scope取值为application时，key值为application名称。即<dubbo:application name="" />
  * scope取值为service时，key值取值为[group:]servie[:version]组合，即组，接口名与版本号组合

* enabled：可选项，指定当前路由骨子额是否立即生效，缺省值为true，表示立即生效
* force：可选项，指定路由结果为空时，是否强制执行，如果不强制执行，路由结果为空则路由规则自动失效，不使用路由，缺省值为false，不强制执行。
* runtime：可选项，指定是否在在每次调用时执行路由规则。若为false，则表示只在提供者地址列表变更时会预先执行路由规则，并将路由结果进行缓存，消费者调用时直接从缓存中获取路由结果
* priority：可选项。用于设置路由规则的优先级，数字越大，优先级越高，越靠前执行，缺省为0

#### (3) 规则体conditions

##### A 格式

路由规则由两个条件组成，分别用于对服务消费者和提供者进行匹配。

[服务消费者匹配条件] => [服务提供者匹配条件]

* 当消费者的URL满足匹配条件时，对消费者执行后面的过滤规则

* =>之后为提供者地址列表的过滤条件，所有的参数和提供者的URL进行对比，消费者最终只拿到过滤后的地址列表。

  例如：host=10.20.153.10=>host=10.20.153.11其表示 IP 为10.20.153.10的服务消费者，只可调用 IP为10.20.153.11 机器上的服务。

* 服务消费者匹配条件为空，表示不对服务消费者进行限制，所有的消费者均将被路由到后面的提供者。

* 服务提供者匹配为空，表示对符合消费者条件的消费者将禁用任何提供者。

  例如：=> host != 10.20.153.11，表示所有的消费者均可调用 IP为10.20.153.11之外的所有主机

  例如：host = 10.20.153.10=> ，表示IP为10.20.153.10的消费者不能调用任何提供者。

##### B 符号支持

* 参数符号
  * method：将调用方法作为路由规则比较对象
  * argument：将调用方法参数作为路由规则比较对象
  * protocol：将调用协议作为路由规则比较对象
  * host：将IP作为路由规则比较对象
  * port：将PORT作为路由规则比较对象
  * address：将ip:prot作为路由规则比较对象
  * application：将应用名称作为路由规则比较对象                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         
* 条件符号
  * =：将匹配上的参数值作为路由条件
  * !=：将不匹配的参数值作为路由条件

##### C 举例

* 白名单

  host  !=10.20.153.10,10.20.153.11 =>

  禁用 ip 为 10.20.153.10与10.20.153.11之外的主机

* 黑名单

  host =10.20.153.10,10.20.153.11 =>

  ip 为10.20.153.10与10.20.153.11的主机将被禁用

* 只暴露一部分的提供者

  => 172.22.3.1\*,172.22.3.\*

  消费者只能访问 ip为172.22.3.1\*与172.22.3.\*

* 为重要的应用提供额外的机器

  application !=kylin => host !=172.22.3.95,172.22.3.96

  应用名称kylin 之外不能访问172.22.3.95,172.22.3.96两台主机

* 读写分离

* 前后台分离

* 隔离不同机房网段

  host !=172.22.3.\* =>172.22.3.\*

  只有172.22.3网段的消费者才可以访问172.22.3提供者，当然也可以访问其他网段的提供者

* 只访问本机的服务

  => host = $host

### 6.8.4 源码解析

##### （1）添加激活RouterFactory到Directory

 router注册过程：

ReferenceBean是一个FactoryBean，当获取同名的bean对象时会调用其getObject()方法

```java
org.apache.dubbo.config.spring.ReferenceBean#getObject();
org.apache.dubbo.config.spring.ReferenceBean#get();
org.apache.dubbo.config.spring.ReferenceBean#init();
org.apache.dubbo.config.spring.ReferenceBean#createProxy();
org.apache.dubbo.rpc.Protocol$Adaptive#refer();
org.apache.dubbo.registry.integration.RegistryProtocol#refer();
// 将构建内置的router放入routerChain,
// AppRouter,MockInvokersSelector,ServiceRouter,TagRouterFactory
org.apache.dubbo.registry.integration.RegistryProtocol#doRefer();
org.apache.dubbo.registry.integration.RegistryDirectory#subscribe();
org.apache.dubbo.registry.zookeeper.ZookeeperRegistry#doSubscribe();
org.apache.dubbo.registry.support.FailbackRegistry#notify();
org.apache.dubbo.registry.support.FailbackRegistry#doNotify();
org.apache.dubbo.registry.support.AbstractRegistry#notify();
org.apache.dubbo.registry.integration.RegistryDirectory#notify();
// 将构建的router添加到RegistryDirectory中
// 在这里假如url中存在compatible_config会被过滤掉
org.apache.dubbo.registry.integration.RegistryDirectory#toRouters();
```

##### (2) 过滤过程

```java
org.apache.dubbo.rpc.proxy.InvocationHandler#invoke();
org.apache.dubbo.rpc.cluster.support.wrapper.MockClusterInvoker#invoke();
org.apache.dubbo.rpc.cluster.support.AbstractClusterInvoker#invoke();
org.apache.dubbo.rpc.cluster.support.AbstractClusterInvoker#list();
org.apache.dubbo.rpc.cluster.directory.AbstractDirectory#list();
org.apache.dubbo.registry.integration.RegistryDirectory#doList();
// 进行过滤
org.apache.dubbo.rpc.cluster.router.condition.ConditionRouter#router();
```

## 6.9 服务降级

### 6.9.1 服务降级源码解析

服务降级源码分析：

服务降级只能是捕获到RpcException，且不是业务异常

```JAVA
org.apache.dubbo.rpc.proxy.InvokerInvocationHandler#invoke();
org.apache.dubbo.rpc.cluster.support.wrapper.MockClusterInvoker#invoke();
org.apache.dubbo.rpc.cluster.support.wrapper.MockClusterInvoker#doMockInvoke();
```

## 6.10 集群容错

集群容错所在的类为：

```
FailbackClusterInvoker
FailfastClusterInvoker
FailoverClusterInvoker
FailsafeClusterINvoker
ForkingClusterInvoker
BroadcastClusterINvoker
```

### 6.10.1 Failover 

故障转义策略，当消费者调用集群中某个服务器失败时，其会自动尝试调用其他服务器，重试次数是根据retries属性指定的

入口为: FailoverClusterInvoker#doInvoke();

调用次数为：重试次数+1

### 6.10.2 Failfast

快速失败策略。消费者端只发起一次调用，若失败则立即包旭哦，通常用于非幂等性写操作，比如新增记录

一次调用失败后立马抛出异常

### 6.10.3 Failsafe

失败安全策略，当消费者调用提供者出现异常时，直接忽略本次消费操作。该操作通常用于执行相对不太重要的服务

### 6.10.4 Failback

失败自动恢复策略。消费者调用提供者失败后，Dubbo会记录该失败的请求，然后会定时发起重试请求，而定时任务执行的次数扔是通过配置文件中的retries指定的

### 6.10.5 Forking

并行策略。消费者对于同一服务并行调用多个提供者服务器，只要有一个成功即调用结束并返回结果。

### 6.10.6 Broadcast

广播策略。广播调用所有提供者，逐个调用，任意一台报错则报错。通常用于通知所有听着更新或缓存日志等本地资源信息

## 6.11 负载均衡

​	