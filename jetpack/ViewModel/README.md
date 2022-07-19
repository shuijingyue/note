## `SavedStateHandle`

- SavedStateHandleController   // SavedStateViewModelFactory
- SavedStateRegistryController // activity
- SavedStateHandle
- SavedStateRegistry
- SavedStateRegistryOwner
- SavedStateProvider

```java
// ComponentActivity.java
@Override
protected void onCreate(@Nullable Bundle savedInstanceState) {
    // Restore the Saved State first so that it is available to
    // OnContextAvailableListener instances
    // ①仅将State取出
    // 实际的恢复操作在②
    mSavedStateRegistryController.performRestore(savedInstanceState); // ①
    mContextAwareHelper.dispatchOnContextAvailable(this);             // ②
    // mSavedStateRegistryController SavedStateRegistryController
    // 状态恢复起始点
    // final SavedStateRegistryController mSavedStateRegistryController = SavedStateRegistryController.create(this);
    ...
}
```

```java
// SavedStateRegistryController.java
@MainThread
public void performRestore(@Nullable Bundle savedState) {
    ...
    // SavedStateRegistry mRegistry
    // mRegistry在构造方法中创建
    mRegistry.performRestore(lifecycle, savedState);
}
```

```java
// SavedStateRegistry
void performRestore(@NonNull Lifecycle lifecycle, @Nullable Bundle savedState) {
    if (mRestored) {
        throw new IllegalStateException("SavedStateRegistry was already restored.");
    }
    if (savedState != null) {
        // SAVED_COMPONENTS_KEY = "androidx.lifecycle.BundlableSavedStateRegistry.key"
        mRestoredState = savedState.getBundle(SAVED_COMPONENTS_KEY);
    }
    ...
    mRestored = true;
}
```

最终的目的是将存储在savedState中的值取出，并将其赋值给mRestoredState,以便通过ViewModelProvider
获取ViewModel时将保存的状态通过key取出并传递给SavedStateHandle

实际状态恢复的起始点：SavedStateViewModelFactory.create

```java
// SavedStateViewModelFactory.java
@NonNull
@Override
public <T extends ViewModel> T create(@NonNull String key, @NonNull Class<T> modelClass) {
    boolean isAndroidViewModel = AndroidViewModel.class.isAssignableFrom(modelClass);
    Constructor<T> constructor;
    if (isAndroidViewModel && mApplication != null) {
        constructor = findMatchingConstructor(modelClass, ANDROID_VIEWMODEL_SIGNATURE);
    } else {
        constructor = findMatchingConstructor(modelClass, VIEWMODEL_SIGNATURE);
    }
    // doesn't need SavedStateHandle
    if (constructor == null) {
        return mFactory.create(modelClass);
    }

    // 状态恢复的起点
    // mSavedStateRegistry
    // mSavedStateRegistry = owner.getSavedStateRegistry();
    // owner ComponentActivity or Fragment
    // savedInstanceState保存在mSavedStateRegistry中
    SavedStateHandleController controller = SavedStateHandleController.create(
            mSavedStateRegistry, mLifecycle, key, mDefaultArgs);
    try {
        T viewmodel;
        if (isAndroidViewModel && mApplication != null) {
            viewmodel = constructor.newInstance(mApplication, controller.getHandle());
        } else {
            viewmodel = constructor.newInstance(controller.getHandle());
        }
        viewmodel.setTagIfAbsent(TAG_SAVED_STATE_HANDLE_CONTROLLER, controller);
        return viewmodel;
    } catch (IllegalAccessException e) {
        throw new RuntimeException("Failed to access " + modelClass, e);
    } catch (InstantiationException e) {
        throw new RuntimeException("A " + modelClass + " cannot be instantiated.", e);
    } catch (InvocationTargetException e) {
        throw new RuntimeException("An exception happened in constructor of "
                + modelClass, e.getCause());
    }
}
```

```java
static SavedStateHandleController create(SavedStateRegistry registry, Lifecycle lifecycle,
        String key, Bundle defaultArgs) {
    // 通过key获取保存的状态
    Bundle restoredState = registry.consumeRestoredStateForKey(key);
    // 将restoredState赋值给SavedStateHandle.mRegular
    // defaultArgs为getIntent().getExtras() or null
    // 该SavedStateHandle已经将保存的状态恢复
    SavedStateHandle handle = SavedStateHandle.createHandle(restoredState, defaultArgs);
    SavedStateHandleController controller = new SavedStateHandleController(key, handle);
    controller.attachToLifecycle(registry, lifecycle);
    tryToAddRecreator(registry, lifecycle);
    return controller;
}
```

状态何时保存的？

```java
// ComponentActivity.java
@CallSuper
@Override
protected void onSaveInstanceState(@NonNull Bundle outState) {
    ...
    mSavedStateRegistryController.performSave(outState);
}
```

```java
// SavedStateRegistryController.java
@MainThread
public void performSave(@NonNull Bundle outBundle) {
    mRegistry.performSave(outBundle);
}
```

```java
