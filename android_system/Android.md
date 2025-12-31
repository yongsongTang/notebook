### ART

 Android 5.0前Dalvik(DVM)，5.0之后ART( Android Runtime).

java虚拟机：指令集是基于栈

android虚拟机：指令集基于寄存器



JIT  	just in time 

AOT	ahead of time 







#### Handle Looper MessageQueue

##### Handle构造

可以指定Looper对象，如果不指定就是当前线程的Looper对象，如果当前线程没有Looper对象抛异常。它将向Looper的消息队列传递消息或Runable对象，并在Looper的线程上执行他们

```java
public Handler(@Nullable Callback callback, boolean async) {
    ...
    mLooper = Looper.myLooper(); // 当前线程Looper
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread " + Thread.currentThread()
                    + " that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async; // 是否是异步(默认false) 不受同步屏障的影响
    mIsShared = false; // 共享不能进行一些危险操作
```

##### Handle 发送消息(sendMessage)

设置msg的target(handle)，将触发时间一起放入队列

> 深度睡眠时将会延后触发

```java
private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg,
        long uptimeMillis) {
    msg.target = this; // msg.target 当前handle
    msg.workSourceUid = ThreadLocalWorkSource.getUid();

    if (mAsynchronous) { // 如果handle构造时 异步的(async=true), 消息设置异步标志位
        msg.setAsynchronous(true);
    }
    // uptimeMillis: 应该被处理的绝对时间 SystemClock.uptimeMillis() + delay(未指定时为0)
  	// 放在message队列最前面时,uptimeMillis是0
    return queue.enqueueMessage(msg, uptimeMillis); 
}
```

##### MessageQueue 消息入队列

按时间(在那个时间处理这个消息)先后顺序向队列中插入msg，时间已经到了或者需要插队(AtFrontOfQueue)放在队列头。如果需要唤醒触发唤醒操作(nativeWake(mPtr))，队列中没有消息也没有idle handle需要处理时Looper线程将会阻塞(epoll_wait(-1)),这里就是唤醒线程，继续从队列中取消息。

```java
boolean enqueueMessage(Message msg, long when) {
    ...
    synchronized (this) {
      	...
        if (mQuitting) {
            ...
            msg.recycle(); // msg放入消息池(Message链表) 复用Message
            return false;
        }
        ...  
        msg.when = when;
        Message p = mMessages; // 链表头(第一个要处理的msg)
        boolean needWake;
        if (p == null || when == 0 || when < p.when) { // 链表头插入元素
            // New head, wake up the event queue if blocked. 
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            // Inserted within the middle of the queue.  Usually we don't have to wake
            // up the event queue unless there is a barrier at the head of the queue
            // and the message is the earliest asynchronous message in the queue.
            needWake = mBlocked && p.target == null && msg.isAsynchronous(); // 队列头有异步屏障消息
            Message prev;
            for (;;) { // 遍历按时间先后排序  在链表中间插入msg
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }

        // We can assume mPtr != 0 because mQuitting is false.
        if (needWake) {
            nativeWake(mPtr); // epoll fd INPUT 唤醒epoll_wait从队列中取消息返回
        }
    }
    return true;
}
```

##### MessageQueue 队列取消息

循环从队列中取message，如果第一个message是屏障(msg != null && msg.target == null)，找队列中异步消息处理(msg设置了异步标志位)，此时不处理其他消息，只到屏障消失。

> 屏障消息并不支持应用使用
>
> ```java
> @UnsupportedAppUsage
> @TestApi
> public int postSyncBarrier() {
>     return postSyncBarrier(SystemClock.uptimeMillis());
> }
> ```

1. 消息处理时间还没到，计算等待的时间，nativePollOnce触发等待(消息队列时间先后顺序排序)

2. 时间已经到了直接返回msg

3. 如果队列中为空，运行Idle Handler

   > 通过messagequeue.addIdleHandler添加Idle Handler处理事件

4. 如果消息队列为空，也没有要运行的Idle Handler则阻塞线程，直到有新消息添加

