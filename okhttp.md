# Okhttp

## 责任链

当我发起一次网络请求时，不论同步还是异步最终都会调用到RealCall中getResponseWithInterceptorChain方法，该方法按照一定顺序依次将各种拦截器添加到interceptors，CallServerInterceptor是最后一个添加的。需要注意的是，除CallServerInterceptor外的所有拦截器，在intercept方法中都调用了chain.proceed方法。其中chain类型是RealInterceptorChain，index在不断地指向下一个拦截器。

```kotlin
internal fun getResponseWithInterceptorChain(): Response {
  // Build a full stack of interceptors.
  val interceptors = mutableListOf<Interceptor>()
  interceptors += client.interceptors
  interceptors += RetryAndFollowUpInterceptor(client)
  interceptors += BridgeInterceptor(client.cookieJar)
  interceptors += CacheInterceptor(client.cache)
  interceptors += ConnectInterceptor
  if (!forWebSocket) {
    interceptors += client.networkInterceptors
  }
  interceptors += CallServerInterceptor(forWebSocket)
  ...
  val chain = RealInterceptorChain(index = 0, request = originalRequest)
  val response = chain.proceed(originalRequest)
  return response
}
```

RealInterceptorChain中的proceed方法

```kotlin
override fun proceed(request: Request): Response {
  ...
  // Call the next interceptor in the chain.
  val next = copy(index = index + 1, request = request)
  val interceptor = interceptors[index]
  //当前拦截器的intercept，参数还是RealInterceptorChain类型
  //RealInterceptorChain.proceed() <===> interceptor.intercept(RealInterceptorChain类型,next)
  val response = interceptor.intercept(next) 
  return response
}
```


