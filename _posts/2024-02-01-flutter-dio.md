---
layout: post
title:  "Flutter Dio网络框架分析"
categories: [ Jekyll ]
image: assets/images/demo1.jpg
---
# 前言
对新技术保持敏锐是一个程序猿最基本的素养，其中Flutter是新技术中的佼佼者，那么对于晦涩难懂的框架源码我们该如何学习呢？

以Flutter中的Dio为例，如果想要了解Dio的源码设计，我们从哪里开始分析？直接阅读源码？

相信大多数同学阅读源码都存在如下两个问题：
* 该从哪处下手
* 学了忘，忘了学

造成这样的原因是因为你没有一个结构化思维，没有理解网络框架的本质。

如果你仍处于以上这种状态，那么接着往下看，这篇文章将非常适合你，我将带领大家对Dio框架进行抽丝剥茧，你可以学到的不仅是框架的设计，同样也是网络框架的基本设计规范，相信大家掌握以后不管是Android还是iOS亦或是Web的网络框架,你都能用同样的系统思维方式去分析。

# 导读
> 文章基于Dio 4.0.6进行分析，根据通用的网络请求流程来一步一步拆解Dio框架，不拘泥于源码解析，偏重于系统性分析：
>
> * 第一章 ***Dio请求流程介绍*** 系统的向读者介绍通用的的网络请求流程和Dio的架构设计，让读者对其有一个整体的结构化思维。
> * 第二章 ***输入数据*** 介绍Dio框架中输入数据时涉及的基本类。
> * 第三章 ***数据处理*** 基于第二章中输入的数据分析Dio中的数据处理流程。
> * 第四章 ***输出数据*** 简述Dio将数据输出给调用者的过程。
> * 第五、六章 基于前面的内容做一个总结。

# 目录

[1. Dio请求流程介绍]()     
[2. 输入数据]()
* [2.1 Api入口]()

[3. 数据处理]()
* [3.1 数据预处理]()
* [3.2 Uri解析]()
* [3.3 DNS解析]()
* [3.4 建立TCP连接]()
* [3.5 IP传输数据]()
* [3.6 服务器接收并处理数据]()
* [3.7 客户端接收并处理返回数据]()

[4. 数据输出]()
* [将数据返回给调用者]()

[5. 其他]()
[6. 总结]()

# 1.Dio请求流程介绍

> 网络框架的请求流程和计算机的运行原理本质上是一样的，主要分为三大流程：
>* 输入数据
>* 数据处理
>* 输出数据
>
>输入数据就像是计算机通过I/O接口输入指令及数据，数据处理好比CPU和内存对数据进行存储运算，输出数据则是通过I/O接口输出到电脑屏幕上。


**网络框架图**

![网络框架.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9f4ebce42fcf45a09aeebd9a1711f326~tplv-k3u1fbpfcp-watermark.image?)


**Dio框架图**

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/286adecf982647e0b66b0c124a7831ae~tplv-k3u1fbpfcp-watermark.image?)

Dio中 **输入数据** 通过App初始化配置和调用get或post方法传入的参数的组合。

**数据处理** 的过程相对复杂，但本质上就是固定步骤的指令操作数据的过程。

**输出数据**是将数据处理完成后返回给调用者。

下三章我们将分析下这三大流程的具体逻辑。

# 2.输入数据
### Api入口

> Dio框架输入数据主要有下面两个入口：  
1.初始化配置  
2.业务调用

**初始化配置** 相当于数据预加载的过程，通常在App启动时进行，其本质是为了在 **业务调用** 时尽量简单，性能更佳。

