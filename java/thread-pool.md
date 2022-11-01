## ThreadPoolExecutor

shotdown 
1. will reject new Task, 
2. allow previously submitted tasks to execute before
terminating

shotdownNow
1. prevent waiting tasks from starting
2. stop the running tasks

一旦终止
1. executor中不会有运行的任务
2. 不会有任务等待执行
3. 不能提交新的任务

没有用的ExecutorService应该被终止,以便回收其占用的资源

submit基于execute,创建并返回一个能够取消或等待完成的Future实例

invokeAny/invokeAll提供最通用的执行批量任务的方式,执行collection实例(List,Map,Set)容纳的任务,
至少等待一个或者全部任务完成

>ExecutorCompletionService

ThreadPoolExecutor

解决了两个问题

1. 改善处理大量异步任务的性能
2. 提供了一种绑定和管理在执行任务集合时消耗的资源(包括线程)的方法。

当提交了任务之后

1. 如果当前程数小于最大核心线程数,即便当前有空闲的线程,也会创建一个新的线程
去处理该任务
2. 如果当前核心线程数小于最大线程数,只有当队列满了时候才会创建新的线程

corePoolSize == maximumPoolSize fixed-size thread pool
maximumPoolSize == +Infinity(Integer.MAX_VALUE) 容纳任意数量的并发任务

一般corePoolSize和maximumPoolSize在构造方法中赋值,也可以通过setCorePoolSize和
setMaximumPoolSize动态设置

尽管核心线程只有在新的任务到来时才会初始化并创建,但是可以通过`prestartCoreThread`和`prestartAllCoreThreads`
方法来动态的修改

**new thread**

ThreadFactory 如果没有指定ThreadFactory,则使用Executors.defaultThreadFactory

DefaultThreadFactory特点:

创建的线程的优先级一样, 线程组一样, 没有守护状态(non-daemon status)

提供特定的ThreadFactory可以更改线程的名称, threadGroup, 优先级和守护状态等等

线程要处理"modifyThread" permission, 如果使用ThreadExecutorPool的工作线程或者其他线程没有处理此权限,
服务可能会被降级: 配置更改可能不会及时生效，终止线程池后可能处于可以终止但尚未完成的状态

**keep-alive times**

如果当前线程池内数的线程数量超过了核心线程数,多余的线程如果空闲时间一旦超过了keepAliveTime就会
终止,这样可以有效的减少线程池不再处于活动状态时的资源消耗. 如果线程池之后被激活, 新的线程又会被创建
可以通过setKeepAliveTime来动态的修改keepAliveTimes参数. 可以将其为Long.MAX_VALUE来防止
线程在Executor shotdown之前终止. Keep-alive机制默认只适用于超出核心线程数的线程(非核心线程),
但是可以通过`allowCoreThreadTimeOut`方法将time-out机制作用于核心线程

**queue**

任何阻塞队列都可以被传递并用于保存提交的任务, 队列与线程池线程数量数量的关系:

1. 如果正在运行的线程数小于核心线程数, 优先创建新的线程而不会将任务加入队列
2. 如果当前运行的线程数大于核心线程数, 优先将任务加入队列而不是开启新的线程
3. 如果当前的队列无法入队, 除非超过最大线程数, 否则就会创建一个新的线程,如果无法入队且当前线程数
超过了最大线程数则任务会被丢弃

三个入队的一般策略

Direct handoffs SynchronousQueue 任务之间存在依赖
Unbounded queues LinkedBlockingQueue 任务之间独立
Bounded queues ArrayBlockingQueue 避免资源耗尽,但是很难调优和控制. 会导致阻塞,降低吞吐量

**reject tasks**

Executor已经shotdown之后,再次通过submit方法提交的任务会被reject, 如果队列满并且已达最大线程数
,任务也会被reject. 只要发生了reject, 就会调用RejectedExecutionHandler.rejectedExecution方法

提供了四个预定义的处理策略

1. 默认的ThreadPoolExecutor.AbortPolicy, 一旦发生了reject 就会抛出一个
RejectedExecutionException runtime异常
2. ThreadPoolExecutor.CallerRunsPolicy 在提交任务的线程执行任务. 提供一个简单的反馈控制机制
用于降低提交任务的速率
3. ThreadPoolExecutor.DiscardPolicy 直接被丢弃,无其他任何操作
4. ThreadPoolExecutor.DiscardOldestPolicy 如果线程没有shotdown, 队头的任务会被丢弃,
重复入队操作直至入队成功

也可以定义并使用其他的RejectedExecutionHandler, 但是当策略设计为仅在特定容量或排队策略下工
作时要格外小心

**Hook methods**

提供可被重写的保护方法 `beforeExecute` & `afterExecute` 分别在任务执行前后调用

用途: 设置执行环境

- reinitializing ThreadLocals
- gathering statistics
- adding log entries.

`terminated`可以被重写用于在Executor完全停止后执行特定的操作

如果 hook, callback, BlockingQueue methods抛出异常后,内部的工作线程可能会失败, 突然终止
或者被替换

**Queue Maintain**

getQueue方法可以获取工作队列用于监控和debug. 使用这个方法用于其他任何用途都是非常不建议的

当大量任务被取消时, 可以使用`remove`和`purge`方法协助存储资源的回收

**Reclamation**

在程序中, 即便没有被调用terminate方法,一个pool不再被引用或者没有保存线程也会被回收. 可以通过
设置恰当的keep-alive时间, 将核心线程数设置为`< 0`的值或者设置allowCoreThreadTimeOut来允许
线程终止

## ScheduledThreadPoolExecutor

一个ThreadPoolExecutor实例能够让一条命令延迟一段时间执行,或周期性的执行

需要多个工作线程的场景或者需要ThreadPoolExecutor灵活的可拓展能力时,
ScheduledThreadPoolExecutor是比Timer更好的选择

延迟任务不会在启动之后立即执行, 延时任务的实时性是无法保证的. 两个调度时间完全相同的任务的先后
执行顺序取决于其submit的先后顺序(FIFO)

一个已经提交的任务在执行前被取消, 其执行会被阻止. 默认情况下, 被取消的任务在未到达延迟时间之前
是不会自动从队列中删除的. 虽然这可以进行进一步的检查和监视，但它也可能导致取消的任务无限制地保留

通过`scheduleAtFixedRate`或`scheduleWithFixedDelay`反复执行或者周期性的执行某个任务,
该任务的执行时间不会重叠. 虽然任务可能会由不同的线程执行,但是先执行的任务早于后执行的任务生效

尽管`ScheduledThreadPoolExecutor`继承自`ThreadPoolExecutor`, 但是继承的一些tuning方法
对于`ScheduledThreadPoolExecutor`来说没用. `ScheduledThreadPoolExecutor`的行为跟
fixed-size with unbound queue的ThreadPoolExecutor一致, 调整maximumPoolSize没有任何
作用. 另外,将corePoolSize设置为0或者使用allowCoreThreadTimeOut并不是一个好的做法,这可能会
导致线程池符合运行条件后没有可用的工作线程
