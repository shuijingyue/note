init进程

PhoneWindow

1. 挂载和创建文件系统
2. 解析rc文件
3. 等待处理事件
   1. 执行action
   2. 检测重启进程
   3. 接收SIGCHILD信号，执行对应的方法

system/core/rootdir/init.rc

```bash
...
on late-init
    ...
    trigger zygote-start
    ...
...    
```

system/core/rootdir/init.zygote64_32.rc

```bash
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server --socket-name=zygote
    ...
```

frameworks/base/cmds/app_process/app_main.cpp
*注意：此时zygote进程创建，以下代码的执行均在zygote进程中*

1. 初始化AndroidRuntime
2. 注册android jni方法
3. 执行ZygoteInit#main方法

```c++
// service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server --socket-name=zygote
while (i < argc) {
    const char* arg = argv[i++];
    if (strcmp(arg, "--zygote") == 0) {
        zygote = true;
        niceName = ZYGOTE_NICE_NAME;
    } else if (strcmp(arg, "--start-system-server") == 0) {
        startSystemServer = true;
    } else if (strcmp(arg, "--application") == 0) {
        application = true;
    } else if (strncmp(arg, "--nice-name=", 12) == 0) {
        niceName.setTo(arg + 12);
    } else if (strncmp(arg, "--", 2) != 0) {
        className.setTo(arg);
        break;
    } else {
        --i;
        break;
    }
}

...

if (zygote) {
    // zygote
    runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
} else if (className) {
    // app
    runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
} else {
    fprintf(stderr, "Error: no class name or --zygote supplied.\n");
    app_usage();
    LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
}
```

frameworks/base/core/jni/AndroidRuntime.cpp

```c++
/*
 * Start the Android runtime.  This involves starting the virtual machine
 * and calling the "static void main(String[] args)" method in the class
 * named by "className".
 *
 * Passes the main function two arguments, the class name and the specified
 * options string.
 */
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
{
    // ...

    //const char* kernelHack = getenv("LD_ASSUME_KERNEL");
    //ALOGD("Found LD_ASSUME_KERNEL='%s'\n", kernelHack);

    /* start the virtual machine */
    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);
    JNIEnv* env;
    if (startVm(&mJavaVM, &env, zygote, primary_zygote) != 0) {
        return;
    }
    onVmCreated(env);

    /*
     * Register android functions.
     */
    if (startReg(env) < 0) {
        ALOGE("Unable to register all android natives\n");
        return;
    }

    // ...

    /*
     * Start VM.  This thread becomes the main thread of the VM, and will
     * not return until the VM exits.
     */
    char* slashClassName = toSlashClassName(className != NULL ? className : "");
    // className com.android.internal.os.ZygoteInit
    jclass startClass = env->FindClass(slashClassName);
    if (startClass == NULL) {
        ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
        /* keep going */
    } else {
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
        if (startMeth == NULL) {
            ALOGE("JavaVM unable to find main() in '%s'\n", className);
            /* keep going */
        } else {
            env->CallStaticVoidMethod(startClass, startMeth, strArray);

#if 0
            if (env->ExceptionCheck())
                threadExitUncaughtException(env);
#endif
        }
    }
    free(slashClassName);

    ALOGD("Shutting down VM\n");
    if (mJavaVM->DetachCurrentThread() != JNI_OK)
        ALOGW("Warning: unable to detach main thread\n");
    if (mJavaVM->DestroyJavaVM() != 0)
        ALOGW("Warning: VM did not shut down cleanly\n");
}
```

com.android.internal.os.ZygoteInit.java

```java
/**
  * This is the entry point for a Zygote process.  It creates the Zygote server, loads resources,
  * and handles other tasks related to preparing the process for forking into applications.
  *
  * This process is started with a nice value of -20 (highest priority).  All paths that flow
  * into new processes are required to either set the priority to the default value or terminate
  * before executing any non-system code.  The native side of this occurs in SpecializeCommon,
  * while the Java Language priority is changed in ZygoteInit.handleSystemServerProcess,
  * ZygoteConnection.handleChildProc, and Zygote.childMain.
  *
  * @param argv  Command line arguments used to specify the Zygote's configuration.
  */
@UnsupportedAppUsage
public static void main(String[] argv) {
    // ...

    Runnable caller;
    try {
        // ...
        for (int i = 1; i < argv.length; i++) {
            if ("start-system-server".equals(argv[i])) {
                startSystemServer = true;
            } else if ("--enable-lazy-preload".equals(argv[i])) {
                enableLazyPreload = true;
            } else if (argv[i].startsWith(ABI_LIST_ARG)) {
                abiList = argv[i].substring(ABI_LIST_ARG.length());
            } else if (argv[i].startsWith(SOCKET_NAME_ARG)) {
                zygoteSocketName = argv[i].substring(SOCKET_NAME_ARG.length());
            } else {
                throw new RuntimeException("Unknown command line argument: " + argv[i]);
            }
        }

        // ...

        zygoteServer = new ZygoteServer(isPrimaryZygote); // 主要用于socket通信

        if (startSystemServer) {
            Runnable r = forkSystemServer(abiList, zygoteSocketName, zygoteServer);

            // {@code r == null} in the parent (zygote) process, and {@code r != null} in the
            // child (system_server) process.
            if (r != null) {
                r.run();
                return;
            }
        }

        Log.i(TAG, "Accepting command socket connections");

        // The select loop returns early in the child process after a fork and
        // loops forever in the zygote.
        caller = zygoteServer.runSelectLoop(abiList);
    } catch (Throwable ex) {
        Log.e(TAG, "System zygote died with fatal exception", ex);
        throw ex;
    } finally {
        if (zygoteServer != null) {
            zygoteServer.closeServerSocket();
        }
    }

    // We're in the child process and have exited the select loop. Proceed to execute the
    // command.
    if (caller != null) {
        caller.run();
    }
}
```
## SystemServer的诞生
ZygoteInit.java#forkSystemServer

```java
/**
    * Prepare the arguments and forks for the system server process.
    *
    * @return A {@code Runnable} that provides an entrypoint into system_server code in the child
    * process; {@code null} in the parent.
    */
private static Runnable forkSystemServer(String abiList, String socketName,
        ZygoteServer zygoteServer) {
    // ...

    int pid;

    try {
        // 参数设置

        /* Request to fork the system server process */
        pid = Zygote.forkSystemServer(
                parsedArgs.mUid, parsedArgs.mGid,
                parsedArgs.mGids,
                parsedArgs.mRuntimeFlags,
                null,
                parsedArgs.mPermittedCapabilities,
                parsedArgs.mEffectiveCapabilities);
    } catch (IllegalArgumentException ex) {
        throw new RuntimeException(ex);
    }

    /* For child process */
    if (pid == 0) {
        if (hasSecondZygote(abiList)) {
            waitForSecondaryZygote(socketName);
        }

        zygoteServer.closeServerSocket();
        return handleSystemServerProcess(parsedArgs);
    }

    return null;
}
```

