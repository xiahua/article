安卓开发领域中，很多重要的问题都有很好的开源解决方案，例如Square公司提供网络请求 OkHttp , Retrofit 和现在非常流行的异步处理框架Rxjava。Square 公司开源的 Retrofit 更是以其简易的接口配置、强大的扩展支持、优雅的代码结构受到大家的喜爱。
# 1.初识Retrofit
认识Retrofit首先要从怎么使用开发，其次我们在使用过程深入了解其内部原理。最好下载源码去阅读。本文的例子来自 [Retrofit 官方网站](http://square.github.io/retrofit/)
## 1.1 Retrofit 概览
Retrofit是一个 RESTful 的 HTTP 网络请求框架的封装  。Retrofit 2.0 开始内置 OkHttp，Retrofit专注于接口的封装，OkHttp专注于网络请求的高效。
我们App通过 Retrofit 请求网络，实际上是使用 Retrofit 接口进行封装请求参数、Header、Url 等信息（Header信息可以用OkHttp的Interceptor实现），之后由 OkHttp 完成后续的请求操作，在服务端返回数据之后，OkHttp 将原始的结果交给 Retrofit， Retrofit根据用户的需求对结果进行解析的过程。

![](http://upload-images.jianshu.io/upload_images/1177171-1e700d3b45df4e43.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 1.2 Retrofit使用
你先需要在你的 build.gradle 中添加依赖：
``` Java
compile 'com.squareup.retrofit2:retrofit:2.2.0'
```
首先我们要构造Retrofit:
``` Java
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://api.github.com/")
    .addConverterFactory(GsonConverterFactory.create())
    .build();
```
**builder 模式，外观模式（门面模式）**，这两个设计模式，可以看看 [stay 的 Retrofit分析-经典设计模式案例](http://www.jianshu.com/p/fb8d21978e38)这篇文章。

其次在定义 API 接口：
``` Java
public interface GitHubService {
  @GET("users/{user}/repos")
  Call<List<Repo>> listRepos(@Path("user") String user);
}
```
看代码非常简洁，接口当中的 listRepos方法，除了两个注解：@GET 和 @Path，就是我们想要访问的 api 了：
> https://api.github.com/users/{user}/repos

其中，在发起请求时， {user}会被替换为方法的第一个参数 user。
再次 创建Api实例：
```java
GitHubService service = retrofit.create(GitHubService.class);
Call<List<Repo>> repos = service.listRepos("octocat");
```
retrofit.create(GitHubService.class)这样就创建了 API 实例了，接着service.listRepos("octocat") 设置参数。
发起请求有同步和异步：
```java 
//同步
List<Repo> data= repos.execute();
//异步
repos.enqueue(new Callback<List<Repo>>() {
    @Override
    public void onResponse(Call<List<Repo>> call, Response<List<Repo>> response) {
      List<Repo>  data= response.body();
    }

    @Override
    public void onFailure(Call<List<Repo>> call,Throwable t) {

    }
});
```
看着代码是不是很简单。
## 1.3 Url 配置
``` java
@GET("users/{user}/repos")
 Call<List<Repo>> listRepos(@Path("user") String user);
```
上面的代码使用两个注解@GET和 @Path。这些注解都有一个参数 value，用来配置其路径，比如示例中的 users/{user}/repos，我们还注意到在构造 Retrofit 之时我们还传入了一个 baseUrl("https://api.github.com/")，
请求的完整 Url 就是通过 baseUrl 与注解的 value（下面称 “path“ ） 整合起来的，具体整合的规则如下：

* path 是相对路径，baseUrl 是目录形式：
 path = "apath"，baseUrl = "http://host:port/a/b/"
 Url = "http://host:port/a/b/apath"

建议采用这种方式来配置，并尽量使用同一种路径形式。
## 1.4 参数类型
Retrofit支持的协议包括 GET/POST/PUT/DELETE/HEAD/PATCH。发请求时，需要传入参数，Retrofit 通过注解的形式令 Http 请求的参数变得更加直接，而且类型安全。
### 1.4.1 Path
``` java
@GET("group/{id}/users")
Call<List<User>> groupList(@Path("id") int groupId);
```
Path 其实就是 Url 中 {id}替换我们传进去的groupId的值，上面的请求的如下：
> https://api.github.com/group/1/users

### 1.4.2 Query & QueryMap
``` java
@GET("group/{id}/users")
Call<List<User>> groupList(@Path("id") int groupId, @Query("sort") String sort);
```
Query 其实就是 Url 中 ‘?’ 后面的 key-value，上面的请求如下
> https://api.github.com/group/1/users?sort=des

如果我有很多个 Query，每个都这么写岂不是很累？而且根据不同的情况，有些字段可能不传，这与方法的参数要求显然也不相符。于是， QueryMap 横空出世了，使用方法很简单，如下：
```java 
@GET("group/{id}/users")
Call<List<User>> groupList(@Path("id") int groupId, @QueryMap Map<String, String> options);
```
### 1.4.3 Body
post请求的body需要发送一个对象，这是可以用Body，Body的作用是把对象转换成需要的字符串发送到服务器，比如服务端需要的是关于某一个自定义对象的JSON数据格式。
``` java
@POST("users/new")
Call<User> createUser(@Body User user);
```
怎样把对象转换为json 数据是通过converter 实现的。

### 1.4.4 Field & FieldMap
POST使用场景很多，如何用 POST 提交表单的场景:
``` java
@FormUrlEncoded
@POST("user/edit")
Call<User> updateUser(@Field("first_name") String first, @Field("last_name") String last);
```
其实也很简单，我们只需要@FormUrlEncoded，然后用 Field 声明了表单的项，这样提交表单就跟普通的函数调用一样简单直接了。
如果你也多个Field ，不想一个一个写，同样有可以用——FieldMap。
### 1.4.5 Part & PartMap
我们经常需要上传文件，Part主要用于文件上传，有Retrofit 会方便很多。
``` java
    @Multipart
    @POST("upload")
    Call<ResponseBody> upload(@Part("description") RequestBody description,@Part MultipartBody.Part file);
```
定义文件上传的接口使用了@Multipart，后面只需要在参数中增加 Part 就可以了。这里的 Part 和 Field 究竟有什么区别，其实从功能上讲，无非就是客户端向服务端发起请求携带参数的方式不同，并且前者可以携带的参数类型更加丰富，包括数据流。也正是因为这一点，我们可以通过这种方式来上传文件，下面我们就给出这个接口的使用方法：
``` java
File file = new File(filename);
RequestBody requestFile =RequestBody.create(MediaType.parse("application/otcet-stream"), file);
MultipartBody.Part body =MultipartBody.Part.createFormData("aFile", file.getName(), requestFile);
 
String descriptionString = "This is a description";
RequestBody description =RequestBody.create(MediaType.parse("multipart/form-data"), descriptionString);
 
Call<ResponseBody> call = service.upload(description, body);
call.enqueue(new Callback<ResponseBody>() {
  @Override
  public void onResponse(Call<ResponseBody> call, Response<ResponseBody> response) {
    System.out.println("success");
  }
 
  @Override
  public void onFailure(Call<ResponseBody> call, Throwable t) {
    t.printStackTrace();
  }
});
```
如有用多个文件上传可以建多个Part 或使用PartMap。
### 1.4.6 Headers
``` java
@Headers("Cache-Control: max-age=640000")
@GET("widget/list")
Call<List<Widget>> widgetList();
```
Headers 主要用于添加头控制,Cache-Control 控制是否保存数据，下次再规定时间内是否去服务器请求。

# 2 Retrofit源码解析
Retrofit 的基本用法和概念介绍了一下，如果你的目标是学会如何使用它，那么下面的内容你可以不用看了
##  2.1 Api创建
``` java
GitHubService service = retrofit.create(GitHubService.class);
```
Retrofit是如何创建Api的，是通过Retrofit类的create的方法。源码如下
``` java
public <T> T create(final Class<T> service) {
    Utils.validateServiceInterface(service);
    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, Object... args)
              throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
           //DefaultMethod 是 Java 8 的概念，是定义在 interface 当中的有实现的方法
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
             //每一个接口最终实例化成一个 ServiceMethod，并且会缓存
            ServiceMethod serviceMethod = loadServiceMethod(method);
            //Retrofit 与 OkHttp 完全耦合
            OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
          //发起请求，并解析服务端返回的结果
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
  }
```
创建 API 实例使用的是**动态代理技术**，关于动态代理的详细介绍，可以查看 [codeKK 公共技术点之 Java 动态代理](http://a.codekk.com/detail/Android/Caij/%E5%85%AC%E5%85%B1%E6%8A%80%E6%9C%AF%E7%82%B9%E4%B9%8B%20Java%20%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86)这篇文章。
简而言之，就是动态生成接口的实现类（当然生成实现类有缓存机制），并创建其实例（称之为代理），代理把对接口的调用转发给 InvocationHandler实例，而在 InvocationHandler的实现中，除了执行真正的逻辑（例如再次转发给真正的实现类对象），我们还可以进行一些有用的操作，例如统计执行时间、进行初始化和清理、对接口调用进行检查等。

为什么要用动态代理？因为对接口的所有方法的调用都会集中转发到 InvocationHandler#invoke函数中，我们可以集中进行处理，更方便了。你可能会想，我也可以手写这样的代理类，把所有接口的调用都转发到 InvocationHandler#invoke，当然可以，但是可靠地自动生成岂不更方便,可以方便为什么不方便。

## 2.2 Api使用
create方法真正我们要看是下面三行代码：
``` java
ServiceMethod serviceMethod = loadServiceMethod(method);
OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
return serviceMethod.callAdapter.adapt(okHttpCall);
```
在分析具体三行代码可以看下[Stay 在 Retrofit分析-漂亮的解耦套路](http://www.jianshu.com/p/45cb536be2f4) 这篇文章中分享的流程图，完整的流程概览建议仔细看看这篇文章：
![retrofit](http://upload-images.jianshu.io/upload_images/1177171-8ba68b3bea4fc57e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 2.3 ServiceMethod<T>
> Adapts an invocation of an interface method into an HTTP call. 
把对接口方法的调用转为一次 HTTP 调用。

一个 ServiceMethod 对象对应于一个 API interface 的一个方法，loadServiceMethod(method) 方法负责加载 ServiceMethod：
``` java
ServiceMethod loadServiceMethod(Method method) {
    ServiceMethod result;
    synchronized (serviceMethodCache) {
      result = serviceMethodCache.get(method);
      if (result == null) {
        result = new ServiceMethod.Builder(this, method).build();
        serviceMethodCache.put(method, result);
      }
    }
    return result;
  }
```
这里实现了缓存逻辑，同一个 API 的同一个方法，只会创建一次。这里由于我们每次获取 API 实例都是传入的 class 对象，而 class 对象是进程内单例的，所以获取到它的同一个方法 Method 实例也是单例的，所以这里的缓存是有效的。

我们再看 ServiceMethod 的构造函数：
``` java
ServiceMethod(Builder<T> builder) {
    this.callFactory = builder.retrofit.callFactory();
    this.callAdapter = builder.callAdapter;
    this.baseUrl = builder.retrofit.baseUrl();
    this.responseConverter = builder.responseConverter;
    this.httpMethod = builder.httpMethod;
    this.relativeUrl = builder.relativeUrl;
    this.headers = builder.headers;
    this.contentType = builder.contentType;
    this.hasBody = builder.hasBody;
    this.isFormEncoded = builder.isFormEncoded;
    this.isMultipart = builder.isMultipart;
    this.parameterHandlers = builder.parameterHandlers;
  }
```
 这里用了buider 模式，这里很多参数，有的参数一看就明白 像baseUrl，headers ，contyentType等，讲解主要关注四个成员：callFactory，callAdapter，responseConverter 和 parameterHandlers。
* callFactory 负责创建 HTTP 请求，HTTP 请求被抽象为了 okhttp3.Call 类，它表示一个已经准备好，可以随时执行的 HTTP 请求；
* callAdapter 把 retrofit2.Call<T> 转为 T（注意和 okhttp3.Call 区分开来，retrofit2.Call<T> 表示的是对一个 Retrofit 方法的调用），这个过程会发送一个 HTTP 请求，拿到服务器返回的数据（通过 okhttp3.Call 实现），并把数据转换为声明的 T 类型对象（通过 Converter<F, T> 实现）；
* responseConverter 是 Converter<ResponseBody, T> 类型，负责把服务器返回的数据（JSON、XML、二进制或者其他格式，由 ResponseBody 封装）转化为 T 类型的对象；
* parameterHandlers 则负责解析 API 定义时每个方法的参数，并在构造 HTTP 请求时设置参数；

### 2.3.1 callFactory
this.callFactory = builder.retrofit.callFactory()，所以 callFactory实际上由 Retrofit类提供，而我们在造 Retrofit
 对象时，可以指定 callFactory，如果不指定，将默认设置为一个 okhttp3.OkHttpClient。

### 2.3.2 CallAdapter
``` java
    private CallAdapter<?> createCallAdapter() {
      Type returnType = method.getGenericReturnType();
      if (Utils.hasUnresolvableType(returnType)) {
        throw methodError(
            "Method return type must not include a type variable or wildcard: %s", returnType);
      }
      if (returnType == void.class) {
        throw methodError("Service methods cannot return void.");
      }
      Annotation[] annotations = method.getAnnotations();
      try {
        return retrofit.callAdapter(returnType, annotations);
      } catch (RuntimeException e) { // Wide exception range because factories are user code.
        throw methodError(e, "Unable to create call adapter for %s", returnType);
      }
    }

```
可以看到，callAdapter 还是由 Retrofit 类提供。在 Retrofit 类内部，将遍历一个 CallAdapter.Factory 列表，让工厂们提供，如果最终没有工厂能（根据 returnType 和 annotations）提供需要的 CallAdapter，那将抛出异常。而这个工厂列表我们可以在构造 Retrofit 对象时进行添加。
### 2.3.2 responseConverter
``` java
 private Converter<ResponseBody, T> createResponseConverter() {
      Annotation[] annotations = method.getAnnotations();
      try {
        return retrofit.responseBodyConverter(responseType, annotations);
      } catch (RuntimeException e) { // Wide exception range because factories are user code.
        throw methodError(e, "Unable to create converter for %s", responseType);
      }
    }
```

同样，responseConverter还是由 Retrofit类提供，而在其内部，逻辑和创建 callAdapter基本一致，通过遍历 Converter.Factory 列表，看看有没有工厂能够提供需要的 responseBodyConverter。工厂列表同样可以在构造 Retrofit对象时进行添加。
### 2.3.3，parameterHandlers
每个参数都会有一个 ParameterHandler，由 ServiceMethod#parseParameter方法负责创建，其主要内容就是解析每个参数使用的注解类型（诸如 Path，Query，Field等），对每种类型进行单独的处理。构造 HTTP 请求时，我们传递的参数都是字符串，那 Retrofit 是如何把我们传递的各种参数都转化为 String 的呢？还是由 Retrofit 类提供 converter！

Converter.Factory除了提供上一小节提到的 responseBodyConverter，还提供 requestBodyConverter 和 stringConverter，API 方法中除了 @Body 和 @Part类型的参数，都利用 stringConverter 进行转换，而 @Body和 @Part类型的参数则利用 requestBodyConverter 进行转换。

这三种 converter 都是通过“询问”工厂列表进行提供，而工厂列表我们可以在构造 Retrofit对象时进行添加。

### 2.3.4，工厂让各个模块得以高度解耦
上面提到了三种工厂：okhttp3.Call.Factory，CallAdapter.Factory和 Converter.Factory，分别负责提供不同的模块，至于怎么提供、提供何种模块，统统交给工厂，Retrofit 完全不掺和，它只负责提供用于决策的信息，例如参数/返回值类型、注解等。

这不正是我们苦苦追求的**高内聚低耦合**效果吗？解耦的第一步就是**面向接口编程**，模块之间、类之间通过接口进行依赖，**创建怎样的实例，则交给工厂负责**，工厂同样也是接口，添加（Retrofit doc 中使用 install 安装一词，非常贴切）怎样的工厂，则在最初构造 Retrofit 对象时决定，各个模块之间完全解耦，**每个模块只专注于自己的职责**，全都是套路，值得反复玩味、学习与模仿。

除了上面重点分析的这四个成员，ServiceMethod中还包含了 API 方法的 url 解析等逻辑，包含了众多关于泛型和反射相关的代码，有类似需求的时候，也非常值得学习模仿。

## 2.4 OkHttpCall
OkHttpCall实现了 retrofit2.Call，我们通常会使用它的 execute() 和 enqueue(Callback<T> callback)接口。前者用于同步执行 HTTP 请求，后者用于异步执行。

### 2.4.1 execute()
``` java
 public Response<T> execute() throws IOException {
   //这个 call 是真正的 OkHttp 的 call，本质上 OkHttpCall 只是对它做了一层封装
    okhttp3.Call call;

    synchronized (this) {
   //处理重复执行的逻辑
      if (executed) throw new IllegalStateException("Already executed.");
      executed = true;

      if (creationFailure != null) {
        if (creationFailure instanceof IOException) {
          throw (IOException) creationFailure;
        } else {
          throw (RuntimeException) creationFailure;
        }
      }

      call = rawCall;
      if (call == null) {
        try {
          call = rawCall = createRawCall();
        } catch (IOException | RuntimeException e) {
          creationFailure = e;
          throw e;
        }
      }
    }

    if (canceled) {
      call.cancel();
    }

    return parseResponse(call.execute());
  }

  private okhttp3.Call createRawCall() throws IOException {
    Request request = serviceMethod.toRequest(args);
    okhttp3.Call call = serviceMethod.callFactory.newCall(request);
    if (call == null) {
      throw new NullPointerException("Call.Factory returned null.");
    }
    return call;
  }

  Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
    ResponseBody rawBody = rawResponse.body();

    // Remove the body's source (the only stateful object) so we can pass the response along.
    rawResponse = rawResponse.newBuilder()
        .body(new NoContentResponseBody(rawBody.contentType(), rawBody.contentLength()))
        .build();

    int code = rawResponse.code();
    if (code < 200 || code >= 300) {
      try {
        // Buffer the entire body to avoid future I/O.
        ResponseBody bufferedBody = Utils.buffer(rawBody);
        return Response.error(bufferedBody, rawResponse);
      } finally {
        rawBody.close();
      }
    }

    if (code == 204 || code == 205) {
      return Response.success(null, rawResponse);
    }

    ExceptionCatchingRequestBody catchingBody = new ExceptionCatchingRequestBody(rawBody);
    try {
      T body = serviceMethod.toResponse(catchingBody);
      return Response.success(body, rawResponse);
    } catch (RuntimeException e) {
      // If the underlying source threw an exception, propagate that rather than indicating it was
      // a runtime exception.
      catchingBody.throwIfCaught();
      throw e;
    }
  }
```
主要包括三步：

* 创建 okhttp3.Call，包括构造参数；
* 执行网络请求；
* 解析网络请求返回的数据；

createRawCall() 函数中，我们调用了 serviceMethod.toRequest(args) 来创建 okhttp3.Request，而在后者中，我们之前准备好的 parameterHandlers 就派上了用场。

然后我们再调用 serviceMethod.callFactory.newCall(request) 来创建 okhttp3.Call，这里之前准备好的 callFactory 同样也派上了用场，由于工厂在构造 Retrofit 对象时可以指定，所以我们也可以指定其他的工厂（例如使用过时的 HttpURLConnection 的工厂），来使用其它的底层 HttpClient 实现。

我们调用 okhttp3.Call#execute() 来执行网络请求，这个方法是阻塞的，执行完毕之后将返回收到的响应数据。收到响应数据之后，我们进行了状态码的检查，通过检查之后我们调用了 serviceMethod.toResponse(catchingBody) 来把响应数据转化为了我们需要的数据类型对象。在 toResponse 函数中，我们之前准备好的 responseConverter 也派上了用场.

### 2.4.2  enqueue(Callback<T> callback)
这里的异步交给了 okhttp3.Call#enqueue(Callback responseCallback)来实现，并在它的 callback 中调用 parseResponse解析响应数据，并转发给传入的 callback。

## 2.5 CallAdapter
终于到了最后一步了，CallAdapter<T>#adapt(Call<R> call)函数负责把 retrofit2.Call<R> 转为 T。这里 T当然可以就是 retrofit2.Call<R>，这时我们直接返回参数就可以了，实际上这正是 DefaultCallAdapterFactory创建的 CallAdapter 的行为。至于其他类型的工厂返回的 CallAdapter的行为，这里暂且不表，后面再单独分析。

至此，一次对 API 方法的调用是如何构造并发起网络请求、以及解析返回数据，这整个过程大致是分析完毕了。对整个流程的概览非常重要，结合 stay 画的流程图，应该能够比较轻松地看清整个流程了。

## 2.6 retrofit-adapters 模块
retrofit 模块内置了 DefaultCallAdapterFactory 和 ExecutorCallAdapterFactory，它们都适用于 API 方法得到的类型为 retrofit2.Call 的情形，前者生产的 adapter 啥也不做，直接把参数返回，后者生产的 adapter 则会在异步调用时在指定的 Executor 上执行回调。

retrofit-adapters 的各个子模块则实现了更多的工厂：GuavaCallAdapterFactory，Java8CallAdapterFactory 和 RxJavaCallAdapterFactory。这里我主要分析 RxJavaCallAdapterFactory，下面的内容就需要一些 [RxJava](http://gank.io/post/560e15be2dca930e00da1083)的知识了.

RxJavaCallAdapterFactory#get 方法中对返回值的类型进行了检查，只支持 rx.Single，rx.Completable 和 rx.Observable，这里我主要关注对 rx.Observable 的支持。

RxJavaCallAdapterFactory#getCallAdapter 方法中对返回值的泛型类型进行了进一步检查，例如我们声明的返回值类型为 Observable<List<Repo>>，泛型类型就是 List<Repo>，这里对 retrofit2.Response 和 retrofit2.adapter.rxjava.Result 进行了特殊处理，有单独的 adapter 负责进行转换，其他所有类型都由 SimpleCallAdapter 负责转换。

那我们就来看看 SimpleCallAdapter#adapt：
``` java
@Override
public <R> Observable<R> adapt(Call<R> call) {
  Observable<R> observable = Observable.create(new CallOnSubscribe<>(call))
      .lift(OperatorMapResponseToBodyOrError.<R>instance());
  if (scheduler != null) {
    return observable.subscribeOn(scheduler);
  }
  return observable;
}
```
这里创建了一个 Observable，它的逻辑由 CallOnSubscribe 类实现，同时使用了一个 OperatorMapResponseToBodyOrError 操作符，用来把 retrofit2.Response 转为我们声明的类型，或者错误异常类型。

这里创建了一个 Observable，它的逻辑由 CallOnSubscribe 类实现，同时使用了一个 OperatorMapResponseToBodyOrError 操作符，用来把 retrofit2.Response 转为我们声明的类型，或者错误异常类型。

我们接着看 CallOnSubscribe#call：
``` java
@Override
public void call(final Subscriber<? super Response<T>> subscriber) {
  // Since Call is a one-shot type, clone it for each new subscriber.
  Call<T> call = originalCall.clone();

  // Wrap the call in a helper which handles both unsubscription and backpressure.
  RequestArbiter<T> requestArbiter = new RequestArbiter<>(call, subscriber);
  subscriber.add(requestArbiter);
  subscriber.setProducer(requestArbiter);
}
```
代码很简短，只干了三件事：
* clone 了原来的 call，因为 okhttp3.Call 是只能用一次的，所以每次都是新 clone 一个进行网络请求；
* 创建了一个叫做 RequestArbiter 的 producer，别被它的名字吓懵了，它就是个 producer；
* 把这个 producer 设置给 subscriber；

简言之，大部分情况下 Subscriber 都是被动接受 Observable push 过来的数据，但要是 Observable 发得太快，Subscriber 处理不过来，那就有问题了，所以就有了一种 Subscriber 主动 pull 的机制，而这种机制就是通过 Producer 实现的。给 Subscriber 设置 Producer 之后（通过 Subscriber#setProducer 方法），Subscriber 就会通过 Producer 向上游根据自己的能力请求数据（通过 Producer#request 方法），而 Producer 收到请求之后（通常都是 Observable 管理 Producer，所以“相当于”就是 Observable 收到了请求），再根据请求的量给 Subscriber 发数据。

那我们就看看 RequestArbiter#request：
``` java
@Override
public void request(long n) {
  if (n < 0) throw new IllegalArgumentException("n < 0: " + n);
  if (n == 0) return; // Nothing to do when requesting 0.
  if (!compareAndSet(false, true)) return; // Request was already triggered.

  try {
    Response<T> response = call.execute();
    if (!subscriber.isUnsubscribed()) {
      subscriber.onNext(response);
    }
  } catch (Throwable t) {
    Exceptions.throwIfFatal(t);
    if (!subscriber.isUnsubscribed()) {
      subscriber.onError(t);
    }
    return;
  }
  if (!subscriber.isUnsubscribed()) {
    subscriber.onCompleted();
  }
}
```
实际干活的逻辑就是执行 call.execute()，并把返回值发送给下游。


而 OperatorMapResponseToBodyOrError#call也相当简短：
``` java
@Override
public Subscriber<? super Response<T>> call(final Subscriber<? super T> child) {
  return new Subscriber<Response<T>>(child) {
    @Override
    public void onNext(Response<T> response) {
      if (response.isSuccessful()) {
        child.onNext(response.body());
      } else {
        child.onError(new HttpException(response));
      }
    }

    @Override
    public void onCompleted() {
      child.onCompleted();
    }

    @Override
    public void onError(Throwable e) {
      child.onError(e);
    }
  };
}
```
关键就是调用了 response.body() 并发送给下游。这里，body() 返回的就是我们声明的泛型类型了，至于 Retrofit 是怎么把服务器返回的数据转为我们声明的类型的，这就是 responseConverter 的事了。
总结下RxJavaCallAdapterFactory 执行路径就是：

  * Observable.subscribe，触发 API 调用的执行；
  * CallOnSubscribe#call，clone call，创建并设置 producer；
  * RequestArbiter#request，subscriber 被设置了 producer 之后最终调用 request，在 request 中发起请求，把结果发给下游；
  * OperatorMapResponseToBodyOrError$1#onNext，把 response 的 body 发给下游；
  *  最终就到了我们 subscribe 时传入的回调里面了；
## 2.7，retrofit-converters 模块
retrofit 模块内置了 BuiltInConverters，只能处理 ResponseBody， RequestBody 和 String类型的转化（实际上不需要转）。而 retrofit-converters 中的子模块则提供了 JSON，XML，ProtoBuf 等类型数据的转换功能，而且还有多种转换方式可以选择。这里我主要关注 GsonConverterFactory。
代码非常简单：
``` java
 @Override
  public Converter<ResponseBody, ?> responseBodyConverter(Type type, Annotation[] annotations,
      Retrofit retrofit) {
    TypeAdapter<?> adapter = gson.getAdapter(TypeToken.get(type));
    return new GsonResponseBodyConverter<>(gson, adapter);
  }

  @Override
  public Converter<?, RequestBody> requestBodyConverter(Type type,
      Annotation[] parameterAnnotations, Annotation[] methodAnnotations, Retrofit retrofit) {
    TypeAdapter<?> adapter = gson.getAdapter(TypeToken.get(type));
    return new GsonRequestBodyConverter<>(gson, adapter);
  }
```
根据目标类型，利用 Gson#getAdapter获取相应的 adapter，转换时利用 Gson 的 API 即可。

# 3 总结
以上就是retrofit 2.0 使用和代码部分功能分析，把retrofit 大致流程走了一边。如有有不对的请指出。

