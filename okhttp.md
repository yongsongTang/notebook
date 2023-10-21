# Okhttp

## 责任链模

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


![整个链递归调用经过](/home/tys-matebook/Downloads/okhttp_link.png)

1. 创建RealInterceptorChain对象并调用该对象proceed方法。也就是调用第0个位置，拦截器的intercept方法。

2. 在各个拦截器的intercept方法中又调用RealInterceptorChain的proceed方法，相当于有回到了1，只是此时的
   index是0+1。如此循环往复，直到遇到CallServerInterceptor拦截器。

3. CallServerInterceptor拦截器，真正发起网络请求，获取结果，不在调用proceed方法。整个递归循环就此介绍，将网络请求结果返回到上一个拦截器。

   > 请求(request): 0 -> 1 -> 2 -> 3 ..... CallServerInterceptor
   >
   > 响应(Response): CallServerInterceptor  ....  -&gt; 3 -&gt; 2 -&gt; 1 -&gt; 0  




## 链接池

一个链接只发起一次http请求，请求结束后关闭链接。这样逻辑简单，也不会跟服务器建立多的链接，但是每次发起请求都将要经历建立链接，断开链接的过程。能不能建立链接后将链接保存起来，下次发起请求时继续使用这个链接。怎么才能呢做到既复用链接，也不会建立太多链接浪费资源。

http/2协议支持一个连接同时发起多路http请求。okhttp中一个http/2连接可以并发128路请求。

### 复用链接



### 清理链接

正常思维是建立Lru缓存，超过一定数量后将较少使用的链接关闭。okhttp的做法是，每个链接中都有一个弱引用集合(mutableListOf<Reference<RealCall>>())，其中保存着所有使用该链接发起的请求。当集合为空时说明该链接空闲，否则说明该链接有使用。