Zygote.java#forkSystemServer

```java
static int forkSystemServer(int uid, int gid, int[] gids, int runtimeFlags,
        int[][] rlimits, long permittedCapabilities, long effectiveCapabilities) {
    ZygoteHooks.preFork();
    // nativeForkSystemServer native方法
    int pid = nativeForkSystemServer(
            uid, gid, gids, runtimeFlags, rlimits,
            permittedCapabilities, effectiveCapabilities);

    // Set the Java Language thread priority to the default value for new apps.
    Thread.currentThread().setPriority(Thread.NORM_PRIORITY);

    ZygoteHooks.postForkCommon();
    return pid;
}
```

frameworks/base/core/jni/com_android_internal_os_Zygote.cpp

```c++
static jint com_android_internal_os_Zygote_nativeForkSystemServer(
        JNIEnv* env, jclass, uid_t uid, gid_t gid, jintArray gids,
        jint runtime_flags, jobjectArray rlimits, jlong permitted_capabilities,
        jlong effective_capabilities) {
  // ...

  pid_t pid = zygote::ForkCommon(env, true,
                                 fds_to_close,
                                 fds_to_ignore,
                                 true);
  if (pid == 0) {
    // ...
  } else if (pid > 0) {
    // ...
  }
  return pid;
}

// Utility routine to fork a process from the zygote.
pid_t zygote::ForkCommon(JNIEnv* env, bool is_system_server,
                         const std::vector<int>& fds_to_close,
                         const std::vector<int>& fds_to_ignore,
                         bool is_priority_fork,
                         bool purge) {
  // ...

  pid_t pid = fork();

  if (pid == 0) {
    // ...
  } else {
    ALOGD("Forked child process %d", pid);
  }

  // We blocked SIGCHLD prior to a fork, we unblock it here.
  UnblockSignal(SIGCHLD, fail_fn);

  return pid;
}
```

此时SystemServer已通过Zygote进程fork生成

## Zygote的初始化

ZygoteInit.java#handleSystemServerProcess

```java
/**
* Finish remaining work for the newly forked system server process.
*/
private static Runnable handleSystemServerProcess(ZygoteArguments parsedArgs) {
    // ...

    if (parsedArgs.mInvokeWith != null) {
        // ...
    } else {
        ClassLoader cl = getOrCreateSystemServerClassLoader();
        if (cl != null) {
            Thread.currentThread().setContextClassLoader(cl);
        }

        /*
            * Pass the remaining arguments to SystemServer.
            */
        return ZygoteInit.zygoteInit(parsedArgs.mTargetSdkVersion,
                parsedArgs.mDisabledCompatChanges,
                parsedArgs.mRemainingArgs, cl);
    }

    /* should never reach here */
}
```

ZygoteInit.java#zygoteInit

```java
public static Runnable zygoteInit(int targetSdkVersion, long[] disabledCompatChanges,
        String[] argv, ClassLoader classLoader) {
    if (RuntimeInit.DEBUG) {
        Slog.d(RuntimeInit.TAG, "RuntimeInit: Starting application from zygote");
    }

    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ZygoteInit");
    RuntimeInit.redirectLogStreams();

    RuntimeInit.commonInit();
    // native 方法，主要用于开启binder线程池
    ZygoteInit.nativeZygoteInit();
    // SystemServer java层入口
    return RuntimeInit.applicationInit(targetSdkVersion, disabledCompatChanges, argv,
            classLoader);
}
```

frameworks/base/core/jni/AndroidRuntime.cpp#com_android_internal_os_ZygoteInit_nativeZygoteInit

```c++
static void com_android_internal_os_ZygoteInit_nativeZygoteInit(JNIEnv* env, jobject clazz)
{
    // gCurRuntime 为 AppRuntime实例
    gCurRuntime->onZygoteInit();
}
```

frameworks/base/cmds/app_process/app_main.cpp#onZygoteInit

```c++
virtual void onZygoteInit()
{
    sp<ProcessState> proc = ProcessState::self();
    ALOGV("App process: starting thread pool.\n");
    proc->startThreadPool(); // 创建binder线程池
}
```

## SystemServer java层入口

RuntimeInit.java#applicationInit
```java
protected static Runnable applicationInit(int targetSdkVersion, long[] disabledCompatChanges,
        String[] argv, ClassLoader classLoader) {
    // ...

    // Remaining arguments are passed to the start class's static main
    return findStaticMain(args.startClass, args.startArgs, classLoader);
}
```

RuntimeInit.java#findStaticMain

```java
protected static Runnable findStaticMain(String className, String[] argv,
            ClassLoader classLoader) {
    Class<?> cl;

    try {
        cl = Class.forName(className, true, classLoader);
    } catch (ClassNotFoundException ex) {
        throw new RuntimeException(
                "Missing class when invoking static main " + className,
                ex);
    }

    Method m;
    try {
        m = cl.getMethod("main", new Class[] { String[].class });
    } catch (NoSuchMethodException ex) {
        throw new RuntimeException(
                "Missing static main on " + className, ex);
    } catch (SecurityException ex) {
        throw new RuntimeException(
                "Problem getting static main on " + className, ex);
    }

    int modifiers = m.getModifiers();
    if (! (Modifier.isStatic(modifiers) && Modifier.isPublic(modifiers))) {
        throw new RuntimeException(
                "Main method is not public and static on " + className);
    }

    /*
        * This throw gets caught in ZygoteInit.main(), which responds
        * by invoking the exception's run() method. This arrangement
        * clears up all the stack frames that were required in setting
        * up the process.
        */
    return new MethodAndArgsCaller(m, argv);
}
```

MethodAndArgsCaller

```java
/**
* Helper class which holds a method and arguments and can call them. This is used as part of
* a trampoline to get rid of the initial process setup stack frames.
*/
static class MethodAndArgsCaller implements Runnable {
    /** method to call */
    private final Method mMethod;

    /** argument array */
    private final String[] mArgs;

    public MethodAndArgsCaller(Method method, String[] args) {
        mMethod = method;
        mArgs = args;
    }

    public void run() {
        try {
            mMethod.invoke(null, new Object[] { mArgs });
        } catch (IllegalAccessException ex) {
            throw new RuntimeException(ex);
        } catch (InvocationTargetException ex) {
            Throwable cause = ex.getCause();
            if (cause instanceof RuntimeException) {
                throw (RuntimeException) cause;
            } else if (cause instanceof Error) {
                throw (Error) cause;
            }
            throw new RuntimeException(ex);
        }
    }
}
```

