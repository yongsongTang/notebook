# JVM



### 内存结构

<font style="background: blue">方法区</font>

> 类信息(版本、字段、方法、接口)，静态变量，常量
>  

<font style="background: blue">堆</font>

> 对象实例，数组
>
> 新生代   
>
> > eden(8) 	survivor0(1) 	 survivor1(1)
> 
>  老年代

线程共享

------

<font style="background: gray">虚拟机栈</font>

> 栈帧： 支持方法运行调用。每个线程在运行方法时，都会为这个方法创建一个栈帧
>
> > 局部变量表：变量值存储空间，传递的参数，方法内局部变量。编译后确定表容量
> >
> > 操作数栈：方法执行过程中各种指令压入弹出。编译后确定深度
> >
> > 返回地址：方法执行完成后退出(正常退出/异常退出)。返回调用的位置，继续执行。
> >
> > > 正常退出：PC计数器
> > >
> > > 异常退出：异常处理器表
> >
> > 动态链接：方法调用过程中动态链接。一个class文件调用其他方法，需要将这些符号引用转化为其所在内存地址的直接引用，这些引用存在方法区中。每个栈帧都包含一个指向运行时常量池中该栈帧所属方法的符号引用。

<font style="background: gray">本地方法栈</font>

<font style="background: gray">程序计算器</font>

> 在线程切换 循环 跳转 记录方法执行到的位置，返回时根据程序计数器位置继续向下执行指令。

线程私有，随着线程的创建而创建，线程的结束而死亡。

------

#### GC回收机制

GC触发时机

1. 堆内存分配时，剩余空间不足
2. System.gc()



#### 可达性分析

GC Root对象：

1. 虚拟机栈中局部变量表引用的对象

2. 方法区中静态引用指向对象

3. 活跃状态中的线程对象(线程未结束，线程中引用对象)

4. Native方法中JNI引用对象

   

GC Root为根节点，开始向下搜索，如果从GC Root到某个对象不可达，则证明此对象是不会在被使用的。

垃圾回收线程运行的同时，其他线程也是在运行的，如何准确的将可回收线程标记出来。简单粗暴的方式就是将其他线程全部暂停直到垃圾收集结束(Stop The World)。这种方式在单核(没有线程切换消耗，专心收垃圾)，管理内存小(对象引用图不复杂，查找引用链时间短)时还行。如果堆越大，存储对象越多，对象结构图越复杂，标记产生停顿时间越长。怎么做才能尽可能减少停顿时间呢。

##### 三色标记算法

白色：尚未被垃圾收集器扫描。开始阶段都是白色。扫描结束仍为白色说明该对象不可达

灰色：该对象已经被垃圾收集器扫描，但最少还有一个引用没被扫描过

黑色：该对象所有引用都已经扫描过，无需重新扫描。黑色对象不可能直接指向某个白色对象(不经过灰色对象)