Dio中 **初始化配置** 示例：
```
final dio = Dio(); // 初始化默认的配置

void configureDio() {
  // 通过BaseOptions设置默认配置
  dio.options.baseUrl = 'https://api.pub.dev';
  dio.options.connectTimeout = Duration(seconds: 5);
  dio.options.receiveTimeout = Duration(seconds: 3);
  
  
  //添加拦截器
  //自定义拦截器,请求、返回、异常三种拦截器
  dio.interceptors.add(RequestInterceptor());
  dio.interceptors.add(ErrorInterceptor());
  dio.interceptors.add(ResponceInterceptor());
  //日志记录
  dio.interceptors.add(LogInterceptor(responseBody: true));

  
 // 设置代理
 (dio.httpClientAdapter as DefaultHttpClientAdapter).onHttpClientCreate = (client) {
  client.badCertificateCallback = (X509Certificate cert, String host, int port) =>     true;
   // config the http client
   client.findProxy = (uri) {
    //proxy all request to localhost:8888
    return "PROXY $proxy";
    };
  // you can also create a new HttpClient to dio
  // return new HttpClient();
  };

  // 通过BaseOptions构造函数设置
  final options = BaseOptions(
    baseUrl: 'https://api.pub.dev',
    connectTimeout: Duration(seconds: 5),
    receiveTimeout: Duration(seconds: 3),
  );
  final anotherDio = Dio(options);
}
```
**业务调用** 示例：
```

发送一个get请求

void request() async {
    Response response;
    //get方法的两种方式
    response = await dio.get('/test?id=12&name=dio');
    print(response.data.toString());
    // 另外一种
    response = await dio.get(
        '/test',
        queryParameters: {'id': 12, 'name': 'dio'},
      );
      print(response.data.toString());
    }

    //调用时也可通过Options设置一些基本属性
    Options options = Options(
      //连接服务器超时时间，单位是毫秒.
      sendTimeout: this.mConnectTimeout,
      //响应流上前后两次接受到数据的间隔，单位为毫秒。
      receiveTimeout: this.mReceiveTimeout,
      //Http请求头.
      headers: mHeaders,
      contentType: "application/json",
      //接受四种类型 `json`, `stream`, `plain`, `bytes`. 默认是 `json`,
      responseType: this.mResponseType,
    );

    //发送一个post请求
    response = await dio.post('/test', data: {'id': 12, 'name': 'dio'，
    , options: options});
}   
```
通过代码示例，发现Dio输入数据时主要涉及的类有以下几种：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/521414d4e24f4e18923c2f1a34a84e65~tplv-k3u1fbpfcp-watermark.image?)

### Dio
Dio类作为框架的入口类，主要有如下两个功能：
* 组合了不同类型的配置数据
* 提供get、post等业务调用方法

具体代码如下所示：
```
import 'entry_stub.dart'
// ignore: uri_does_not_exist
    if (dart.library.html) 'entry/dio_for_browser.dart'
// ignore: uri_does_not_exist
    if (dart.library.io) 'entry/dio_for_native.dart';

abstract class Dio {
    //工厂方法创建实现类
  factory Dio([BaseOptions? options]) => createDio(options);
   //下面是基础配置类 
  late BaseOptions options;

  Interceptors get interceptors;

  late HttpClientAdapter httpClientAdapter;

  late Transformer transformer;
  
   //下面是主要调用方法，还包含put、download等方法
  Future<Response<T>> get<T>(
      String path, {
      Map<String, dynamic>? queryParameters,
      Options? options,
      CancelToken? cancelToken,
      ProgressCallback? onReceiveProgress,
   })；

   Future<Response<T>> post<T>( 
        String path, {
        data,
        Map<String, dynamic>? queryParameters,
        Options? options,
        CancelToken? cancelToken,
        ProgressCallback? onSendProgress,
        ProgressCallback? onReceiveProgress,
  })；
}  
```
可以看到，Dio是一个抽象类，createDio是工厂方法，其实现类通过平台区分导入，在移动端中导入的是 ***dio_for_native.dart***，这个文件中 createDio创建的是DioForNative对象。
```
class DioForNative with DioMixin implements Dio {
}
```

**DioForNative 中提供的get、post等方法主要实现在其继承的DioMixin中。**

除了基本的配置数据外，Dio还主要包含get、post方法，在调用这两个方法时传入的参数数据主要包含如下：
* path--请求路径
* params或data--请求参数
* options--请求时传入的配置可以和基础的BaseOption合并，调用时传入的数据会覆盖之前设置的数据
* cancelToken--用于取消当前请求的标记
* onSendProgress和onReceiveProgress是记录一个发送和获取数据的过程，可用于加载进度条，打印日志等

### BaseOption、Option
BaseOption和Option作用类似，都用做基础数据配置，最终会合并成一个对象RequestOptions。