```java
Message next() {
    ...
    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    int nextPollTimeoutMillis = 0;
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        } 
      	// nextPollTimeoutMillis 	下一个msg在等待这么长时间后应该被处理。
      	// -1:无线等待(队列空并且IdleHandler也为空时) 0:立即返回 否则对应时间等待
        nativePollOnce(ptr, nextPollTimeoutMillis); // epoll_wait

        synchronized (this) {
            // Try to retrieve the next message.  Return if found.
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null && msg.target == null) {
              	// 如果第一个是屏障，队列中找同步消息(消息设置了同步标志位)处理，此时不处理其他消息
                // Stalled by a barrier.  Find the next asynchronous message in the queue.
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous()); // 队列中找同步消息处理
            }
            if (msg != null) {
                if (now < msg.when) {
                    // Next message is not ready.  Set a timeout to wake up when it is ready.
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // Got a message.
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null; // 断开链表节点
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                    msg.markInUse();
                    return msg;
                }
            } else {
                // No more messages.  
                nextPollTimeoutMillis = -1; // 阻塞，只到有对应事件就绪
            }

            // Process the quit message now that all pending messages have been handled.
            if (mQuitting) {
                dispose(); 
                return null;
            }

            // If first time idle, then get the number of idlers to run.
            // Idle handles only run if the queue is empty or if the first message
            // in the queue (possibly a barrier) is due to be handled in the future.
            if (pendingIdleHandlerCount < 0
                    && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size(); // messagequeue.addIdleHandler添加
            }
            if (pendingIdleHandlerCount <= 0) {
                // No idle handlers to run.  Loop and wait some more.
                mBlocked = true; 
                continue;
            }

            if (mPendingIdleHandlers == null) {
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }

        // Run the idle handlers. 处理idle handle (在没有message时干的事情)
        // We only ever reach this code block during the first iteration.
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; // release the reference to the handler
            boolean keep = false;
            try {
                keep = idler.queueIdle();
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }
            if (!keep) {
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }

        // Reset the idle handler count to 0 so we do not run them again.
        pendingIdleHandlerCount = 0;

        // While calling an idle handler, a new message could have been delivered
        // so go back and look again for a pending message without waiting.
        nextPollTimeoutMillis = 0;
    }
}
```

##### 同步屏障

在next取消息时，遇到同步屏障时(第一个Message是同步屏障)，此时只取异步消息执行，忽略所有同步消息。提高View绘制的优先级。

View.requestLayout(),View.invalidate()时触发。

```java
// ViewRootImpl.java
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}
```



##### Message处理

从MessageQueue中取出的消息，交给msg的target，也就是Handle.dispatchMessage处理。

```java
// Looper.loop 从队列中取消息并处理
public static void loop() { 
    final Looper me = myLooper();
		...
    for (;;) {
        if (!loopOnce(me, ident, thresholdOverride)) {
                return; // Looper.quit()
         }
    }
 }


private static boolean loopOnce(final Looper me,
        final long ident, final int thresholdOverride) {
    Message msg = me.mQueue.next(); // might block c
    if (msg == null) {
        // No message indicates that the message queue is quitting.
        return false;
    }
		...
    try {
        msg.target.dispatchMessage(msg); // msg.target ==>handle 
    } catch (Exception exception) {
        ...
        throw exception;
    } finally {
        ...
    }
		...
    msg.recycleUnchecked(); // msg放入消息池 复用
    return true;
}

public void dispatchMessage(@NonNull Message msg) {
    if (msg.callback != null) {
        handleCallback(msg); // message的callback，handle.post(Runable r)
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg); // 复写的handleMessage方法
    }
}
```

启动应用入口ActivityThread，主线程(Main Thread)调用Looper.looper()开启循环从MessageQueue中取消息