执行点ZygoteInit.java#main

```java
if (startSystemServer) {
    Runnable r = forkSystemServer(abiList, zygoteSocketName, zygoteServer);

    // {@code r == null} in the parent (zygote) process, and {@code r != null} in the
    // child (system_server) process.
    if (r != null) {
        // mMethod.invoke(null, new Object[] { mArgs });
        // 执行SytemServer.main
        r.run();
        return;
    }
}
```

## SystemServer java层初始化

SystemServer.java#main

```java
public static void main(String[] args) {
    new SystemServer().run();
}

private void run() {
    TimingsTraceAndSlog t = new TimingsTraceAndSlog();
    try {
        // ...

        // Initialize the system context.
        createSystemContext();

        // ...
    } finally {
        t.traceEnd();  // InitBeforeStartServices
    }

    // Setup the default WTF handler
    RuntimeInit.setDefaultApplicationWtfHandler(SystemServer::handleEarlySystemWtf);

    // Start services.
    try {
        t.traceBegin("StartServices");
         // startBootPhase 设置启动阶段标志，用于在对应的阶段执行对应的回调
        startBootstrapServices(t);
        startCoreServices(t);
        startOtherServices(t);
    } catch (Throwable ex) {
        Slog.e("System", "******************************************");
        Slog.e("System", "************ Failure starting system services", ex);
        throw ex;
    } finally {
        t.traceEnd(); // StartServices
    }

    // ...

    // Loop forever.
    Looper.loop();
    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```
### Application和SystemContext的创建

SystemServer#createSystemContext

```java
private void createSystemContext() {
    ActivityThread activityThread = ActivityThread.systemMain();
    mSystemContext = activityThread.getSystemContext();
    mSystemContext.setTheme(DEFAULT_SYSTEM_THEME);

    final Context systemUiContext = activityThread.getSystemUiContext();
    systemUiContext.setTheme(DEFAULT_SYSTEM_THEME);
}
```

ActivityThread#systemMain

```java
public static ActivityThread systemMain() {
    ThreadedRenderer.initForSystemProcess();
    ActivityThread thread = new ActivityThread();
    thread.attach(true, 0);
    return thread;
}
```

```java
private void attach(boolean system, long startSeq) {
    sCurrentActivityThread = this;
    mConfigurationController = new ConfigurationController(this);
    mSystemThread = system;
    if (!system) {
        // ...
    } else {
        // Don't set application object here -- if the system crashes,
        // we can't display an alert, we just want to die die die.
        android.ddm.DdmHandleAppName.setAppName("system_process",
                UserHandle.myUserId());
        try {
            mInstrumentation = new Instrumentation();
            mInstrumentation.basicInit(this);
            ContextImpl context = ContextImpl.createAppContext(
                    this, getSystemContext().mPackageInfo);
            mInitialApplication = context.mPackageInfo.makeApplication(true, null);
            mInitialApplication.onCreate();
        } catch (Exception e) {
            throw new RuntimeException(
                    "Unable to instantiate Application():" + e.toString(), e);
        }
    }

    // ...
}
```

```java
@UnsupportedAppUsage
static ContextImpl createAppContext(ActivityThread mainThread, LoadedApk packageInfo) {
    return createAppContext(mainThread, packageInfo, null);
}

static ContextImpl createAppContext(ActivityThread mainThread, LoadedApk packageInfo,
        String opPackageName) {
    if (packageInfo == null) throw new IllegalArgumentException("packageInfo");
    ContextImpl context = new ContextImpl(null, mainThread, packageInfo,
        ContextParams.EMPTY, null, null, null, null, null, 0, null, opPackageName);
    context.setResources(packageInfo.getResources());
    context.mContextType = isSystemOrSystemUI(context) ? CONTEXT_TYPE_SYSTEM_OR_SYSTEM_UI
            : CONTEXT_TYPE_NON_UI;
    return context;
}
```

LoadedApk.java#makeApplication

```java
@UnsupportedAppUsage
public Application makeApplication(boolean forceDefaultAppClass,
        Instrumentation instrumentation) {
    if (mApplication != null) {
        return mApplication;
    }

    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "makeApplication");

    Application app = null;

    String appClass = mApplicationInfo.className;
    if (forceDefaultAppClass || (appClass == null)) {
        appClass = "android.app.Application";
    }

    try {
        final java.lang.ClassLoader cl = getClassLoader();
        // ...

        ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
        // The network security config needs to be aware of multiple
        // applications in the same process to handle discrepancies
        NetworkSecurityConfigProvider.handleNewApplication(appContext);
        // 反射创建application
        app = mActivityThread.mInstrumentation.newApplication(
                cl, appClass, appContext);
        appContext.setOuterContext(app);
    } catch (Exception e) {
        if (!mActivityThread.mInstrumentation.onException(app, e)) {
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            throw new RuntimeException(
                "Unable to instantiate application " + appClass
                + " package " + mPackageName + ": " + e.toString(), e);
        }
    }
    // ...

    return app;
}
```

Instrumentation.java#newApplication

```java
public Application newApplication(ClassLoader cl, String className, Context context)
        throws InstantiationException, IllegalAccessException, 
        ClassNotFoundException {
    Application app = getFactory(context.getPackageName())
            .instantiateApplication(cl, className);
    app.attach(context);
    return app;
}
```

AppComponentFactory.java#instantiateApplication

```java
public @NonNull Application instantiateApplication(@NonNull ClassLoader cl,
        @NonNull String className)
        throws InstantiationException, IllegalAccessException, ClassNotFoundException {
    return (Application) cl.loadClass(className).newInstance();
}
```

### ATMS 和 AMS的创建和注册

SystemServer.java#startBootstrapServices

```java
/**
* Starts the small tangle of critical services that are needed to get the system off the
* ground.  These services have complex mutual dependencies which is why we initialize them all
* in one place here.  Unless your service is also entwined in these dependencies, it should be
* initialized in one of the other functions.
*/
private void startBootstrapServices(@NonNull TimingsTraceAndSlog t) {
    // ...

    // Activity manager runs the show.
    t.traceBegin("StartActivityManager");
    // TODO: Might need to move after migration to WM.
    ActivityTaskManagerService atm = mSystemServiceManager.startService(
            ActivityTaskManagerService.Lifecycle.class).getService();
    mActivityManagerService = ActivityManagerService.Lifecycle.startService(
            mSystemServiceManager, atm);
    mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
    mActivityManagerService.setInstaller(installer);
    mWindowManagerGlobalLock = atm.getGlobalLock();
    t.traceEnd();

    // ...
}
```