### Interceptor
Interceptor的设计借鉴的是Android OkHttp中插值器形式，使用的是责任链模式，这种优秀的设计使得开发不仅能介入网络请求请求前的数据处理，还可以对返回结果或错误异常进行拦截。
```
class Interceptor {

  // 发送请求前拦截  
  void onRequest(
    RequestOptions options,
    RequestInterceptorHandler handler,
  ) =>
      handler.next(options);
  
  //在结果返回给调用者前拦截
  void onResponse(
    Response response,
    ResponseInterceptorHandler handler,
  ) =>
      handler.next(response);
  
  //发生错误返回给调用者时拦截
  void onError(
    DioError err,
    ErrorInterceptorHandler handler,
  ) =>
      handler.next(err);
}
```

Interceptor在Dio框架中是通过队列保存的，队列是“FIFO”模式，也就是先添加的Interceptor会优先处理，后续添加相同的逻辑会覆盖之前的处理，通常会在onRequest中添加一些headers等操作，onResponse或onError中对结果处理成调用者想要的方式，记住onResponse和onError是互斥的，只有一个会被调用。


### DefaultHttpClientAdapter
DefaultHttpClientAdapter是默认的网络请求配置适配类，这里使用适配器模式将底层网络框架的选择交给开发者，默认实现主要是基于HttpClient（Flutter官方提供的网络库），也可用来设置代理进行抓包，其中核心是fetch方法:
```
class DefaultHttpClientAdapter implements HttpClientAdapter {
  /// [Dio] will create HttpClient when it is needed.
  /// If [onHttpClientCreate] is provided, [Dio] will call
  /// it when a HttpClient created.
  OnHttpClientCreate? onHttpClientCreate;

  HttpClient? _defaultHttpClient;

    @override
    Future<ResponseBody> fetch(
      RequestOptions options,
      Stream<Uint8List>? requestStream,
      Future? cancelFuture,
    ) async {
      if (_closed) {
        throw Exception(
        "Can't establish connection after [HttpClientAdapter] closed!");
      }
      var _httpClient = _configHttpClient(cancelFuture, options.connectTimeout);
      var reqFuture = _httpClient.openUrl(options.method, options.uri);
    }
}  
```
通过 **dio.httpClientAdapter = 开发者自行实现的HttpClientAdapter**开发者可自行配置网络请求部分，需要自行实现fetch方法。

**第二章**我们主要介绍了 **Dio输入数据** 时框架中主要涉及的类，但是对于 **http请求** 来说，这些数据还需要经过一定的 **加工** 才能转换成最终的http协议所需要的数据规范，那么第三章将为我们揭开Dio是如何在保证逻辑严谨的情况下进行 **数据处理** 的。


# 3.数据处理

上一章中输入的数据和Http消息报文的数据格式并不相符，要想变成符合Http协议要求的数据格式，就需要将开发者输入的数据转换成如下图规范的消息数据格式，而网络框架数据处理的本质便是如此。

http的消息结构如下：  
![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6467a74233054c9bbea865ec677ccd76~tplv-k3u1fbpfcp-watermark.image?)

网络框架 **数据处理** 流程本质就是get或post方法调用过程，简单的流程如下：  
1.将BaseOption和Options合并成RequestOptions对象  
2.遍历并调用所有拦截器的onRequst方法，这个方法中重要的参数就是RequestOptions  
3.调用dispatch方法开始传输数据  
4.遍历并调用所有拦截器onResponce方法，这个方法中有一个重要的参数就是Responce  
5.遍历并调用所有拦截器onError方法，这个方法中有一个重要的参数就是DioError  
6.最后将结果Response返回给调用者


### 3.1 数据预处理
**数据预处理**是将输入的数据进行打包、合并的过程其中拦截器中还可以添加、修改请求配置数据。对应到流程中的1，2两个步骤。

Dio get和post实现方法是在DioMixin类中，可以看到不管是get还是post，都会调用到request方法中，实际开发者直接调用request也是可以的，只是需要多一个请求方法**method**的参数。

**将BaseOption和Options合并成RequestOptions对象**

```
@override
Future<Response<T>> request<T>(
  String path, {
  data,
  Map<String, dynamic>? queryParameters,
  CancelToken? cancelToken,
  Options? options,
  ProgressCallback? onSendProgress,
  ProgressCallback? onReceiveProgress,
}) async {
  options ??= Options();
  //将BaseOptions和Options合并成RequestOptions对象
  var requestOptions = options.compose(
    this.options,
    path,
    data: data,
    queryParameters: queryParameters,
    onReceiveProgress: onReceiveProgress,
    onSendProgress: onSendProgress,
    cancelToken: cancelToken,
  );
  requestOptions.onReceiveProgress = onReceiveProgress;
  requestOptions.onSendProgress = onSendProgress;
  requestOptions.cancelToken = cancelToken;

  return fetch<T>(requestOptions);
}
```
request方法首先将初始化时输入的BaseOptions对象和调用时传入的Options对象合并成RequestOptions对象，而RequestOptions将作为后续数据传递过程中的重要媒介。这里Options类有点多，容易对分析框架造成干扰，为避免影响分析，读者请仔细甄别。


