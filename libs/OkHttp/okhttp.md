okHttpClient.newCall() -> RealCall
```kotlin

class RealCall(
  val client: OkHttpClient,
  /** The application's original request unadulterated by redirects or auth headers. */
  val originalRequest: Request,
  val forWebSocket: Boolean
) : Call
```

Dispatcher.runningSyncCalls.add(call)

client.dispatcher: Dispatcher

RealCall.getResponseWithInterceptorChain

```
interceptors += client.interceptor // 先添加自定义的拦截器

interceptors += client.interceptors
interceptors += RetryAndFollowUpInterceptor(client)
interceptors += BridgeInterceptor(client.cookieJar)
interceptors += CacheInterceptor(client.cache)
interceptors += ConnectInterceptor
if (!forWebSocket) {
  interceptors += client.networkInterceptors
}
interceptors += CallServerInterceptor(forWebSocket)
```

interceptors的数量6 + n

生成RealInterceptorChain

```kotlin
// index初始值0
val next = copy(index = index + 1, request = request)
val interceptor = interceptors[index] // 取得当前需要处理的interceptor
```

## request流程

RetryAndFollowUpInterceptor {
  intercept() {
    RealCall.enterNetworkInterceptorExchange // 给RealCall.exchangeFinder赋值
    // 如果在执行到RetryAndFollowUpInterceptor.intercept时调用了call.cancel,将会抛出异常
    ...
    followUpRequest() {
      // 重试处理各种异常状态码
    }
  }
}

401 
407

308 307 300 301 302 303
408超时
503
421

BridgeInterceptor {
  intercept() {
    // 给request添加headers
    // Content-Type Content-Length Transfer-Encoding Host Connection Accept-Encoding
    // Cookie User-Agent
  }
}

CacheInterceptor {

}

ConnectInterceptor {
  Intercept {
    // 初始化RealCall的exchange
    // RealCall.interceptorScopedExchange = result
    // RealCall.exchange = result
    val exchange = realChain.call.initExchange(chain)
    // 初始化chain.exchange
    val connectedChain = realChain.copy(exchange = exchange)
  }
}

response流程

CallServerInterceptor {

}

## 连接建立过程

ExchangeFinder.findHealthConnection ->
ExchangeFinder.findConnection ->
val newConnection = RealConnection(connectionPool, route) ->
newConnection.connect(
    connectTimeout,
    readTimeout,
    writeTimeout,
    pingIntervalMillis,
    connectionRetryEnabled,
    call,
    eventListener
)

RouteSelector.address 通过RealCall.createAddress创建
在Builder中创建
    internal var hostnameVerifier: HostnameVerifier = OkHostnameVerifier
    internal var certificatePinner: CertificatePinner = CertificatePinner.DEFAULT

OkHttpClient.init {
    // 默认的sslSocketFactory 
    this.x509TrustManager = Platform.get().platformTrustManager()
    this.sslSocketFactoryOrNull = Platform.get().newSslSocketFactory(x509TrustManager!!)
}

dns
// Builder中创建
// 默认使用java原生
// InetAddress.getAllByName(hostname).toList()
internal var dns: Dns = Dns.SYSTEM 

RouteSelector实例化时会执行resetNextProxy

## 响应流程