![链递归调用经过](https://raw.githubusercontent.com/yongsongTang/PicGo/main/img/okhttp_link.svg)

1. 创建RealInterceptorChain对象并调用该对象proceed方法。也就是调用第0个位置，拦截器的intercept方法。

2. 在各个拦截器的intercept方法中又调用RealInterceptorChain的proceed方法，相当于有回到了1，只是此时的
   index是0+1。如此循环往复，直到遇到CallServerInterceptor拦截器。

3. CallServerInterceptor拦截器，真正发起网络请求，获取结果，不在调用proceed方法。整个递归循环就此介绍，将网络请求结果返回到上一个拦截器。

   > 请求(request): 0 -> 1 -> 2 -> 3 ..... CallServerInterceptor
   >
   > 响应(Response): CallServerInterceptor  ....  -&gt; 3 -&gt; 2 -&gt; 1 -&gt; 0  

### 思考过程

1. 一个方法递归循环，压栈时获取参数，出栈时获取结果

2. 两个方法递归循环，a方法中调用方法b，b方法中调用方法a。如此反复，压栈时获取参数，出栈时获取结果。

   > 方法b：放出去由外部实现，但是实现中必须得调用方法a，否则循环会结束。
   >
   > 方法a：固定为找到下一个方法b并调用

3. 如果a，b不是方法，是类。b类似Intercept  a类似Chain

   

### 代码描述

```kotlin
data class Request_(var url: String)
data class Response_(var ret: String)

interface Interceptor_ {
    interface Chain {
        fun request(): Request_

        fun process(request: Request_): Response_
    }
    fun intercept(chain: Chain): Response_
}

class RealInterceptorChain(
    private val interceptors: List<Interceptor_>,
    private val index: Int,
    private val request: Request_
) : Interceptor_.Chain {

    override fun request(): Request_ = this.request

    override fun process(request: Request_): Response_ {
        // 调用当前位置Interceptor.intercept()  把下一个Interceptor位置传下去
        val interceptor = interceptors[index]
        val c = RealInterceptorChain(interceptors, index + 1, request());
        return interceptor.intercept(c)
    }
}

class MyInterceptor(private val index: Int) : Interceptor_ {
    override fun intercept(chain: Interceptor_.Chain): Response_ {
        println("$index request:${chain.request().url}")
        chain.request().url += "_${index}" // 修改参数(request)

        val response_ = chain.process(chain.request()) // 触发下一拦截器方法
        println("$index response:${response_.ret}")
        response_.ret += "-${index}" // 修改结果(response)
        return response_
    }
}

class MyInterceptorEnd : Interceptor_ {
    override fun intercept(chain: Interceptor_.Chain): Response_ {
        println("最后一个 发起请求获取结果 ${chain.request().url}")
        // 真正发起网络请求获取结果,不在继续向下(不在调用chain.process方法)
        return Response_(ret = "result")
    }
}
```

```kotlin
        val interceptors = mutableListOf<Interceptor_>()
        for (i in 0..2) {
            interceptors.add(i, MyInterceptor(i));
        }
        interceptors += MyInterceptorEnd()

        val request = Request_(url = "https://a")
        val realInterceptorChain = RealInterceptorChain(interceptors, 0, request)
        val response = realInterceptorChain.process(request)
        println("最终结果 ${response.ret}")
        
//        0 request:https://a
//        1 request:https://a_0
//        2 request:https://a_0_1
//        最后一个 发起请求获取结果 https://a_0_1_2
//        2 response:result
//        1 response:result-2
//        0 response:result-2-1
//        最终结果 result-2-1-0
```




## 链接池

一个链接只发起一次http请求，请求结束后关闭链接。这样逻辑简单，也不会跟服务器建立多的链接，但是每次发起请求都将要经历建立链接，断开链接的过程。能不能建立链接后将链接保存起来，下次发起请求时继续使用这个链接。怎么才能呢做到既复用链接，也不会建立太多链接浪费资源。okhttp的做法是，每个链接中都有一个弱引用集合mutableListOf<Reference<RealCall>>()，其中保存着所有使用该链接发起的请求。当集合为空时说明该链接空闲，否则说明该链接有使用，当满足一点条件时空闲链接将会从链接池中移除，关闭链接。类似引用计数

http/2协议支持一个连接同时发起多路http请求。okhttp中一个http/2连接可以并发128路请求。

### 链接复用

在发起请求后，会先去找连接池中找有没有符合条件的链接，如果有则当前Call使用该链接，同时将当前Call加入到弱引用集合，说明该链接多了一个使用。如果没有则新建链接，当前Call使用新建的链接，并将新建链接放入连接池，同时将当前Call加入到该链接的弱引用集合中。

```kotlin
private fun findConnection(
  connectTimeout: Int,
  readTimeout: Int,
  writeTimeout: Int,
  pingIntervalMillis: Int,
  connectionRetryEnabled: Boolean
): RealConnection {
  if (call.isCanceled()) throw IOException("Canceled")

  // Attempt to reuse the connection from the call.
  val callConnection = call.connection // This may be mutated by releaseConnectionNoEvents()!
    //call不是新建的吗，怎么会存在connection，还要复用呢？？？
  if (callConnection != null) {
	...
  }
  // Attempt to get a connection from the pool. 
  if (connectionPool.callAcquirePooledConnection(address, call, null, false)) {
    val result = call.connection!!
    eventListener.connectionAcquired(call, result)
      //连接池中找到符合要求的
    return result
  }
	...
  // Connect. Tell the call about the connecting call so async cancels work.
  val newConnection = RealConnection(connectionPool, route)
  call.connectionToCancel = newConnection
  synchronized(newConnection) {
      //新建链接放入连接池
    connectionPool.put(newConnection)
      //新建链接的弱引用集合中加入该call
    call.acquireConnectionNoEvents(newConnection)
  }
  eventListener.connectionAcquired(call, newConnection)
  return newConnection
}
```

符合条件就是说有一链接是可以用于给定地址的，其判断过程下方法:

```kotlin
// RealConnection.kt
internal fun isEligible(address: Address, routes: List<Route>?): Boolean {
  // If this connection is not accepting new exchanges, we're done.
  //calls就是那个弱引用队列
  //该链接使用数 >= 允许最大分配数 || 不允许在进行数据交换
  //allocationLimit：
  // http_1_1协议时值为1，一个连接同时只能发起一个http请求
  // http/2xi协议时值为128，一个连接最多并发128个http请求
  //noNewExchanges:true表示该链接不允许在进行数据交互,在读、写出错时会设置成true  
  if (calls.size >= allocationLimit || noNewExchanges) return false

  // 代理 端口
  if (!this.route.address.equalsNonHost(address)) return false
  if (address.url.host == this.route().address.url.host) {
     //匹配上了
    return true // This connection is a perfect match.
  }
  //url都不一样了，为啥下面还有代码,还能匹配？？
  //不同url可能是相同的ip地址，一台物理机上跑着各种服务，每个服务有自己的域名。一个http/2协议链接，可以并发处理n个http请求。

  // our connection coalescing requirements are met. See also:
  // https://hpbn.co/optimizing-application-delivery/#eliminate-domain-sharding
  // https://daniel.haxx.se/blog/2016/08/18/http2-connection-coalescing/

  // 1. This connection must be HTTP/2.
  if (http2Connection == null) return false

  // 2. The routes must share an IP address.
  if (routes == null || !routeMatchesAny(routes)) return false

  // 3. This connection's server certificate's must cover the new host.
  if (address.hostnameVerifier !== OkHostnameVerifier) return false
  if (!supportsUrl(address.url)) return false

  // 4. Certificate pinning must match the host.
  try {
    address.certificatePinner!!.check(address.url.host, handshake()!!.peerCertificates)
  } catch (_: SSLPeerUnverifiedException) {
    return false
  }

  return true // The caller's address can be carried by this connection.
}
```



### 链接清理

每一次请求结束后，不管正常还是异常结束都会将Call从该链接的弱引用集合中移除,在移除后发现集合为空，说明还连接已经为空此时会触发清理连接的任务cleanupTask。在创建新链接添加到链接池时，也会触发清理连接任务。

```kotlin
//ReallCall.kt
internal fun getResponseWithInterceptorChain(): Response {
  // Build a full stack of interceptors.
	...
  var calledNoMoreExchanges = false
  try {
    val response = chain.proceed(originalRequest)
 	...
    return response
  } catch (e: IOException) {
    calledNoMoreExchanges = true
    throw noMoreExchanges(e) as Throwable //从弱引用队列中移除Call(请求和响应body已经关闭情况下)
  } finally {
    if (!calledNoMoreExchanges) {
      noMoreExchanges(null) //从弱引用队列中移除Call(请求和响应body已经关闭情况下)
    }
  }
}
```

清理连接任务会遍历所有链接，找到空闲时间最长的是那个链接longestIdleConnection，最长空闲了多长时间longestIdleDurationNs，有多少链接处于空闲idleConnectionCount。然后跟设定阈值比较是否满足移除条件，如果满足条件则从连接池中移除该链接,关闭链接。如果不满足条件，则计算出应该移除的时间点在延迟后继续触发。代码如下:

```kotlin
//RealConnectionPool.kt
fun cleanup(now: Long): Long {
  var inUseConnectionCount = 0 //处于空闲的连接数
  var idleConnectionCount = 0
  var longestIdleConnection: RealConnection? = null //处于空闲状态下时间最长的链接
  var longestIdleDurationNs = Long.MIN_VALUE //处于空闲状态下最长时间

  // Find either a connection to evict, or the time that the next eviction is due.
  for (connection in connections) {
   ...  
   //遍历链接找出，有多少空闲 空闲最久的是谁 空闲最久是多长时间 
  }

  when {
      //空闲时间>=5min || 空闲数>5
    longestIdleDurationNs >= this.keepAliveDurationNs //默认5min
        || idleConnectionCount > this.maxIdleConnections -> { //默认5个
	   ...	
	  //longestIdleConnection 从连接池中移除 关闭链接 	
      // Clean up again immediately.
      return 0L  //不延迟 立即执行下一次
    }
    idleConnectionCount > 0 -> {
      //空闲数>0  剩余允许空闲的时间 = keepAliveDurationNs - longestIdleDurationNs
      return keepAliveDurationNs - longestIdleDurationNs //到点了来清理
    }
    inUseConnectionCount > 0 -> {
      return keepAliveDurationNs //全部在使用中 延迟5min在来看
    }
    else -> {
      // No connections, idle or in use.
      return -1 //没有链接 不用执行下一次
    }
  }
}
```



