## Spring的BeanDefinition的获取

---

类图

BeanDefinitionReader，从名字可以看出，它负责从某处获取BeanDefinition，某处是指定义的某个路径location，或者是更上一层的包装Resource。

```java
public interface BeanDefinitionReader {

    //BeanDefinition的注册中心，一般都是BeanFactory
    //其实就是一个存放BeanDefinition的容器，实现中数据结构为Map<beanName, BeanDefinition>
    BeanDefinitionRegistry getRegistry();

    //获取ResourceLoader
    ResourceLoader getResourceLoader();

    //加载Bean的ClassLoader，如果为null，则意味着不加载Bean，仅仅注册BeanDefinition
    ClassLoader getBeanClassLoader();

    //如果一个Bean没有指定名字，则使用该名字生成器来生成名字
    BeanNameGenerator getBeanNameGenerator();

    //从resource从加载BeanDefinition，加载后放到BeanDefinitionRegistry中
    int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException;
    //从多个从resource从加载BeanDefinition
    int loadBeanDefinitions(Resource... resources) throws BeanDefinitionStoreException;

    //从指定的路径location中加载BeanDefinition，这个location可能是location pattern
    int loadBeanDefinitions(String location) throws BeanDefinitionStoreException;
    //从指定的多个路径location中加载BeanDefinition
    int loadBeanDefinitions(String... locations) throws BeanDefinitionStoreException;
}
```

**AbstractBeanDefinitionReader**

BeanDefinitionReader的抽象实现类，它的构造器很有意思。Environment是Spring3.1中引入的概念，所以，AbstractBeanDefinitionReader中的environment变量是此时加入的。

```java
protected AbstracregistrytBeanDefinitionReader(BeanDefinitionRegistry registry) {
    //一般来说，这个 registry就是实现了BeanDefinitionRegistry接口的Bean工厂
    this.registry = registry;

    //如果registry本身就实现了ResourceLoader，就使用registry作为resourceLoader
    if (this.registry instanceof ResourceLoader) {
        this.resourceLoader = (ResourceLoader) this.registry;
    }else {//否则，使用PathMatchingResourcePatternResolver作为resourceLoader
        this.resourceLoader = new PathMatchingResourcePatternResolver();
    }

    //如果registry本身实现了EnvironmentCapable，即它是有环境属性的，则使用registry作为它自身的environment
    if (this.registry instanceof EnvironmentCapable) {
        this.environment = ((EnvironmentCapable) this.registry).getEnvironment();
    }
    else {//否则，使用StandardEnvironment作为environment
        this.environment = new StandardEnvironment();
    }
}
```

此外，它默认使用DefaultBeanNameGenerator作为Bean的名字生成器，具体的逻辑在BeanDefinitionReaderUtils中：

```java
public static String generateBeanName(
            BeanDefinition definition, BeanDefinitionRegistry registry, boolean isInnerBean)
            throws BeanDefinitionStoreException {
    //获取Bean的全类名，比如“com.maxwell.example.service.UserServiceImpl”
    String generatedBeanName = definition.getBeanClassName();
    //generatedBeanName为空，则看它是否有父BeanDefinition或者是不是FactoryBean
    if (generatedBeanName == null) {
        if (definition.getParentName() != null) {
            generatedBeanName = definition.getParentName() + "$child";
        }
        else if (definition.getFactoryBeanName() != null) {
            generatedBeanName = definition.getFactoryBeanName() + "$created";
        }
    }
    //如果此时，generatedBeanName仍然没有获取到，则抛异常
    if (!StringUtils.hasText(generatedBeanName)) {
        throw new BeanDefinitionStoreException("Unnamed bean definition specifies neither " +
                "'class' nor 'parent' nor 'factory-bean' - can't generate bean name");
    }
    //此时，拿到了生成beanName的类名，比如“com.maxwell.example.service.UserServiceImpl”
    String id = generatedBeanName;
    if (isInnerBean) {
        // 如果是内部Bean，则在后面追加一个用哈希值生成的16进制的字符串
        id = generatedBeanName + GENERATED_BEAN_NAME_SEPARATOR + ObjectUtils.getIdentityHexString(definition);
    }else {
        // 如果是Top-level的bean，则在后面追加一个序号，这个序号是从0开始的
        int counter = -1;
        while (counter == -1 || registry.containsBeanDefinition(id)) {
            counter++;
            id = generatedBeanName + GENERATED_BEAN_NAME_SEPARATOR + counter;
        }
    }
    //最后的bean名字，如“com.maxwell.example.service.UserServiceImpl#0”
    return id;
}
```

**XmlBeanDefinitionReader**

读取在Xml文件中定义的 BeanDefinition，早期的Spring开发一般都会使用Xml文件来定义各种Bean。

**PropertiesBeanDefinitionReader**

读取 在配置文件中定义的 BeanDefinition，这种格式很少见，可参考Spring注释中该形式的示例。

按道理，所有可以去获取BeanDefinition的类都应该实现BeanDefinitionReader接口，但是在BeanDefinitionReader的注释中，有这样一段话：a bean definition reader不一定要实现该接口，该接口仅仅作为那些遵从标准命名约定的reader的建议。

> Note that a bean definition reader does not have to implement this interface.
>
> It only serves as suggestion for bean definition readers that want to follow standard naming conventions.

按照我的想法，既然有XmlBeanDefinitionReader去读取Xml中定义的BeanDefinition，有PropertiesBeanDefinitionReader去读取配置文件中定义的BeanDefiniton，就应该有类似ClassBeanDefinitionReader这种名字的reader去读取Class文件中通过注解定义的BeanDefiniton才对啊。很遗憾没有。TODO 为什么。

## 参考

---

[What are inner beans in Spring?](https://stackoverflow.com/questions/40042493/what-are-inner-beans-in-spring)[  
](https://fangjian0423.github.io/2017/06/15/spring-bean-register-note/)