```java
public static void main(String[] args) {
    ...
    Looper.prepareMainLooper(); // 当前线程初始化Looper对象 sThreadLocal.set(new Looper(quitAllowed))
		...
    Looper.loop(); // 循环阻塞从MessageQueue(Looper)中取消息 交给Handel处理(dispatchMessage)

    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

##### Message 复用

handle.obtainMessage()获取Message时将会调用下面这个方法，从消息池取Message返回(链表头)

```java
private static Message sPool;  // 全局
public static Message obtain() {
    synchronized (sPoolSync) {
        if (sPool != null) {
            Message m = sPool;
            sPool = m.next;
            m.next = null;
            m.flags = 0; // clear in-use flag
            sPoolSize--;
            return m;
        }
    }
    return new Message();
}
```



在消息处理结束,即handle.dispatchMessage(msg)后，将msg数据清空放入复用池(插入链表头)，复用池对多保存50个Message

```java
@UnsupportedAppUsage
void recycleUnchecked() {
    // Mark the message as in use while it remains in the recycled object pool.
    // Clear out all other details.
    flags = FLAG_IN_USE;
    what = 0;
    arg1 = 0;
    arg2 = 0;
    obj = null;
    replyTo = null;
    sendingUid = UID_NONE;
    workSourceUid = UID_NONE;
    when = 0;
    target = null;
    callback = null;
    data = null;
    synchronized (sPoolSync) {
        if (sPoolSize < MAX_POOL_SIZE) { // MAX_POOL_SIZE=50
            next = sPool;
            sPool = this;
            sPoolSize++;
        }
    }
}
```



#### 时钟

##### System.currentTimeMillis()

标准时间(1970毫秒值)，可由用户或者网络设置，因此可能向前、向后跳跃。当与现实时间对应时使用(日历)。时间间隔应该使用不同的时钟。

##### SystemClock.uptimeMillis()

系统启动毫秒值，当系统进入深度睡眠时，时钟将会停止(不包括深度睡眠时间)。但是不会受到时间变化影响，适用不跨越休眠的时间间隔。

> - Thread.sleep(millis) 和 Object.wait(millis)使用uptimeMillis时钟，如果设备进入睡眠，剩下时间将会被推迟直到设备唤醒。
>
> - Handler也是使用uptimeMillis为时间基准，通常用于图形界面。
>
>   > 设备进入睡眠后延时任务被推迟。Time spent in deep sleep will add an additional delay to execution. 
>   >
>   > epoll_wait在设备进入深度睡眠时 等的时间更长，epoll_wait指定超时时间，实际唤醒时间比指定等待时间更长。
>
> - [AlarmManager](https://developer.android.com/reference/android/app/AlarmManager)可触发一次或重复事件，在设备睡眠或者应用未运行

##### SystemClock.elapsedRealtime()

系统启动时间，包括深度睡眠时间。适合计算间隔



#### StartActivity

ActivityA --> ActivityManagerService(检查调用权限，解析Intent，Activity栈)

ActivityManagerService --> ApplicationThread (binder)

ActivityThread.handleLaunchActivity()

```java
public Activity handleLaunchActivity(ActivityClientRecord r,
        PendingTransactionActions pendingActions, Intent customIntent) {
    // If we are getting ready to gc after going to the background, well
    // we are back active so skip it.
    unscheduleGcIdler(); // idle handle 中  BinderInternal.forceGc(reason);
  	...
    // Initialize before creating the activity
    if (ThreadedRenderer.sRendererEnabled
            && (r.activityInfo.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0) {
        HardwareRenderer.preload();
    }
   	...
    final Activity a = performLaunchActivity(r, customIntent);
  	...
    return a;
}

 private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ActivityInfo aInfo = r.activityInfo;
       	...
        ContextImpl appContext = createBaseContextForActivity(r); //  ContextImpl.createActivityContext
        Activity activity = null;
        java.lang.ClassLoader cl = appContext.getClassLoader();
        activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent); // 反射创建Activity
        ...
        try {
            // 没有的话创建Application
            Application app = r.packageInfo.makeApplicationInner(false, mInstrumentation);
            if (activity != null) {
               	...
                appContext.setOuterContext(activity);
              	// Activity.attach, Activity和Window(PhoneWindow)建立起关系
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.activityConfigCallback,
                        r.assistToken, r.shareableActivityToken);
              	...
                // Assigning the activity to the record before calling onCreate() allows
                // ActivityThread#getActivity() lookup for the callbacks triggered from
                // ActivityLifecycleCallbacks#onActivityCreated() or
                // ActivityLifecycleCallback#onActivityPostCreated().
                r.activity = activity;
                if (r.isPersistable()) {
                  // Activity.OnCreate
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
            		...   
            }
            r.setState(ON_CREATE);
        } catch (SuperNotCalledException e) {
            throw e;
        } catch (Exception e) {
         ...  
        }
        return activity;
    }