SystemServiceManager.java#startService

```java
/**
    * Creates and starts a system service. The class must be a subclass of
    * {@link com.android.server.SystemService}.
    *
    * @param serviceClass A Java class that implements the SystemService interface.
    * @return The service instance, never null.
    * @throws RuntimeException if the service fails to start.
    */
public <T extends SystemService> T startService(Class<T> serviceClass) {
    try {
        final String name = serviceClass.getName();
        Slog.i(TAG, "Starting " + name);
        Trace.traceBegin(Trace.TRACE_TAG_SYSTEM_SERVER, "StartService " + name);

        // Create the service.
        if (!SystemService.class.isAssignableFrom(serviceClass)) {
            throw new RuntimeException("Failed to create " + name
                    + ": service must extend " + SystemService.class.getName());
        }
        final T service;
        try {
            Constructor<T> constructor = serviceClass.getConstructor(Context.class);
            service = constructor.newInstance(mContext);
        } catch (InstantiationException ex) {
            throw new RuntimeException("Failed to create service " + name
                    + ": service could not be instantiated", ex);
        } catch (IllegalAccessException ex) {
            throw new RuntimeException("Failed to create service " + name
                    + ": service must have a public constructor with a Context argument", ex);
        } catch (NoSuchMethodException ex) {
            throw new RuntimeException("Failed to create service " + name
                    + ": service must have a public constructor with a Context argument", ex);
        } catch (InvocationTargetException ex) {
            throw new RuntimeException("Failed to create service " + name
                    + ": service constructor threw an exception", ex);
        }

        startService(service);
        return service;
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
    }
}

public void startService(@NonNull final SystemService service) {
    // Register it.
    // mServices: ArrayList<SystemService>
    mServices.add(service);
    // Start it.
    long time = SystemClock.elapsedRealtime();
    try {
        service.onStart();
    } catch (RuntimeException ex) {
        throw new RuntimeException("Failed to start service " + service.getClass().getName()
                + ": onStart threw an exception", ex);
    }
    warnIfTooLong(SystemClock.elapsedRealtime() - time, service, "onStart");
}
```

```java
public static final class Lifecycle extends SystemService {
    private final ActivityTaskManagerService mService;

    public Lifecycle(Context context) {
        super(context);
        mService = new ActivityTaskManagerService(context);
    }

    @Override
    public void onStart() {
        publishBinderService(Context.ACTIVITY_TASK_SERVICE, mService);
        mService.start();
    }

    @Override
    public void onUserUnlocked(@NonNull TargetUser user) {
        synchronized (mService.getGlobalLock()) {
            mService.mTaskSupervisor.onUserUnlocked(user.getUserIdentifier());
        }
    }

    @Override
    public void onUserStopped(@NonNull TargetUser user) {
        synchronized (mService.getGlobalLock()) {
            mService.mTaskSupervisor.mLaunchParamsPersister
                    .onCleanupUser(user.getUserIdentifier());
        }
    }

    public ActivityTaskManagerService getService() {
        return mService;
    }
}
```

SystemService.java#publishBinderService

```java
 /**
    * Publish the service so it is accessible to other services and apps.
    *
    * @param name the name of the new service
    * @param service the service object
    */
protected final void publishBinderService(@NonNull String name, @NonNull IBinder service) {
    publishBinderService(name, service, false);
}

/**
    * Publish the service so it is accessible to other services and apps.
    *
    * @param name the name of the new service
    * @param service the service object
    * @param allowIsolated set to true to allow isolated sandboxed processes
    * to access this service
    */
protected final void publishBinderService(@NonNull String name, @NonNull IBinder service,
        boolean allowIsolated) {
    publishBinderService(name, service, allowIsolated, DUMP_FLAG_PRIORITY_DEFAULT);
}

/**
    * Publish the service so it is accessible to other services and apps.
    *
    * @param name the name of the new service
    * @param service the service object
    * @param allowIsolated set to true to allow isolated sandboxed processes
    * to access this service
    * @param dumpPriority supported dump priority levels as a bitmask
    *
    * @hide
    */
protected final void publishBinderService(String name, IBinder service,
        boolean allowIsolated, int dumpPriority) {    
    ServiceManager.addService(name, service, allowIsolated, dumpPriority);
}
```
## launcher的启动

startOtherServices ==> mActivityManagerService.systemReady

```java
/**
    * Ready. Set. Go!
    */
public void systemReady(final Runnable goingCallback, @NonNull TimingsTraceAndSlog t) {
    t.traceBegin("PhaseActivityManagerReady");
    mSystemServiceManager.preSystemReady();
    synchronized(this) {
        // ...
        mActivityTaskManager.onSystemReady(); // atms相关初始化
        // ...
        mSystemReady = true;
        t.traceEnd();
    }

    // ...

    synchronized (this) {
        // ...

        if (bootingSystemUser) {
            t.traceBegin("startHomeOnAllDisplays");
            mAtmInternal.startHomeOnAllDisplays(currentUserId, "systemReady");
            t.traceEnd();
        }

        // ...
    }
}
```

```java
@Override
public boolean startHomeOnAllDisplays(int userId, String reason) {
    synchronized (mGlobalLock) {
        return mRootWindowContainer.startHomeOnAllDisplays(userId, reason);
    }
}
```

```java
boolean startHomeOnAllDisplays(int userId, String reason) {
    boolean homeStarted = false;
    for (int i = getChildCount() - 1; i >= 0; i--) {
        final int displayId = getChildAt(i).mDisplayId;
        homeStarted |= startHomeOnDisplay(userId, reason, displayId);
    }
    return homeStarted;
}

boolean startHomeOnDisplay(int userId, String reason, int displayId) {
    return startHomeOnDisplay(userId, reason, displayId, false /* allowInstrumenting */,
            false /* fromHomeKey */);
}

boolean startHomeOnDisplay(int userId, String reason, int displayId, boolean allowInstrumenting,
        boolean fromHomeKey) {
    // Fallback to top focused display or default display if the displayId is invalid.
    if (displayId == INVALID_DISPLAY) {
        final Task rootTask = getTopDisplayFocusedRootTask();
        displayId = rootTask != null ? rootTask.getDisplayId() : DEFAULT_DISPLAY;
    }

    final DisplayContent display = getDisplayContent(displayId);
    return display.reduceOnAllTaskDisplayAreas((taskDisplayArea, result) ->
                    result | startHomeOnTaskDisplayArea(userId, reason, taskDisplayArea,
                            allowInstrumenting, fromHomeKey),
            false /* initValue */);
}

boolean startHomeOnTaskDisplayArea(int userId, String reason, TaskDisplayArea taskDisplayArea,
        boolean allowInstrumenting, boolean fromHomeKey) {
    // ...
    mService.getActivityStartController().startHomeActivity(homeIntent, aInfo, myReason,
            taskDisplayArea);
    return true;
}
```

