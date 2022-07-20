## source code detail

Fragment.mHost赋值

`FragmentStateManager`
`attach`

`FragmentLayoutInflaterFactory`
`onCreateView`


## Transaction Lifecycle

![Fragment状态与Lifecycle.State以及生命周期方法的关系](https://upload-images.jianshu.io/upload_images/12169089-c96a7848e78ad43f.jpg)

1. add
```log
2022-04-21 16:08:08.069 11595-11595/program.playio D/LifecycleFragment: onAttach
2022-04-21 16:08:08.069 11595-11595/program.playio D/LifecycleFragment: onCreate
2022-04-21 16:08:08.070 11595-11595/program.playio D/LifecycleFragment: onCreateView
2022-04-21 16:08:08.073 11595-11595/program.playio D/LifecycleFragment: onViewCreated
2022-04-21 16:08:08.073 11595-11595/program.playio D/LifecycleFragment: onStart
2022-04-21 16:08:08.074 11595-11595/program.playio D/LifecycleFragment: onResume
```

2. remove
```log
2022-04-21 16:08:41.740 12035-12035/program.playio D/LifecycleFragment: onPause
2022-04-21 16:08:41.741 12035-12035/program.playio D/LifecycleFragment: onStop
2022-04-21 16:08:41.743 12035-12035/program.playio D/LifecycleFragment: onDestroyView
2022-04-21 16:08:41.745 12035-12035/program.playio D/LifecycleFragment: onDestroy
2022-04-21 16:08:41.746 12035-12035/program.playio D/LifecycleFragment: onDetach
```

**detach 和 destroy 是否执行取决于有没有加入回退栈**, 加入回退栈后remove不会回调destory detach
```log
2022-04-21 16:09:18.966 12125-12125/program.playio D/LifecycleFragment: onPause
2022-04-21 16:09:18.967 12125-12125/program.playio D/LifecycleFragment: onStop
2022-04-21 16:09:18.969 12125-12125/program.playio D/LifecycleFragment: onDestroyView
```

3. detach
```log
2022-04-21 16:10:05.669 12125-12125/program.playio D/LifecycleFragment: onPause
2022-04-21 16:10:05.671 12125-12125/program.playio D/LifecycleFragment: onStop
2022-04-21 16:10:05.675 12125-12125/program.playio D/LifecycleFragment: onDestroyView
```

4. attach
```log
2022-04-21 16:10:35.196 12125-12125/program.playio D/LifecycleFragment: onCreateView
2022-04-21 16:10:35.198 12125-12125/program.playio D/LifecycleFragment: onViewCreated
2022-04-21 16:10:35.200 12125-12125/program.playio D/LifecycleFragment: onStart
2022-04-21 16:10:35.202 12125-12125/program.playio D/LifecycleFragment: onResume
```

## trap
1. 第一次添加fragment触发的onResume中isVisible并不准确,可能为false
2. destoryView detach中更新view无效,因为view已经detach

