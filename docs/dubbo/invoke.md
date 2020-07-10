# dubbo服务远程调用
 在调用dubbo服务时发现执行方法时实际上调用的是invoker的代理类，那么这个代理类是怎么生成的呢？
![invoker handler](https://img-blog.csdnimg.cn/20190119134228836.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1kxODYzMDI0Njc5MzE5NDUzMA==,size_16,color_FFFFFF,t_70)
 本文讲下使用dubbo注解方式（**@Reference**）的服务引用过程

## 示例
```
import com.alibaba.dubbo.config.annotation.Reference;
import com.momo.dubbo.provider.IHelloService;
import com.momo.dubbo.provider.ISayBeyService;
import org.springframework.stereotype.Component;
/**
 * HelloConsumerService
 *
 * @author huangtao
 * @date 2019/1/12
 * desc：
 */
@Component("helloConsumerService")
public class HelloConsumerService {
    @Reference
    private IHelloService helloService;
    @Reference
    private ISayBeyService sayBeyService;
    public String geyHello() {
        return helloService.sayHello();
    }
    public String geyHello(String word) {
        return helloService.sayHello(word);
    }
    public String getSayBey(String word) {
        return sayBeyService.sayHello(word);
    }
}
```


```
@Configuration
@EnableDubbo(scanBasePackages = "com.momo.sentinel")
@ComponentScan(value = {"com.momo.sentinel"})
public class DubboConfig {
    @Bean
    public RegistryConfig registryConfig() {
        RegistryConfig registryConfig = new RegistryConfig();
        registryConfig.setAddress("zookeeper://localhost:2181");
        return registryConfig;
    }
    @Bean
    public ApplicationConfig applicationConfig() {
        ApplicationConfig applicationConfig = new ApplicationConfig();
        applicationConfig.setName("dubbo-consumer-demo");
        return applicationConfig;
    }
    @Bean
    public ConsumerConfig consumerConfig() {
        ConsumerConfig consumerConfig = new ConsumerConfig();
        return consumerConfig;
    }
    @Bean
    public AnnotationBean annotationBean() {
        AnnotationBean annotationBean = new AnnotationBean();
        annotationBean.setPackage("com.momo.sentinel");
        return annotationBean;
    }
    @Bean
    public SentinelResourceAspect sentinelResourceAspect() {
        return new SentinelResourceAspect();
    }
}
```
这里注入bean采用了dubbo的Reference注解
## 调用方(consumer)注册
ReferenceAnnotationBeanPostProcessor
![ReferenceAnnotationBeanPostProcessor](https://img-blog.csdnimg.cn/20190119134333381.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1kxODYzMDI0Njc5MzE5NDUzMA==,size_16,color_FFFFFF,t_70)
来看看spring对方法的定义 postProcessPropertyValues：
```
	@Override
	public PropertyValues postProcessPropertyValues(
			PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeansException {
		return pvs;
	}
```
postProcessPropertyValues方法的作用在属性中被设置到目标实例之前调用，可以修改属性的设置
1. pvs参数表示参数属性值(从BeanDefinition中获取)
2. pds代表参数的描述信息(比如参数名，类型等描述信息)
3. bean参数是目标实例
4. beanName是目标实例在Spring容器中的name
5. 返回值是PropertyValues，可以使用一个全新的PropertyValues代替原先的PropertyValues用来覆盖属性设置或者直接在参数pvs上修改。如果返回值是null，那么会忽略属性设置这个过程(所有属性不论使用什么注解，最后都是null)

postProcessPropertyValues执行的结果：
![postProcessPropertyValues](https://img-blog.csdnimg.cn/2019011913440052.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1kxODYzMDI0Njc5MzE5NDUzMA==,size_16,color_FFFFFF,t_70)
 metadata.inject(bean, beanName, pvs) 修改了bean的属性
来看下ReferenceFieldElement的inject方法：

```
        @Override
        protected void inject(Object bean, String beanName, PropertyValues pvs) throws Throwable {
            Class<?> referenceClass = field.getType();
            //创建注入的bean
            referenceBean = buildReferenceBean(reference, referenceClass);
            ReflectionUtils.makeAccessible(field);
            //设置访问器Accessor
            field.set(bean, referenceBean.getObject());
        }
    }
```
![buildReferenceBean](https://img-blog.csdnimg.cn/20190119134432622.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1kxODYzMDI0Njc5MzE5NDUzMA==,size_16,color_FFFFFF,t_70)
build方法：
![build](https://img-blog.csdnimg.cn/2019011913451593.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1kxODYzMDI0Njc5MzE5NDUzMA==,size_16,color_FFFFFF,t_70)
这个方法里的doBulid方法初始化了个referenceBean 里买呢记录了dubbo的配置属性，核心方法configureBean前5个方法都是在为bean初始化做准备
```
    protected void configureBean(B bean) throws Exception {
        preConfigureBean(annotation, bean);
        configureRegistryConfigs(bean);
        configureMonitorConfig(bean);
        configureApplicationConfig(bean);
        configureModuleConfig(bean);
        postConfigureBean(annotation, bean);
    }
```

postConfigureBean 的前三个方法依然在准备生成bena需要的上下文、接口名、配置信息
```
    protected void postConfigureBean(Reference annotation, ReferenceBean bean) throws Exception {
        bean.setApplicationContext(applicationContext);
        configureInterface(annotation, bean);
        configureConsumerConfig(annotation, bean);
        bean.afterPropertiesSet();
    }
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190119134539461.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1kxODYzMDI0Njc5MzE5NDUzMA==,size_16,color_FFFFFF,t_70)
最终生成的是个包含dubbo配置信息、接口信息的ReferenceBean
![referenceBean.getObject()](https://img-blog.csdnimg.cn/20190119134618692.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1kxODYzMDI0Njc5MzE5NDUzMA==,size_16,color_FFFFFF,t_70)
referenceBean.getObject()实际上得到的是InvokerInvocationHandler：invoker执行器
![get](https://img-blog.csdnimg.cn/20190119134703163.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1kxODYzMDI0Njc5MzE5NDUzMA==,size_16,color_FFFFFF,t_70)
getObject 里面的核心方法是init，init核心方法的createProxy
来看看createProxy方法代码：
![createProxy](https://img-blog.csdnimg.cn/20190119134800302.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1kxODYzMDI0Njc5MzE5NDUzMA==,size_16,color_FFFFFF,t_70)
StubProxyFactoryWrapper：代理工厂包装类
![StubProxyFactoryWrapper](https://img-blog.csdnimg.cn/20190119134839830.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1kxODYzMDI0Njc5MzE5NDUzMA==,size_16,color_FFFFFF,t_70)
JavassistProxyFactory getProxy方法生成代理类：
```
    public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
        return Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
    }
```
到此生成InvokerInvocationHandler代理类完成。
在 **ReferenceConfig.createProxy(map)** 方法中，涉及远程引用服务的代码如下：
```
    @SuppressWarnings({"unchecked", "rawtypes", "deprecation"})
    private T createProxy(Map<String, String> map) {
        URL tmpUrl = new URL("temp", "localhost", 0, map);
        //是否是本地引用
        final boolean isJvmRefer;
        if (isInjvm() == null) {
            if (url != null && url.length() > 0) { // if a url is specified, don't do local reference
                isJvmRefer = false;
            } else {
                // by default, reference local service if there is
                isJvmRefer = InjvmProtocol.getInjvmProtocol().isInjvmRefer(tmpUrl);
            }
        } else {
            isJvmRefer = isInjvm();
        }
       //本地服务引用
        if (isJvmRefer) {
            URL url = new URL(Constants.LOCAL_PROTOCOL, Constants.LOCALHOST_VALUE, 0, interfaceClass.getName()).addParameters(map);
            invoker = refprotocol.refer(interfaceClass, url);
            if (logger.isInfoEnabled()) {
                logger.info("Using injvm service " + interfaceClass.getName());
            }
        } else {
            //定义直连地址，可以是服务提供者的地址，也可以是注册中心的地址
            if (url != null && url.length() > 0) { // user specified URL, could be peer-to-peer address, or register center's address.
                // 拆分地址成数组，使用 ";" 分隔。
                String[] us = Constants.SEMICOLON_SPLIT_PATTERN.split(url);
                // 循环数组，添加到 `url` 中。
                if (us != null && us.length > 0) {
                    for (String u : us) {
                        // 创建 URL 对象
                        URL url = URL.valueOf(u);
                        // 设置默认路径
                        if (StringUtils.isEmpty(url.getPath())) {
                            url = url.setPath(interfaceName);
                        }
                        // 注册中心的地址，带上服务引用的配置参数
                        if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                            urls.add(url.addParameterAndEncoded(Constants.REFER_KEY, StringUtils.toQueryString(map)));
                        } else {
                            // 服务提供者的地址
                            urls.add(ClusterUtils.mergeUrl(url, map));
                        }
                    }
                }
                // 注册中心
            } else { // assemble URL from register center's configuration
                // 加载注册中心 URL 数组
                List<URL> us = loadRegistries(false);
                // 循环数组，添加到 `url` 中。
                if (CollectionUtils.isNotEmpty(us)) {
                    for (URL u : us) {
                        // 加载监控中心 URL
                        URL monitorUrl = loadMonitor(u);
                        // 服务引用配置对象 `map`，带上监控中心的 URL
                        if (monitorUrl != null) {
                            map.put(Constants.MONITOR_KEY, URL.encode(monitorUrl.toFullString()));
                        }
                        // 注册中心的地址，带上服务引用的配置参数
                        urls.add(u.addParameterAndEncoded(Constants.REFER_KEY, StringUtils.toQueryString(map)));
                    }
                }
                if (urls.isEmpty()) {
                    throw new IllegalStateException("No such any registry to reference " + interfaceName + " on the consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion() + ", please config <dubbo:registry address=\"...\" /> to your spring config.");
                }
            }
            // 单 `urls` 时，引用服务，返回 Invoker 对象
            if (urls.size() == 1) {
                invoker = refprotocol.refer(interfaceClass, urls.get(0));
            } else {
                // 循环 `urls` ，引用服务，返回 Invoker 对象
                List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
                URL registryURL = null;
                for (URL url : urls) {
                    // 引用服务
                    invokers.add(refprotocol.refer(interfaceClass, url));
                    // 使用最后一个注册中心的 URL
                    if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                        registryURL = url; // use last registry url
                    }
                }
                // 有注册中心
                if (registryURL != null) { // registry url is available
                    // 对有注册中心的 Cluster 只用 AvailableCluster
                    // use RegistryAwareCluster only when register's cluster is available
                    URL u = registryURL.addParameter(Constants.CLUSTER_KEY, RegistryAwareCluster.NAME);
                    // The invoker wrap relation would be: RegistryAwareClusterInvoker(StaticDirectory) -> FailoverClusterInvoker(RegistryDirectory, will execute route) -> Invoker
                    invoker = cluster.join(new StaticDirectory(u, invokers));
                    // 无注册中心
                } else { // not a registry url, must be direct invoke.
                    invoker = cluster.join(new StaticDirectory(invokers));
                }
            }
        }
        //启动时检查
        Boolean c = check;
        if (c == null && consumer != null) {
            c = consumer.isCheck();
        }
        if (c == null) {
            c = true; // default true
        }
        if (c && !invoker.isAvailable()) {
            // make it possible for consumer to retry later if provider is temporarily unavailable
            initialized = false;
            throw new IllegalStateException("Failed to check the status of the service " + interfaceName + ". No provider available for the service " + (group == null ? "" : group + "/") + interfaceName + (version == null ? "" : ":" + version) + " from the url " + invoker.getUrl() + " to the consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion());
        }
        if (logger.isInfoEnabled()) {
            logger.info("Refer dubbo service " + interfaceClass.getName() + " from url " + invoker.getUrl());
        }
        /**
         * @since 2.7.0
         * ServiceData Store
         */
        MetadataReportService metadataReportService = null;
        if ((metadataReportService = getMetadataReportService()) != null) {
            URL consumerURL = new URL(Constants.CONSUMER_PROTOCOL, map.remove(Constants.REGISTER_IP_KEY), 0, map.get(Constants.INTERFACE_KEY), map);
            metadataReportService.publishConsumer(consumerURL);
        }
        //创建 Service 代理对象
        // create service proxy
        return (T) proxyFactory.getProxy(invoker);
    }
```
## 服务调用
MockClusterInvoker：
![MockClusterInvoker](https://img-blog.csdnimg.cn/20190119134925297.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1kxODYzMDI0Njc5MzE5NDUzMA==,size_16,color_FFFFFF,t_70)
为什么是FailoverCluster？
FailoverCluster是缺省值
![FailoverCluster](https://img-blog.csdnimg.cn/20190119134949634.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1kxODYzMDI0Njc5MzE5NDUzMA==,size_16,color_FFFFFF,t_70)
FailoverClusterInvoker：
![FailoverClusterInvoker](https://img-blog.csdnimg.cn/20190119135007884.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1kxODYzMDI0Njc5MzE5NDUzMA==,size_16,color_FFFFFF,t_70)ProtocolFilterWrapper：![ProtocolFilterWrapper](https://img-blog.csdnimg.cn/20190119135030302.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1kxODYzMDI0Njc5MzE5NDUzMA==,size_16,color_FFFFFF,t_70)

DubboInvoker：最终执行目标方法
![DubboInvoker](https://img-blog.csdnimg.cn/20190119135057415.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1kxODYzMDI0Njc5MzE5NDUzMA==,size_16,color_FFFFFF,t_70)
ProtocolFilterWrapper：过滤器链
![ProtocolFilterWrapper](https://img-blog.csdnimg.cn/20190119135115538.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1kxODYzMDI0Njc5MzE5NDUzMA==,size_16,color_FFFFFF,t_70)
在构建调用链时方法先获取Filter列表，然后创建与Fitler数量一样多Invoker结点，接着将这些结点串联在一起，构成一个链表，最后将这个链的首结点返回，随后的调用中，将从首结点开始，依次调用各个结点，完成调用后沿调用链返回。这里各个Invoker结点的串联是通过与其关联的invoke方法来完成的。

在invoke方法内部，通过调用与该invoker关联的filter中的invoke方法来实现结点的连接。调用时将next传入invoke方法，在调用时首先会调用该结点对应的filter的invoke方法，接着调用传入参数next的invoke方法。Next的invoke方法同样会调用自己所关联的filter的invoke方法，这样就完成了结点的串联。可以看到链表的最后一个结点就是buildInvokerChain 方法的入参invoker。最终buildInvokerChain方法通过链表头插法完成调用链的创建。因此在真正的调用请求处理前会经过若干filter进行预处理。

过滤器链的创建可以看作是职责链模式（Chain of Responsibility Pattern）的一个实现。这样系统中增加一个新的过滤器预处理请求时，无须修改原有系统的代码，只需重新建调用链即可。

ProtocolListenerWrapper：监听器链
![ProtocolListenerWrapper](https://img-blog.csdnimg.cn/20190119135134929.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1kxODYzMDI0Njc5MzE5NDUzMA==,size_16,color_FFFFFF,t_70)
各类协议protocol类均是由ProtocolListenerWrapper类封装的，在服务的暴露和引用过程中，都是调用该类的export和refer方法，在这些方法中完成监听器链的创建。
完整的时序图如下：
![远程调用时序图](https://img-blog.csdnimg.cn/20190119135154122.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1kxODYzMDI0Njc5MzE5NDUzMA==,size_16,color_FFFFFF,t_70)