```java
void startHomeActivity(Intent intent, ActivityInfo aInfo, String reason,
        TaskDisplayArea taskDisplayArea) {
    // ...
    mLastHomeActivityStartResult = obtainStarter(intent, "startHomeActivity: " + reason)
            .setOutActivity(tmpOutRecord)
            .setCallingUid(0)
            .setActivityInfo(aInfo)
            .setActivityOptions(options.toBundle())
            .execute(); // 普通app的页面也是通过此方法开启
    // ...
}
```

```java
/**
    * Resolve necessary information according the request parameters provided earlier, and execute
    * the request which begin the journey of starting an activity.
    * @return The starter result.
    */
int execute() {
    try {
        // ...

        int res;
        synchronized (mService.mGlobalLock) {
            final boolean globalConfigWillChange = mRequest.globalConfig != null
                    && mService.getGlobalConfiguration().diff(mRequest.globalConfig) != 0;

            final Task rootTask = mRootWindowContainer.getTopDisplayFocusedRootTask();
            // ...
            res = executeRequest(mRequest);

            // ...
        }
    } finally {
        onExecutionComplete();
    }
}

private int executeRequest(Request request) {
    // ...

    mLastStartActivityResult = startActivityUnchecked(r, sourceRecord, voiceSession,
            request.voiceInteractor, startFlags, true /* doResume */, checkedOptions, inTask,
            restrictedBgActivity, intentGrants);

    // ...

    return mLastStartActivityResult;
}

/**
    * Start an activity while most of preliminary checks has been done and caller has been
    * confirmed that holds necessary permissions to do so.
    * Here also ensures that the starting activity is removed if the start wasn't successful.
    */
private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, boolean doResume, ActivityOptions options, Task inTask,
            boolean restrictedBgActivity, NeededUriGrants intentGrants) {
    int result = START_CANCELED;
    final Task startedActivityRootTask;
    // ...
    try {
        mService.deferWindowLayout();
        Trace.traceBegin(Trace.TRACE_TAG_WINDOW_MANAGER, "startActivityInner");
        result = startActivityInner(r, sourceRecord, voiceSession, voiceInteractor,
                startFlags, doResume, options, inTask, restrictedBgActivity, intentGrants);
    } finally {
        // ...
    }

    postStartActivityProcessing(r, result, startedActivityRootTask);

    return result;
}

int startActivityInner(final ActivityRecord r, ActivityRecord sourceRecord,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        int startFlags, boolean doResume, ActivityOptions options, Task inTask,
        boolean restrictedBgActivity, NeededUriGrants intentGrants) {
    // ...

    final ActivityRecord targetTaskTop = newTask
            ? null : targetTask.getTopNonFinishingActivity();
    if (targetTaskTop != null) {
        // Recycle the target task for this launch.
        startResult = recycleTask(targetTask, targetTaskTop, reusedTask, intentGrants);
        if (startResult != START_SUCCESS) {
            return startResult;
        }
    } else {
        mAddingToTask = true;
    }

    // ...
}

int recycleTask(Task targetTask, ActivityRecord targetTaskTop, Task reusedTask,
        NeededUriGrants intentGrants) {
    // ...
    resumeTargetRootTaskIfNeeded();
    // ...
}

private void resumeTargetRootTaskIfNeeded() {
    if (mDoResume) {
        final ActivityRecord next = mTargetRootTask.topRunningActivity(
                true /* focusableOnly */);
        if (next != null) {
            next.setCurrentLaunchCanTurnScreenOn(true);
        }
        if (mTargetRootTask.isFocusable()) {
            mRootWindowContainer.resumeFocusedTasksTopActivities(mTargetRootTask, null,
                    mOptions, mTransientLaunch);
        } else {
            mRootWindowContainer.ensureActivitiesVisible(null, 0, !PRESERVE_WINDOWS);
        }
    } else {
        ActivityOptions.abort(mOptions);
    }
    mRootWindowContainer.updateUserRootTask(mStartActivity.mUserId, mTargetRootTask);
}
```

```java
boolean resumeFocusedTasksTopActivities(
            Task targetRootTask, ActivityRecord target, ActivityOptions targetOptions,
            boolean deferPause) {
    if (!mTaskSupervisor.readyToResume()) {
        return false;
    }

    boolean result = false;
    if (targetRootTask != null && (targetRootTask.isTopRootTaskInDisplayArea()
            || getTopDisplayFocusedRootTask() == targetRootTask)) {
        // app页面启动一般走这个流程
        result = targetRootTask.resumeTopActivityUncheckedLocked(target, targetOptions,
                deferPause);
    }

    for (int displayNdx = getChildCount() - 1; displayNdx >= 0; --displayNdx) {
        // ...
        result |= resumedOnDisplay[0];
        if (!resumedOnDisplay[0]) {
            // In cases when there are no valid activities (e.g. device just booted or launcher
            // crashed) it's possible that nothing was resumed on a display. Requesting resume
            // of top activity in focused root task explicitly will make sure that at least home
            // activity is started and resumed, and no recursion occurs.
            // 设备刚启动或者launcher崩溃了 确保至少有一个activity启动
            final Task focusedRoot = display.getFocusedRootTask();
            if (focusedRoot != null) {
                result |= focusedRoot.resumeTopActivityUncheckedLocked(target, targetOptions);
            } else if (targetRootTask == null) {
                result |= resumeHomeActivity(null /* prev */, "no-focusable-task",
                        display.getDefaultTaskDisplayArea());
            }
        }
    }

    return result;
}
```

