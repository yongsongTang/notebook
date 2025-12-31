Android 架构

#### 特点

移动设备资源有限，应用组件(Activity Sevices BroadCastReceiver ContentProvider)，进程可能随时会被终止以便为新进程腾空间。

> 组件可以不按顺序单独启动，组件之间不相互依赖 



#### 原则

允许应用**扩展/收缩**提升应用**稳健**且**方便测试**和**阅读**。架构定义应用各部分的界线以及各部分应当承担的责任

##### 1. [关注点分离](https://zh.wikipedia.org/wiki/%E5%85%B3%E6%B3%A8%E7%82%B9%E5%88%86%E7%A6%BB)

   > 将程序分割成不同部分,各部分有各自的关注焦点

##### 2. 数据模型驱动界面

   >应用数据独立于应用界面和其他组件，便于测试，更稳定可靠

##### 3. 单一数据源

   >只有某一个数据源可以修改或转换该数据。所有更改集中到一处，保护数据防止篡改，便于跟踪发现bug。
   >
   >可以是ViewModel、界面、也可以是单独操作的类

##### 4. 单向数据流

   > 状态朝一个方向流动，修改数据的事件朝反方向流动
   >
   > e.用户事件(按钮)从界面流向单一数据源，在数据源中修改并且以不可变更方式公开。应用数据通常从数据源流向界面



#### 架构

##### 界面层

屏幕上显示数据，无论是用户互动（点击按钮）还是外部输入（网络响应,界面都应该随之变化，反映这些变化。

1. 使用应用数据，将其转换为界面可以轻松呈现的数据 
2. 使用界面可呈现的数据，转换为界面元素
3. 界面元素响应用户输入事件

##### 数据层

应用数据和业务逻辑



#### MVVM

Model：与数据源的交互，数据的获取、存储和操作。以及和数据相关的业务逻辑

View：呈现数据和捕获用户交互

ViewModel：作为 View 和 Model 之间的桥梁，处理界面交互和Ui相关逻辑，调用Modle层方法更新数据



用户在View上的操作传递给ViewModel，ViewMode处理用户输入，执行逻辑更新Model。Model数据变化通过ViewModel反映到View上

ViewModel 从 Model 获取数据，并通过 LiveData 暴露给 View，View 观察 ViewModel 中的 LiveData，当数据发生变化时自动更新 UI

View	--->	ViewModel	<--->	Model

``` kotlin
// Model 关于类的操作(User)
data class User(val id: Int, val name: String, val email: String)

interface UserRepository {
    fun getUser(userId: Int): User
}

class UserRepositoryImpl : UserRepository {
    override fun getUser(userId: Int): User {
        // 模拟从数据源获取用户数据
        return User(userId, "John Doe", "john.doe@example.com")
    }
}

// ViewModel 链接View和Model
class UserViewModel(private val userRepository: UserRepository) : ViewModel() {
    private val _user = MutableLiveData<User>()
    val user: LiveData<User> get() = _user

    fun loadUser(userId: Int) {
        _user.value = userRepository.getUser(userId)
    }
}

object UserViewModelFactor : ViewModelProvider.Factory {
    override fun <T : ViewModel> create(modelClass: Class<T>): T {
        return UserViewModel( UserRepositoryImpl() ) as T
    }
}

// View
class MainActivity : ComponentActivity() {
    companion object {
        private const val TAG = "MainActivity"
    }

    private val viewModel: UserViewModel by viewModels { UserViewModelFactor }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        window.decorView.setBackgroundColor(Color.GRAY)
        setContentView(R.layout.main_activity_layout)

        viewModel.user.observe(this) { // Activity生命周期和观察者绑定在一起，onDestory时移除观察者
            Log.d(TAG, "onCreate() called $it")
        }
        viewModel.loadUser(1)
    }
}
```



#### MVP

Model：数据的获取、存储操作以及数据有关业务

View：显示数据和捕获用户交互，

Presenter：View和Model的中介，从而model层获取数据并传递给view

View 通过接口与 Presenter 交互。View 接口定义了更新 UI 的方法，Presenter 接口定义了处理用户交互的方法。

Presenter 直接调用 Model 的方法获取或保存数据

View  <-->  Presenter  <-->  Model



```kotlin
// model
data class User(val id: Int, val name: String, val email: String)

interface UserRepository {
    fun getUser(userId: Int): User
}

class UserRepositoryImpl : UserRepository {
    override fun getUser(userId: Int): User {
        // 模拟从数据源获取用户数据
        return User(userId, "John Doe.", "john.doe@example.com")
    }
}

// View
interface IUserView {
    fun show_xx(user: User)
}

class MainActivity : ComponentActivity(), IUserView {
    private lateinit var presenterUser: PresenterUser

    companion object {
        private const val TAG = "MainActivity"
    }
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.main_activity_layout)
        presenterUser = PresenterUser(this, UserRepositoryImpl())
        presenterUser.getUser(1)
    }

    override fun show_xx(user: User) {
        Log.d(TAG, "show_xx() called with: user = $user")
    }
}

// Presenter
class PresenterUser(val view: IUserView, val userRepository: UserRepository) { // 通过接口和界面交互
    fun getUser(id: Int) {
        val user = userRepository.getUser(id)
        view.show_xx(user)
    }
}

```

MVP：通过接口和界面交互，直观容易理解，但是产生更多的样板代码(接口定义)，需要自己手动管理生命周期

MVVM：通过LiveData和界面交互自动更新ui，同时生命周期和界面绑定，减少内存泄漏风险。依赖LiveData



#### 设计模式



##### 创建型：

工厂模式：

抽象工厂模式：

单例模式：

原型模式：通过复制现有对象来创建新的对象



##### 结构型：

适配器模式：将一个类的接口转换成客户希望的另外一个接口

桥接模式：抽象部分与实现部分分离，

组合模式：将多个对象组合在一起，简化客户端对组合对象和单个对象的处理。增加设计复杂违反单一职责

装饰模式：动态给对象添加职责

外观模式：定义统一高层接口，简化复杂系统使用，隐藏系统的复杂性

享元模式：共享细粒度对象

代理模式：为其他对象提供一种代理，以控制对这个对象的访问



##### 行为型：

责任链模式：多个对象有机会处理同一请求，将这些对象连成一条链。沿着这条链传递请求

命令模式：一个请求封装成一个对象，请求参数化

解释器模式：解析和执行特定语言的表达式

迭代器模式：提供一种方法顺序访问聚合对象中的各个元素

中介模式：中介对象封装一系列对象的交互，各个对象不需要显示的相互引用

备忘录模式：不破坏封装的情况下，捕获和恢复对象内部状态

观察者模式：一对多的关系，对象状态改变，依赖它的对象得到通知

状态模式：对象内部状态改变时，其行为也得到改变

策略模式：定义一系列策略，将每个封装起来，使他们可以相互替换。 

模板模式：定义一个操作中的骨架，将一些操作延迟到子类中。子类不改变结构既可重定义某些特定步骤

访问者模式：不改变访问对象类的前提下定义新的操作，数据结构稳定但是经常需要定义新的操作。



