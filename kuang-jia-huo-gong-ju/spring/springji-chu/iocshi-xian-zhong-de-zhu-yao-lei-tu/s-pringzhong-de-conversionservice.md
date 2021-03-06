# Spring的类型转换器

---

![](/assets/屏幕快照 2018-11-07 下午10.52.29.png)

Converter则是Spring中的类型转换器，将S类型的source对象转换为T类型的对象

```java
public interface Converter<S, T> {

    T convert(S source);
}
```

为了将不同类型转换器集中起来，Spring定义了ConverterRegistry来添加或移除转换器：

```java
 public interface ConverterRegistry {

    void addConverter(Converter<?, ?> converter);

    <S, T> void addConverter(Class<S> sourceType, Class<T> targetType, Converter<? super S, ? extends T> converter);

    void addConverter(GenericConverter converter);

    void addConverterFactory(ConverterFactory<?, ?> factory);

    void removeConvertible(Class<?> sourceType, Class<?> targetType);
}
```

Spring将类型转换抽象为一种服务，叫做ConversionService，既然叫类型转换服务，则需要提供类型转换的方法：

```java
public interface ConversionService {
    //sourceType是否可以转换为targetType
    boolean canConvert(@Nullable Class<?> sourceType, Class<?> targetType);

    //sourceType是否可以转换为targetType
    boolean canConvert(@Nullable TypeDescriptor sourceType, TypeDescriptor targetType);

    //将source对象转换为targetType类型的对象
    <T> T convert(@Nullable Object source, Class<T> targetType);

    //将sourceType类型的source对象转换为targetType类型的对象
    Object convert(@Nullable Object source, @Nullable TypeDescriptor sourceType, TypeDescriptor targetType);
}
```

ConfigurableConversionService意为可配置的转换服务，既然可配置，则表示可以任意增减转换器：

```java
public interface ConfigurableConversionService extends ConversionService, ConverterRegistry {

}
```

GenericConversionService是ConfigurableConversionService的基本实现，

 DefaultConversionService则在GenericConversionService的基础上，内部添加了常用的类型转换器。 

** TODO Spring的Formatter接口 **

### 参考

---

[Spring源码：Converter及TypeConverter类解析](#)