```java
/**
    * Ensure that the top activity in the root task is resumed.
    *
    * @param prev The previously resumed activity, for when in the process
    * of pausing; can be null to call from elsewhere.
    * @param options Activity options.
    * @param deferPause When {@code true}, this will not pause back tasks.
    *
    * @return Returns true if something is being resumed, or false if
    * nothing happened.
    *
    * NOTE: It is not safe to call this method directly as it can cause an activity in a
    *       non-focused root task to be resumed.
    *       Use {@link RootWindowContainer#resumeFocusedTasksTopActivities} to resume the
    *       right activity for the current system state.
    */
@GuardedBy("mService")
boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options,
        boolean deferPause) {
    if (mInResumeTopActivity) {
        // Don't even start recursing.
        return false;
    }

    boolean someActivityResumed = false;
    try {
        // Protect against recursion.
        mInResumeTopActivity = true;

        if (isLeafTask()) {
            if (isFocusableAndVisible()) {
                someActivityResumed = resumeTopActivityInnerLocked(prev, options, deferPause);
            }
        } else {
            int idx = mChildren.size() - 1;
            while (idx >= 0) {
                // ...

                someActivityResumed |= child.resumeTopActivityUncheckedLocked(prev, options,
                        deferPause);
                // Doing so in order to prevent IndexOOB since hierarchy might changes while
                // resuming activities, for example dismissing split-screen while starting
                // non-resizeable activity.
                if (idx >= mChildren.size()) {
                    idx = mChildren.size() - 1;
                }
            }
        }

        // ...
    } finally {
        mInResumeTopActivity = false;
    }

    return someActivityResumed;
}

/** @see #resumeTopActivityUncheckedLocked(ActivityRecord, ActivityOptions, boolean) */
@GuardedBy("mService")
boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
    return resumeTopActivityUncheckedLocked(prev, options, false /* skipPause */);
}

private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options,
        boolean deferPause) {
    // ...

    if (next.attachedToProcess()) {
        // ...
    } else {
        // ...
        mTaskSupervisor.startSpecificActivity(next, true, true);
    }

    return true;
}
```

```java
void startSpecificActivity(ActivityRecord r, boolean andResume, boolean checkConfig) {
    // Is this activity's application already running?
    final WindowProcessController wpc =
            mService.getProcessController(r.processName, r.info.applicationInfo.uid);

    boolean knownToBeDead = false;
    if (wpc != null && wpc.hasThread()) {
        try {
            // 进程存在
            realStartActivityLocked(r, wpc, andResume, checkConfig);
            return;
        } catch (RemoteException e) {
            Slog.w(TAG, "Exception when starting activity "
                    + r.intent.getComponent().flattenToShortString(), e);
        }

        // If a dead object exception was thrown -- fall through to
        // restart the application.
        knownToBeDead = true;
    }

    r.notifyUnknownVisibilityLaunchedForKeyguardTransition();

    final boolean isTop = andResume && r.isTopRunningActivity();
    // 进程不存在 mService为ATMS
    mService.startProcessAsync(r, knownToBeDead, isTop, isTop ? "top-activity" : "activity");
}
```

```java
void startProcessAsync(ActivityRecord activity, boolean knownToBeDead, boolean isTop,
        String hostingType) {
    try {
        if (Trace.isTagEnabled(TRACE_TAG_WINDOW_MANAGER)) {
            Trace.traceBegin(TRACE_TAG_WINDOW_MANAGER, "dispatchingStartProcess:"
                    + activity.processName);
        }
        // Post message to start process to avoid possible deadlock of calling into AMS with the
        // ATMS lock held.
        final Message m = PooledLambda.obtainMessage(ActivityManagerInternal::startProcess,
                mAmInternal, activity.processName, activity.info.applicationInfo, knownToBeDead,
                isTop, hostingType, activity.intent.getComponent());
        mH.sendMessage(m);
    } finally {
        Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
    }
}
```

ActivityServiceManager.LocalService#startProcess

```java
@Override
public void startProcess(String processName, ApplicationInfo info, boolean knownToBeDead,
        boolean isTop, String hostingType, ComponentName hostingName) {
    try {
        if (Trace.isTagEnabled(Trace.TRACE_TAG_ACTIVITY_MANAGER)) {
            Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "startProcess:"
                    + processName);
        }
        synchronized (ActivityManagerService.this) {
            // If the process is known as top app, set a hint so when the process is
            // started, the top priority can be applied immediately to avoid cpu being
            // preempted by other processes before attaching the process of top app.
            startProcessLocked(processName, info, knownToBeDead, 0 /* intentFlags */,
                    new HostingRecord(hostingType, hostingName, isTop),
                    ZYGOTE_POLICY_FLAG_LATENCY_SENSITIVE, false /* allowWhileBooting */,
                    false /* isolated */);
        }
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    }
}
```

ActivityServiceManager.java#startProcessLocked

```java
@GuardedBy("this")
final ProcessRecord startProcessLocked(String processName,
        ApplicationInfo info, boolean knownToBeDead, int intentFlags,
        HostingRecord hostingRecord, int zygotePolicyFlags, boolean allowWhileBooting,
        boolean isolated) {
    return mProcessList.startProcessLocked(processName, info, knownToBeDead, intentFlags,
            hostingRecord, zygotePolicyFlags, allowWhileBooting, isolated, 0 /* isolatedUid */,
            null /* ABI override */, null /* entryPoint */,
            null /* entryPointArgs */, null /* crashHandler */);
}
```

ProcessList.java#startProcessLocked

```java
@GuardedBy("mService")
ProcessRecord startProcessLocked(String processName, ApplicationInfo info,
        boolean knownToBeDead, int intentFlags, HostingRecord hostingRecord,
        int zygotePolicyFlags, boolean allowWhileBooting, boolean isolated, int isolatedUid,
        String abiOverride, String entryPoint, String[] entryPointArgs, Runnable crashHandler) {
    // ...
    final boolean success =
            startProcessLocked(app, hostingRecord, zygotePolicyFlags, abiOverride);
    checkSlow(startTime, "startProcess: done starting proc!");
    return success ? app : null;
}

@GuardedBy("mService")
boolean startProcessLocked(ProcessRecord app, HostingRecord hostingRecord,
        int zygotePolicyFlags, String abiOverride) {
    return startProcessLocked(app, hostingRecord, zygotePolicyFlags,
            false /* disableHiddenApiChecks */, false /* disableTestApiChecks */,
            abiOverride);
}
```