**遍历并调用所有拦截器的onRequst方法，这个方法中重要的参数就是RequestOptions**
```
@override
Future<Response<T>> fetch<T>(RequestOptions requestOptions) async {

    // 对初始化中所有拦截器进行遍历并调用onRequest方法
    interceptors.forEach((Interceptor interceptor) {
       var fun = interceptor is QueuedInterceptor
          ? interceptor._handleRequest
          : interceptor.onRequest;
      future = future.then(_requestInterceptorWrapper(fun));
    });
   
    // 请求传输数据部分
    future = future.then(_requestInterceptorWrapper((
      RequestOptions reqOpt,
      RequestInterceptorHandler handler,
    ) {
      requestOptions = reqOpt;
      _dispatchRequest(reqOpt)
          .then((value) => handler.resolve(value, true))
          .catchError((e) {
        handler.reject(e as DioError, true);
      });
    }));
    
    // 对初始化中所有拦截器进行遍历并调用onResponse方法
    interceptors.forEach((Interceptor interceptor) {
      var fun = interceptor is QueuedInterceptor
          ? interceptor._handleResponse
          : interceptor.onResponse;
      future = future.then(_responseInterceptorWrapper(fun));
    });

    // 对初始化中所有拦截器进行遍历并调用onError方法
    interceptors.forEach((Interceptor interceptor) {
      var fun = interceptor is QueuedInterceptor
          ? interceptor._handleError
          : interceptor.onError;
      future = future.catchError(_errorInterceptorWrapper(fun));
    });
}
```
第一步合成RequestOptions对象后，RequestOptions对象就作为第二步拦截器中onRequest方法中的重要参数，每一个拦截器都可以对请求配置数据options进行拦截添加或修改。

**数据预处理**完成后接下来下来就需要知道数据传输的地址了，而弄清楚地址的第一步就是**Uri解析**

### 3.2 Uri解析
输入的数据中有一个URL参数，它是BaseUrl和path的组合，比如 www.baidu.com/home/index?id=1234&name=baidu。

而Uri解析的目的是为了得到如下地址相关的数据：
* http或者https，区分协议类型
* 域名如 www.baidu.com ，用于DNS解析，指定服务器网络地址和主机地址的关键参数
* path如 /home/index，指定获取后台数据的路径
* params如id、name等，针对不同接口传入不同的参数类型，主要是后来用来做逻辑判断

这些数据都保存在一个Uri对象中,所以本质上Uri解析是将Url拆解成一个对象的形式存储，方便后续获取，Dio中关于Uri解析的关键类在**RequestOptions**中，关键方法是的**uri** get方法：

```
class DefaultHttpClientAdapter implements HttpClientAdapter {
  /// [Dio] will create HttpClient when it is needed.
  /// If [onHttpClientCreate] is provided, [Dio] will call
  /// it when a HttpClient created.
  OnHttpClientCreate? onHttpClientCreate;

  HttpClient? _defaultHttpClient;

    @override
    Future<ResponseBody> fetch(
      RequestOptions options,
      Stream<Uint8List>? requestStream,
      Future? cancelFuture,
    ) async {
      if (_closed) {
        throw Exception(
        "Can't establish connection after [HttpClientAdapter] closed!");
      }
      var _httpClient = _configHttpClient(cancelFuture, options.connectTimeout);
      //建立TCP链接，options.uri表示
      var reqFuture = _httpClient.openUrl(options.method, options.uri);
    }
}  

//通过options.uri便可获得所需的Uri数据
/// generate uri
Uri get uri {
  var _url = path;
  if (!_url.startsWith(RegExp(r'https?:'))) {
    _url = baseUrl + _url;
    var s = _url.split(':/');
    if (s.length == 2) {
      _url = s[0] + ':/' + s[1].replaceAll('//', '/');
    }
  }
  var query = Transformer.urlEncodeMap(queryParameters, listFormat);
  if (query.isNotEmpty) {
    _url += (_url.contains('?') ? '&' : '?') + query;
  }
  // Normalize the url.
  return Uri.parse(_url).normalizePath();
}
```

