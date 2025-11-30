# Zygote 进程

## init.rc

init进程启动后，按顺序扫描目录下的rc文件，进行合并。然后执行.rc文件中定义的`on early-init`,`on init`,`on late-init`块。

扫描目录顺序：

1. /system/etc/init/      系统核心
2. /system_ext/etc/init/  系统扩展 与系统紧密相关，但是又不属于系统
3. /vendor/etc/init/    供应商  芯片/设备制造 硬件驱动服务
4. /odm/etc/init/       odm 基于芯片供应商（Vendor）的方案，小范围的修改和调整 传感器配置，屏幕校准
5. /product/etc/init/   产品    手机或者电视，区分不同产品型号  

- 对于 service 定义，如果多个文件定义了*同名服务*，后加载的会覆盖先加载的。比如，/vendor中的定义会覆盖 /system中的同名服务定义
- 对于 on 动作块，如果多个文件定义了同名事件（如 on init），它们的命令列表会合并追加在一起，形成一个更长的命令序列。  
- service执行，fork出子进程，在子进程中设置环境变量，通过`execv()`执行对应命令，同时父进程继续执行下一个命令。
  
> `system/core/init/init.cpp` 看详细.rc文件解析执行过程。
> `execv()`原地替换(替换代码段)，继承进程状态，只在执行失败时返回-1，执行成功不返回。

```shell
on init
    # 日志 system/logging/logd/logd.rc
    start logd 

    # 低内存杀守护进程 system/memory/lmkd/lmkd.rc
    start lmkd

    # 系统 servicemanager 
    # frameworks/native/cmds/servicemanager/servicemanager.rc定义 
    start servicemanager
    # 硬件 servicemanager
    start hwservicemanager
    start vndservicemanager

on late-init
    # zygote
    trigger zygote-start
    trigger boot

on boot
    class_start hal
    # core分类里面包括 surfaceflinger，bootanim
    class_start core    

```

`zygote-start`定义

```shell
on zygote-start
    # 64-bit 64位fork出来是64位
    start zygote
    # 32-bit 32位fork出来是32位
    start zygote_secondary

# zygote 定义
# 1.zygote进程。 1.1 jvm创建 1.2 fork system_service进程 1.3 zyoget socket usap池维护
# 2.system_service进程。ams pms wms等各种server的创建发布
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server --socket-name=zygote
    class main
    socket zygote stream 660 root system
    socket usap_pool_primary stream 660 root system
    
# zygote_secondary 定义
service zygote_secondary /system/bin/app_process32 -Xzygote /system/bin --zygote --socket-name=zygote_secondary --enable-lazy-preload
    class main
    socket zygote_secondary stream 660 root system
    socket usap_pool_secondary stream 660 root system    
```

在`late-init`中触发`zygote-start`，开启两个service，zygote(64位)和zygote_secondary(32位)。servcie内容就是执行`/system/bin/app_process64 xxx`命令。

## app_process

找打app_process对应的bp文件，会发现他是`frameworks/base/cmds/app_process/app_main.cpp`编译出来的。主要干的事情是，创建AndroidRuntime(准备java运行环境)，运行指定java类的Main方法。

