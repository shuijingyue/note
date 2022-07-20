kernel/common/init/main.c

system/core/init/main.cpp

system/core/init/first_stage_init.cpp
system/core/init/subcontext.cpp
system/core/init/selinux.cpp
system/core/init/init.cpp

FirstStageMain

SetupSelinux 最小权限原则

SecondStageMain
1. 清理僵尸进程
2. 解析init.rc
3. 进入循环等待

system/core/rootdir/init.rc

on zygote-start
  start zygote

system/core/rootdir/init.zygote32.rc
system/core/rootdir/init.zygote64.rc

frameworks/base/cmds/app_process/app_main.cpp

AppRuntime AndroidRuntime
```c++
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
{
    // ....
    /* start the virtual machine */
    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);
    JNIEnv* env;
    // 启动jvm
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
    jclass startClass = env->FindClass(slashClassName);
    if (startClass == NULL) {
        ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
        /* keep going */
    } else {
        // 调用ZygoteInit.java
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
        //...
    }
    //...
}
```

ZygoteInit.java

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
        ZygoteServer zygoteServer = null;
        // ...
        try {

            // ...
            Zygote.initNativeState(isPrimaryZygote);

            ZygoteHooks.stopZygoteNoThreadCreation();

            zygoteServer = new ZygoteServer(isPrimaryZygote);

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