可以看到，通过对输入数据时传入的baseUrl和path参数组合，再经过Uri解析转换，就可以得到消息结构中的部分关键数据了。

但是对于操作系统支持的http协议来说，域名是一个陌生的概念，真正识别的服务器网络地址和主机地址的是ip地址和端口号，那么要想知道域名对应的服务器地址和端口号，就需要进行DNS解析，在下一节将介绍**DNS解析**相关的内容。

### 3.3 DNS解析

通过Uri解析我们得到了域名等相关的信息，但不管是操作系统还是其他网络设备，都不能识别域名而只能识别ip地址和端口号，所以需要一种能通过域名查询ip地址（如192.168.1.1）和端口号（如88）的服务，这个服务就是**DNS解析**。

>**下面我们解答关于域名和ip地址常见的两个疑问**
>
>1，为什么不直接使用ip地址和端口号？  
答：当然可以用，但是因为ip地址不易分辨且难以记忆，比如“wwww.baidu.com“就比较符合正常人的思维习惯。
>
>2，Http协议为什么不直接使用域名？  
答：域名占用的字节数相对ip地址更大，对整体传输的效率会打折扣，且不如ip这种数字地址容易分配。

DNS解析的过程通常在socket中，对于开发者来说并不需要直接接触，但部分优秀的框架也会自行实现DNS解析的逻辑，Dio中进行DNS解析是在HttpClient框架中的socket组件，根据平台（Android/Web）不同实现也不一，这里我们不详细展开，借助一张关于HttpClient的时序图。

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c758c0d3aab04b0bb8cfe920396ea16c~tplv-k3u1fbpfcp-watermark.image?)

在获取到ip地址和端口号以后，接下来就需要通过获取到的信息**建立TCP连接**了。

### 3.4 建立TCP连接

准备工作完成后，通过一系列调用来到DefaultHttpClientAdapter的fetch方法中，其中_httpClient.openUrl就是**建立TCP连接**通道的流程：

```
class DefaultHttpClientAdapter implements HttpClientAdapter {
  /// [Dio] will create HttpClient when it is needed.
  /// If [onHttpClientCreate] is provided, [Dio] will call
  /// it when a HttpClient created.
  OnHttpClientCreate? onHttpClientCreate;

  HttpClient? _defaultHttpClient;

    @override
    Future<ResponseBody> fetch(
      RequestOptions options,
      Stream<Uint8List>? requestStream,
      Future? cancelFuture,
    ) async {
      if (_closed) {
        throw Exception(
        "Can't establish connection after [HttpClientAdapter] closed!");
      }
      var _httpClient = _configHttpClient(cancelFuture, options.connectTimeout);
      var reqFuture = _httpClient.openUrl(options.method, options.uri);
    }
}  
```
下面是时序图：
![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b22a7d9da62247ce8d8549eeda9f2542~tplv-k3u1fbpfcp-watermark.image?)

openUrl方法的参数是请求方式method（get、post等）和**Uri解析**后生成的uri对象，可以看出Dio框架只是收集整理数据，核心逻辑仍然在**HttpClient**中。

