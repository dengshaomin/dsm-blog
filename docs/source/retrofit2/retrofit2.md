***
implementation 'com.squareup.retrofit2:retrofit:2.8.1'

github项目地址：​https://github.com/square/retrofit
***

retrofit应该是我们商业项目中用的最多的一个网络框架了，功能齐全、架构清晰、扩展性好，底层使用okhttp做网络处理。使用[动态代理](https://juejin.cn/post/6844903983761326093)的方式解析api接口这算是retrofit一个比较核心的点，官方对retrofit的使用介绍：https://square.github.io/retrofit/。 使用最简单的例子为切入点对其进行分析

```
var retrofit: Retrofit? = null
var api: IPost? = null
private fun request() {
        if (retrofit == null) {
            //创建retrofit
            retrofit = Retrofit.Builder().baseUrl("http://www.kuaidi100.com/")
                .build()
            //动态代理创建api
            api = (retrofit as Retrofit).create(IPost::class.java)
        }
        //调用request获取call,其实还只是解析接口，并没有真正进行接口调用
        var call = api?.request(object : HashMap<String, String>() {
            init {
                //请求参数
                put("type", "yuantong")
                put("postid", "11111111111")
            }
        })
        call?.enqueue(object : Callback<PostBean> {
            override fun onFailure(call: Call<PostBean>, t: Throwable) {
                //成功回调
            }

            override fun onResponse(call: Call<PostBean>, response: Response<PostBean>) {
                //失败回调
            }
        })
    }
//接口定义
interface IPost {
    @GET("query")
    fun request(@QueryMap map: Map<String, String>): Call<PostBean>
}
```

### 1.retrofit的创建
```
retrofit = Retrofit.Builder().baseUrl("http://www.kuaidi100.com/")
                .build()
```
通过建造者模式创建，支持配置callFactory、url、convertFactory、callAdapter、callbackExecutor等，并通过build完成内部参数的构建

```
public final class Retrofit {
  private final Map<Method, ServiceMethod<?>> serviceMethodCache = new ConcurrentHashMap<>();
  //调用工厂，默认使用OkHttpClient
  final okhttp3.Call.Factory callFactory;
  //请求host
  final HttpUrl baseUrl;
  //数据转换器，一般使用FastJsonConvertFactory
  final List<Converter.Factory> converterFactories;
  
  final List<CallAdapter.Factory> callAdapterFactories;
  //设置回调所在的线程默认为主线程MainThreadExecutor，所以在回调中更新UI是不需要再切线程的
  final @Nullable Executor callbackExecutor;
  final boolean validateEagerly;
}

public Retrofit build() {
      ...
      okhttp3.Call.Factory callFactory = this.callFactory;
      if (callFactory == null) {
        //没有设置callfactory默认使用okhttpclient
        callFactory = new OkHttpClient();
      }

      Executor callbackExecutor = this.callbackExecutor;
      if (callbackExecutor == null) {
        //没有设置默认使用MainThreadExecutor
        callbackExecutor = platform.defaultCallbackExecutor();
      }

      // 整合calladapter
      List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>(this.callAdapterFactories);
      callAdapterFactories.addAll(platform.defaultCallAdapterFactories(callbackExecutor));

      // 整合convertfactory
      List<Converter.Factory> converterFactories = new ArrayList<>(
          1 + this.converterFactories.size() + platform.defaultConverterFactoriesSize());
      converterFactories.add(new BuiltInConverters());
      converterFactories.addAll(this.converterFactories);
      converterFactories.addAll(platform.defaultConverterFactories());
      
      return new Retrofit(callFactory, baseUrl, unmodifiableList(converterFactories),
          unmodifiableList(callAdapterFactories), callbackExecutor, validateEagerly);
    }
```
### 2.retrofit.create api的创建

通过动态代理的方式生成另一个class，当api内部的方法被调用时会InvocationHandler，然后通过loadservicemethod对api进行解析，组要是对注解的解析，并返回ServiceMethod
```
//创建api
api = (retrofit as Retrofit).create(IPost::class.java)
//调用通过动态代理生成类的内部方法
var call = api?.request(object : HashMap<String, String>() {
            init {
                put("type", "yuantong")
                put("postid", "11111111111")
            }
        })

#retrofit.java
public <T> T create(final Class<T> service) {
    validateServiceInterface(service);
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          //获取android平台
          private final Platform platform = Platform.get();
          private final Object[] emptyArgs = new Object[0];
          @Override public @Nullable Object invoke(Object proxy, Method method,
              @Nullable Object[] args) throws Throwable {
            //当内部方法被调用将回调到这里
            return loadServiceMethod(method).invoke(args != null ? args : emptyArgs);
          }
        });
  }
  
ServiceMethod<?> loadServiceMethod(Method method) {
    //方法的解析需要消耗一定时间，这里做了一个缓存
    ServiceMethod<?> result = serviceMethodCache.get(method);
    if (result != null) return result;
    //单例DCL
    synchronized (serviceMethodCache) {
      result = serviceMethodCache.get(method);
      if (result == null) {
        //解析注解
        result = ServiceMethod.parseAnnotations(this, method);
        //解析完毕加入缓存
        serviceMethodCache.put(method, result);
      }
    }
    return result;
  }
```
ServiceMethod.parseAnotations对注解的解析分为两步，一是通过RequetFactory解析api所有的注解，二是通过HttpServiceMethod通过参数匹配calladapter以及convertfactory

```
static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Method method) {
  //解析api所有注解
  RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);
  //获取返回类型并校验返回类型的正确性
  Type returnType = method.getGenericReturnType();
  if (Utils.hasUnresolvableType(returnType)) {
    throw methodError(method,
        "Method return type must not include a type variable or wildcard: %s", returnType);
  }
  if (returnType == void.class) {
    throw methodError(method, "Service methods cannot return void.");
  }
  //选择合适的calladapter以及convertfactory
  return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);
}

#RequestFactory.java
final class RequestFactory {
  static RequestFactory parseAnnotations(Retrofit retrofit, Method method) {
    return new Builder(retrofit, method).build();
  }
  
  static final class Builder {
    Builder(Retrofit retrofit, Method method) {
      this.retrofit = retrofit;
      this.method = method;
      //获取方法的注解：@GET("query")
      this.methodAnnotations = method.getAnnotations();
      //获取参数类型：Map<String,String>
      this.parameterTypes = method.getGenericParameterTypes();
      //获取参数的注解：@QueryMap
      this.parameterAnnotationsArray = method.getParameterAnnotations();
    }
}

#HttpServiceMethod.java
abstract class HttpServiceMethod<ResponseT, ReturnT> extends ServiceMethod<ReturnT> {
  static <ResponseT, ReturnT> HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(
      Retrofit retrofit, Method method, RequestFactory requestFactory) {
      //判断是否kotlin的suspend方法
    boolean isKotlinSuspendFunction = requestFactory.isKotlinSuspendFunction;
    //获取所有注解
     Annotation[] annotations = method.getAnnotations();
    Type adapterType;
    if (isKotlinSuspendFunction) {
      ...
      //kotlin的处理逻辑
    } else {
      //获取method的返回类型
      adapterType = method.getGenericReturnType();
    }
    //获取到匹配的calladapter
    CallAdapter<ResponseT, ReturnT> callAdapter =
        createCallAdapter(retrofit, method, adapterType, annotations);
    Type responseType = callAdapter.responseType();
    ...
    //选取合适的reponseconvert
    Converter<ResponseBody, ResponseT> responseConverter =
        createResponseConverter(retrofit, method, responseType);
    okhttp3.Call.Factory callFactory = retrofit.callFactory;
    if (!isKotlinSuspendFunction) {
      return new CallAdapted<>(requestFactory, callFactory, responseConverter, callAdapter);
    } else if (continuationWantsResponse) {
      //noinspection unchecked Kotlin compiler guarantees ReturnT to be Object.
      return (HttpServiceMethod<ResponseT, ReturnT>) new SuspendForResponse<>(requestFactory,
          callFactory, responseConverter, (CallAdapter<ResponseT, Call<ResponseT>>) callAdapter);
    } else {
      //noinspection unchecked Kotlin compiler guarantees ReturnT to be Object.
      return (HttpServiceMethod<ResponseT, ReturnT>) new SuspendForBody<>(requestFactory,
          callFactory, responseConverter, (CallAdapter<ResponseT, Call<ResponseT>>) callAdapter,
          continuationBodyNullable);
    }
  }
```

###   3.CallAdapter

先看一下callAdapter的匹配规则，上面的例子是没有配置自定义 calladapter的，这样callAdapterFactories中就只有系统默认的两个adapter：CompletableFutureCallAdapterFactory以及DefaultCallAdapterFactory，主要是通过api的返回类型来匹配的，上面示例中返回类型为Call<PostBean>所以匹配上了DefaultCallAdapterFactory

```
#Retrofit.java
public CallAdapter<?, ?> nextCallAdapter(@Nullable CallAdapter.Factory skipPast, Type returnType,
    Annotation[] annotations) {
  ...
  int start = callAdapterFactories.indexOf(skipPast) + 1;
  for (int i = start, count = callAdapterFactories.size(); i < count; i++) {
    //各自的get方法通过返回值进行匹配
    CallAdapter<?, ?> adapter = callAdapterFactories.get(i).get(returnType, annotations, this);
    if (adapter != null) {
      return adapter;
    }
  }
  //没有找到adapter直接抛出异常
  throw new IllegalArgumentException(builder.toString());
}
```

如果我们在创建retrofit的时候配置常用的RxJavaCallAdapterFactory，api的返回类型需要改成Observable，这样通过返回值匹配也同样能匹配到RxJavaCallAdapterFactory。rxjava配合retrofit能使调用过程变得更简单、清晰所以建议配套使用。
```
retrofit = Retrofit.Builder().baseUrl("http://www.kuaidi100.com/")
                //增加rxjavacalladapter
                .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
                .build()
//接口定义更改
interface IPost {
    @GET("query")
    fun request(@QueryMap map: Map<String, String>): Observable<PostBean>
}
//调用方式的改变
api.request.subscribeOn(Schedulers.newThread())
		.subscribeOn(AndroidSchedulers.mainThread())
		.subscribe(new Subscriber<Location>() {
			@Override
			public void onCompleted() {
				Log.d(TAG, "onCompleted: ");
			}
			@Override
			public void onError(Throwable e) {
				Log.d(TAG, "onError: ");
			}
			@Override
			public void onNext(Location location) {
				Log.d(TAG, "onNext: " + location);
			}
		});
```

### 4.ConvertFactory

convert的匹配规则同样在Retrofit中，通过判断responseBodyConverter函数的返回值是否为null来完成匹配，如果在convertFactories没有找到合适的也会和callAdapter一样抛出异常

```
#retrofit.java
public <T> Converter<ResponseBody, T> nextResponseBodyConverter(
    @Nullable Converter.Factory skipPast, Type type, Annotation[] annotations) {
  Objects.requireNonNull(type, "type == null");
  Objects.requireNonNull(annotations, "annotations == null");

  int start = converterFactories.indexOf(skipPast) + 1;
  for (int i = start, count = converterFactories.size(); i < count; i++) {
  //通过response判断
    Converter<ResponseBody, ?> converter =
        converterFactories.get(i).responseBodyConverter(type, annotations, this);
    if (converter != null) {
      //noinspection unchecked
      return (Converter<ResponseBody, T>) converter;
    }
  }
  //没有合适的适配器抛出异常
  throw new IllegalArgumentException(builder.toString());
}
```

retrofit默认提供两个responseconvert：BuiltInConverters和OptionalConverterFactory，示例中的返回值是 PostBean导致这两个都没匹配上，而我们自己又没有设置对应的factory所以抛出了异常。修改示例代码在retrofit构建时添加自定义FastJsonConvertFactory，其中requestBodyConverter和reponseBodyConverter都不为null，所以匹配上了这个适配器。

```
retrofit = Retrofit.Builder()
    .baseUrl("http://www.kuaidi100.com/")
    .addConverterFactory(FastJsonConvertFactory())
    .build()
    
#FastJsonConvertFactory.java
public class FastJsonConvertFactory extends Converter.Factory {
    public static FastJsonConvertFactory create() {
        return new FastJsonConvertFactory();
    }
    @Override
    public Converter<?, RequestBody> requestBodyConverter(Type type, Annotation[] parameterAnnotations, Annotation[] methodAnnotations, Retrofit retrofit) {
        return new FastJsonRequestConverter<>();
    }
    @Override
    public Converter<ResponseBody, ?> responseBodyConverter(Type type, Annotation[] annotations, Retrofit retrofit) {
        return new FastJsonResponseConverter<>(type);
    }
}
```

### 5.发起网络请求

上面所有的操作都是给最后的请求做准备，真正拉起网络请求的步骤为call.enqueue

```
//异步调用
call?.enqueue(object : Callback<PostBean> {
    override fun onFailure(call: Call<PostBean>, t: Throwable) {
    }

    override fun onResponse(call: Call<PostBean>, response: Response<PostBean>) {
    }
})
```
示例中默认我们并没有配置callAdapter所以使用默认的DefaultCallAdapterFactory，call最终提供者是Okhttp中的RealCall(关于okhttp在下一章专门介绍)；

```
@Override public void enqueue(final Callback<T> callback) {
  Objects.requireNonNull(callback, "callback == null");
  //代理是OkHttpCall，最终是由OkhttpCient提供的RealCall
  delegate.enqueue(new Callback<T>() {
    @Override public void onResponse(Call<T> call, final Response<T> response) {
      //使用配置的callbackExecutor在主线程回调结果
      callbackExecutor.execute(() -> {
        if (delegate.isCanceled()) {
          // Emulate OkHttp's behavior of throwing/delivering an IOException on cancellation.
          callback.onFailure(ExecutorCallbackCall.this, new IOException("Canceled"));
        } else {
          callback.onResponse(ExecutorCallbackCall.this, response);
        }
      });
    }
    @Override public void onFailure(Call<T> call, final Throwable t) {
      callbackExecutor.execute(() -> callback.onFailure(ExecutorCallbackCall.this, t));
    }
  });
}
```










