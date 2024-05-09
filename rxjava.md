### ReactiveX

[^1] 通过使用可观察序列来组成异步和基于事件的程序的库

<font color=red>扩展了[观察者模式](https://zh.wikipedia.org/wiki/%E8%A7%82%E5%AF%9F%E8%80%85%E6%A8%A1%E5%BC%8F)，以支持数据和事件序列</font>，并添加了操作符，可以声明方式将序列组合在一起。同时抽象出底层线程、同步、线程安全、并发数据结构和非阻塞 I/O 等问题。

------



#### 上下游(观察被观察)关系是如何建立起来的？

1. 既为主题也为观察者，一个主题一个观察者。没有一对多

2. 每个操作符有自己对应的主题类和观察者类


```kotlin
Observable.create { it.onNext(1) } // 创建 ObservableCreate source(上游)是{}里内容   	|
    .map { } // 创建 ObservableMap 上游是 ObservableCreate                          	|
    .observeOn(Schedulers.io()) // 创建 ObservableObserveOn 上游是 ObservableMap    	|
    .doOnNext { } // 创建 ObservableDoOnEach 上游是 ObservableObserveOn                  🔽
                                                                                   
// 创建各种操作符对应的主题(可观察者)，并指明主题对应的上游(source)
```

{ } -> ObservableCreate -> ObservableMap-> ObservableObserveOn -> ObservableDoOnEach

​	

```kotlin
Observable.create { it.onNext(1) } // {}是上游 触发{ }里内容        	                                 🔼		
    .map { } // 上游(ObservableCreate).subscribeActual(下游/观察者DoOnEachObserver)							
    .observeOn(Schedulers.io()) // 上游(ObservableMap).subscribeActual(下游/观察者ObserveOnObserver)	 |
    .doOnNext { } // 上游(ObservableObserveOn).subscribeActual(下游/观察者(DoOnEachObserver))	         |
    .subscribe() // ObservableDoOnEach.subscribeActual(下游/观察者(空))                                    |

// subscribe()时 指明各主题对应的下游
```

<font color=red>subscribe() </font> -----> subscribeActual方法。 subscribeActual方法会调用上游(source)的<font color=red>subscribe() </font>

subscribe() -> ObservableDoOnEach(subscribeActual) -> ObservableObserveOn -> ObservableMap -> ObservableCreate -> { }



#### 线程切换实现？

也就是observeOn、subscribeOn操作符的实现，在数据向下传递或者向上订阅时通过Schedulers(线程池提交任务)进行线程切换。

observeOn：在数据向下传递，流经该节点时(onNext/onError/onComplete)进行线程切换 ==> 下游所有事件消费端(onNext/onError/onComplete)都进行了线程切换

```kotlin
@Override
public void onNext(T t) {
    if (done) { // 事件是否结束error或者Complete
        return;
    }
    if (sourceMode != QueueDisposable.ASYNC) {
        queue.offer(t);
    }
    schedule(); // 线程切换
}
@Override
public void onError(Throwable t) {
    if (done) {
        RxJavaPlugins.onError(t);
        return;
    }
    error = t;
    done = true;
    schedule();
}
@Override
public void onComplete() {
    if (done) {
        return;
    }
    done = true;
    schedule();
}
```



subscribeOn：在事件向上订阅，经过该节点时(subscribeActual)进行线程切换 ==> 上游所有订阅(subscribe)和数据向下传(onNext/onError/onComplete)递都进行了线程切换

```kotlin
public void subscribeActual(final Observer<? super T> observer) {
    final SubscribeOnObserver<T> parent = new SubscribeOnObserver<>(observer);
		...
    parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
}
```



#### 代码描述

```kotlin
interface Observer_<T> {
    fun onNext(t: T)
    fun complete()
}

abstract class Subject<T> {

    abstract fun subscribeActual(observer: Observer_<T>)

    fun subscribe() {
        subscribeActual(object : Observer_<T> {
            override fun onNext(t: T) { /* empty */ }
            
            override fun complete() {}
        })
    }

    companion object {
        fun <T> created(subscribe: (Observer_<T>) -> Unit): Subject<T> {
            return CreateSubject(subscribe)
        }
    }

    // 各种操作符
    fun doNext(next: Consumer<T>): Subject<T> = DoOnEachSubject(this, next, null)

    fun doComplete(complete: Action): Subject<T> = DoOnEachSubject(this, null, complete)
    //Schedulers.io() Schedulers.computation() 线程池提交任务
    fun observeOn(executors: ExecutorService): Subject<T> = ObserveOnSubject(this, executors)

    fun subscribeOn(executors: ExecutorService): Subject<T> = SubscribeOnSubject(this, executors)
}

class CreateSubject<T>(val subscribe: (Observer_<T>) -> Unit) : Subject<T>() {
    override fun subscribeActual(observer: Observer_<T>) {
        println("顶层订阅, 向下发生数据 ${Thread.currentThread().name}") // 向下发送数据 ==> 调用下游(观察者)对应方法
        val topObservableEmitter = CreateSubjectEmitter(observer)
        subscribe(topObservableEmitter) // 发射器交出去,外部控制顶端数据向下发送
    }

    class CreateSubjectEmitter<T>(private val observer: Observer_<T>) : Observer_<T> {
        override fun onNext(t: T) = observer.onNext(t)

        override fun complete() = observer.complete()
    }

}

class DoOnEachSubject<T>(
    val source: Subject<T>, val next: Consumer<T>?, val complete: Action?
) : Subject<T>() {
    override fun subscribeActual(observer: Observer_<T>) {
        println("donNext/Complete订阅,调用source的订阅 ${Thread.currentThread().name}")
        source.subscribeActual(DoOnEachObserver(observer, next, complete))
    }

    class DoOnEachObserver<T>(
        val downStream: Observer_<T>, val next: Consumer<T>?, val complete: Action?
    ) : Observer_<T> {
        override fun onNext(t: T) {
            next?.accept(t)
            downStream.onNext(t)
        }

        override fun complete() {
            complete?.run()
            downStream.complete()
        }
    }
}

class ObserveOnSubject<T>(val source: Subject<T>, val executors: ExecutorService) : Subject<T>() {
    override fun subscribeActual(observer: Observer_<T>) {
        println("ObserveOn订阅,调用source的订阅 ${Thread.currentThread().name}")
        source.subscribeActual(ObserveOnObserver(observer, executors))
    }

    class ObserveOnObserver<T>(
        val observer: Observer_<T>, val executors: ExecutorService
    ) : Observer_<T> {
        override fun onNext(t: T) {
            // observer.onNext(t) 切换到其他线程执行,及其下游都将会在切换后的线程中执行
            executors.execute { observer.onNext(t) }
        }

        override fun complete() {
            // observer.complete() 切换到其他线程执行
            executors.execute { observer.complete() }
        }
    }
}

class SubscribeOnSubject<T>(val source: Subject<T>, val executors: ExecutorService) : Subject<T>() {
    override fun subscribeActual(observer: Observer_<T>) {
        println("SubscribeOn订阅, 切换线程后-调用source的订阅 ${Thread.currentThread().name}")
        // 从这个点的订阅(subscribe)开始切换线程 ==> 上游订阅以及数据向下传递线程都已经切换
        executors.execute {
            source.subscribeActual(SubscribeOnObserver(observer))
        }
    }

    class SubscribeOnObserver<T>(val observer: Observer_<T>) : Observer_<T> {
        override fun onNext(t: T) = observer.onNext(t)

        override fun complete() = observer.complete()
    }
}
```



```kotlin
Subject.created {
    it.onNext(1)
    it.onNext(3)
    it.complete()
}.doNext {
    println("111 ${Thread.currentThread().name} next:$it")
}.observeOn(Executors.newCachedThreadPool { r ->
    return@newCachedThreadPool Thread(r, "MyCache-thread")
}).doNext {
    println("222 ${Thread.currentThread().name} next:$it")
}.doComplete {
    println("222 ${Thread.currentThread().name} doComplete")
}.subscribeOn(Executors.newFixedThreadPool(1) { r ->
    return@newFixedThreadPool Thread(r, "MySinge-thread")
}).subscribe()

//        SubscribeOn订阅, 切换线程后-调用source的订阅 Test worker
//        donNext/Complete订阅,调用source的订阅 MySinge-thread
//        donNext/Complete订阅,调用source的订阅 MySinge-thread
//        ObserveOn订阅,调用source的订阅 MySinge-thread
//        donNext/Complete订阅,调用source的订阅 MySinge-thread
//        顶层订阅, 向下发生数据 MySinge-thread
//        111 MySinge-thread next:1
//        111 MySinge-thread next:3
//        222 MyCache-thread next:1
//        222 MyCache-thread next:3
//        222 MyCache-thread doComplete

```

1. 所有订阅都发生在MySinge-thread线程，应为subscribeOn写在最下游的
2. 数据向下传递在没到达observeOn节点前还是MySinge-thread线程，经过后切换到MyCache-thread线程



[^1]:https://reactivex.io/intro.html