```

#### Activity Window  View

Activity.setContextView  --> 
	Window(PhoneWindow).setContextView  // 创建DecorView并且将Activity的布局添加进去

> ```java
> // PhoneWindow.java
> public void setContentView(View view, ViewGroup.LayoutParams params) {
>   // 如果继承AppCompatActivity将会在AppCompatDelegateImpl.java 中创建DecoreView  
>     if (mContentParent == null) { 
>         installDecor(); // 创建DecoreView
>     } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
>         mContentParent.removeAllViews();
>     }
> 
>     if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
>         view.setLayoutParams(params);
>         final Scene newScene = new Scene(mContentParent, view);
>         transitionTo(newScene);
>     } else {
>         mContentParent.addView(view, params); // mContentParent内容部分 添加Activitr布局
>     }
>     mContentParent.requestApplyInsets();
>     final Callback cb = getCallback();
>     if (cb != null && !isDestroyed()) {
>         cb.onContentChanged();
>     }
>     mContentParentExplicitlySet = true;
> }
> ```

显示时

ActivityThread.java.handleResumeActivity()  -> 
	WindowManagerImpl.java -> addView() ->  new ViewRootImpl
		ViewRootImpl.java -> setView(DecorView)  // DecorView的mParent是ViewRootimpl, 其他view是在VIewGroup.addView时指定

```java
// ViewGropu.java
private void addViewInner(View child, int index, LayoutParams params,
        boolean preventRequestLayout) {
  // tell our children
    if (preventRequestLayout) {
        child.assignParent(this);
    } else {
        child.mParent = this;
    }
}
```



#### View绘制流程

绘制/动画/输入时层层向上传递到达最顶层ViewRootImpl，触发ViewRootImpl的遍历任务调度(测量，布局，绘制)，此时并不会真正执行而是把它放到链表中(Choreographer.mCallbackQueues[CALLBACK_TRAVERSAL] 按照执行时间排序)，Choreographer接收硬件屏幕刷新信号(Vsync)从mCallbackQueues中取出已经到时间的任务执行。



##### ViewRootImpl

视图层次结构的顶端(DecoreView的父视图)，显示时(Resume)由WindowManagerImpl创建，实现View和WindowManager之间所需的协议.View触发invalidate，requestLayout通过层层向上传递到达ViewRootImpl，ViewRootImpl收到消息后触发"遍历调度"

```java
// ViewRootImpl.java
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier(); // 同步屏障 界面刷新
   // Choreographer安排触发时机(协调屏幕刷新和绘制)，mTraversalRunnable是触发performTraversals方法(测量 布局 绘制) 
   // 收到VSYNC信号(硬件屏幕刷新周期) 执行mTraversalRunnable进行绘制   
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null); 
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}

private void performTraversals() {
  ...
  // View(DecorView).measure(int, int)  
  // 遍历每个子View测量 触发 onMeasure(int widthMeasureSpec, int heightMeasureSpec)方法，设置View宽高
  performMeasure(childWidthMeasureSpec, childHeightMeasureSpec); // 测量  2位模式 30位值
  
  ...
  // View(DecorView).layout(int left, int top, int right, int bottom) 
  // ViewGropu.onLayout(boolean changed, int left, int top, int right, int bottom)  
  // 遍历每一个子View	布局 触发onLayout(boolean changed, int left, int top, int right, int bottom)
  performLayout(lp, mWidth, mHeight); // 布局
  
  ...
  // drawSoftware(Surface, AttachInfo, int, int, boolean, Rect dirty, Rect surfaceInsets) //dirty:绘制区域
  // View(DecoreView).draw(Canvas) 
  // ViewGroup.dispatchDraw()
  // 遍历触发每一个子View的 onDraw(Canvas)方法  
	performDraw(); // 绘制

}
```



##### VSYNC(Vertical Synchronization)

协调屏幕刷新和绘制的时间点(Choreographer)，接受应用输入 input，动画，绘制缓存在CallbackQueue中，接受硬件VSYNC信号从CallbackQueue取出调用run方法执行

屏幕刷新率：显示器每刷新一次称为一个刷新周期。在这个周期内，显示器从顶部到底部逐行更新图像。60HZ就是16.6ms刷新一次

绘制频率：GPU每秒能合成绘制多少帧

> 绘制频率低于屏幕刷新率，绘制不能在16.6ms中完成，屏幕重复显示旧的帧。出现卡顿不流畅
>
> 绘制频率高于屏幕刷新率，绘制很快在16.6ms以内完成，那么有些帧将不会被显示，用户看不到完整的帧，导致不连贯跳跃

VSYNC机制强行将绘制(CPU GPU共同操作)和屏幕刷新拖拽到同一起点，

> 1. 屏幕撕裂：双缓冲，一个用于显示，一个用于绘制。在VSYNC时，交换缓冲区。每次显示都是完整数据(不会在显示中数据变化)
> 2. 卡顿：不同步，在过了刷新点之后，请求绘制。要想在下一个刷新点能够显示那么留给绘制的时间将不足16.6ms，即使绘制的时间小于16.6ms也只能在下一个刷新周期显示，丢帧卡顿
> 3. 资源浪费：控制帧率使其与显示器的刷新率匹配，防止GPU过度渲染，消耗更多CPU GPU资源，减少功耗和热量。屏幕无法显示超过刷新频率的内容



```java
// FrameDisplayEventReceiver.java Choreographer的内部类
// 收到SYNC信号时
@Override
public void onVsync(long timestampNanos, long physicalDisplayId, int frame,
        VsyncEventData vsyncEventData) {
    		...
        mTimestampNanos = timestampNanos;
        mFrame = frame;
        mLastVsyncEventData.copyFrom(vsyncEventData);
        Message msg = Message.obtain(mHandler, this); // run()  doFrame
        msg.setAsynchronous(true);
        mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
}