```
abstract class HttpClient {
  static const int defaultHttpPort = 80;
  @Deprecated("Use defaultHttpPort instead")
  static const int DEFAULT_HTTP_PORT = defaultHttpPort;

  static const int defaultHttpsPort = 443;
  @Deprecated("Use defaultHttpsPort instead")
  static const int DEFAULT_HTTPS_PORT = defaultHttpsPort;
  
  Future<HttpClientRequest> openUrl(String method, Uri url);
}  
```
HttpClient是一个抽象类，其实现在http_imp.dart文件中_HttpClient类的_openUrl方法。
```
Future<_HttpClientRequest> _openUrl(String method, Uri uri) {
  return _getConnection(uri, uri.host, port, proxyConf, isSecure, profileData)
      .then((_ConnectionInfo info) {
    _HttpClientRequest send(_ConnectionInfo info) {
      profileData?.requestEvent('Connection established');
      return info.connection
          .send(uri, port, method.toUpperCase(), info.proxy, profileData);
    }

    // If the connection was closed before the request was sent, create
    // and use another connection.
    if (info.connection.closed) {
      return _getConnection(
              uri, uri.host, port, proxyConf, isSecure, profileData)
          .then(send);
    }
    return send(info);
  }, onError: (error) {
    profileData?.finishRequestWithError(error.toString());
    throw error;
  });
}

// Get a new _HttpClientConnection, from the matching _ConnectionTarget.
Future<_ConnectionInfo> _getConnection(
    Uri uri,
    String uriHost,
    int uriPort,
    _ProxyConfiguration proxyConf,
    bool isSecure,
    _HttpProfileData? profileData) {
  Iterator<_Proxy> proxies = proxyConf.proxies.iterator;

  Future<_ConnectionInfo> connect(error, stackTrace) {
    if (!proxies.moveNext()) return Future.error(error, stackTrace);
    _Proxy proxy = proxies.current;
    String host = proxy.isDirect ? uriHost : proxy.host!;
    int port = proxy.isDirect ? uriPort : proxy.port!;
    return _getConnectionTarget(host, port, isSecure)
        .connect(uri, uriHost, uriPort, proxy, this, profileData)
        // On error, continue with next proxy.
        .catchError(connect);
  }

  return connect(HttpException("No proxies given"), StackTrace.current);
}

```
通过层层方法的调用，我们最终找到真正进行TCP连接的是关于Https的SecureSocket和http的Socket的startConnect方法中
```
external static Future<ConnectionTask<Socket>> _startConnect(host, int port,
    {sourceAddress, int sourcePort = 0});
```

上面的external表示Android/iOS平台有各自的socket代码实现，但最终的逻辑是相同的，有**DNS解析**和TCP三次握手，其中https的SecureSocket在进行TCP连接之前还增加了一个证书校验过程,也就是https的四次握手。

到这里，TCP连接通道已经建立好了，那么接下来就是数据传输过程了，下一节我们将先介绍Dio框架中基于IP协议的数据传输过程。


### 3.5 IP传输数据

建立好TCP通道后，IP协议开发发挥作用了，是时候传输数据了
```
class DefaultHttpClientAdapter implements HttpClientAdapter {
  /// [Dio] will create HttpClient when it is needed.
  /// If [onHttpClientCreate] is provided, [Dio] will call
  /// it when a HttpClient created.
  OnHttpClientCreate? onHttpClientCreate;

  HttpClient? _defaultHttpClient;

    @override
    Future<ResponseBody> fetch(
      RequestOptions options,
      Stream<Uint8List>? requestStream,
      Future? cancelFuture,
    ) async {
      if (_closed) {
        throw Exception(
        "Can't establish connection after [HttpClientAdapter] closed!");
      }
      //创建HttpClient，建立TCPl连接
      var _httpClient = _configHttpClient(cancelFuture, options.connectTimeout);
      var reqFuture = _httpClient.openUrl(options.method, options.uri);
      
      late HttpClientRequest request;
      request = await reqFuture;
        //开始IP传输数据
      var future = request.close();
    }
}  
```
DefaultHttpClientAdapter中传输数据的方法入口就是request.close（很想吐槽close这个方法名），request建立TCP连接时调用_httpClient.openUrl返回的HttpClientRequest对象，调用流程如下图所示（借图）：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/90656761608d49909c4cc165f7520a17~tplv-k3u1fbpfcp-watermark.image?)

```

// HttpClientRequest的实现类_HttpClientRequest中的close方法
Future<HttpClientResponse> close() {
  if (!_aborted) {
    // It will send out the request.
    super.close();
  }
  return done;
}

// 上面的super.close()调用的是_HttpClientRequest间接继承的_StreamSinkImpl类中的close方法
Future close() {
  if (_isBound) {
    throw StateError("StreamSink is bound to a stream");
  }
  if (!_isClosed) {
    _isClosed = true;
    var controller = _controllerInstance;
    if (controller != null) {
      controller.close();
    } else {
      _closeTarget();
    }
  }
  return done;
}

void _closeTarget() {
  _target.close().then(_completeDoneValue, onError: _completeDoneError);
}
```

