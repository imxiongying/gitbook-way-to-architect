## ConfigurationClassParser

---

ConfigurationClassParser的作用是解析被@Configuration注解标注的类。

用户使用@Configuration标注的类，会被解析为ConfigurationClass，然后ConfigurationClassBeanDefinitionReader从ConfigurationClass中解析出BeanDefinition。

ConfigurationClassParser是在ConfigurationClassPostProcessor中被使用，而ConfigurationClassPostProcessor这个BeanFactoryPostProcessor是当发现xml文件中含有以下标签时被注册的：

```xml
<context:annotation-config/> or
<context:component-scan/>
```

 ConfigurationClassPostProcessor中的处理逻辑为：

* 检查BeanDefinition是否为配置类，有两种配置类
* * full是指：类上标注了@Configration
  * lite是指：类上标注了@Import或者类中含有标注了@Bean的方法
* 因为涉及到配置类的覆盖问题，对发现的配置类进行排序

* 从每一个配置类中解析出BeanDefinition，如果解析出的BeanDefinition仍然为配置类，则继续解析该配置类。

其中，每一个配置类中解析出BeanDefinition的主要代码如下：

```java
ConfigurationClassParser parser = new ConfigurationClassParser(
                this.metadataReaderFactory, 
                this.problemReporter, 
                this.environment,
                this.resourceLoader, 
                this.componentScanBeanNameGenerator, 
                registry);
Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
do {
    //从当前配置类拿到ConfigurationClass对象
    parser.parse(candidates);
    parser.validate();
    Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
    //从ConfigurationClass对象中解析出BeanDefinition
    if (this.reader == null) {
        this.reader = new ConfigurationClassBeanDefinitionReader(
                registry, this.sourceExtractor, this.resourceLoader, this.environment,
                this.importBeanNameGenerator, parser.getImportRegistry());
    }
    this.reader.loadBeanDefinitions(configClasses);

}
while (!candidates.isEmpty());
```

从一个配置类可能获取到多个ConfigurationClass对象，比如配置类上通过@Import标签引入了其他配置类。

```java
protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass)throws IOException {
	// 处理内部类，因为内部类有可能也加了@Configration注解
	processMemberClasses(configClass, sourceClass);
	// 处理@PropertySource注解：将配置信息加载到环境中
	for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
			sourceClass.getMetadata(), PropertySources.class,
			org.springframework.context.annotation.PropertySource.class)) {
		if (this.environment instanceof ConfigurableEnvironment) {
			processPropertySource(propertySource);
		}
	}
	// 处理@ComponentScan注解，
	// 因为该注解允许重复，一个类上可能标记了多个@ComponentScan注解，所以这里是个集合
	Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
			sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
	if (!componentScans.isEmpty() &&
			!this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
		for (AnnotationAttributes componentScan : componentScans) {
			// 使用ComponentScanAnnotationParser来解析
			// 它会从@ComponentScan注解中拿到basePackages，然后交给ClassPathBeanDefinitionScanner来执行真正的扫描和加载BeanDefinition
			Set<BeanDefinitionHolder> scannedBeanDefinitions =
					this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
			// 检查扫描出来的BeanDefiniton中是否有配置类，有的话就递归再解析
			for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
				if (ConfigurationClassUtils.checkConfigurationClassCandidate(
						holder.getBeanDefinition(), this.metadataReaderFactory)) {
					parse(holder.getBeanDefinition().getBeanClassName(), holder.getBeanName());
				}
			}
		}
	}
	// 处理@Import注解：先解析出@Import中value中写的类，这些类又分为3类
	// 第一类：ImportSelector的实现类，则调用该实现类的selectImports去获取要加载的BeanDefiniton的类名
	// 第二类：ImportBeanDefinitionRegistrar的实现类，则调用该类的registerBeanDefinitions方法加载BeanDefiniton
	// 第三类：其他的类，则当做配置类去处理（又会递归）
	processImports(configClass, sourceClass, getImports(sourceClass), true);
	// 处理@ImportResource注解：解析在xml中定义的Bean的BeanDefiniton
	AnnotationAttributes importResource =
			AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
	if (importResource != null) {
		String[] resources = importResource.getStringArray("locations");
		Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
		for (String resource : resources) {
			String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
			configClass.addImportedResource(resolvedResource, readerClass);
		}
	}

	// 处理使用@Bean注解的方法：这里仅仅是记录下来这些方法（封装为BeanMethod），没有进行进一步处理
	Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
	for (MethodMetadata methodMetadata : beanMethods) {
		configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
	}

	// 处理该类实现的接口中的默认方法（java1.8引入的默认方法）：将其作为BeanMethod
	processInterfaces(configClass, sourceClass);

	// 处理该类的父类：如果有父类，则递归处理
	if (sourceClass.getMetadata().hasSuperClass()) {
		String superclass = sourceClass.getMetadata().getSuperClassName();
		if (superclass != null && !superclass.startsWith("java") &&
				!this.knownSuperclasses.containsKey(superclass)) {
			this.knownSuperclasses.put(superclass, configClass);
			// Superclass found, return its annotation metadata and recurse
			return sourceClass.getSuperClass();
		}
	}
	// 没有父类，处理结束
	return null;
}
```

可以看出，ConfigurationClassParser这个BeanDefinitonReader是比前面提到的ClassPathBeanDefinitionScanner更为上层的封装，它可以完整地解析出几乎所有的注解。

## 参考

---

[ 一分钟学会spring注解之@Import注解](http://blog.51cto.com/4247649/2118354)

[【译】Spring 4 @PropertySource和@Value注解示例](https://www.cnblogs.com/chenpi/p/6212534.html)

[Java inner class and static nested class](https://stackoverflow.com/questions/70324/java-inner-class-and-static-nested-class)

[Spring（32）——ImportSelector介绍](http://elim.iteye.com/blog/2428994)