@Override
public void run() {
    mHavePendingVsync = false;
    doFrame(mTimestampNanos, mFrame, mLastVsyncEventData);
}
```

```java
// Choreographer.java 处理一帧内是事情
void doFrame(long frameTimeNanos, int frame,
        DisplayEventReceiver.VsyncEventData vsyncEventData) {
  AnimationUtils.lockAnimationClock(frameTimeNanos / TimeUtils.NANOS_PER_MS);

	mFrameInfo.markInputHandlingStart();
  doCallbacks(Choreographer.CALLBACK_INPUT, frameIntervalNanos); // 输入

  mFrameInfo.markAnimationsStart();
  doCallbacks(Choreographer.CALLBACK_ANIMATION, frameIntervalNanos); // 动画
  doCallbacks(Choreographer.CALLBACK_INSETS_ANIMATION, frameIntervalNanos);

  mFrameInfo.markPerformTraversalsStart();
  doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameIntervalNanos); // 界面显示(遍历view 刷新界面)

  doCallbacks(Choreographer.CALLBACK_COMMIT, frameIntervalNanos);  
}
```

##### 触摸事件

ViewRootImp接收到触摸事件

​	DecorView.java.dispatchTouchEvent ->
​			Activity.java.dispatchTouchEvent ->  // 返回false 说明View中没有消费事件，将会触发Activity.onTouchEvent
​				PhoneWindow.superDispatchTouchEvent ->
​					DecorView.java.superDispatchTouchEvent ->
​						ViewGroup.java.dispatchTouchEvent   

**dispatchTouchEvent**

View中：触发onTouchEvent消费事件

ViewGroup中：倒序遍历调用子View的dispatchTouchEvent, 子View消费了事件停止遍历

**onInterceptTouchEvent**

```java
final boolean intercepted;
if (actionMasked == MotionEvent.ACTION_DOWN // down或者mFirstTouchTarget不为空
        || mFirstTouchTarget != null) { // 任何子View消费了任何事件不为空
    final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
    if (!disallowIntercept) {
        intercepted = onInterceptTouchEvent(ev);
    }
```

1. 在down事件拦截时，表明父视图希望接管整个触摸事件序列的处理。这样，子视图不会接收到后续的任何触摸事件。后续事件将交给父视图的onTouchEvent

2. 在move事件拦截时，父视图希望在触摸事件序列进行中的特定条件下接管触摸事件的处理。处理事件的子视图会收到一个 `ACTION_CANCEL` 事件，表示它不再接收后续的触摸事件，父视图接管后续的触摸事件处理。

   

```java
// DecoreView.java
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
    // ActivityThread中创建Activity时，调用Activity.attach方法时设置的，Activity实现Window.Callback 
    final Window.Callback cb = mWindow.getCallback();  
    return cb != null && !mWindow.isDestroyed() && mFeatureId < 0 //触发Activity中dispatchTouchEvent 
            ? cb.dispatchTouchEvent(ev) : super.dispatchTouchEvent(ev); 
}
```

``` java
// Activity.java
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        onUserInteraction();
    }
    if (getWindow().superDispatchTouchEvent(ev)) { // Phonew.superDispatchTouchEvent()
        return true;
    }
    return onTouchEvent(ev); // 如果没有任何View消费这个事件，Activity的onTouchEvent
}
```



##### ANR(application not response)





#### Binder

共享内存：直接将内存区域共享给多个进程，映射相同的物理内存区域到自己的虚拟空间，实现数据共享(mmp, ashmem-Android Shared Memory)

Binder：







#### Context

提供访问应用环境，资源，系统服务的抽象类

##### ContextWrapper

Context代理实现类，所有操作都委托给另一个Context(mBase)。可以继承这个类，修改行为。



##### ContextThemeWrapper



##### Application(applicationContext)

继承ContextWrapper，生命周期与应用生命周期一致，从应用启动到应用结束。

> 全局持有的资源，e.单例对象、数据库、SharedPreferences
> 不与界面直接交互的操作,全局的广播接收器

##### Activity Context

> Activity生命周期
> 与界面交互，比如显示dialog





#### Androd内存泄漏

- 长生命周期对象持有短生命周期对象的引用

- 静态变量持有上下文引用(静态变量持有Context)

- 未取消的注册的监听事件(广播)

- Handle引用(内部类默认持有外部类引用),导致外部类(Activity)无法回收。

  > 静态Handle持有外部类Activity的弱引用。
  >
  > 现在都用ViewModel了

Context，ApplicationContext

Register-UnRegister(广播)

Handle







