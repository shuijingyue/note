Fragment

```java
安全访问getViewLifecycleOwner的方式: onCreate返回`@NonNull View`

public LifecycleOwner getViewLifecycleOwner() {
    if (mViewLifecycleOwner == null) {
        throw new IllegalStateException("Can't access the Fragment View's LifecycleOwner when "
                + "getView() is null i.e., before onCreateView() or after onDestroyView()");
    }
    return mViewLifecycleOwner;
}
```

addObserver

```java
State initialState = mState == DESTROYED ? DESTROYED : INITIALIZED;
// addObserver之后 mState要么是destroy,要么是initialized

// 同一个observer不会添加多次
```

LifecycleRegistry

handleLifecycleEvent & setCurrentState

moveToState -> sync -> backwardPass / forwardPass