```cpp
    class AppRuntime : public AndroidRuntime{
        // VM 创建后,运行静态Main方法前。 startVm
        virtual void onVmCreated(JNIEnv* env){
            if (mClassName.empty()) {
                return; // Zygote. Nothing to do here.
            }
            char* slashClassName = toSlashClassName(mClassName.c_str());
            mClass = env->FindClass(slashClassName); // 类引用
            if (mClass == NULL) {
                ALOGE("ERROR: could not find class '%s'\n", mClassName.c_str());
            }
            free(slashClassName);
            mClass = reinterpret_cast<jclass>(env->NewGlobalRef(mClass)); // 类全局引用
        }

        // om_android_internal_os_RuntimeInit_nativeFinishInit
        // com.android.internal.os.RuntimeInit的Main方法结束后回到这里    
        virtual void onStarted(){
            // 回旋镖 cpp -> java -> cpp, 在这里真正启动要执行的java类。
            sp<ProcessState> proc = ProcessState::self();
            ALOGV("App process: starting thread pool.\n");
            proc->startThreadPool();

            AndroidRuntime* ar = AndroidRuntime::getRuntime();
            // 给定类、参数的静态main方法
            ar->callMain(mClassName, mClass, mArgs);

            IPCThreadState::self()->stopProcess();
            hardware::IPCThreadState::self()->stopProcess();
        }

        // usap进程特化结束
        virtual void onZygoteInit(){
            sp<ProcessState> proc = ProcessState::self();
            ALOGV("App process: starting thread pool.\n");
            proc->startThreadPool();
        }
    }

    // frameworks/base/cmds/app_process/app_main.cpp
    int main(int argc, char* const argv[]){
        AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
        ...
        
//    "/system/bin/app_process64" "-Xzygote" "/system/bin" "--zygote" "--start-system-server" "--socket-name=zygote"
//    "app_process" "/system/bin" "com.binderjava.MainBinder"
        while (i < argc) {
            const char* arg = argv[i++];
            if (strcmp(arg, "--zygote") == 0) {
                zygote = true;
                niceName = ZYGOTE_NICE_NAME; // 64位：zygote64  32位：zygote
            } else if (strcmp(arg, "--start-system-server") == 0) {
                startSystemServer = true;
            } else if (strcmp(arg, "--application") == 0) {
                application = true;
            } else if (strncmp(arg, "--nice-name=", 12) == 0) {
                niceName = (arg + 12);
            } else if (strncmp(arg, "--", 2) != 0) { // 不是 --开头
                className = arg;
                break;
            } else {
                --i;
                break;
            }
        }

        if (zygote) {
            // --zygote, Start in zygote mode
            runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
        } else if (!className.empty()) {
            // 运行一个外部指定的class类。 app_process /System/bin com.android.commands.am.Am "$@"   
            // 比如：am instrument java单元测试
            runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
        }

    }
```

创建AppRuntime，参数解析，最后调用`runtime.start()`，并且传入了一个java类和解析后的参数。

```cpp
    // frameworks/base/core/jni/AndroidRuntime.cpp
    void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote){
        // 这里很多 vm相关参数配置
        ...
        if (startVm(&mJavaVM, &env, zygote, primary_zygote) != 0) {
            return;
        }
        // VM 创建后 startVm，app_main中有实现
        onVmCreated(env);
        ...
        jclass startClass = env->FindClass(slashClassName);
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main", "([Ljava/lang/String;)V");
        env->CallStaticVoidMethod(startClass, startMeth, strArray);
    }
```

在`AndroidRuntime::start`方法中，启动了vm，找到传过来的java类的入口(main方法)并运行。

### 运行指定java类

```java
    // frameworks/base/core/java/com/android/internal/os/RuntimeInit.java
    public static final void main(String[] argv) {
            preForkInit();
            if (argv.length == 2 && argv[1].equals("application")) {
                if (DEBUG) Slog.d(TAG, "RuntimeInit: Starting application");
                redirectLogStreams();
            } else {
                if (DEBUG) Slog.d(TAG, "RuntimeInit: Starting tool");
            }
            commonInit();

            /**
             *  Now that we're running in interpreted code, call back into native code
             *  to run the system.
             * */
            nativeFinishInit();
        }
```

```cpp
    // frameworks/base/core/jni/AndroidRuntime.cpp
    static void com_android_internal_os_RuntimeInit_nativeFinishInit(JNIEnv* env, jobject clazz){
    gCurRuntime->onStarted(); // AppRuntime::onStarted
}
```

执行普通java类时，传入是`com.android.internal.os.ZygoteInit`java类，其main方法一些初始化之后，又回到cpp层`onStarted`方法，在`onStarted`方法中才真正调用目标类的main方法。回旋镖的方式。(cpp->java->cpp)调用目标类的main方法。`am instrument xx`单元测试就是这种方式。
有人说这么做是为了分层，java相关初始化在`RuntimeInit.main`方法中完成，cpp层单纯就是运行指定java类的main方法。

### 运行Zygote进程

当指定`--zygote`参数后，传入是`com.android.internal.os.ZygoteInit`java类。其main方法中fork出system_server进程，接收来自zygote socket的消息并处理。

这是第一个java进程，其他app进程也是由他fork出来的。fork出来的子进程持有和父进程相同的资源，不用每个app进程都去加载相同的资源，提高响应速度。当然也有些从父进程继承而来，但是子进程并不需要的资源，不得不将其关闭。比如：`gUsapPoolEventFD`,`mZygoteSocket`。