```java
/**
    * @return {@code true} if process start is successful, false otherwise.
    */
@GuardedBy("mService")
boolean startProcessLocked(ProcessRecord app, HostingRecord hostingRecord,
        int zygotePolicyFlags, boolean disableHiddenApiChecks, boolean disableTestApiChecks,
        String abiOverride) {
    // ...

    try {
        // ...
        return startProcessLocked(hostingRecord, entryPoint, app, uid, gids,
                runtimeFlags, zygotePolicyFlags, mountExternal, seInfo, requiredAbi,
                instructionSet, invokeWith, startTime);
    } catch (RuntimeException e) {
        // ...
        return false;
    }
}

 @GuardedBy("mService")
boolean startProcessLocked(HostingRecord hostingRecord, String entryPoint, ProcessRecord app,
        int uid, int[] gids, int runtimeFlags, int zygotePolicyFlags, int mountExternal,
        String seInfo, String requiredAbi, String instructionSet, String invokeWith,
        long startTime) {
    // ...
    if (mService.mConstants.FLAG_PROCESS_START_ASYNC) {
        if (DEBUG_PROCESSES) Slog.i(TAG_PROCESSES,
                "Posting procStart msg for " + app.toShortString());
        mService.mProcStartHandler.post(() -> handleProcessStart(
                app, entryPoint, gids, runtimeFlags, zygotePolicyFlags, mountExternal,
                requiredAbi, instructionSet, invokeWith, startSeq));
        return true;
    } else {
        try {
            final Process.ProcessStartResult startResult = startProcess(hostingRecord,
                    entryPoint, app,
                    uid, gids, runtimeFlags, zygotePolicyFlags, mountExternal, seInfo,
                    requiredAbi, instructionSet, invokeWith, startTime);
            handleProcessStartedLocked(app, startResult.pid, startResult.usingWrapper,
                    startSeq, false);
        } catch (RuntimeException e) {
            // ...
        }
        return app.getPid() > 0;
    }
}

private Process.ProcessStartResult startProcess(HostingRecord hostingRecord, String entryPoint,
        ProcessRecord app, int uid, int[] gids, int runtimeFlags, int zygotePolicyFlags,
        int mountExternal, String seInfo, String requiredAbi, String instructionSet,
        String invokeWith, long startTime) {
    try {
        // ...
        boolean regularZygote = false;
        if (hostingRecord.usesWebviewZygote()) {
            // ...
        } else if (hostingRecord.usesAppZygote()) {
            // ...
        } else {
            regularZygote = true;
            startResult = Process.start(entryPoint,
                    app.processName, uid, uid, gids, runtimeFlags, mountExternal,
                    app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet,
                    app.info.dataDir, invokeWith, app.info.packageName, zygotePolicyFlags,
                    isTopApp, app.getDisabledCompatChanges(), pkgDataInfoMap,
                    allowlistedAppDataInfoMap, bindMountAppsData, bindMountAppStorageDirs,
                    new String[]{PROC_START_SEQ_IDENT + app.getStartSeq()});
        }

        // ...
        return startResult;
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    }
}
```

```java
public static ProcessStartResult start(@NonNull final String processClass,
                                        @Nullable final String niceName,
                                        int uid, int gid, @Nullable int[] gids,
                                        int runtimeFlags,
                                        int mountExternal,
                                        int targetSdkVersion,
                                        @Nullable String seInfo,
                                        @NonNull String abi,
                                        @Nullable String instructionSet,
                                        @Nullable String appDataDir,
                                        @Nullable String invokeWith,
                                        @Nullable String packageName,
                                        int zygotePolicyFlags,
                                        boolean isTopApp,
                                        @Nullable long[] disabledCompatChanges,
                                        @Nullable Map<String, Pair<String, Long>>
                                                pkgDataInfoMap,
                                        @Nullable Map<String, Pair<String, Long>>
                                                whitelistedDataInfoMap,
                                        boolean bindMountAppsData,
                                        boolean bindMountAppStorageDirs,
                                        @Nullable String[] zygoteArgs) {
    return ZYGOTE_PROCESS.start(processClass, niceName, uid, gid, gids,
                runtimeFlags, mountExternal, targetSdkVersion, seInfo,
                abi, instructionSet, appDataDir, invokeWith, packageName,
                zygotePolicyFlags, isTopApp, disabledCompatChanges,
                pkgDataInfoMap, whitelistedDataInfoMap, bindMountAppsData,
                bindMountAppStorageDirs, zygoteArgs);
}
```

```java
public final Process.ProcessStartResult start(@NonNull final String processClass,
                                                final String niceName,
                                                int uid, int gid, @Nullable int[] gids,
                                                int runtimeFlags, int mountExternal,
                                                int targetSdkVersion,
                                                @Nullable String seInfo,
                                                @NonNull String abi,
                                                @Nullable String instructionSet,
                                                @Nullable String appDataDir,
                                                @Nullable String invokeWith,
                                                @Nullable String packageName,
                                                int zygotePolicyFlags,
                                                boolean isTopApp,
                                                @Nullable long[] disabledCompatChanges,
                                                @Nullable Map<String, Pair<String, Long>>
                                                        pkgDataInfoMap,
                                                @Nullable Map<String, Pair<String, Long>>
                                                        allowlistedDataInfoList,
                                                boolean bindMountAppsData,
                                                boolean bindMountAppStorageDirs,
                                                @Nullable String[] zygoteArgs) {
    // TODO (chriswailes): Is there a better place to check this value?
    if (fetchUsapPoolEnabledPropWithMinInterval()) {
        informZygotesOfUsapPoolStatus();
    }

    try {
        return startViaZygote(processClass, niceName, uid, gid, gids,
                runtimeFlags, mountExternal, targetSdkVersion, seInfo,
                abi, instructionSet, appDataDir, invokeWith, /*startChildZygote=*/ false,
                packageName, zygotePolicyFlags, isTopApp, disabledCompatChanges,
                pkgDataInfoMap, allowlistedDataInfoList, bindMountAppsData,
                bindMountAppStorageDirs, zygoteArgs);
    } catch (ZygoteStartFailedEx ex) {
        Log.e(LOG_TAG,
                "Starting VM process through Zygote failed");
        throw new RuntimeException(
                "Starting VM process through Zygote failed", ex);
    }
}

private Process.ProcessStartResult startViaZygote(@NonNull final String processClass,
                                                    @Nullable final String niceName,
                                                    final int uid, final int gid,
                                                    @Nullable final int[] gids,
                                                    int runtimeFlags, int mountExternal,
                                                    int targetSdkVersion,
                                                    @Nullable String seInfo,
                                                    @NonNull String abi,
                                                    @Nullable String instructionSet,
                                                    @Nullable String appDataDir,
                                                    @Nullable String invokeWith,
                                                    boolean startChildZygote,
                                                    @Nullable String packageName,
                                                    int zygotePolicyFlags,
                                                    boolean isTopApp,
                                                    @Nullable long[] disabledCompatChanges,
                                                    @Nullable Map<String, Pair<String, Long>>
                                                            pkgDataInfoMap,
                                                    @Nullable Map<String, Pair<String, Long>>
                                                            allowlistedDataInfoList,
                                                    boolean bindMountAppsData,
                                                    boolean bindMountAppStorageDirs,
                                                    @Nullable String[] extraArgs)
                                                    throws ZygoteStartFailedEx {
    ArrayList<String> argsForZygote = new ArrayList<>();

    // ...

    synchronized(mLock) {
        // The USAP pool can not be used if the application will not use the systems graphics
        // driver.  If that driver is requested use the Zygote application start path.
        // 与 zygote 通过socket通信，通知zygote fork进程
        // ZygoteServer.java的runSelectLoop中处理
        return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi),
                                            zygotePolicyFlags,
                                            argsForZygote);
    }
}
```