通过一系列调用，核心逻辑在 _target这个变量中，_target是StreamConsumer，HttpOutgoing则继承自StreamConsumer，所以核心逻辑就是在http_imp.dart文件中HttpOutgoing的close方法中，HttpOutgoing中保存有socket对象，可用于发送数据。
```
class _HttpOutgoing implements StreamConsumer<List<int>> {

    Future close() {
        Future finalize() {
            return socket.flush().then((_) {
              _doneCompleter.complete(socket);
              return outbound;
            }, onError: (error, stackTrace) {
              _doneCompleter.completeError(error, stackTrace);
              if (_ignoreError(error)) {
                return outbound;
              } else {
                throw error;
              }
            });
        }
        
        var future = writeHeaders();
        if (future != null) {
          return _closeFuture = future.whenComplete(finalize);
        }
        return _closeFuture = finalize();
    }
}
```
socket.flush()就是发送数据的过程，数据通过TCP建立的通道传输给服务器，服务器接收到客户端输入的数据后，也需要对数据进行并将数据返回给客户端。

### 3.6 服务器接收并处理数据

服务器接收数据相当于服务器的数据输入，只不过数据是经过客户端Dio框架加工而成。而服务器拿到数据也需要对数据进行处理，处理流程如下：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ba9224fa821347839680c74221a9a0b7~tplv-k3u1fbpfcp-watermark.image?)

由于涉及到后台逻辑，此处不做详细展开，有兴趣的同学可以自行google。

### 3.7 客户端接收并处理返回数据
上一节中后台通过TCP返回数据，客户端通过TCP接收数据，在接收到数据以后就需要对数据进行处理。

首先我们来看下标准的返回数据格式：
![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/81bc1815383b4043b33f6b6241561b72~tplv-k3u1fbpfcp-watermark.image?)

可以看到通过socket返回的数据格数如上图所示，如果此时直接返回给调用者，调用方需要不停的进行io操作读取字符串，使用起来异常麻烦，这就需要框架对数据进行加工处理，返回调用者自己想要的数据形式。

在建立TCP连接openUrl时创建了_HttpClientConnection对象，构造函数为Socket注册了onData事件的回调，即_HttpParser用于接收后台返回的数据。因此每当Socket有数据进来时，都会触发_HttpParser的onData进行处理。
![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/62d86fa157384dffa1138c5bc3c44664~tplv-k3u1fbpfcp-watermark.image?)

```
  _HttpClientConnection(this.key, this._socket, this._httpClient,
      [this._proxyTunnel = false, this._context])
      : _httpParser = new _HttpParser.responseParser() {
    _httpParser.listenToStream(_socket);

    _subscription = _httpParser.listen((incoming) {......}
```

处理完成后，通过层层调用至_HttpClientRequest的_responseCompleter，最终获取到**HttpClientResponse**对象。

# 4.输出数据
#### 4.1 返回responce并断开连接
上一章数据处理完成后最终得到的是HttpClientResponse对象，Dio框架则通过将**HttpClientResponse**包装成**Response**对象返回给调用者，如果发生网络请求错误或异常则会将错误包装成**DioError**对象，这其中还涉及到Intercetor队列中onResponce和onError的拦截处理。
```
void onResponse(
  Response response,
  ResponseInterceptorHandler handler,
) =>
    handler.next(response);
    
void onError(
  DioError err,
  ErrorInterceptorHandler handler,
) =>
    handler.next(err);
```

# 5.其他
至此Dio的大致框架已经清晰，文章并不拘泥了源码的细节，主要是从系统的网络请求流程上分析，除上述流程外，读者对于框架中可能还会存在如下部分疑惑：
* Cookie和Session
* 加密Basice和Digest
* Gzip
* 代理、网关和隧道
* 缓存
* https

限于篇幅，本篇并上面部分做详细介绍，下一篇笔者将基于Dio框架对以上部分做详细分析。

# 6.总结

综上所述，我们首先介绍了Dio框架的数据输入流程，有初始化配置和业务调用两类，其次介绍了框架对输入数据进行处理的过程，最后就需要将处理完的数据输出给调用者

**输入数据**
一般分为两种：
* 初始化配置
* 业务调用

**数据处理** 的过程比较复杂，详细在后续讲解，
Dio的数据处理流程大致分为如下7个步骤:
* 数据预处理
* URL解析
* DNS解析
* 建立TCP连接
* IP传输数据
* 服务器接收并处理数据
* 客户端接收并处理返回数据

**输出数据**是将完整的数据返回给调用端

文章对Dio框架进行了一个基本的解构，主要是想让读者拥有一个网络框架系统性的思维，你是否想过，Android/iOS/Web的网络框架设计流程是否也是如此？根据上述流程，基于现有的技术栈能否自己手动撸一个网络框架？答案是一定的。

> 建议对照流程手动分析Dio框架