载资源有：

1. `preloadClasses()`，加载由`/system/etc/preloaded-classes`文件指定的java类。比如：`android.app.Activity`，`android.view.View`
2. `cacheNonBootClasspathClassLoaders()`，用到又没有放在`BOOTCLASSPATH`下的java类。比如：`/system_ext/framework/androidx.window.extensions.jar`Jetpack折叠、多屏相关，`/system/framework/android.hidl.base-V1.0-java.jar`hidl定义生成的java部分代码。用到但是又不属于核心的部分，更新的时候方便点
3. `preloadResources()`，系统资源配置。比如：当前语言，屏幕密度宽高，系统主题中用到的颜色，样式，图像
4. `nativePreloadAppProcessHALs()`，HAL模块。比如：`GraphicBufferMapper`，绘制，视频编解码缓冲
5. `maybePreloadGraphicsDriver()`，GPU驱动加载。比如：OpenGL or Vulkan
6. `preloadSharedLibraries()`，共享native库，比如：libjnigraphics.so，libandroid.so
7. `preloadTextResources()`，字体排版相关
8. `prepareWebViewInZygote()`，WebView相关

> - `fork();` fork的下一行代码子进程，父进程都会执行。但是你可以通过返回值判断是子进程还是父进程。
> - fork时并没有立即复制父进程内存，而是父子进程指向相同物理内存。当父进程或者子进程要写共享内存时，才真正复制该页的内容，写到自己的副本上。这个叫写时复制(copy of write COW)。

#### fork system_service进程(forkSystemServer)

```java
    // system_service 启动参数
    String[] args = {
                "--setuid=1000", // uid
                "--setgid=1000", // gid
                "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1023,"
                        + "1024,1032,1065,3001,3002,3003,3005,3006,3007,3009,3010,3011,3012",
                "--capabilities=" + capabilities + "," + capabilities,
                "--nice-name=system_server",
                "--runtime-args",
                "--target-sdk-version=" + VMRuntime.SDK_VERSION_CUR_DEVELOPMENT,
                "com.android.server.SystemServer",
    };
```

system_service进程启动参数是硬编码的，fork出子进程设置设置uid，gid，进程名。找到`com.android.server.SystemServer`类的main方法并返回Runnable对象。这也就是system_server进程的入口。

> 子进程都是返回一个叫`MethodAndArgsCaller`的Runable对象，它内部保存类(来自于参数)的main方法，和对应的参数。这个main方法就是子进程入口。

#### 接收处理来自zygote socket指令，usap池维护(runSelectLoop)

先解释下usap功能，**usap**(unspecialized app process)未专有化的app进程。传统app启动，通过`mZygoteSocket`连接到Zygote进程，并发送要启动的应用信息。zygote进程fork出子进程，特化加载运行自己的app代码。

加入usap功能就是，提前fork出子进程，并不立即特化，而是阻塞在`usapPoolSocket.accept()`等待认领。这些进程就叫usap进程，这些usap进程在cpp层有个数组记录pid和pipFD(写端)叫usap tab表。在使用时不在连接`mZygoteSocket`，而是连接`mUsapPoolSocket`唤醒usap进程，根据发过来参数特化形成app进程。这样少了fork子进程这个步骤，提高应用启动速度。

循环poll等待`mZygoteSocket`，`sessionSocketRawFDs`，`mUsapPoolEventFD`，`usapPipeFDs`事件触发。

1. 通过sessionSocketRawFDs发送消息到zygote进程，解析消息并执行。比如：是否开启usap池，查询zygote进程id。*Zygote.forkAndSpecialize*这个好像是debug的时候会触发，*Zygote.forkSimpleApps*这个不知道什么时候触发。
2. usap进程退出，消耗时接收来自mUsapPoolEventFD,usapPipeFDs的消息，维护usap池。

- mZygoteSocket：其他进程和zygote进程通信的socket，init.rc文件配置创建。
- sessionSocketRawFDs：连接mZygoteSocket后的session socket。
- mUsapPoolEventFD：cpp层，监听usap进程退出信号(`SIGCHLD`)，从usap table表成功移除时，写1，zygote收到信号维护usap池。
- usapPipeFDs：usap进程被消耗，特化时，写进程id。zygote进程收到管道消息维护usap池。
  
