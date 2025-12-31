# system_server 进程

由zygote进程fork出来，创建发布各种Service，其中包括ActivityManagerService，PackageManager，WindowManagerService， ContenService。各种Service跑在system_server进程。

```java
    // 
    private void run() {
        Looper.prepareMainLooper();
        ...
        // ActivityManagerService  PackageManager
        startBootstrapServices(t);
            
        //
        startCoreServices(t);
            
        // ContentService WindowManagerService InputManagerService  
        startOtherServices(t);

        // system_server 进程 主线程
        Looper.loop();
    }

    // 1.ActivityManagerService 添加到ServiceManager
    //frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java  setSystemProcess()
    ServiceManager.addService(Context.ACTIVITY_SERVICE, this, /* allowIsolated= */ true,
                    DUMP_FLAG_PRIORITY_CRITICAL | DUMP_FLAG_PRIORITY_NORMAL | DUMP_FLAG_PROTO);

    // 2.PackageManagerService 添加到ServiceManager
    //frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java   main()
    PackageManagerService m = new PackageManagerService(injector, factoryTest,
                PackagePartitions.FINGERPRINT, Build.IS_ENG, Build.IS_USERDEBUG,
                Build.VERSION.SDK_INT, Build.VERSION.INCREMENTAL);
    ...
    // IPackageManagerImpl extends IPackageManagerBase; IPackageManagerBase extends IPackageManager.Stub
    IPackageManagerImpl iPackageManager = m.new IPackageManagerImpl(); 
    ServiceManager.addService("package", iPackageManager);


    // 3.WindowManagerService 添加到ServiceManager
    //frameworks/base/services/java/com/android/server/SystemServer.java startOtherServices()
    wm = WindowManagerService.main(context, inputManager, !mFirstBoot,
                    new PhoneWindowManager(), mActivityManagerService.mActivityTaskManager);
    ServiceManager.addService(Context.WINDOW_SERVICE, wm, /* allowIsolated= */ false,
                    DUMP_FLAG_PRIORITY_CRITICAL | DUMP_FLAG_PROTO);

    // ContentService 添加到ServiceManager
    //frameworks/base/services/core/java/com/android/server/content/ContentService.java Lifecycle.onStart() 
    mService = new ContentService(getContext(), factoryTest);
            publishBinderService(ContentResolver.CONTENT_SERVICE_NAME, mService);            
```

system_service进程创建发布各种Service后，最终主线程停在消息循环(`Looper.loop();`)。不仅仅是四大组件的创建发布，各种Service之间相互依赖，其他Service也伴随着创建启动。
