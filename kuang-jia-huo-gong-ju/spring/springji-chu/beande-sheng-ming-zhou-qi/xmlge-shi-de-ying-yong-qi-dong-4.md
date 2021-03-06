**第四步**：postProcessBeanFactory\(beanFactory\)：在BeanFactory完成标准的配置后，让子类进行一些定制化的设置。

对应到ClassPathXmlApplicationContext，它其实没有进行实现。

**第五步**：invokeBeanFactoryPostProcessors\(beanFactory\)：回调注册的BeanFactoryPostProcessor。

```java
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
    PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

    // Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
    // (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
    if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }
}
```

由于这里并没有设置BeanFactoryPostProcessor，因此相当于什么都没有做。如果想要添加BeanFactoryPostProcessor，则需要手动context.addBeanFactoryPostProcessor\(postProcessor\)，或者在xml中定义我们自己的BeanFactoryPostProcessor。

 关键的执行逻辑都在PostProcessorRegistrationDelegate这个代理类中，我们在xml中定义的BeanFactoryPostProcessor会在这里得到调用，这句代码很有欺骗性，乍一看，getBeanFactoryPostProcessors\(\)为空，觉得不会触发任何BeanFactoryPostProcessor，但是invokeBeanFactoryPostProcessors方法还有一个参数是beanFactory，这个beanFactory中其实已经包含了我们想要定义的BeanFactoryPostProcessor。

PostProcessorRegistrationDelegate的主要功能有两个：一是执行BeanFactoryPostProcessor，而是注册BeanPostProcessor。

由于代码过长，注释详见[PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors](https://github.com/maxwellyue/JavaLanguage/blob/master/spring/src/main/java/com/maxwell/learning/spring/sourcecode/InvokeBeanFactoryPostProcessors.java)。

对于BeanFactoryPostProcessor，它只有一个扩展接口BeanDefinitionRegistryPostProcessor。而这个扩展接口主要用来对已经加载完成的BeanDefinition进行修改，所以要首先执行它的扩展方法postProcessBeanDefinitionRegistry的逻辑。

所以，就不难理解上面代码的逻辑：①首先回调方法参数中传入的BeanDefinitionRegistryPostProcessor的postProcessBeanDefinitionRegistry，然后回调beanFactory中找到的BeanDefinitionRegistryPostProcessor的postProcessBeanDefinitionRegistry。②回调方法中传入的以及在beanFactory中找到的BeanDefinitionRegistryPostProcessor的postProcessBeanFactory方法；③对于BeanFactoryPostProcessor的直接实现类，先调用方法入参中传入的，再回调在beanFactory中找到的。

此外，无论是BeanFactoryPostProcessor还是它的扩展接口BeanDefinitionRegistryPostProcessor，当有多个时，是按照如下顺序调用的：

`实现了PriorityOrdered > 实现了Ordered > 即未实现PriorityOrdered也未实现Ordered`

PriorityOrdered和Ordered这两个接口都是表示顺序的，只是PriorityOrdered更能说明优先级这个语义。

```java
public interface Ordered {

    //最高优先级
    int HIGHEST_PRECEDENCE = Integer.MIN_VALUE;

    //最低优先级
    int LOWEST_PRECEDENCE = Integer.MAX_VALUE;

    //值越大，优先级越低；值越小，优先级越大；值相同，则顺序是任意的。
    int getOrder();
}
public interface PriorityOrdered extends Ordered {

}
```

PostProcessorRegistrationDelegate的第二个功能，后面用到的时候 再说。

 总结一下第4和第5步的主要工作：

①对beanFactory做定制化的配置，ClassPathXmlApplicationContext并没有做。

②回调自定义的BeanFactoryPostProcessor。