![](https://raw.githubusercontent.com/yongsongTang/PicGo/main/img/%E4%B8%89%E8%89%B2%E6%A0%87%E8%AE%B0%20(2).png)

1. 初始标记

   > 将GC Root能直接关联的对象标记为灰色。Stop The World
   >
   > 根节点枚举必须在一个能保障一致性的快照中进行，整个枚举期间执行子系统看起来就像是被冻结在某个时间点上。如果不停下来，所有分析都是基于一个不准确的引用关系

2. 并发标记

   > 从灰色对象开始遍历对象图，(只是这一层，广度优先)将灰色对象直接关联的对象标记为灰色。如果该对象所有引用都扫描完成，则将该对象标记为黑色
   >
   > > | 应该 | 实际 | 原因                                                         |                            |
   > > | :--: | ---- | ------------------------------------------------------------ | -------------------------- |
   > > | 白色 | 黑色 | 在标记为灰色之后引用链断裂                                   | 能接受,等待下一次扫描      |
   > > | 黑色 | 白色 | 以下两条件同时满足时会出现对象消失[^1]<br />1. 插入一条或多条从黑色对象到白色对象新引用<br />2. 删除了全部从灰色对象到该白色对象的引用 | 对象消失，回收使用中的内存 |

3. 重新标记

   > 由于并发标记阶段，对象间引用关系发生变化。确保在并发标记期间发生的引用关系变化被正确处理。Stop The World 
   >
   > > 1. 增量更新，黑色对象插入新的指向白色对象引用关系时，将新插入的引用记录下来。并发扫描结束后，将记录中的黑色对象为根，重新扫描一次。黑色对象一旦插入白色对象就变回灰色对象了。
   > > 2. 原始快照，灰色对象要删除指向白色对象引用关系时，将这个要删除引用记录下来。并发扫描结束后，将记录过的引用中的灰色对象为根，重新扫描一次。无论引用关系是否删除，都会按照刚开始那一刻的对象图快照进行扫描。

jvm中的引用

强引用(Strong Reference)， 软引用(Soft Reference)，弱引用(Weak Reference)，虚引用(Phantom Reference)

| 强引用 | 决不能回收                     | Object o = new Objext()                                      |
| ------ | ------------------------------ | :----------------------------------------------------------- |
| 软引用 | 内存不足时回收                 | SoftReference<Object> softRef = new SoftReference<Object>()  |
| 弱引用 | 垃圾回收时回收                 | WeakReference<Object> weakRef = new WeakReference<>(new Object()); |
| 虚引用 | 对象被垃圾回收器回收时收到通知 | ReferenceQueue<Object> referenceQueue = new ReferenceQueue<>();   PhantomReference<Object> phantomRef = new PhantomReference<>(new Object(), referenceQueue); |

软引用，弱引用通常用于缓存内存敏感数据对象。虚引用用来管理内存释放。



#### 分代回收策略

核心理论

1. 弱分代假说，大多数对象生命周期都很短，很快会被回收，只有少部分具有长生命周期。

   > 将堆内存划分为不同的代，不同代有不同回收频率和回收算法


2. 强制不变量假说，如果某个对象在一代中存活下来，那么他在另一代中存活下来的概率也很高。

   > 年轻代：较小空间   轻量级垃圾回收算法(标记复制，大部分对象很快被回收 复制少量存活对象)          
   >
   > 老年代：较大空间   更复杂耗时垃圾回收算法(标记压缩，标记存活对象 清除未标记对象 将存活对象紧凑移向一端)



##### 新生代

Eden(8)  	Survivor0(1)	  Survivor1(1)

新创建对象存放Eden区(大多数)，Eden区满时进行垃圾回收，将Eden区垃圾清除，存活对象复制到S0，此时S1区域为空。下一次Eden区满时再执行垃圾回收，将Eden区和S0区的垃圾对象清除，并将存活对象复制到S1，此时S0区域为空。如此返回几次(15)存活的对象说明有较长的生命周期，则将他们转移到老年代。

> 如果空闲区域(S0或S1)不足以装下所有存活对象，则直接转移到老年代

新生代GC(Minor gc)，大多数对象朝生夕死所以回收频率高，回收速度快短暂停顿

##### 老年代

新生代存活时间足够长，移动到老年代。如果对象比较大，新生代空间不足，则直接将大对象分配到老年代。

> 老年代引用到新生代对象，此时新生代GC，需要查询老年代的引用情况。老年代生命周期长，存活对象多。为了加快速度维护了一个card table，记录老年代对新生代的引用信息。

老年代GC(Major gc，Full gc)，这些对象生命周期长频率低，更复杂垃圾回收算法，可能需要移动大量对象停顿时间长



### 类加载

java程序运行时将字节码加载到内存中并初始化的过程，加载类文件(.class文件)并在内存中生成对应的Class对象，使得程序能够使用这些类。 并不是一次加载所有.class文件，而是运行过程中动态加载相应类到内存中。

类加载时机

1. 调用类构造方法(new 反射 初始化子类前先加载父类)

2. 调用类中静态变量或静态方法

类卸载时机，类卸载由jvm控制



##### JVM中3个类加载器

启动类加载器(BootstrapClassLoader)：加载java核心类库，jvm一部分c++实现，加载java核心类库(java.lang 字符串 线程 ClassLoader)

扩展类加载器(ExtClassLoader)：JDK 1.9 之后，改名为 PlatformClassLoader，加载jvm扩展类库，通常是jdk引入的三方库

应用类加载器(AppClassLoader): 加载应用程序的类，自己写的类三方jar包中类

##### 双亲委派机制

1. 检查Class是否已经被加载
2. 如果没有加载过，判断parent是否为空，不为空加载任务给到parent
3. parent为空则用BootstrapClassLoader加载
4. 如果parent和BootstrapClassLoader都没加载成功，用当前ClassLoader的findClass方法

```kotlin
protected Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException
{
        // First, check if the class has already been loaded
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            try {
                if (parent != null) {
                    c = parent.loadClass(name, false);
                } else {
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }

            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.
                c = findClass(name);
            }
        }
        return c;
}
```

Test t = new Test();

1. 用AppClassLoader加载这个类，加载任务委派给parent(ExtClassLoader)

2. ExtClassLoader的parent为空，所以加载任务委派给BootstrapClassLoad

3. parent和BootstrapClassLoad都有没成功加载Test类，调用自己(AppClassLoader)的findClass方法


```kotlin
println("1 ${Test::class.java.classLoader}") // AppClassLoader
println("2 ${Test::class.java.classLoader?.parent}") // PlatformClassLoader
println("3 ${Test::class.java.classLoader?.parent?.parent}") // null
```

父类加载失败 退回给子类加载。

> 双亲委派机制 是java推荐的 并不是强制，可以继承java.lang.ClassLoader 实现自己的类加载器。若果保持双亲委派机制就只重写findClass，如果破坏双亲委派机制就重写loadClass



##### Android下的ClassLoader 

PathClassLoader：ClassLoader 实现，该实现对本地文件系统中的文件和目录列表进行操作，但不会尝试从网络加载类。 Android 使用此类作为其系统类加载器和应用程序类加载器。

DexClassLoader：从classes.dex、.jar、 apk文件中加载。可用于执行作为应用程序未安装部分的代码。 热修复(下载更新dex， 重启应用更改代码行为，不用重装app)

> 在api 26前 需要一个应用程序私有的科协目录来存储类。Context.getCodeCacheDir()

```kotlin
Log.i(TAG, ("1 ${Test::class.java.classLoader}")) // PathClassLoader
Log.i(TAG, ("2 ${Test::class.java.classLoader?.parent}")) // BootClassLoader
Log.i(TAG, ("3 ${Test::class.java.classLoader?.parent?.parent}")) // null
```



##### ClassLoader加载字节码过程

1. 装载

   加载字节码文件(.class)，根据字节流 生成java.lang.Class对象

   > 1. ClassLoader通过一个类的全名(包名+类名)来查找字节码文件(class、jar、zip、网络)
   > 2. 解析字节码文件，生成代表该类的java.lang.Class对象。后续对该类的访问都通过这个类对象

2. 链接

   > 1. 验证：对要加载的类进行验证 确保符合字节码格式 语义
   >
   > 2. 准备：为类的静态变量分配内存 赋值(根据变量类型赋值)
   >
   >    基本类型（int、long、short、char、byte、boolean、float、double）的默认值为 0；
   >
   >    引用类型 null
   >
   > 3. 解析：符号引用转换为直接引用，将常量池中的类、接口名、字段名、方法名转换为具体内存地址

3. 初始化

   静态变量赋予指定的值，执行类的静态代码块。确保类的所有静态成员都被赋值

   > ```java
   > static { System.loadLibrary("");}
   > ```

##### 调用顺序(静态代码块/非静态代码块/构造函数)

```java
class Parent {
    static {
        // 多次创建对象也只会调用一次(类加载 初始化时调用)
        System.out.println("Parent 静态代码块");
    }

    {
        // 对象创建一次 调用一次（kotlin没有这种写法，放到构造函数中）
        System.out.println("Parent 非静态代码块");
    }

    public Parent() {
        System.out.println("Parent 构造函数");
    }
}

class Child extends Parent {
    static {
        System.out.println("Child 静态代码块");
    }

    {
        System.out.println("Child 非静态代码块");
    }

    public Child() {
        System.out.println("Child 构造函数");
    }
}
```

```java
Child c = new Child();
System.out.println("========================");
c = new Child();

//        Parent 静态代码块  
//        Child 静态代码块		
//        Parent 非静态代码块
//        Parent 构造函数
//        Child 非静态代码块
//        Child 构造函数
//        ========================
//        Parent 非静态代码块
//        Parent 构造函数
//        Child 非静态代码块
//        Child 构造函数
```

静态类代码块(类加载初始化时调用)  ->  非静态代码块 -> 构造函数



#### 内存模型

定义jvm如何与计算机内存进行交互，多线程中变量可见性，顺序性的行为规范。



##### 缓存一致性

多处理器系统中，各个处理器(核)中缓存的数据与主内存中数据保持一致。每个处理器都有自己的缓存用于加速对内存的访问，当多个处理器访问同一块内存区域时，可能出现缓存数据与主内存数据不一致的问题。(e.一个线程已经把值赋值成a 刷到主内存，但另一个线程缓存中还是b)

工作内存：cpu中寄存器和高速缓冲区的描述

主内存：线程之间共享变量存储

线程是cpu调度的最小单位，一个设备通常都是多cpu多核，所以同一时刻有多个线程在被不同的cpu执行。每个线程都有自己的工作内存，工作内存中存储了该线程读/写变量的副本。

##### 指令重排

对字节码指令进行重新排序，已提高计算机性能。单线程下不会用什么问题，应为重排后执行结果与重排前执行结果是一致的。


```java
class ReorderExample {
    private int x = 0;
    private boolean y = false;

    public void write() {
        x = 1;       // 写操作1
        y = true;    // 写操作2
    }
    public void read() {
        if (y) {     // 读操作1
            System.out.println("====x: " + x);  // 读操作2
        }
    }
}
```

两个线程分别调用read和write方法，由于指令重拍 y = true 提前 x = 1操作。这样及时y=true由于x还未赋值也会出现出人意料的结果。



##### happens-before

是内存模型中的一种规则，描述两个操作的内存可见性。如果操作A happens-before 操作B， 则A操作的结果对于B操作是可见的。也就是说操作B可以看到操作A的结果。

jvm定义了几种规则自动符合happens-before规则

1. 程序次序规则

   同一线程中，按照程序代码顺序执行

   > int a = 10;  // 1
   > b = b + 1;   // 2
   >
   > 由于b对a没有依赖所有可能发生指令重排
   >
   > int a = 10;  // 1
   > b = b + a;   // 2

2. 锁规则

   同一个对象的锁，解锁操作happens-before加锁操作。也就是说一个线程释放了锁，另一线程获取了该锁，那么在第一个线程释放锁之前做的所有操作对于第二个线程获取锁后都是可见的。

3. 变量规则(volatile变量)

   对一个 volatile 变量的写操作 happens-before 后续对该变量的读操作。也就是说一个线程对volatile变量进行写操作，那么其他线程一定能看到写之后的结果。

4. 传递性规则
   A happens-before B，B happens-before C 则 A happens-before C

5. 线程启动规则
   线程A中启动(start)线程B，那么在线程B中可以看到，线程A在启动B之前对共享变量的修改

   ```kotlin
   i = 9
   val t = thread {
       println("==== $i")
   }
   t.join()
   ```

   主线程在启动子线程之前修改共享变量i，根据线程启动规则 子线程对共享变量可见(修改后的值)

6. 线程终止规则
   线程中所有操作都 happens-before 该线程终止。也就是说线程A等待(join方法)线程B终止，线程B中所有操作在线程A等待结束后都可见。

7. 对象终结规则

   对象终结前，对象构造函数中所有操作已经执行完毕。初始化 happens-befor finalize()

通过happens-befor规则判断是否存在竞争



#### 锁

##### Synchronized 

- 修饰实例方法
  锁对象是当前实例，只有同一个对象调用锁方法才会出现互斥。不同对象调用锁方法不会有互斥效果

- 修饰静态方法
  锁对象是该类的Class对象，调用不同实例的锁方法也会出现互斥现象

- 修饰代码块
  锁对象是括号中指定的对象

可重入锁：同一线程可以多次获取同一把锁，不会被自己所持有的锁阻塞

实现原理

Synchronized同步方法、代码块是与对象相关联的。 java对象在内存中布局分为3部分。对象头，实例数据，对齐填充数据。对象头中有部分(mark word标记文字 )存储锁信息。java每个对象头中保存一个(ObjectMonitor对象)，所以每个对象都可以作为锁对象

锁状态 	 		    25bit 				 	4bit		  		 1bit		 		 	2bit
无锁				hash code值		对象分代年龄		是否偏向锁			锁标志
偏向锁
轻量级锁
重量级锁

- 当一个线程进入Synchronized方法或者代码块时，会尝试获取对象锁
- 如果锁是空闲的，则线程通过CAS操作将锁置为自己(偏向锁)，Owner指向该线程
- 如果锁已经被其他线程占用，当前线程进入到锁等待队列(EntrySet)，进入阻塞状态(Blocking)
- 如果线程调用了wait方法，则会释放锁对象，Owner变为null同时加到的等待队列(WaitSet),进入Waiting状态，等待唤醒
- 如果线程调用notifyf方法将会唤醒WaitSet中的线程，从新加入到EntrySet。notify并不会释放锁
- 线程释放锁(执行完成或者wait)后，jvm会选择等待队列中的某个线程唤醒，唤醒的线程获取锁

ObjectMonitor同步机制是jvm对操作系统互斥锁(Mutex Lock)的管理，期间都会从用户空间转入内核空间操作。也就是说Synchronized实现锁是重量级锁。为了减少对ObjectMonitor对象的访问，减少线程上下文切换。做了优化，锁自旋，轻量级锁，偏向锁

###### 锁自旋
线程的阻塞和唤醒需要cpu从用户态进入内核态，频繁的阻塞和唤醒对cpu来说是一件负担很重的事情，影响并发性能。自旋锁就是让线程等待一段时间，忙等待(循环获取锁 比如CAS操作)，看持有锁的线程是否很快释放锁。

> 自旋锁适用多核处理器，竞争不激烈(短暂临界 锁定时间短)。如果竞争激烈长时间获取不到锁，造成cpu空转

###### 轻量级锁

同一代码块虽然有多个不同的线程去执行，但是这些线程交替请求这把锁，也就是不存在锁竞争的情况。这种情况锁会保持在轻量级锁的状态，避免重量级锁的阻塞和唤醒。通过锁自旋避免真正的加锁

> 线程交替执行同步代码块，如果存在同一时间访问同一锁的情况就会导致轻量级锁膨胀为重量级锁

###### 偏向锁

当一个现场首次申请锁时，会被标记为偏向锁(记录锁状态 线程id)，当这个线程再次请求该锁时，就可以直接获取到该锁。但如果有其他线程也需要获取该锁则偏向锁就失效了，升级成轻量级锁。偏向锁主要用于同一线程访问同步代码块，略过了轻量级锁和重量级锁加锁释放锁过程。

> 适合无竞争的情况，即同一代码块被同一线程频繁访问



##### ReentrantLock

加锁(lock和解锁(unlock)都是手动完成手动完成(Synchronized离开代码块后自动解锁)

可重入锁：同一线程可多次获取同一把锁

公平(FairSync)：

```kotlin
final boolean initialTryLock() {
    Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (!hasQueuedThreads() && compareAndSetState(0, 1)) { // 等待队列没有
            setExclusiveOwnerThread(current);
            return true; // 持有锁
        }
    } else if (getExclusiveOwnerThread() == current) { // 重入
        if (++c < 0) // overflow  // 重入时state也在 ++
            throw new Error("Maximum lock count exceeded");
        setState(c);
        return true; // 持有锁
    }
    return false; // 获取锁失败，调用AQS下的请求方法(acquire)进行排队 阻塞
}
```

非公平(NonfairSync)：

```kotlin
final boolean initialTryLock() {
    Thread current = Thread.currentThread();
    if (compareAndSetState(0, 1)) { // 没有等待队列判断 直接CAS
        setExclusiveOwnerThread(current);
        return true;
    } else if (getExclusiveOwnerThread() == current) { // 重入
        int c = getState() + 1;
        if (c < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(c);
        return true;
    } else
        return false;
}
```



###### AQS(Abstract Queued Synchronizer)

提供构建锁同步框架。AQS 主要用于实现像 ReentrantLock、Semaphore、CountDownLatch 等同步器

**状态(int state)**：一个int值状态，提供相应的get、set、CAS操作。ReentrantLock中state用于记录同一个线程持有锁的次数(重入)，unlock时state-1，当state为0时释放资 唤醒等待线程。所以ReentrantLock的lock和unlock一定是成对出现。CountDownLatch中state用于倒数的计数。

**队列(双向链表)**：每个请求锁的线程，在获取锁失败时，自旋继续尝试获取锁，如果获取锁还是失败。会放入这个队列，阻塞线程，直到超时、线程中断、或者被唤醒。

**独占和共享模式**：独占模式，一个资源在同一时间只能被一个线程持有，共享模式，资源在同一时间可以被多个线程持有。
在AQS中并没有独占共享的意思，只是分成两类并且提供对应的获取锁(**`tryAcquire`**)释放锁(**`tryRelease`**)方法。真正体现独占共享是在子类的实现。在ReentrantLock(独享模式)中，tryAcquire只要有他线程持有锁(state不等于0)就返回false，acquire方法进入队列阻塞排队。在CountDownLatch(中)，tryAcquireShared只要倒计数没有结束(state不等于0)，acquire方法进入队列阻塞排队。

CAS(Compare And Swap)操作

比较某个内存地址是否是预期值，是则替换该值否则不做更改。依赖操作系统处理器指令来实现院子操作



##### ReentrantReadWriteLock

读锁共享，写锁独享。同一个sync(AQS), state高16为记录共享锁数，低16位记录独享锁数



#### 线程池

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          RejectedExecutionHandler handler) 
```

corePoolSize:核心线程数

maximumPoolSize：最大线程数。线程池最大能执行的线程数

keepAliveTime：线程池中线程空闲时间，当空闲时间达到这个值，线程会被销毁直到线程数剩下corePoolSize

workQueue：任务等待队列。当任务达到corePoolSize时，新的任务将会被保存到workQueue

handler：当workQueue队列满了并且线程数达到maximumPoolSize，线程池通过该策略处理

```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0)); // 初始RUNNING状态
private static final int COUNT_BITS = Integer.SIZE - 3;  // 高3为线程池状态, 低29位线程运行数
private static final int COUNT_MASK = (1 << COUNT_BITS) - 1;

