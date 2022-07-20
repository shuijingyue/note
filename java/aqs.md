AQS提供了一个可用于实现blocking locks和相关的synchronizer, 这个框架依赖FIFO wait queue
AQS被设计为大多数依赖single atomic int 来表示状态的synchronizer的基础
AQS的子类必须重写用于更改状态的protected methods, 重写的方法定义了在获取或释放对象时当前状态
的含义。AQS的其他methods管理队列和阻塞机制
AQS的子类可以维护其他状态字段，但是只有使用getState、setState和compareAndSetState方法操作
的原子更新的int值才会被跟踪。
AQS的子类应该被声明为non-public内部类用于实现其外部类的同步属性

AQS没有实现任何synchronization interface. AQS声明了一些方法, 比如
`acquireInterruptibly`, 具体的锁和相关的同步器可以通过适当的调用这些方法来实现自己的公共方法

AQS默认支持独占(排他)模式,也支持共享模式. 如果是排他模式, 其他线程执行acquire操作将会失败. 
多个线程在共享模式下执行acquire操作可能(不一定)成功. 

AQS本身并不会区分排他和共享模式, 在定义上, 如果是共享模式, 一个线程执行了acquire操作之后, 
必须确定下一个等待线程(如果存在)是否也可以执行acquire. 等待线程在不同的模式下共享同一个FIFO 
queue. 通常AQS的子类只会支持一种模式, 但这两种模式都可以发挥作用，例如在ReadWriteLock中。仅
支持一种模式的子类不能重写未使用到的方法.

AQS内声明了一个嵌套类ConditionObject, 作为Condition接口的实现, 

AQS内的方法对于追踪拥有排他同步器的线程非常有用,我们鼓励您使用它们——这使得监控和诊断工具能够协助
用户确定哪些线程持有锁。
 
尽管AQS基于FIFO queue,但是并不自动强制使用FIFO acquire策略

```java
// 非公平锁
static final class NonfairSync extends Sync {
	...
	final void lock() {
		if (compareAndSetState(0, 1))
			setExclusiveOwnerThread(Thread.currentThread());
		else
			acquire(1);
		}
  ...
}
```
