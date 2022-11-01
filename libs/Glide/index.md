Glide.with进行的操作

1. 如果Glide尚未实例化,则执行Glide的实例化并初始化
2. 通过glide的`requestManagerRetriver`属性获取RequestManager, 
  1. `supportFragmentGet(Context,FragmentManager,Fragment,boolean)` with传入fragment
  parentHint 为传入的fragment, with传入Activity, parentHint为null
  2. 生成RequestManagerFragment, 设置其parentFragmentHint, 将其挂载到当前的activity或者
  fragment中
  fragment -> childFragmentManager -> requestManagerFragment -> parentFragmentHint
  activity -> fragmentManager -> requestManagerFragment

虽然挂载到activity的requestManagerFragment.parentFragmentHint为null,但其rootRequestManagerFragment
不为null, 其rootRequestManagerFragment为自身, 其rootRequestManagerFragment属性设置时机
为onAttach

挂载到activity的requestManagerFragment只会经历一次registerFragmentWithRoot
而挂载到fragment的requestManagerFragment会经历两次registerFragmentWithRoot

RequestManagerFragment.requestManagerTreeNode

```java
  // Glide.java
  @NonNull
  private static RequestManagerRetriever getRetriever(@Nullable Context context) {
    // requestManagerRetriever
    return Glide.get(context).getRequestManagerRetriever();
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

   private SupportRequestManagerFragment getSupportRequestManagerFragment(
      @NonNull final FragmentManager fm, @Nullable Fragment parentHint) {
    SupportRequestManagerFragment current = pendingSupportRequestManagerFragments.get(fm);
    if (current == null) {
      current = (SupportRequestManagerFragment) fm.findFragmentByTag(FRAGMENT_TAG);
      if (current == null) {
        current = new SupportRequestManagerFragment(); // 此处会给lifecycle赋值
        current.setParentFragmentHint(parentHint); // 设置rootRequestManagerFragment
        pendingSupportRequestManagerFragments.put(fm, current);
        // attach之后会会再次设置rootRequestManagerFragment
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

  // 在RequestManager将自身注册到lifecycle中
  // Our usage is safe here.
  @SuppressWarnings("PMD.ConstructorCallsOverridableMethod")
  RequestManager(
      Glide glide,
      Lifecycle lifecycle, // RequestManagerFragment.lifecycle
      RequestManagerTreeNode treeNode,
      RequestTracker requestTracker,
      ConnectivityMonitorFactory factory,
      Context context) {
    // ...
    
    if (Util.isOnBackgroundThread()) {
      Util.postOnUiThread(addSelfToLifecycle);
    } else {
      lifecycle.addListener(this);
    }
    lifecycle.addListener(connectivityMonitor);

    // ...
  }
  // 至此,RequestManager通过lifecycle作为桥梁与Fragment的产生了关联
```

RequestManager.load

```java
RequestBuilder<Drawable>

// resourceClass Drawable.class
return new RequestBuilder<>(glide, this, resourceClass, context);

// requestBuilder.model = "https://com.play.wa/app.png"
// requestBuilder.isModelSet = true
```

RequestBuilder.into
```java
// 返回值 DrawableImageViewTarget
// target: DrawableImageViewTarget
// targetListener null
// requestOptions this(RequestBuilder)
// callbackExecutor Executors.mainThreadExecutor()
Request request = buildRequest(target, targetListener, options, callbackExecutor);
// request: SingleRequest

target.getRequest // target 为 DrawableImageViewTarget

// ViewTarget.getRequest => view.getTag(tagId)
// 检测是否已经设置过一次
// 分支1 如果已经存在request且是相同(通过isEquivalentTo)的request
// 已存在的request.isRuning为false,执行request.begin;
// 分支2
// 用view.tag记录request
// requestManager.track(target, request);
```

targetTracker与requestTracker均在构造方法中new出来
每个requestManager拥有唯一的targetTracker和requestTracker
targetTracker记录target
requestTracker记录request 并执行 request.begin

SingleRequest

status = Status.WAITING_FOR_SIZE
overriderWidth overrideHeight默认值UNSET(-1)
如果size有效,直接调用onSizeReady回调

```java
// ViewTarget.sizeDeterminer
// 通过ViewTreeObserver(preDraw)获取size
```

onSizeReady内通过Engine开始真正的加载

state = Status.RUNNING

engine.load

Engine

```text
Class<?> resourceClass Object.class
Class<R> transcodeClass Drawable.class
```

创建EngineKey

loadFromMemory

loadFromActiveResources // ActiveResources
loadFromCache // LruResourceCache

engine.waitForExistingOrStartNewJob // 磁盘缓存 + 请求网络

Jobs存储EngineJob

EngineJob分为两类, 通过onlyRetrieveFromCache区分

`EngineJob<Drawable>` `DecodeJob<Drawable>`

```java
class EngineJob<R> implements DecodeJob.Callback<R>, Poolable

class DecodeJob<R>
    implements DataFetcherGenerator.FetcherReadyCallback,
        Runnable,
        Comparable<DecodeJob<?>>,
        Poolable
```

engineJob.addCallback(cb, callbackExecutor)

cb SingleRequest
callbackExecutor Executors.mainThreadExecutor

engineJob.start(encodeJob)

DecodeJob.run

runReason = RunReason.INITIALIZE
```java
// DiskCacheStrategy
public static final DiskCacheStrategy AUTOMATIC =
    new DiskCacheStrategy() {
      @Override
      public boolean isDataCacheable(DataSource dataSource) {
        return dataSource == DataSource.REMOTE;
      }

      @Override
      public boolean isResourceCacheable(
          boolean isFromAlternateCacheKey, DataSource dataSource, EncodeStrategy encodeStrategy) {
        return ((isFromAlternateCacheKey && dataSource == DataSource.DATA_DISK_CACHE)
                || dataSource == DataSource.LOCAL)
            && encodeStrategy == EncodeStrategy.TRANSFORMED;
      }

      @Override
      public boolean decodeCachedResource() {
        return true;
      }

      @Override
      public boolean decodeCachedData() {
        return true;
      }
    };
```

stage = getNextStage(INITIALIZE) -> Stage.RESOURCE_CACHE

currentGenerator = ResourceCacheGenerator

stage = Stage.DATA_CACHE

currentGenerator = DataCacheGenerator

state = Stage.SOURCE

currentGenerator = SourceGenerator

state = State.FINISHED

```java
// SourceGenerator
```

```java
.append(String.class, InputStream.class, new DataUrlLoader.StreamFactory<String>())
.append(String.class, InputStream.class, new StringLoader.StreamFactory())
.append(String.class, ParcelFileDescriptor.class, new StringLoader.FileDescriptorFactory())
.append(String.class, AssetFileDescriptor.class, new StringLoader.AssetFileDescriptorFactory())
```
  