// runState is stored in the high-order bits
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;

// Packing and unpacking ctl
private static int runStateOf(int c)     { return c & ~COUNT_MASK; }
private static int workerCountOf(int c)  { return c & COUNT_MASK; }
private static int ctlOf(int rs, int wc) { return rs | wc; }
```

##### 线程池状态：

RUNNING：接受新任务并且处理队列中任务

SHUTDOWN：不接受新任务，但是处理队列中任务。  调用shutdown()后，RUNNING -> SHUTDOWN

STOP：不接受新任务，不处理队列中任务，中断正在处理的任务。调用shutdownNow()后，(RUNNING or SHUTDOWN) -> STOP

TIDYING：所有任务已终止，workerCount 为零，线程转换到状态 TIDYING，运行terminated()钩子方法

TERMINATED：terminated()方法运行结束

##### 执行流程

调用 execute 或者 submit提交任务到线程池时

1. 线程池中运行的线程数量还没有达到 corePoolSize 大小时，线程池会创建一个新线程执行提交的任务，无论之前创建的线程是否处于空闲状态。

2. 线程池中运行的线程数量已经达到 corePoolSize 大小时，线程池会把任务加入到等待队列中，直到某一个线程空闲了，线程池会根据我们设置的等待队列规则(是否运行核心线程超时，超时时间)，从队列中取出一个新的任务执行。

   > ```jade
   > // 从队列中取任务
   > private Runnable getTask() {
   > 	  ...
   >     for (;;) {
   >         int c = ctl.get();
   > 				...
   >         int wc = workerCountOf(c);
   > 
   >         // 允许核心线程超时 || 运行线程数大于核心线程数  ==> 线程退出
   >         boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
   >         
   > 				if ((wc > maximumPoolSize || (timed && timedOut))
   >                 && (wc > 1 || workQueue.isEmpty())) {
   >                 if (compareAndDecrementWorkerCount(c))
   >                     return null; // 线程退出
   >                 continue;
   >         }
   >         
   >         try {
   >             Runnable r = timed ?
   >                 workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) : // keepAliveTime时间线程退出
   >                 workQueue.take(); // 阻塞等待任务添加
   >             if (r != null)
   >                 return r;
   >             timedOut = true;
   >         } catch (InterruptedException retry) {
   >             timedOut = false;
   >         }
   >     }
   > }
   > ```

3. 线程数大于 corePoolSize 数量但是还没有达到最大线程数 maximumPoolSize，并且等待队列已满，则线程池会创建新的线程来执行任务。

4. 如果提交的任务，无法被核心线程直接执行，又无法加入等待队列，又无法创建“非核心线程”直接执行，线程池将根据拒绝处理器定义的策略处理这个任务。

*线程在控制中断时用了AQS*



##### 线程状态：

NEW：线程刚创建，还没start()

RUNNABLE：运行状态，已经调用start()方法可能正在运行也可能在等待cpu调度

BLOCKED：线程正在等待监视器锁（monitor lock），在进入同步代码块或同步方法时被阻塞。

> ```kotlin
> @Synchronized
> fun synMethod() { } // 同步方法
> 
> val objectLock = Object();
> synchronized(objectLock) { } // 同步代码块
> ```

WAITING：等待状态，等待其他线程执行特点的操作

> Object.wait，Thread.join，LockSupport.park 不带超时

TIMED_WAITING： 等待状态，等待其他线程执行特定操作。但是有时间限制

> Thread.sleep(1)，Object.wait(1)，Thread.join(1)，LockSupport.parkNanos(1) 带时间限制

TERMINATED：终止状态，线程执行完成或者由于异常退出

线程中断：当线程处于WAITING或者TIME_WAITING时 响应中断并且抛出InterruptedException异常。当线程处于RUNNABLE状态时并不会改变状态 只是设置中断标志为。Thread.interrupted()检查并且清除中断状态，Thread.isInterrupted()检查但不清除中断状态








[^1]: 深入理解Java虚拟机-周志明
