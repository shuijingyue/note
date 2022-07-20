在xml中定义的`navGraph`和`defaultNavHost`在NavHostFragment的onInflate方法中解析并保存
```java
this.graphId = graphId
defaultNavHost = true
```

NavBackStackEntry持有lifecycle: LifecycleRegistry

记录当前的NavBackStackEntry的 Lifecycle.State 与 hostLifecycleState不同, 
hostLifecycleState与NavHostFragment保持一致, 而NavBackStackEntry的lifecycle.state受到
maxLifecycle的制约, 与NavHostFragment不一定相同