```java
Runnable runSelectLoop(String abiList) {
    // ...

    while (true) {
        // ...

        int pollReturnValue;
        try {
            pollReturnValue = Os.poll(pollFDs, pollTimeoutMs); // epoll机制等待
        } catch (ErrnoException ex) {
            throw new RuntimeException("poll failed", ex);
        }

        if (pollReturnValue == 0) {
            // ...
        } else {
            boolean usapPoolFDRead = false;

            while (--pollIndex >= 0) {
                if ((pollFDs[pollIndex].revents & POLLIN) == 0) {
                    continue;
                }

                if (pollIndex == 0) {
                    // ...
                } else if (pollIndex < usapPoolEventFDIndex) {
                    // Session socket accepted from the Zygote server socket

                    try {
                        ZygoteConnection connection = peers.get(pollIndex);
                        boolean multipleForksOK = !isUsapPoolEnabled()
                                && ZygoteHooks.isIndefiniteThreadSuspensionSafe();
                        final Runnable command =
                                connection.processCommand(this, multipleForksOK);

                        // ...

                } else {
                    // ...
                }
            }

            // ....
        }

        // ...
    }
}
```

```java
Runnable processCommand(ZygoteServer zygoteServer, boolean multipleOK) {
    ZygoteArguments parsedArgs;

    try (ZygoteCommandBuffer argBuffer = new ZygoteCommandBuffer(mSocket)) {
        while (true) {
            // ...

            if (parsedArgs.mInvokeWith != null || parsedArgs.mStartChildZygote
                    || !multipleOK || peer.getUid() != Process.SYSTEM_UID) {
                // Continue using old code for now. TODO: Handle these cases in the other path.
                pid = Zygote.forkAndSpecialize(parsedArgs.mUid, parsedArgs.mGid,
                        parsedArgs.mGids, parsedArgs.mRuntimeFlags, rlimits,
                        parsedArgs.mMountExternal, parsedArgs.mSeInfo, parsedArgs.mNiceName,
                        fdsToClose, fdsToIgnore, parsedArgs.mStartChildZygote,
                        parsedArgs.mInstructionSet, parsedArgs.mAppDataDir,
                        parsedArgs.mIsTopApp, parsedArgs.mPkgDataInfoList,
                        parsedArgs.mAllowlistedDataInfoList, parsedArgs.mBindMountAppDataDirs,
                        parsedArgs.mBindMountAppStorageDirs);

                try {
                    if (pid == 0) {
                        // in child
                        zygoteServer.setForkChild();

                        zygoteServer.closeServerSocket();
                        IoUtils.closeQuietly(serverPipeFd);
                        serverPipeFd = null;

                        return handleChildProc(parsedArgs, childPipeFd,
                                parsedArgs.mStartChildZygote);
                    } else {
                        // In the parent. A pid < 0 indicates a failure and will be handled in
                        // handleParentProc.
                        IoUtils.closeQuietly(childPipeFd);
                        childPipeFd = null;
                        handleParentProc(pid, serverPipeFd);
                        return null;
                    }
                } finally {
                    IoUtils.closeQuietly(childPipeFd);
                    IoUtils.closeQuietly(serverPipeFd);
                }
            } else {
                // ...
            }
        }
    }
    // ...
}

private Runnable handleChildProc(ZygoteArguments parsedArgs,
        FileDescriptor pipeFd, boolean isZygote) {
    /*
        * By the time we get here, the native code has closed the two actual Zygote
        * socket connections, and substituted /dev/null in their place.  The LocalSocket
        * objects still need to be closed properly.
        */

    closeSocket();

    Zygote.setAppProcessName(parsedArgs, TAG);

    // End of the postFork event.
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    if (parsedArgs.mInvokeWith != null) {
        // ...
    } else {
        if (!isZygote) {
            // 跟fork SystemServer的后续流程一致
            return ZygoteInit.zygoteInit(parsedArgs.mTargetSdkVersion,
                    parsedArgs.mDisabledCompatChanges,
                    parsedArgs.mRemainingArgs, null /* classLoader */);
        } else {
            // ...
        }
    }
}
```

mAtmInternal.resumeTopActivities

主Zygote 从Zygote


mDecor -> DecorView

mContext -> Activity

getApplicationContext ContextImpl = getBaseContext

```java
// PhoneWindow
protected DecorView generateDecor(int featureId) {
    Context context;
    // mUseDecorContext = true 在构造函数中设置为true
    if (mUseDecorContext) {
        // 获取的到context为在ActivityThread.performLaunchActivity生成的ContextImpl
        Context applicationContext = getContext().getApplicationContext();
        if (applicationContext == null) {
            context = getContext();
        } else {
            // DecorContext extends ContextThemeWrapper
            context = new DecorContext(applicationContext, this);
            if (mTheme != -1) {
                context.setTheme(mTheme);
            }
        }
    } else {
        context = getContext();
    }
    return new DecorView(context, featureId, this, getAttributes());
}
```
ViewRootImpl 如何与 DecorView建立起关联
handleResumeActivity

window.addView => root = new ViewRootImpl() => root.setView()

而setView中会建立事件分发的通道, InputChannel

事件在应用层首先通过ViewPostImeInputState分发给DecorView
```
mView.dispatchPointerEvent
```
decorView将事件分发给Activity

```java
// DecorView
// cb即为activity
final Window.Callback cb = mWindow.getCallback();
return cb != null && !mWindow.isDestroyed() && mFeatureId < 0
        ? cb.dispatchTouchEvent(ev) : super.dispatchTouchEvent(ev);
```

activity将事件分发给PhoneWindow

```java
if (ev.getAction() == MotionEvent.ACTION_DOWN) {
    onUserInteraction();
}
if (getWindow().superDispatchTouchEvent(ev)) {
    return true;
}
return onTouchEvent(ev);
```

PhoneWindow又将事件分发回DecorView

```java
return mDecor.superDispatchTouchEvent(event);
```

DecorView将事件分发给子view

```java
// 即调用ViewGroup的dispatchTouchEvent
return super.dispatchTouchEvent(ev)
```
