# 降级详解

## 自动开关降级

自动降级是根据系统负载、资源使用情况、SLA等指标进行降级。

#### **超时降级**

当访问的数据库/http服务/远程调用响应慢或者长时间响应慢，且该服务不是核心服务的话可以在超时后自动降级；比如商品详情页上有推荐内容/评价，但是推荐内容/评价暂时不展示对用户购物流程不会产生很大的影响；对于这种服务是可以超时降级的。如果是调用别人的远程服务，和对方定义一个服务响应最大时间，如果超时了则自动降级。

#### **统计失败次数降级**

有时候依赖一些不稳定的API，比如调用外部机票服务，当失败调用次数达到一定阀值自动降级；然后通过异步线程去探测服务是否恢复了，则取消降级。

#### **故障降级**

比如要调用的远程服务挂掉了（网络故障、DNS故障、http服务返回错误的状态码、rpc服务抛出异常），则可以直接降级。降级后的处理方案有：默认值（比如库存服务挂了，返回默认现货）、兜底数据（比如广告挂了，返回提前准备好的一些静态页面）、缓存（之前暂存的一些缓存数据）。

#### **限流降级**

当我们去秒杀或者抢购一些限购商品时，此时可能会因为访问量太大而导致系统崩溃，此时开发者会使用限流来进行限制访问量，当达到限流阀值，后续请求会被降级；降级后的处理方案可以是：排队页面（将用户导流到排队页面等一会重试）、无货（直接告知用户没货了）、错误页（如活动太火爆了，稍后重试）。

## **人工开关降级**

场景：在大促期间通过监控发现线上的一些服务存在问题，这个时候需要暂时将这些服务摘掉；还有有时候通过任务系统调用一些服务，但是服务依赖的数据库可能存在：网卡被打满了、挂掉了或者很多慢查询，此时需要暂停下任务系统让服务方进行处理；还有发现突然调用量太大，可能需要改变处理方式（比如同步转换为异步）；

此时就可以使用开关来完成降级。

开关可以存放到配置文件、数据库、Redis/ZooKeeper；如果不是存放在本地，可以定期同步开关数据（比如1秒同步一次）。然后通过判断某个KEY的值来决定是否降级。

另外对于新开发的服务想上线进行灰度测试；但是不太确定该服务的逻辑是否正确，此时就需要设置开关，当新服务有问题可以通过开关切换回老服务。还有多机房服务，如果某个机房挂掉了，此时需要将一个机房的服务切到另一个机房，此时也可以通过开关完成切换。

还有一些是因为功能问题需要暂时屏蔽掉某些功能，比如商品规格参数数据有问题，数据问题不能用回滚解决，此时需要开关控制降级。

## **读服务降级**

对于读服务降级一般采用的策略有：暂时切换读（降级到读缓存、降级到走静态化）、暂时屏蔽读（屏蔽读入口、屏蔽某个读服务）。读服务一般会按照如下顺序去读取数据：接入层缓存--&gt;应用层本地缓存--&gt;分布式缓存--&gt;RPC服务/DB。我们会在接入层、应用层设置开关，当分布式缓存、RPC服务/DB有问题自动降级为不调用。当然这种情况适用于对读一致性要求不高的场景。

页面降级、页面片段降级、页面异步请求降级都是读服务降级，目的是丢卒保帅（比如因为这些服务也要使用核心资源、或者占了带宽影响到核心服务）或者因数据问题暂时屏蔽。

**还有一种是页面静态化场景**

动态化降级为静态化：比如平时网站可以走动态化渲染商品详情页，但是到了大促来临之际可以将其切换为静态化来减少对核心资源的占用，而且可以提升性能；其他还有如列表页、首页、频道页都可以这么玩；可以通过一个程序定期的推送静态页到缓存或者生成到磁盘，出问题时直接切过去；

静态化降级为动态化：比如当使用静态化来实现商品详情页架构时，平时使用静态化来提供服务，但是因为特殊原因静态化页面有问题了，需要暂时切换回动态化来保证服务正确性。

以上都保证出问题了有预案，用户还是可以使用网站，不影响用户购物。

## **写服务降级**

写服务在大多数场景下是不可降级的，不过可以通过一些迂回战术来解决问题。比如将同步操作转换为异步操作，或者限制写的量/比例。

比如扣减库存一般这样操作：

方案1：

1、扣减DB库存，2、扣减成功后更新Redis中的库存；

方案2：

1、扣减Redis库存，2、同步扣减DB库存，如果扣减失败则回滚Redis库存；

前两种方案非常依赖DB，假设此时DB性能跟不上则扣减库存就会遇到问题；因此我们可以想到方案3：

1、扣减Redis库存，2、正常同步扣减DB库存，性能扛不住时降级为发送一条扣减DB库存的消息，然后异步进行DB库存扣减实现最终一致即可；

这种方式发送扣减DB库存消息也可能成为瓶颈；这种情况我们可以考虑方案4：

1、扣减Redis库存，2、正常同步扣减DB库存，性能扛不住时降级为写扣减DB库存消息到本机，然后本机通过异步进行DB库存扣减来实现最终一致性。

也就是说正常情况可以同步扣减库存，在性能扛不住时降级为异步；另外如果是秒杀场景可以直接降级为异步，从而保护系统。还有如下单操作可以在大促时暂时降级将下单数据写入Redis，然后等峰值过去了再同步回DB，当然也有更好的解决方案，但是更复杂。

还有如用户评价，如果评价量太大，也可以把评价从同步写降级为异步写。当然也可以对评价按钮进行按比例开放（比如一些人的看不到评价操作按钮）。比如评价成功后会发一些奖励，在必要的时候降级同步到异步。

## **多级降级**

缓存是离用户最近越高效；而降级是离用户越近越能对系统保护的好。因为业务的复杂性导致越到后端QPS/TPS越低。

**页面JS降级开关**：主要控制页面功能的降级，在页面中通过JS脚本部署功能降级开关，在适当时机开启/关闭开关；

**接入层降级开关**：主要控制请求入口的降级，请求进入后会首先进入接入层，在接入层可以配置功能降级开关，可以根据实际情况进行自动/人工降级；这个可以参考《[京东商品详情页服务闭环实践](http://jinnianshilongnian.iteye.com/blog/2258111)》，尤其在后端应用服务出问题时，通过接入层降级从而给应用服务有足够的时间恢复服务；

**应用层降级开关**：主要控制业务的降级，在应用中配置相应的功能开关，根据实际业务情况进行自动/人工降级。  


## 内容来源

《亿级流量网站架构核心技术》：降级特效
