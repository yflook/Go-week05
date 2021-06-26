# 第五周作业：
1. 总结限流，熔断，降级的常用方式，重试的注意事项，负载均衡的常用方式。



## 限流
* 所有的可用性问题一定不是单点解决的, 一定是立体式防御, 一定是从客户端到服务端两个纬度去做方案  
* 限流是指一段时间内, 定义某个客户或应用可以接收或处理多个请求的技术, 可以过滤掉产生峰值的客户和微服务  
* 令牌桶、漏桶 针对当个节点, 无法分布式限流  
* QPS 限流  
  * 不同的请求可能需要数量不一的资源来处理  
  * 静态QPS限流不准  
  * ps.通过计算每qps的cpu成本来限流  
* 给每个用户设置限制  
  * 全局过载时针对异常控制, 比如异常账号操作  
  * 一定程度的"超卖"配额  
* 按照优先级丢弃  
* 拒绝请求也需要成本  


## 熔断
* 熔断 - 客户端限流  
* 为了限制操作的持续时间  
* 当某个用户超过资源配额时, 后端任务会快速拒绝请求, 返回配额不足的错误, 但是拒绝回复仍会消耗一定资源, 有可能因为不断拒绝请求而导致过载  
* max(0, (requests - K * accepts) / (requests + 1))  


## 降级
* 丢弃不重要的请求, 提供一个降级的服务, 对某几个服务可进行空回复  
  * 基于cpu、错误的降级回复, 回复一些mock值  
  * 进入降级时, 不反悔一个复杂数据, 而是从一些缓存中捞取或者直接空回复  
  * 降级一般在bff或者gate way层做, 防止缓存污染  
  * 降级在意外流量或者意外负载时候触发  
  * 降级在不定时演练, 保证功能可用  
* 降级的本质: 提供有损服务  
  * ui模块化, 非核心模块降级  
    * BFF层聚合API, 模块降级  
  * 页面上一次缓存副本  
  * 默认值、热门推荐等。  
  * 流量拦截+定期数据缓存(过期副本策略)  
  * 页面降级、延迟服务、写/读降级、缓存降级(local cache)  
  * 抛异常、返回约定协议、Mock数据、Fallback处理  


## 重试
* 当请求返回错误 (例: 配额不足、超时、内部错误等), 对于backend部分节点过载的情况下, 倾向于立刻重试, 但是需要留意重试带来的流量放大  
  * 限制重试次数(内网中一般不超过两次)和基于重试分布的策略(重试比例10%)  
  * 随机化、指数型增长的重试周期: exponential ackoff + jitter  
  * client侧记录重试次数直方图, 传递到server, 进行分布判定, 交由server判定拒绝  
  * 只应该在失败这层重试,当重试仍然失败, 全局约定错误码"过载, 无需重试", 避免级联重试  


## 负载均衡
* 某个服务的负载会完全均匀的分发给所有后端任务, 最忙和最不忙的节点永远消耗同样数量的CPU  
  * 均衡的流量分发  
  * 可靠的识别异常节点  
  * scale-out, 增加同质节点扩容  
  * 减少错误, 提高可用性 (N+2冗余)  
* backend之间的load差异比较大  
  * 每个请求的处理成本不同  
  * 物理机环境的差异  
    * 服务器很难强同质性  
    * 存在内存资源争用(内存缓存、带宽、IO等)  
  * 性能因素  
    * FullGC  
    * JVM JIT  
  * JSQ(最闲轮训)  
    * 缺乏服务端全局视图, 目标: 需要综合考虑 负载+可用性  
  * the choice-of-2  
    * 选择backend: CPU, client: health, inflight, latency 作为指标, 使用一个简单的线性方程进行打分  
    * 对新启动的节点使用常量惩罚值(penalty), 以及使用探针方式最小化放量, 进行预热  
    * 打分较低的节点, 避免进入"永久黑名单"而无法恢复, 使用统计衰减的方式, 让节点指标逐渐恢复到初始状态  
    * 指标计算结合moving average, 使用时间衰减, 计算vt = v(t-1) * b + at * (1-b), b为若干次幂的倒数即: Math.Exp((-span) / 600ms)  