> mUsapPoolSocket socket这个也是init.rc文件中配置创建的。新建的usap进程阻塞在`mUsapPoolSocket.accept()`，其他进程连接发送启动参数过来，特化形成特定的app进程。
> 特化：设置uid，gid，SELinux信息，进程名，data目录，更高优先级等等。 `frameworks/base/core/java/com/android/internal/os/Zygote.java` `specializeAppProcess()`方法。

```java
    //frameworks/base/core/java/com/android/internal/os/ZygoteServer.java runSelectLoop()
    ArrayList<FileDescriptor> socketFDs = new ArrayList<>();
    ArrayList<ZygoteConnection> peers = new ArrayList<>();
    socketFDs.add(mZygoteSocket.getFileDescriptor());
    peers.add(null);

    while (true) {
        int[] usapPipeFDs = null;
        // poll 监听的fd数组
        StructPollfd[] pollFDs;

        // 第一次 还没有开启usap， mUsapPoolEnabled=false
        if (mUsapPoolEnabled) {
            // cpp层table表，所有usap进程， read pipe 管道读端
            usapPipeFDs = Zygote.getUsapPipeFDs();
            // +1 是 mUsapPoolEventFD; UsapPoolEventFD
            pollFDs = new StructPollfd[socketFDs.size() + 1 + usapPipeFDs.length];
        } else {
            pollFDs = new StructPollfd[socketFDs.size()];
        }

        int pollIndex = 0;
        for (FileDescriptor socketFD : socketFDs) {
            pollFDs[pollIndex] = new StructPollfd();
            // 1.mZygoteSocket + session socket
            pollFDs[pollIndex].fd = socketFD;
            pollFDs[pollIndex].events = (short) POLLIN;
            ++pollIndex;
        }

        final int usapPoolEventFDIndex = pollIndex;

        if (mUsapPoolEnabled) {
            pollFDs[pollIndex] = new StructPollfd();
            // 2.usap event 事件fd mUsapPoolEventFD
            pollFDs[pollIndex].fd = mUsapPoolEventFD;
            pollFDs[pollIndex].events = (short) POLLIN;
            ++pollIndex;

            for (int usapPipeFD : usapPipeFDs) {
                // 3.usap进程 pipeFD 管道读端， 写端在子进程
                FileDescriptor managedFd = new FileDescriptor();
                managedFd.setInt$(usapPipeFD);
                pollFDs[pollIndex] = new StructPollfd();
                pollFDs[pollIndex].fd = managedFd;
                pollFDs[pollIndex].events = (short) POLLIN;
                ++pollIndex;
            }
        }
        ...
        pollReturnValue = Os.poll(pollFDs, pollTimeoutMs); // Linux poll

        if (pollReturnValue == 0) {                
            // 指定时间没有一个fd就绪（超时） 填充usap池
            mUsapPoolRefillTriggerTimestamp = INVALID_TIMESTAMP;
            mUsapPoolRefillAction = UsapPoolRefillAction.DELAYED;
        } else {
            // poll事件发生，处理产生事件的fd
            while (--pollIndex >= 0) {
                if ((pollFDs[pollIndex].revents & POLLIN) == 0) {
                    continue;
                }
                if (pollIndex == 0) {
                    // mZygoteSocket 有新的连接
                    ZygoteConnection newPeer = acceptCommandPeer(abiList);
                    // 连接添加到 peers
                    peers.add(newPeer);
                    // 新连接的socket（session socket）添加到 socketFDs
                    socketFDs.add(newPeer.getFileDescriptor());
                }else if (pollIndex < usapPoolEventFDIndex){
                    // session socket，消息来了
                    // peers和socketFDs是同步的，相同位置描述的是同一个session socket(除位置0)
                    // pollFDs按照socketFDs顺序添加的的，所以
                    // peers.get(pollIndex)获取的就是对应就绪的socket的 ZygoteConnection
                    ZygoteConnection connection = peers.get(pollIndex);
                    
                    // zygote进程处理发过来的消息
                    final Runnable command = connection.processCommand(this, multipleForksOK);
                    ...
                } else {
                    // pollIndex >= usapPoolEventFDIndex 
                    // mUsapPoolEventFD 或者 pipFD 读8字节
                    if (readBytes == Zygote.USAP_MANAGEMENT_MESSAGE_BYTES) {
                            DataInputStream inputStream = new DataInputStream(new ByteArrayInputStream(buffer));
                            messagePayload = inputStream.readLong();
                    }

                    if (pollIndex > usapPoolEventFDIndex) {
                        // pipeFD 管道 ==> messagePayload是进程id

                        //frameworks/base/core/java/com/android/internal/os/Zygote.java
                        //usap进程 childMain 方法，根据参数特化，消耗usap进程时写管道
                        int readBytes = Os.read(pollFDs[pollIndex].fd, buffer, 0, buffer.length);
                        // cpp 层，对应进程从table表移除
                        Zygote.removeUsapTableEntry((int) messagePayload);
                    }
                    // 这里有个隐藏的来自 mUsapPoolEventFD 的事件  pollIndex == usapPoolEventFDIndex

                    // frameworks/base/core/jni/com_android_internal_os_Zygote.cpp，信号处理函数 SigChldHandler()中发出, 写的是int 1
                    // app进程退出(正常或异常 SIGCHLD信号监听) && 成功从table表移除时，写usapPoolEventFDIndex 
                        
                    // 一个usap进程在，特化的时从table中移除一次usap pipe管道消息。退出的时候，从table中又移除一次 UsapPoolEventFD  ==>
                    // 防止usap进程在未特化前就退出的情况

                    // usap池有被消耗
                    usapPoolFDRead = true;
                }

            }

            // usap 进程被消耗后, 计算usap池填充时机
            // 1.立即填充usap池 2.延迟填充usap池 3.不填充usap池
            if (usapPoolFDRead) {
                // 当前usap池大小，就是usap table表大小 
                int usapPoolCount = Zygote.getUsapPoolCount();
                if (usapPoolCount < mUsapPoolSizeMin) {
                    // 当前usap进程数 < 最小值(默认1) 立即填充usap池                        
                    mUsapPoolRefillAction = UsapPoolRefillAction.IMMEDIATE;
                } else if (mUsapPoolSizeMax - usapPoolCount >= mUsapPoolRefillThreshold) {
                    // 剩余可填充的usap进程数 > 阈值(最大可填充数/2) 延迟填充usap池                        
                    mUsapPoolRefillTriggerTimestamp = System.currentTimeMillis();
                } 
            }
        }

        // usap池需要填充
        if (mUsapPoolRefillAction != UsapPoolRefillAction.NONE) {
            // 所有session socket（连接mZygoteSocket后的socket）
            int[] sessionSocketRawFDs = socketFDs.subList(1, socketFDs.size())
                            .stream().mapToInt(FileDescriptor::getInt$).toArray();
            final boolean isPriorityRefill =
                        mUsapPoolRefillAction == UsapPoolRefillAction.IMMEDIATE;

            // fork 出来的子进程继承了父进程的，sessionSocketRawFDs，mZygoteSocket，mUsapPoolEventFD，mUsapPoolSocket
            // mZygoteSocket(init.rc 配置创建)，mUsapPoolEventFD(来自native) cpp层可以获取到，所以没有向下传递
            // mUsapPoolSocket 是子进程需要
            final Runnable command = fillUsapPool(sessionSocketRawFDs, isPriorityRefill);  // fork出usap进程，阻塞在 usapPoolSocket.accept();
            if (command != null) {
                // 子进程，当有进程链接mUsapPoolSocket时(init.rc配置创建的)，usap进程唤醒 根据发过来的参数特化，从这里返回
                return command;
            } else if (isPriorityRefill) {
                // 父进程(zygote) 填充到最小值， 安排延迟填充
                mUsapPoolRefillTriggerTimestamp = System.currentTimeMillis();
            }
        }
    }
```

> epoll在添加fd时，`epoll_ctl(int epfd, EPOLL_CTL_ADD, int fd, struct epoll_event *_Nullable event)`，`epoll_event`中指定监听的事件类型，和自定义的一些数据data，在事件响应时获取到关联的自定义数据data。
> poll在添加fd时，`poll(struct pollfd *fds, nfds_t nfds, int timeout);`只有事件类型，没有数据data，在事件响应时只能获取到对应的fd，通过fd找到对应的事件处理函数。
