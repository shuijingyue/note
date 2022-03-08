Glide.with进行的操作

```java
  // Glide.java
  @NonNull
  public static RequestManager with(@NonNull FragmentActivity activity) {
    return getRetriever(activity).get(activity);
  }
```

```java
  // Glide.java
  @NonNull
  private static RequestManagerRetriever getRetriever(@Nullable Context context) {
    ...
    return Glide.get(context).getRequestManagerRetriever();
  }
```

```java
  // Glide.java
  @NonNull
  public RequestManagerRetriever getRequestManagerRetriever() {
    // GlideBuilder.build初始化
    return requestManagerRetriever; 
  }
```

```java
  // RequestManagerRetriever.java
  public RequestManager get(@NonNull FragmentActivity activity) {
    if (Util.isOnBackgroundThread()) {
      return get(activity.getApplicationContext());
    } else {
      assertNotDestroyed(activity);
      frameWaiter.registerSelf(activity);
      FragmentManager fm = activity.getSupportFragmentManager();
      return supportFragmentGet(activity, fm, /*parentHint=*/ null, isActivityVisible(activity));
    }
  }
```

```java
  // RequestManagerRetriever.java
  private static FrameWaiter buildFrameWaiter(GlideExperiments experiments) {
    // O 8
    // Q 10
    // x < 8 || x >= 10
    // HARDWARE_BITMAPS_SUPPORTED SDK_INT >= O
    // BLOCK_HARDWARE_BITMAPS_WHEN_GL_CONTEXT_MIGHT_NOT_BE_INITIALIZED SDK_INT < Q
    // 除8 9之外的版本
    if (!HardwareConfigState.HARDWARE_BITMAPS_SUPPORTED
        || !HardwareConfigState.BLOCK_HARDWARE_BITMAPS_WHEN_GL_CONTEXT_MIGHT_NOT_BE_INITIALIZED) {
      return new DoNothingFirstFrameWaiter();
    }
    //android 8 9
    return experiments.isEnabled(WaitForFramesAfterTrimMemory.class)
        ? new FirstFrameAndAfterTrimMemoryWaiter()
        : new FirstFrameWaiter();
  }

  // 这三个的registerSelf都是空实现 有什么用?? 
```

```java
  // pendingSupportRequestManagerFragments HashMap<FragmentManager, SupportRequestManagerFragment>
  // 由于fragment会post到looper的队尾执行,为了避免pending fragment二次添加,用pendingSupportRequestManagerFragments记录
  // 当fragment添加完成之后,记录就没有作用了,需要将记录删除,但fragment的添加并没有提供"添加完成的回调"
  // 所以将移除记录的消息又post到looper的队尾,可以在fragment添加完成后及时地将记录删除
  // 至此,空白fragment已经完成添加
  private SupportRequestManagerFragment getSupportRequestManagerFragment(
      @NonNull final FragmentManager fm, @Nullable Fragment parentHint) {
    // If we have a pending Fragment, we need to continue to use the pending Fragment. Otherwise
    // there's a race where an old Fragment could be added and retrieved here before our logic to
    // add our pending Fragment notices. That can then result in both the pending Fragmeng and the
    // old Fragment having requests running for them, which is impossible to safely unwind.
    SupportRequestManagerFragment current = pendingSupportRequestManagerFragments.get(fm);
    if (current == null) {
      current = (SupportRequestManagerFragment) fm.findFragmentByTag(FRAGMENT_TAG);
      if (current == null) {
        current = new SupportRequestManagerFragment(); // 此处会给lifecycle赋值
        current.setParentFragmentHint(parentHint);
        pendingSupportRequestManagerFragments.put(fm, current);
        fm.beginTransaction().add(current, FRAGMENT_TAG).commitAllowingStateLoss();
        handler.obtainMessage(ID_REMOVE_SUPPORT_FRAGMENT_MANAGER, fm).sendToTarget();
      }
    }
    return current;
  }

  // handler.handleMessage
  case ID_REMOVE_SUPPORT_FRAGMENT_MANAGER:
    FragmentManager supportFm = (FragmentManager) message.obj;
    // 验证是否添加成功,是否只添加了一个fragment,如果添加了两个会抛出异常
    // TODO 如果添加了两个/添加失败的处理
    if (verifyOurSupportFragmentWasAddedOrCantBeAdded(supportFm, hasAttemptedBefore)) {
      attemptedRemoval = true;
      key = supportFm;
      removed = pendingSupportRequestManagerFragments.remove(supportFm);
    }
    break;
```

Fragment和RequestManager如何关联

```java
  //在生成fragment时会给lifecycle赋值
  public SupportRequestManagerFragment() {
    this(new ActivityFragmentLifecycle());
  }

  @VisibleForTesting
  @SuppressLint("ValidFragment")
  public SupportRequestManagerFragment(@NonNull ActivityFragmentLifecycle lifecycle) {
    this.lifecycle = lifecycle;
  }
  //在fragment生命周期钩子方法中会回调lifecycle相关的回调
  public void onStart() {
    super.onStart();
    lifecycle.onStart();
  }

  public void onStop() {
    super.onStop();
    lifecycle.onStop();
  }

  public void onDestroy() {
    super.onDestroy();
    lifecycle.onDestroy();
    unregisterFragmentWithRoot();
  }

  // 在RequestManager将自己作为生命周期监听者注册到lifecycle中
  // 至此,RequestManager通过lifecycle作为桥梁与Fragment的产生了关联
```