性能

- 图片解码速度
- 解码多余的工作量

解码速度快
尽量少的与图片展示无关的额外工作
不阻塞主线程
不产生过多的gc

downsampling 智能缩减像素采样
资源重用
活动状态的activity和fragment优先

不用检查url是否为空, 自动清空ImageView或者设置占位图placeholder.fallback,自动取消请求

Glide唯一的要求是，任何可重用的视图或目标，你可能已经开始加载到以前的位置，要么有一个新的加载开始，或通过clear() API显式清除。