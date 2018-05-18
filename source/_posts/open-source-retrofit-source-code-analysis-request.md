---
title: Retrofit 2 源码分析一 -- 数据请求流程解析
categories: Android开源项目
comments: true
tags: [Android开源项目, Retrofit]
description: Retrofit 2 数据请求流程的简单介绍
date: 2017-11-8 10:00:00
---

分析之前可以参考一下我的前面的博客[Retrofit 2 使用指南 ](http://www.heqiangfly.com/2017/10/22/open-source-retrofit-guide/)，熟悉一下它的基本使用。

本文用到的一些类图

![效果图](http://www.plantuml.com/plantuml/svg/bP51IWD144NtTOeYgsGHJp18H1HtGiW5ofu_qgJj6RkhHL5FuYbS2zv6l8PT4jlPPZUy-V_NWzvabQJbBj3EQm0ljj0q3bw_tp--tZuNH3ugqY0EV2uXy3CnRv6dCMPqkrF68rnHB5ULFuo-PyJxWeAbfM_4xIta3jyhUYKN96U-tb-fJfOvW8lVdJ4PEkjbgaSlnLNmT3B_PIlDWtdmKKBhjZj_O9Qnagdq2BWLHJKXOzozhDTpdGQFLIBwN-7k-3xJ1h6lJ_43)

# 初始化请求流程

先来看一下如何创建一个请求。

```
public interface RequestService {
    @GET("hq/urls")
    Call<RequestManager.TestBean> getData();
}
    ......
    
        Retrofit retrofit = new Retrofit.Builder()
                .client(client)
                .baseUrl("http://172.17.137.68/")
                .addConverterFactory(GsonConverterFactory.create())
                //加入对RxJava2的支持
                //.addCallAdapterFactory(RxJava2CallAdapterFactory.create())
                .build();
        mRequestService = retrofit.create(RequestService.class);
```

异步请求

```
    public void getData(CallBack callBack){
        Call<TestBean> call = mMyService.getData();
        call.enqueue(new Callback<TestBean>() {

            @Override
            public void onResponse(Call<TestBean> call, Response<TestBean> response) {
                Log.e("Test","onResponse = "+response.body().toString());
            }

            @Override
            public void onFailure(Call<TestBean> call, Throwable t) {
                Log.e("Test","onFailure = ");
            }
        });
    }
```

同步请求

```
    public TestBean getDataSync(){
        Call<TestBean> call = mRequestService.getData();
        Response<TestBean> data = null;
        try {
            data = call.execute();
        } catch (IOException e) {
            e.printStackTrace();
        }

        return data != null ? data.body() : null;
    }
```

首先这里是创建一个 `Retrofit` 对象，这里用到了建造者模式。
那么就先来看一下 `Builder().build()` 的源码。
先来看一下 `platform` 这个变量，因为后面会用到它。通过 `Builder` 的构造函数，可以看到它是通过 `Platform.get()` 来实例化的，在 Android 平台上是一个 `Platform.Android` 对象。

```
    public Retrofit build() {
      //  `baseUrl` 是必须设置的。
      if (baseUrl == null) {
        throw new IllegalStateException("Base URL required.");
      }

      // callFactory 是 client() 方法设置的，如果不设置，会生成个默认的 OkHttpClient()。
      okhttp3.Call.Factory callFactory = this.callFactory;
      if (callFactory == null) {
        callFactory = new OkHttpClient();
      }

      // callbackExecutor 是callbackExecutor()方法设置的，如果不设置，会添加平台默认的回调执行器
      Executor callbackExecutor = this.callbackExecutor;
      if (callbackExecutor == null) {
        // 这里会返回一个 Android.MainThreadExecutor 对象
        callbackExecutor = platform.defaultCallbackExecutor();
      }

      List<CallAdapter.Factory> adapterFactories = new ArrayList<>(this.adapterFactories);
      // 这里会把 ExecutorCallAdapterFactory 添加到 adapterFactories 中。这个是 Retrofit 内置的适配器。
      // callbackExecutor 为请求结果返回后调用客户端回调的执行器
      adapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));

      // Make a defensive copy of the converters.
      List<Converter.Factory> converterFactories =
          new ArrayList<>(1 + this.converterFactories.size());

      // 添加内置转换器
      converterFactories.add(new BuiltInConverters());
      converterFactories.addAll(this.converterFactories);

      return new Retrofit(callFactory, baseUrl, converterFactories, adapterFactories,
          callbackExecutor, validateEagerly);
    }
```

再来看一下 `Retrofit.create`。
这部分是重点，因为请求方法的执行其实就是执行到了这里动态代理对象的 `invoke()` 方法。后面会分开来介绍。

```
  public <T> T create(final Class<T> service) {
    // InvocationHandler只能支持interface动态代理，这里检查一下service是否是interface
    Utils.validateServiceInterface(service);
    // validateEagerly默认为false，除非通过validateEagerly()方法配置
    if (validateEagerly) {
      //这里会提前创建 ServiceMethod
      eagerlyValidateMethods(service);
    }
    // 返回一个动态代理对象，这里是重点
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, @Nullable Object[] args)
              throws Throwable {
            // 如果是Object的方法直接执行
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            // Android平台这里返回false，如果是平台默认方法，直接执行
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            // 创建 ServiceMethod
            ServiceMethod<Object, Object> serviceMethod =
                (ServiceMethod<Object, Object>) loadServiceMethod(method);
            // 生成一个 Call 对象，OkHttpCall实现了 Call 接口
            OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
            // 调用适配器的adapter生成一个 范型 对象（请求方法定义的返回类型）。
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
  }
```

# 执行请求流程

比如我们Demo中的一个请求方法的执行 `mRequestService.getData()`，其实执行的方法体就是前面的动态代理对象的 `invoke()` 方法，返回一个 `RequestService` 接口中此方法定义的一个范型对象。
先来看一下如何生成 `ServiceMethod`：
`ServiceMethod` 和我们前面定义的请求方法是一一对应关系的，每个 Method 对应一个 `ServiceMethod`。比如就像例子中我们调用 `RequestService.getData()` 方法就会生成一个对应的 `ServiceMethod` 对象，保存了 baseUrl、请求参数、请求类型等信息。
如果缓存里没有，则新建一个并放入到缓存中。至于这个 `ServiceMethod` 是我们后面再详细分析。

```
  ServiceMethod<?, ?> loadServiceMethod(Method method) {
    ServiceMethod<?, ?> result = serviceMethodCache.get(method);
    if (result != null) return result;

    synchronized (serviceMethodCache) {
      result = serviceMethodCache.get(method);
      if (result == null) {
        result = new ServiceMethod.Builder<>(this, method).build();
        serviceMethodCache.put(method, result);
      }
    }
    return result;
  }
```

接着会根据 `ServiceMethod` 对象和参数生成一个 `OkHttpCall` 对象，然后再根据适配器的 adapter 方法把 `OkHttpCall` 转化为 `ExecutorCallbackCall` （使用内置适配器时）。
这部分接下来也会详细介绍。

## ServiceMethod

`ServiceMethod` 方法也比较简单，一些属性包含了一个Http请求的基本信息。

```
final class ServiceMethod<R, T> {
  // Upper and lower characters, digits, underscores, and hyphens, starting with a character.
  static final String PARAM = "[a-zA-Z][a-zA-Z0-9_-]*";
  static final Pattern PARAM_URL_REGEX = Pattern.compile("\\{(" + PARAM + ")\\}");
  static final Pattern PARAM_NAME_REGEX = Pattern.compile(PARAM);

  final okhttp3.Call.Factory callFactory;
  final CallAdapter<R, T> callAdapter;

  private final HttpUrl baseUrl;
  private final Converter<ResponseBody, R> responseConverter;
  private final String httpMethod;
  private final String relativeUrl;
  private final Headers headers;
  private final MediaType contentType;
  private final boolean hasBody;
  private final boolean isFormEncoded;
  private final boolean isMultipart;
  private final ParameterHandler<?>[] parameterHandlers;

  ServiceMethod(Builder<R, T> builder) {
    ......
  }

  Request toRequest(@Nullable Object... args) throws IOException {
    ......
  }

  R toResponse(ResponseBody body) throws IOException {
    ......
  }
}
```

我们重点来看一下 `ServiceMethod.Builder()` 类。

```
    Builder(Retrofit retrofit, Method method) {
      this.retrofit = retrofit;
      this.method = method;
      this.methodAnnotations = method.getAnnotations();
      this.parameterTypes = method.getGenericParameterTypes();
      this.parameterAnnotationsArray = method.getParameterAnnotations();
    }
```

它的构造函数会传入 `Retrofit` 和 `Method` 对象作为参数。

```
    public ServiceMethod build() {
      // 获取该方法对应的适配器
      callAdapter = createCallAdapter();
      responseType = callAdapter.responseType();
      ......
      // 获取该方法对应的转换器
      responseConverter = createResponseConverter();

      // 解析方法注解，根据方法注解调用不同的方法生成请求参数
      for (Annotation annotation : methodAnnotations) {
        parseMethodAnnotation(annotation);
      }

      ......

      int parameterCount = parameterAnnotationsArray.length;
      parameterHandlers = new ParameterHandler<?>[parameterCount];
      for (int p = 0; p < parameterCount; p++) {
        Type parameterType = parameterTypes[p];
        if (Utils.hasUnresolvableType(parameterType)) {
          ......
        }

        Annotation[] parameterAnnotations = parameterAnnotationsArray[p];
        ......
        // 解析参数
        parameterHandlers[p] = parseParameter(p, parameterType, parameterAnnotations);
      }

      ......

      return new ServiceMethod<>(this);
    }
```

先来看一下 `build` 方法的调用流程（使用内置适配器时）：

```
├── ServiceMethod.Builder().build()
    ├── ServiceMethod.Builder().createCallAdapter()
        ├── Retrofit.callAdapter()
            ├── Retrofit.nextCallAdapter()
                ├── ExecutorCallAdapterFactory.get()
                    └──  new CallAdapter()
    ├── ServiceMethod.Builder().createResponseConverter()
    ├── ServiceMethod.Builder().parseMethodAnnotation()
    └── new ServiceMethod<>(this)
```

### 获取适配器

```
  public CallAdapter<?, ?> nextCallAdapter(@Nullable CallAdapter.Factory skipPast, Type returnType,
      Annotation[] annotations) {
    ......

    int start = adapterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = adapterFactories.size(); i < count; i++) {
      // 获取可以处理当前请求的适配器
      CallAdapter<?, ?> adapter = adapterFactories.get(i).get(returnType, annotations, this);
      if (adapter != null) {
        return adapter;
      }
    }

    ......
    throw new IllegalArgumentException(builder.toString());
  }
```

这里会遍历我们添加的所有适配器，并调用适配器的 `get()` 方法，这个方法会根据当前请求方法的返回值类型来决定是否处理。
这里以内置的适配器来介绍一下：
ExecutorCallAdapterFactory.get():

```
  @Override
  public CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
    // 这个适配器工厂只处理返回值为 Call 的请求方法
    if (getRawType(returnType) != Call.class) {
      return null;
    }
    final Type responseType = Utils.getCallResponseType(returnType);
    // 生成一个 CallAdapter
    return new CallAdapter<Object, Call<?>>() {
      @Override public Type responseType() {
        return responseType;
      }
      // 后面在动态代理类里面调用serviceMethod.callAdapter.adapt(okHttpCall)时会生成一个ExecutorCallbackCall对象
      @Override public Call<Object> adapt(Call<Object> call) {
        return new ExecutorCallbackCall<>(callbackExecutor, call);
      }
    };
  }
```

### 获取转换器

```
├── ServiceMethod.createResponseConverter()
    └── Retrofit.responseBodyConverter(Type type, Annotation[] annotations)
        └── Retrofit.nextResponseBodyConverter()
```

```
  public <T> Converter<ResponseBody, T> nextResponseBodyConverter(
      @Nullable Converter.Factory skipPast, Type type, Annotation[] annotations) {
    ......
    int start = converterFactories.indexOf(skipPast) + 1;
    // 这里遍历所有的转换器，比如我们前面添加了 GsonConverterFactory，那么就有两个转换器，还有个默认的内置转换器 BuiltInConverters
    // 我们看源码知道BuiltInConverters只处理 ResponseBody 和 Void两种返回结果，Demo中我们自定义RequestManager.TestBean返回结果，肯定是指定 GsonConverterFactory来作为转换器
    for (int i = start, count = converterFactories.size(); i < count; i++) {
      Converter<ResponseBody, ?> converter =
          converterFactories.get(i).responseBodyConverter(type, annotations, this);
      if (converter != null) {
        return (Converter<ResponseBody, T>) converter;
      }
    }

    ......
    throw new IllegalArgumentException(builder.toString());
  }
```


### 处理方法注解

这一部分主要是根据方法注解的类型解析生成 `ServiceMethod` 的一些属性，比如：`httpMethod`、`hasBody` 和 `relativeUrl` 等。
`ServiceMethod.Builder.parseMethodAnnotation()`：

```
    private void parseMethodAnnotation(Annotation annotation) {
      if (annotation instanceof DELETE) {
        parseHttpMethodAndPath("DELETE", ((DELETE) annotation).value(), false);
      } else if (annotation instanceof GET) {
        parseHttpMethodAndPath("GET", ((GET) annotation).value(), false);
      } else if (annotation instanceof HEAD) {
        parseHttpMethodAndPath("HEAD", ((HEAD) annotation).value(), false);
        if (!Void.class.equals(responseType)) {
          throw methodError("HEAD method must use Void as response type.");
        }
      } else if (annotation instanceof PATCH) {
        ......//省略，具体代码可以看源码
      } else if (annotation instanceof FormUrlEncoded) {
        if (isMultipart) {
          throw methodError("Only one encoding annotation is allowed.");
        }
        isFormEncoded = true;
      }
    }
```

```
    private void parseHttpMethodAndPath(String httpMethod, String value, boolean hasBody) {
      if (this.httpMethod != null) {
        throw methodError("Only one HTTP method is allowed. Found: %s and %s.",
            this.httpMethod, httpMethod);
      }
      this.httpMethod = httpMethod;
      this.hasBody = hasBody;

      ......//省略，具体代码可以看源码

      this.relativeUrl = value;
      this.relativeUrlParamNames = parsePathParameters(value);
    }
```

## OkHttpCall

这部分主要针对动态代理类中 `OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args)` 这段方法的执行。
前面我们说过，适配器处理对网络请求（内置适配器是生成的 Call 对象）都是委托给 OkHttpCall 来处理的。
先来看一下这个类：

```
final class OkHttpCall<T> implements Call<T> {
  private final ServiceMethod<T, ?> serviceMethod;
  private final @Nullable Object[] args;

  private volatile boolean canceled;

  @GuardedBy("this")
  private @Nullable okhttp3.Call rawCall;
  @GuardedBy("this")
  private @Nullable Throwable creationFailure; // Either a RuntimeException or IOException.
  @GuardedBy("this")
  private boolean executed;

  OkHttpCall(ServiceMethod<T, ?> serviceMethod, @Nullable Object[] args) {
    this.serviceMethod = serviceMethod;
    this.args = args;
  }
  @Override public void enqueue(final Callback<T> callback) {
    checkNotNull(callback, "callback == null");

    okhttp3.Call call;
    Throwable failure;

    synchronized (this) {
      if (executed) throw new IllegalStateException("Already executed.");
      executed = true;

      call = rawCall;
      failure = creationFailure;
      if (call == null && failure == null) {
        try {
          call = rawCall = createRawCall();
        } catch (Throwable t) {
          failure = creationFailure = t;
        }
      }
    }

    if (failure != null) {
      callback.onFailure(this, failure);
      return;
    }

    if (canceled) {
      call.cancel();
    }

    call.enqueue(new okhttp3.Callback() {
      @Override public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse)
          throws IOException {
        Response<T> response;
        try {
          response = parseResponse(rawResponse);
        } catch (Throwable e) {
          callFailure(e);
          return;
        }
        callSuccess(response);
      }

      @Override public void onFailure(okhttp3.Call call, IOException e) {
        callFailure(e);
      }

      private void callFailure(Throwable e) {
        try {
          callback.onFailure(OkHttpCall.this, e);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }

      private void callSuccess(Response<T> response) {
        try {
          callback.onResponse(OkHttpCall.this, response);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }
    });
  }

  @Override public Response<T> execute() throws IOException {
    okhttp3.Call call;

    synchronized (this) {
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
}
```
`execute()` 对应的就是同步请求方法。
主要来分析一下 `enqueue()` 方法，这个方法对应的就是异步请求方法。请求加入队列的方法最终也是由 `okhttp3.Call` 来执行的。那么就要来看一下 `okhttp3.Call` 是如何生成的。

```
  private okhttp3.Call createRawCall() throws IOException {
    // 调用 ServiceMethod.toRequest() 方法生成请求
    Request request = serviceMethod.toRequest(args);
    // 生成 okhttp3.Call
    okhttp3.Call call = serviceMethod.callFactory.newCall(request);
    if (call == null) {
      throw new NullPointerException("Call.Factory returned null.");
    }
    return call;
  }
```

`serviceMethod.callFactory.newCall(request)` 会调用 `Retrofit.callFactory` 来生成请求，`Retrofit.callFactory` 是通过 `Retrofit.client(OkHttpClient client)` 方法来设置的。
因此， `okhttp3.Call` 是最终由 `OkHttpClient` 来创建的，那么请求的执行也是由 `OkHttpClient` 来执行的。 

## CallAdapter 和 CallAdapter.Factory

`CallAdapter`是一个接口，前文所说的适配器都是实现了这个接口。`ExecutorCallAdapterFactory` 的匿名内部类和 `RxJava2CallAdapter`。
这里再来介绍一下 `CallAdapter.Factory ` 类。它是 `CallAdapter` 的工厂类，是一个抽象类，`ExecutorCallAdapterFactory` 和 `RxJava2CallAdapterFactory` 都是继承这个类。

```
public interface CallAdapter<R, T> {
  // 从http的响应数据转换成java类时用到的数据的类型 如Call<T> 中的 T
  // 这个 T 会作为Converter.Factory.responseBodyConverter 的第一个参数
  Type responseType();

  // 这个方法就是动态代理类中执行的方法，委托Call生成一个T对象	
  T adapt(Call<R> call);

  // 用于向Retrofit提供CallAdapter的工厂类
  abstract class Factory {
    // 在这个方法中判断是否是我们支持的类型，returnType 即Call<Requestbody>和Observable<Requestbody>
    // RxJavaCallAdapterFactory 就是判断returnType是不是Observable<?> 类型
    // 不支持时返回null
    public abstract @Nullable CallAdapter<?, ?> get(Type returnType, Annotation[] annotations,
        Retrofit retrofit);

    // 用于根据index获取泛型的参数 如 Call<Requestbody> 中 index为0时的Requestbody，
    // index为 1 时 Map<String, ? extends Runnable> 的 Runnable
    protected static Type getParameterUpperBound(int index, ParameterizedType type) {
      return Utils.getParameterUpperBound(index, type);
    }

    // 用于获取泛型的原始类型 如 Call<Requestbody> 中的 Call
    // 上面的get方法需要使用该方法。
    protected static Class<?> getRawType(Type type) {
      return Utils.getRawType(type);
    }
  }
}
```

## 适配器处理请求流程

适配器处理请求是在动态代理类的 `serviceMethod.callAdapter.adapt(okHttpCall)` 方法处理。我们前面已经介绍过，使用使用内置适配器时，会由 `ExecutorCallAdapterFactory.get()` 方法生成的 `CallAdapter` 对象的 `adapt()` 方法执行。

```
      @Override public Call<Object> adapt(Call<Object> call) {
        return new ExecutorCallbackCall<>(callbackExecutor, call);
      }
```

这个方法返回了 `ExecutorCallAdapterFactor.ExecutorCallbackCall` 对象，以 `Android.MainThreadExecutor` 和 `OkHttpCall` 来作为参数。

```
    ExecutorCallbackCall(Executor callbackExecutor, Call<T> delegate) {
      this.callbackExecutor = callbackExecutor;
      this.delegate = delegate;
    }
```

在创建请求流程一节中我们展示的Demo中：

```
        Call<TestBean> call = mRequestService.getData();
        call.enqueue(new Callback<TestBean>() {

            @Override
            public void onResponse(Call<TestBean> call, Response<TestBean> response) {
                Log.e("Test","onResponse = "+response.body().toString());
            }

            @Override
            public void onFailure(Call<TestBean> call, Throwable t) {

            }
        });
```

`mRequestService.getData()` 返回的其实就是 `ExecutorCallAdapterFactor.ExecutorCallbackCall` 对象，后面的 `call.enqueue` 方式其实就是调用的 `ExecutorCallAdapterFactor.ExecutorCallbackCall.enqueue()` 方法。
返回相应结果时，需要调用客户端设置的回调方法，这个回调方法是在客户端设置的回调执行器中调用的，如果没有设置，那么默认就是 `Android.MainThreadExecutor`。

```
    @Override public void enqueue(final Callback<T> callback) {
      checkNotNull(callback, "callback == null");

      delegate.enqueue(new Callback<T>() {
        @Override public void onResponse(Call<T> call, final Response<T> response) {
          // 结果的执行由客户端设置的回调执行器来执行。
          callbackExecutor.execute(new Runnable() {
            @Override public void run() {
              if (delegate.isCanceled()) {
                // Emulate OkHttp's behavior of throwing/delivering an IOException on cancellation.
                callback.onFailure(ExecutorCallbackCall.this, new IOException("Canceled"));
              } else {
                callback.onResponse(ExecutorCallbackCall.this, response);
              }
            }
          });
        }

        @Override public void onFailure(Call<T> call, final Throwable t) {
          callbackExecutor.execute(new Runnable() {
            @Override public void run() {
              callback.onFailure(ExecutorCallbackCall.this, t);
            }
          });
        }
      });
    }
```

这里我们可以看到，`ExecutorCallbackCall` （使用内置适配器时）中相关的请求操作其实都是委托给 `OkHttpCall` 来执行的。执行的结果通过 `Android.MainThreadExecutor` 在主线程中执行回调函数。
具体的执行流程请看上面一节。

## 处理响应结果

这部分比较简单，主要是调用转换器的 `convert()` 方法把消息体转换成我们想要的数据类。`Response<T>` 这里用到了范型，是在定义请求方法时我们指定的类，比如 `RequestManager.TestBean`。
这里以我们前面Demo中定义的 `GsonConverterFactory` 转换器来看一下执行流程：

```
├── OkHttpCall.parseResponse()
    └── ServiceMethod.toResponse()
        └── GsonResponseBodyConverter.convert()
```

`OkHttpCall.parseResponse()`：

```
  Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
    ResponseBody rawBody = rawResponse.body();

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
      rawBody.close();
      return Response.success(null, rawResponse);
    }

    ExceptionCatchingRequestBody catchingBody = new ExceptionCatchingRequestBody(rawBody);
    try {
      T body = serviceMethod.toResponse(catchingBody);
      return Response.success(body, rawResponse);
    } catch (RuntimeException e) {
      catchingBody.throwIfCaught();
      throw e;
    }
  }
```

至此，Retrofit 数据请求流程部分源码解析已经完成。




