
在xml中定义的`navGraph`和`defaultNavHost`在NavHostFragment的onInflate方法中解析并保存
```java
this.graphId = graphId
defaultNavHost = true
```

`NavBackStackEntry`持有`lifecycle: LifecycleRegistry`

记录当前的NavBackStackEntry的 Lifecycle.State 与 hostLifecycleState不同, 
hostLifecycleState与NavHostFragment保持一致, 而NavBackStackEntry的lifecycle.state受到
maxLifecycle的制约, 与NavHostFragment不一定相同

fragment

## setup
navigation一切的起点 setGraph

```java
// NavHostFragment onCreate
if (graphId != 0) {
    // Set from onInflate()
    navHostController!!.setGraph(graphId)
} else {
   ...
}
// ==> NavContoller onGraphCreated
...
if (_graph != null && backQueue.isEmpty()) {
    val deepLinked =
        !deepLinkHandled && activity != null && handleDeepLink(activity!!.intent)
    if (!deepLinked) {
        // Navigate to the first destination in the graph
        // if we haven't deep linked to a destination
        navigate(_graph!!, startDestinationArgs, null, null)
    }
} else {
    dispatchOnDestinationChanged()
}
// ==> NavContoller navigate
...
if (navOptions?.shouldRestoreState() == true && backStackMap.containsKey(node.id)) {
    navigated = restoreStateInternal(node.id, finalArgs, navOptions, navigatorExtras)
} else {
    ...
    if (navOptions?.shouldLaunchSingleTop() == true &&
        node.id == currentBackStackEntry?.destination?.id
    ) {
        ...
    } else {
        // Not a single top operation, so we're looking to add the node to the back stack
        val backStackEntry = NavBackStackEntry.create(
            context, node, finalArgs, hostLifecycleState, viewModel
        )
        // navigator is GraphNavigator
        navigator.navigateInternal(listOf(backStackEntry), navOptions, navigatorExtras) {
            navigated = true
            addEntryToBackStack(node, finalArgs, it)
        }
    }
}
...
// ==> GraphNavigator navigate ==>
// 根据startDestination获取合适的navigator
val navigator = navigatorProvider.getNavigator<Navigator<NavDestination>>(
    startDestination.navigatorName
)
val startDestinationEntry = state.createBackStackEntry(
    startDestination,
    startDestination.addInDefaultArgs(args)
)
// 通过navigator导航到startDestination
navigator.navigate(listOf(startDestinationEntry), navOptions, navigatorExtras)
// 故一开始backQueue的size为2
```

`onInflate()`当解析xml时会调用，如果是通过transaction添加，不会调用

```java
 @CallSuper
public override fun onInflate(
    context: Context,
    attrs: AttributeSet,
    savedInstanceState: Bundle?
) {
    super.onInflate(context, attrs, savedInstanceState)
    context.obtainStyledAttributes(
        attrs,
        androidx.navigation.R.styleable.NavHost
    ).use { navHost ->
        val graphId = navHost.getResourceId(
            androidx.navigation.R.styleable.NavHost_navGraph, 0
        )
        if (graphId != 0) {
            this.graphId = graphId // 获取graph资源id
        }
    }
    context.obtainStyledAttributes(attrs, R.styleable.NavHostFragment).use { array ->
        val defaultHost = array.getBoolean(R.styleable.NavHostFragment_defaultNavHost, false)
        if (defaultHost) {
            defaultNavHost = true // 判断是否是主导航宿主
        }
    }
}
```

navigate调用链

NavController.navigate() => 
    FragmentNavigator.navigateInternal (在NavController内声明的扩展函数) 
        => FragmentNavigator.navigate
        => ActivityNavigator.navigate

**FragmentNavigator.navigate非常粗暴，直接replace**

NavContoller NavigatorProvider初始化

```java
// NavHostFragment onCreate
navHostController = NavHostController(context)

// NavController
init {
    _navigatorProvider.addNavigator(NavGraphNavigator(_navigatorProvider))
    _navigatorProvider.addNavigator(ActivityNavigator(context))
}

// NavHostFragment onCreate
onCreateNavHostController(navHostController!!)
// NavHostFragment onCreateNavHostController
onCreateNavController(navHostController)
// NavHostFragment onCreateNavController     
navController.navigatorProvider +=
    DialogFragmentNavigator(requireContext(), childFragmentManager)
navController.navigatorProvider.addNavigator(createFragmentNavigator())

```

## navigate

## popBackStack

## restore state

`NavController`持有`lifecycleObserver`, 在`HavHostFragment.onCreate`中将
`lifecycleObserver`注册到了NavHostFragment上

```java
@CallSuper
public override fun onCreate(savedInstanceState: Bundle?) {
    var context = requireContext()
    navHostController = NavHostController(context)
    navHostController!!.setLifecycleOwner(this)
}
```

每个NavBackStackEntry

记录当前的 `NavBackStackEntry` 的 `Lifecycle.State` 与 `hostLifecycleState` 不同, 
`hostLifecycleState` 与 `NavHostFragment`保持一致, 而`NavBackStackEntry`的
`lifecycle.state`受到`maxLifecycle`的制约, 与NavHostFragment不一定相同
