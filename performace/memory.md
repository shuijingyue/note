getprop ro.product.model 手机型号 机型定位 功能过滤

dumpsys window display

settings get secure android_id

service call iphonesubinfo 1

cat /proc/cpuinfo

cat /system/build.prop

adb shell cat /proc/meminfo

> cache + buffer + slab = available

linux动态内存分配
mmap 数据缓写机制 日志系统 logan xlog

KernelStack 跟用户线程绑定
PageTables 虚拟地址与物理地址转化

### adb shell dumpsys meminfo

rss 共享库 so 动态链接库

优先级
foreground
A services
B services

强 弱 软 虚

koom matrix hah

fork进程

mat profile

WeakHashMap

Debug MemoryInfo

爱奇艺 xhook

activityLifecycleCallback onDestory 中调用 gc 会影响Activity的启动,阈值检查 
gc

## 内存泄漏

可以作为gc root的对象

静态变量 单例 内部类 匿名内部类 动画 io close操作

adb shell am dumpheap `<processname>` `<filename>`

```java
Debug.dumpHproData(filename) // 会暂时挂起所有线程
```

libmemunreachale + breakpad

检测Native内存泄漏

malloc_debug

binonic库

ASan 插桩 hook