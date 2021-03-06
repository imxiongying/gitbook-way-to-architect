# BeanFactory

---

![](/assets/屏幕快照 2018-10-21 下午5.16.58.png)

BeanFactory是最顶层也是最基本的工厂接口，定义了基本的获取Bean的方法，并定义了获取Bean属性的方法。

```java
public interface BeanFactory {
    //FactoryBean的转义符
    String FACTORY_BEAN_PREFIX = "&";

    //获取Bean的方法
    Object getBean(String name) throws BeansException;
    <T> T getBean(String name, Class<T> requiredType);
    <T> T getBean(Class<T> requiredType) throws BeansException;
    Object getBean(String name, Object... args) throws BeansException;

    //工具方法
    boolean containsBean(String name);
    boolean isSingleton(String name) throws NoSuchBeanDefinitionException;    
    boolean isPrototype(String name) throws NoSuchBeanDefinitionException;    
    Class<?> getType(String name) throws NoSuchBeanDefinitionException;
    String[] getAliases(String name);
}
```

BeanFactory的直接继承接口：

* **HierarchicalBeanFactory**

* * 为实现bean工厂的层级关系提供支持，定义了`BeanFactory getParentBeanFactory()`方法；比如常见的Web应用中，ContextLoaderListener创建的容器为父容器，而DispatcherServlet创建的容器为子容器。
  * 容器有父子之分，父容器对子容器可见，子容器对父容器不可见。
* **AutowireCapableBeanFactory**

* * 提供自动装配Bean的能力，主要方法为`void autowireBean(Object existingBean)`。
  * ```xml
    //引入Autowire之前
    <bean id="example" class="com.maxwell.learning.spring.Example">
        <property name="a" ref="nameA"/>
        <property name="b" ref="nameB"/>
        <property name="c" ref="nameC"/>
    </bean>
    //引入Autowire之后
    <bean id="example" class="com.maxwell.learning.spring.Example" autowire="byName">
    ```
  * autowire包含4中模式：

  * * no：默认值，Spring建议使用该值，即我们在配置中显式指明其依赖，而不是靠Spring根据类型或名字来判断
    * byName：根据依赖Bean的名字去获取依赖Bean
    * byType：根据依赖Bean的类型去获取依赖Bean
    * constructor：在构造时，按照依赖Bean的类型去获取依赖Bean
* **ListableBeanFactory**

* * BeanFactory可以提供getBeanByName或者getBeanByType的能力，而ListableBeanFactory还可以枚举所有的Bean实例。

**ConfigurableBeanFacory**：提供对BeanFactory的配置功能，比如BeanFactory的父级BeanFactory是哪个，BeanFactory使用哪个ClassLoader去加载Bean，是否需要缓存Bean的元信息，解析Bean时使用哪个EL表达式解析器，BeanFactory有哪些BeanPostProcessor等等。

**ConfigurableListableBeanFactory**

 继承以上所有接口，并定义了一些修改BeanDefinition的方法等，主要是更接近实现层面的工具方法的定义。

**DefaultListableBeanFactory**

实现了以上所有接口（有些是在其父类中实现的），可以理解为最终BeanFactory的实现类。

**XmlBeanFactory**

与父类相比，仅仅重写了获取BeanDefinition的方法：从XML文件中获取BeanDefinition，设计模式中典型的模板方法。



## 参考

---

[Spring 主要涉及类类图](https://blog.csdn.net/strivezxq/article/details/44560771)

[Spring BeanFactory 类图详解](https://blog.csdn.net/qq_34090008/article/details/78772189)

[Spring 父子容器](http://wangxinchun.iteye.com/blog/2341197)

[Spring AutowireCapableBeanFactory学习](https://www.jianshu.com/p/d564335fafab)

[Spring源码学习--@Autowired注解和启动自动扫描的三种方式  ](https://blog.csdn.net/u013412772/article/details/73741710)

[Spring源码学习--ConfigurableBeanFactory接口](https://blog.csdn.net/u013412772/article/details/80819398)

[Spring源码分析——BeanFactory体系之接口详细分析](https://www.cnblogs.com/zrtqsk/p/4028453.html)

[SingletonBeanRegistry](https://blog.csdn.net/weixin_39165515/article/details/77655567)

[Spring Bean 大体架构介绍](https://gavinzhang1.gitbooks.io/spring/content/spring_bean_da_ti_jia_gou_jie_shao.html)

