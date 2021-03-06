# 自动降级的实现：Hystrix

有时，我们需要根据服务的超时时间等对服务进行自动降级。这种场景下，Hystrix就可以发挥它的作用了。

## Hystrix概述

### Hystrix是什么

[Hystrix](https://github.com/Netflix/Hystrix)是Netflix开源的一个容灾框架，解决当外部依赖故障时拖垮业务系统、甚至引起雪崩的问题。在分布式环境中，许多服务依赖关系中的一些不可避免地会失败。 通过Hystrix为应用/服务添加延迟容限和容错逻辑，则控制这些分布式服务之间的交互。 Hystrix通过隔离服务之间的访问、停止它们之间的级联故障、提供备用选项来实现这一点，所有这些都可以提高系统的整体弹性。

### Hystrix可以做什么

Hystrix是为如下功能而设计的： 

* 在依赖方延迟或失败情况下（通常是网络原因），为应用提供处理这些问题的能力
* 避免在复杂的分布式系统中出现雪崩效应（级联故障）
* 失败回退（Fallback）和优雅的服务降级机制
* 监控、报警和其他一些运营操作

更多详见：[Hystrix 官方介绍](https://github.com/Netflix/Hystrix/wiki)

## 使用Hystrix实现自动降级

假如我们有EchoService，其中有个接口为echoTime，它会返回当前时间。

```java
public class EchoService {
    public String echoTime(){
        //模拟抛出异常：（1/5的概率抛出异常）
        if(new Random().nextInt(10) > 7){
            System.out.println("exception:::failure processing echo time");
            throw new RuntimeException();
        }
        //模拟网络耗时（1/2的概率100，1/2的概率1200）
        try {
            long elapsed = new long[]{100, 1200}[new Random().nextInt(2)];
            System.out.println("elapsed:::" + elapsed);
            Thread.sleep(elapsed);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "[normal]time:::" + System.currentTimeMillis();
    }
}
```

 对EchoService的echoTime\(\)编写对应的Command：

```java
public class EchoTimeCommand extends HystrixCommand<String> {
    private EchoService echoService;
    public EchoTimeCommand(EchoService echoService) {
        super(setter());
        this.echoService = echoService;
    }
    @Override
    protected String run() {
        return echoService.echoTime();
    }
    /**
     * 降级方法
     */
    @Override
    protected String getFallback() {
        //此处不应该再进行网络请求
        long defaultTime = 0;
        return "[fallback]time:::" + defaultTime;
    }
    /**
     *
     * 降级策略配置
     *
     * @return
     */
    private static Setter setter() {
        //服务分组
        HystrixCommandGroupKey groupKey = HystrixCommandGroupKey.Factory.asKey("echo_service");
        HystrixCommandKey commandKey = HystrixCommandKey.Factory.asKey("echo_time");
        HystrixCommandProperties.Setter setter = HystrixCommandProperties.Setter()
                //隔离策略，默认THREAD
                .withExecutionIsolationStrategy(HystrixCommandProperties.ExecutionIsolationStrategy.THREAD)
                //是否允许降级，默认允许，如果允许，则在超时或异常时会调用getFallback()进行降级处理
                .withFallbackEnabled(true)
                //getFallback()的并发数，如果超出该值，就不再调用getFallback()方法，而是快速失败，默认为10
                .withFallbackIsolationSemaphoreMaxConcurrentRequests(10)
                //当执行线程执行run()超时时，是否进行中断处理，默认false
                .withExecutionIsolationThreadInterruptOnFutureCancel(true)
                //是否开启/run()方法超时设置，默认true
                .withExecutionTimeoutEnabled(true)
                //run()方法的超时时间（单位毫秒），默认1000
                .withExecutionTimeoutInMilliseconds(1000)
                //run()方法超时的时候，是否中断run()执行，默认true
                .withExecutionIsolationThreadInterruptOnTimeout(false);

        return HystrixCommand.Setter
                .withGroupKey(groupKey)
                .andCommandKey(commandKey)
                .andCommandPropertiesDefaults(setter);
    }
}
```

**执行逻辑**：首先Command会调用run方法，如果run方法超时或抛出异常，且启用降级处理，则会调用getFallback\(\)方法进行降级。 

进行调用测试：

```java
public class Client {
    public static void main(String[] args) throws InterruptedException {
        EchoService service = new EchoService();
        for (;;){
            EchoTimeCommand command = new EchoTimeCommand(service);
            //注意这里，要调用command的execute()方法，而不是直接调用run()
            String time = command.execute();
            System.out.println(time);
        }
    }
}
//输出如下：
elapsed:::1200ms [fallback]time:::0 
elapsed:::100ms [normal]time:::1529733096909 
elapsed:::1200ms [fallback]time:::0 
elapsed:::1200ms [fallback]time:::0 
elapsed:::100ms [normal]time:::1529733099022 
elapsed:::1200ms [fallback]time:::0 
exception:::failure processing echo time [fallback]time:::0 
elapsed:::100ms [normal]time:::1529733100134 
```

TODO：测试调用getFallback\(\)的并发限制数。

## 使用Hystrix实现熔断

Hystrix提供了熔断机制，在熔断后会自动进行降级处理。如果配置了开启熔断功能（默认开启），则Command会先判断服务已经被熔断，如果已经被熔断，则直接调用Command的getFallBack\(\)进行降级处理，在服务被熔断一定时间后（circuitBreakerSleepWindowInMilliseconds），会重新进行允许调用run\(\)方法，如果成功了，则关闭熔断，如果还是没有成功，则继续保持熔断状态。

还是以上面的例子来继续：

修改`EchoService`的`echoTime()`方法，增大超时和异常的概率：

```java
public class EchoService {

    public String echoTime(){
        //模拟抛出异常
        if(new Random().nextInt(10) > 5){
            System.out.println("exception:::failure processing echo time");
            throw new RuntimeException();
        }
        //模拟网络耗时，100或者1200
        try {
            long elapsed = new long[]{100, 1200, 1500}[new Random().nextInt(3)];
            System.out.println("elapsed:::" + elapsed + "ms");
            Thread.sleep(elapsed);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "[normal]time:::" + System.currentTimeMillis();
    }
}
```

修改EchoTimeCommand的配置，设置熔断相关的参数：

```java
public class EchoTimeCommand extends HystrixCommand<String> {

    private EchoService echoService;

    public EchoTimeCommand(EchoService echoService) {
        super(setter());
        this.echoService = echoService;
    }

    @Override
    protected String run() {
        return echoService.echoTime();
    }

    /**
     * 降级方法
     */
    @Override
    protected String getFallback() {
        //此处不应该再进行网络请求
        long defaultTime = 0;
        return "[fallback]time:::" + defaultTime;
    }

    /**
     * 熔断配置
     *
     * @return
     */
    private static Setter setter() {
        HystrixCommandGroupKey groupKey = HystrixCommandGroupKey.Factory.asKey("echo_service");
        HystrixCommandKey commandKey = HystrixCommandKey.Factory.asKey("echo_time");

        HystrixCommandProperties.Setter commandProperties = HystrixCommandProperties.Setter()
                // 是否开启熔断机制，默认为true
                .withCircuitBreakerEnabled(true)
                // 是否强制打开熔断器，默认false
                .withCircuitBreakerForceOpen(false)
                // 是否强制关闭熔断器，默认false
                .withCircuitBreakerForceClosed(false)
                // 至少有n个请求才计算错误比率，默认20
                .withCircuitBreakerRequestVolumeThreshold(3)
                // 对Command进行熔断的出错率阈值，超过这个阈值，则会将打开熔断器
                .withCircuitBreakerErrorThresholdPercentage(50)
                // 统计滚动的时间窗口
                .withMetricsRollingStatisticalWindowInMilliseconds(5000)
                // 开启熔断后的静默时间，超过这个时间，就会重新放一个请求进去，如果请求成功的话就关闭熔断，失败就再等一段时间，默认5000
                .withCircuitBreakerSleepWindowInMilliseconds(2000)
                // 为了便于观察，关闭超时时的线程中断
                .withExecutionIsolationThreadInterruptOnTimeout(false);
        return Setter
                .withGroupKey(groupKey)
                .andCommandKey(commandKey)
                .andCommandPropertiesDefaults(commandProperties);
    }
}
```

 修改测试代码，以便于观察熔断情况：

```java
public class Client {

    public static void main(String[] args) throws InterruptedException {
        EchoService service = new EchoService();
        int count = 0 ;
        for (;;){
            EchoTimeCommand command = new EchoTimeCommand(service);
            String time = command.execute();
            if(command.isCircuitBreakerOpen()){
                count++;
                if(count > 5){
                    Thread.sleep(2000);
                }
                System.out.println("[alert!!! alert!!! alert!!! circuit breaker is opened]");
            }
            System.out.println(time);
        }
    }
}
//输出如下
exception:::failure processing echo time [fallback]time:::0      
elapsed:::1500ms [fallback]time:::0 
exception:::failure processing echo time [fallback]time:::0 
exception:::failure processing echo time [fallback]time:::0 
exception:::failure processing echo time [fallback]time:::0 
exception:::failure processing echo time [fallback]time:::0 
exception:::failure processing echo time [fallback]time:::0 
elapsed:::100ms [normal]time:::1529735983294 
elapsed:::100ms [normal]time:::1529735983401 
elapsed:::1500ms 
[alert!!! alert!!! alert!!! circuit breaker is opened] [fallback]time:::0 
[alert!!! alert!!! alert!!! circuit breaker is opened] [fallback]time:::0 
[alert!!! alert!!! alert!!! circuit breaker is opened] [fallback]time:::0 
[alert!!! alert!!! alert!!! circuit breaker is opened] [fallback]time:::0 
[alert!!! alert!!! alert!!! circuit breaker is opened] [fallback]time:::0 
[alert!!! alert!!! alert!!! circuit breaker is opened] [fallback]time:::0 
exception:::failure processing echo time 
[alert!!! alert!!! alert!!! circuit breaker is opened] [fallback]time:::0 
elapsed:::1200ms 
[alert!!! alert!!! alert!!! circuit breaker is opened] [fallback]time:::0 
exception:::failure processing echo time 
[alert!!! alert!!! alert!!! circuit breaker is opened] [fallback]time:::0 
elapsed:::100ms [normal]time:::1529735993536 
elapsed:::1500ms [fallback]time:::0 
elapsed:::100ms [normal]time:::1529735994655 
elapsed:::1500ms [fallback]time:::0 
exception:::failure processing echo time 
[fallback]time:::0 
```

 熔断器有以下几种状态：

* closed：如果关闭熔断机制或者当前请求失败率没有超过失败率阈值，则熔断器处于closed状态。
* open：如果当前请求失败率超过了失败率阈值，则熔断器处于open状态。
* half-open：当熔断器处于open状态时，不能一直对服务进行熔断，需要在一定的时间后进行重试，这种状态就是half-open，如果重试时，run\(\)成功了，则熔断器会回到closed状态，如果仍然失败，则熔断器回到open状态。

TODO：Hystrix的熔断机制的具体实现













## 参考

[hystrix在spring mvc的使用](http://tech.lede.com/2017/06/15/rd/server/hystrix/)

《亿级流量网站架构核心技术》：降级特技

[hystrix配置解释](https://github.com/Netflix/Hystrix/wiki/Configuration)：

[服务容错模式](https://tech.meituan.com/service-fault-tolerant-pattern.html)





