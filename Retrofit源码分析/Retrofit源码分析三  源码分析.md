# Retrofit源码分析三 源码分析

### 使用方法

我们先来看一下Retrofit的常见使用方法：

```
//创建网络请求接口类
public interface GitHubService {
  @GET("users/{user}/repos")
  Call<List<Repo>> listRepos(@Path("user") String user);
}

//创建Retrofit实例对象
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://api.github.com/")
    .build();

//通过动态代理创建网络接口代理对象
GitHubService service = retrofit.create(GitHubService.class);

//获取Call对象
Call<List<Repo>> repos = service.listRepos("octocat");

//执行同步请求或异步请求
repos.execute();
repos.enqueue(callback)
```

上面是Retrofit的最基本使用方法，当然现在使用最多的还是RxJava2+Retrofit搭配使用，关于RxJava2，大家可以看我的另一篇 [RxJava2源码分析](https://github.com/BlackFlagBin/blackflagbin.github.blog/blob/master/RxJava2%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/RxJava2%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.md) ，当然RxJava2与Retrofit搭配使用的解析我会在稍后分析，这里我们先关注最基本的使用方法。


### 创建网络接口类

这一步的目的就是封装我们网络请求相关的一些参数，没什么好多说的。


### 创建Retrofit实例对象

Retrofit实例对象的创建很明显是采用了Builder模式，Builder模式在 **Java语言中** 创建一个 **有很多可选配置参数** 的对象的时候是很好的一种设计模式。Builder模式有两个重点，一个是在 **Java语言中** 中，另一个是 **有很多可选配置参数**，其实在现在的Android开发中，使用Kotlin开发Android已经很普遍了，熟悉Kotlin语法的小伙伴可能很熟悉了，由于Kotlin中 **默认参数** 的存在，所以在Kotlin中使用Builder模式的意义不大。但由于Java语法的限制，在创建一个 **有很多可选配置参数** 的时候，Builder模式还是首要选择。

我们来看一下Retrofit中的成员变量：

```
public final class Retrofit {
  //缓存封装好的ServiceMethod
  private final Map<Method, ServiceMethod<?, ?>> serviceMethodCache = new ConcurrentHashMap<>();
  //OKHttp中的网络请求工厂
  final okhttp3.Call.Factory callFactory;
  //BaseUrl
  final HttpUrl baseUrl;
  //数据转换器工厂集合
  final List<Converter.Factory> converterFactories;
  //网络请求适配器工厂集合
  final List<CallAdapter.Factory> callAdapterFactories;
  //处理线程切换
  final @Nullable Executor callbackExecutor;
  //不需要关注，默认为false
  final boolean validateEagerly;
  
  //忽略无关代码......
  }
```

**serviceMethodCache** 是一个HashMap，它的Key是Method，代表我们定义的网络请求接口类中的方法，它的Value是ServiceMethod，代表对网络请求接口类中的方法的一个封装，简单看一下它就明白了：

```
final class ServiceMethod<R, T> {

  private final okhttp3.Call.Factory callFactory;
  private final CallAdapter<R, T> callAdapter;

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
  
  //忽略无关代码.......
  }
```

很明显了，ServiceMethod就是对网络请求参数的封装类，包含请求头、相对url、GET请求或者是POST请求等等对请求的配置信息。**serviceMethodCache** 就是一个对ServiceMethod的缓存，可以提高一定的效率。

在Retrofit中，默认的数据转换器工厂就是 **GsonConverterFactory** ，所以其实如果我们是使用 **Gson** 来做数据转换的，其实没有必要去配置。在Android平台，Retrofit中默认的网络请求适配器工厂是 **ExecutorCallAdapterFactory** ，我们简单瞅一眼它是如何被创建的：

```
static class Android extends Platform {
    @Override public Executor defaultCallbackExecutor() {
      return new MainThreadExecutor();
    }

    @Override CallAdapter.Factory defaultCallAdapterFactory(@Nullable Executor callbackExecutor) {
      if (callbackExecutor == null) throw new AssertionError();
      return new ExecutorCallAdapterFactory(callbackExecutor);
    }

    static class MainThreadExecutor implements Executor {
      //注意这里创建了一个位于主线程的Handler
      private final Handler handler = new Handler(Looper.getMainLooper());

      @Override public void execute(Runnable r) {
        //通过handler.post(runnable)实现线程切换
        handler.post(r);
      }
    }
  }
```

可以看到，在创建 **ExecutorCallAdapterFactory** 的同时传入了一个 **callbackExecutor** ，这个 **callbackExecutor**  也是Retrofit中默认的 **callbackExecutor** ，在Android平台中它是 **MainThreadExecutor**类型 ，可以看到，在它的内部创建了一个位于主线程的Handler。我们知道，使用Retrofit的时候不同于直接使用OKHttp，在使用Retrofit的异步网络请求时，网络结果的回调是位于主线程中的，那么Retrofit是如何切换的线程，看过上面的代码应该会心里有个数了，它是通过 **handler.post(r)** 来实现由子线程到主线程的切换。


### 通过动态代理创建网络接口代理对象

看过我的Retrofit源码分析第二篇：代理模式的小伙伴们应该都比较清楚了，在Retrofit的 **create** 方法中其实是使用了动态代理生成了一个代理对象，现在我们就来看一下 **create** 的源码：

```
  public <T> T create(final Class<T> service) {
    //忽略无关代码......
    
    //下面就是JDK给我们提供的动态代理了
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, @Nullable Object[] args)
              throws Throwable {
            //忽略无关代码......
            
            //获取method的网络请求参数的封装类ServiceMethod对象
            ServiceMethod<Object, Object> serviceMethod =
                (ServiceMethod<Object, Object>) loadServiceMethod(method);
                
            //获取对OKHttp中的RealCall的一个包装类OkHttpCall对象
            OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
            //通过网络请求适配器将原始的Call对象转换成需要的对象，比如RxJavaCallAdapter会将Call对象转换成Observable。
            return serviceMethod.adapt(okHttpCall);
          }
        });
  }
```

在 **create** 方法中做了3件事：
* 封装网络请求方法
* 封装原始Call对象
* 通过CallAdapter转换Call对象
封装网络请求方法很容易理解，关于ServiceMethod上面已经说过，不再赘述。封装原始Call对象，对OKHttp源码熟悉的小伙伴应该知道，原始的Call其实就是OkHttp中的ReallCall。如果不熟悉OkHttp的同学可以看我之前的一篇  [OkHttp源码分析](https://github.com/BlackFlagBin/blackflagbin.github.blog/blob/master/OkHttp%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/OkHttp%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.md) 。我们可以简单看一下这个包装类 **OkHttpCall** :

```
final class OkHttpCall<T> implements Call<T> {

    //忽略无关代码......    
    
    @Override public Response<T> execute() throws IOException {
    okhttp3.Call call;

    //熟悉OkHttp源码的同学应该很熟悉了，完全照抄OkHttp中的代码，确保每个Call只会被执行一次
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already executed.");
      executed = true;

      //忽略无关代码......
    
      call = rawCall;
      if (call == null) {
        try {
          //创建OkHttp中的RealCall对象
          call = rawCall = createRawCall();
        } catch (IOException | RuntimeException | Error e) {
          throwIfFatal(e); //  Do not assign a fatal error to creationFailure.
          creationFailure = e;
          throw e;
        }
      }
    }

    if (canceled) {
      call.cancel();
    }
    //解析OkHttp中的RealCall执行同步方法后返回的网络数据
    return parseResponse(call.execute());
  }
  
  
  @Override public void enqueue(final Callback<T> callback) {
    checkNotNull(callback, "callback == null");

    okhttp3.Call call;
    Throwable failure;

    //确保一个Call对象只被执行一次
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already executed.");
      executed = true;

      call = rawCall;
      failure = creationFailure;
      if (call == null && failure == null) {
        try {
          //创建OkHttp中的RealCall对象
          call = rawCall = createRawCall();
        } catch (Throwable t) {
          throwIfFatal(t);
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
    
    //调用OkHttp中的RealCall的异步请求网络方法
    call.enqueue(new okhttp3.Callback() {
      @Override public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse) {
        Response<T> response;
        try {
          response = parseResponse(rawResponse);
        } catch (Throwable e) {
          callFailure(e);
          return;
        }

        try {
          callback.onResponse(OkHttpCall.this, response);
        } catch (Throwable t) {
          t.printStackTrace();
        }
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
    });
  }
}
```

可以看到，Retrofit中的这个 **OKHttpCall** 完全是对OkHttp中的 **RealCall** 的一个包装。在 **OKHttpCall** 中的同步网络方法 **execute** 和异步网络方法 **enQueue** 中其实就做了三件事：获取 **RealCall** 对象，调用 **RealCall** 中的 **execute** 或者  **enQueue** 方法，然后去解析OkHttp返回的原始数据并转换成我们需要的类型，就这么简单。

接下来就 **create** 方法中就只剩下 **通过CallAdapter转换Call对象** 了，我们上面说过，在没有特别配置CallAdapter的时候，默认的CallAdapterFactory是 **ExecutorCallAdapterFactory** ，很明显，CallAdapter是由Factory创建的，那我们看一下这个默认的CallAdapterFactory：

```
final class ExecutorCallAdapterFactory extends CallAdapter.Factory {
    
  //本质是传入的MainThreadExecutor
  final Executor callbackExecutor;

  ExecutorCallAdapterFactory(Executor callbackExecutor) {
    this.callbackExecutor = callbackExecutor;
  }

  @Override
  public CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
    if (getRawType(returnType) != Call.class) {
      return null;
    }
    final Type responseType = Utils.getCallResponseType(returnType);
    
    //直接创建并返回一个CallAdapter的匿名对象
    return new CallAdapter<Object, Call<?>>() {
      @Override public Type responseType() {
        return responseType;
      }

      @Override public Call<Object> adapt(Call<Object> call) {
        //返回默认的CallAdapter转换后的Call对象
        return new ExecutorCallbackCall<>(callbackExecutor, call);
      }
    };
  }

  static final class ExecutorCallbackCall<T> implements Call<T> {
    //本质是传入的MainThreadExecutor
    final Executor callbackExecutor;
    //就是OKHttpCall，从名字也能理解，代理Call对象
    final Call<T> delegate;

    ExecutorCallbackCall(Executor callbackExecutor, Call<T> delegate) {
      this.callbackExecutor = callbackExecutor;
      this.delegate = delegate;
    }

    @Override public void enqueue(final Callback<T> callback) {
      checkNotNull(callback, "callback == null");

       //调用OkHttp中RealCall的异步网络请求
      delegate.enqueue(new Callback<T>() {
        @Override public void onResponse(Call<T> call, final Response<T> response) {
            //注意，OkHttp的异步回调是在子线程的，Retrofit这里实现了线程的切换，本质是调用了handler.post(runnable)
          callbackExecutor.execute(new Runnable() {
            @Override public void run() {
              if (delegate.isCanceled()) {
                callback.onFailure(ExecutorCallbackCall.this, new IOException("Canceled"));
              } else {
                callback.onResponse(ExecutorCallbackCall.this, response);
              }
            }
          });
        }

        @Override public void onFailure(Call<T> call, final Throwable t) {
            //注意，OkHttp的异步回调是在子线程的，Retrofit这里实现了线程的切换，本质是调用了handler.post(runnable)
          callbackExecutor.execute(new Runnable() {
            @Override public void run() {
              callback.onFailure(ExecutorCallbackCall.this, t);
            }
          });
        }
      });
    }

    @Override public boolean isExecuted() {
      return delegate.isExecuted();
    }

    @Override public Response<T> execute() throws IOException {
       //由于同步方法不需要切线程，所以直接执行并返回OKHttpCall的同步网络方法
      return delegate.execute();
    }

    @Override public void cancel() {
      delegate.cancel();
    }

    @Override public boolean isCanceled() {
      return delegate.isCanceled();
    }

    @SuppressWarnings("CloneDoesntCallSuperClone") // Performing deep clone.
    @Override public Call<T> clone() {
      return new ExecutorCallbackCall<>(callbackExecutor, delegate.clone());
    }

    @Override public Request request() {
      return delegate.request();
    }
  }
}
```

上面的代码其实也很清晰了 **ExecutorCallAdapterFactory** 这个CallAdapter工厂类直接创建并返回一个CallAdapter的匿名对象，这个其实就是我们Retrofit的默认CallAdapter，关键是这个CallAdapter的adapt方法，它返回了 **ExecutorCallbackCall** 这个对OkHttpCall的包装类，我们关注这个包装类的 **enqueue** 方法，在这个方法中，它通过调用 `callbackExecutor.execute` 来实现了子线程到主线程的线程切换。这个 **callbackExecutor** 就是 **MainThreadExecutor** ，这个类我们上面提到过， **MainThreadExecutor** 中的 **execute** 方法内部就是 `handler.post(r);` ，这个 **handler** 其实就是主线程的，因此实现了线程切换。


### 获取Call对象并执行同步或异步方法
其实经过上一步的分析，我们已经知道了，我们获取的Call对象，其实是通过动态代理中 `serviceMethod.adapt(okHttpCall)` 返回的Call对象，这个其实就是 **ExecutorCallbackCall** ，这个我们都很清楚了，它是对OkHttp中RealCall的一个包装类，在它的异步方法中通过调用 `callbackExecutor.execute` 实现了线程的切换，当然本质还是通过 **Handler** 机制。

其实到此为止，基本的流程已经分析完毕了，我们到这里也会很清楚Retrofit的定位：**Retrofit并不是一个网络请求的框架，而是一封装网络请求参数，解析网络返回结果的框架**。下面我们来分析一下使用了RxJava2CallAdapter的情况。

### RxJava2CallAdapter的原理

CallAdapter的重点是它的adapt方法，我们来看一下RxJava2CallAdapter中的adapt方法：

```
@Override public <R> Object adapt(Call<R> call) {
    //创建一个发射网络返回结果的Observable
    Observable<Response<R>> responseObservable = new CallObservable<>(call);

    Observable<?> observable;
    if (isResult) {
      observable = new ResultObservable<>(responseObservable);
    } else if (isBody) {
      observable = new BodyObservable<>(responseObservable);
    } else {
      observable = responseObservable;
    }
    
    //默认scheduler为空
    if (scheduler != null) {
      observable = observable.subscribeOn(scheduler);
    }

    if (isFlowable) {
      return observable.toFlowable(BackpressureStrategy.LATEST);
    }
    if (isSingle) {
      return observable.singleOrError();
    }
    if (isMaybe) {
      return observable.singleElement();
    }
    if (isCompletable) {
      return observable.ignoreElements();
    }
    return observable;
  }
```

这段代码本质是创建一个发射网络返回结果的Observable，我们看一下 **CallObservable** 的内部实现，看过RxJava2源码的同学应该都知道，Observable的关键方法是 **subscribeActual**，我们就看一下这个方法内部都做了什么：

```
@Override protected void subscribeActual(Observer<? super Response<T>> observer) {
    // Since Call is a one-shot type, clone it for each new observer.
    Call<T> call = originalCall.clone();
    observer.onSubscribe(new CallDisposable(call));

    boolean terminated = false;
    try {
      //调用了OKHttpCall的同步网络访问方法，并获取网络数据
      Response<T> response = call.execute();
      if (!call.isCanceled()) {
        //将获取到的网络数据发送到观察者observer
        observer.onNext(response);
      }
      if (!call.isCanceled()) {
        terminated = true;
        //发送结束事件
        observer.onComplete();
      }
    } catch (Throwable t) {
      Exceptions.throwIfFatal(t);
      if (terminated) {
        RxJavaPlugins.onError(t);
      } else if (!call.isCanceled()) {
        try {
          //发送错误事件
          observer.onError(t);
        } catch (Throwable inner) {
          Exceptions.throwIfFatal(inner);
          RxJavaPlugins.onError(new CompositeException(t, inner));
        }
      }
    }
  }
```

熟悉RxJava2的同学看完就应该懂了：RxJava2CallAdapter通过adapt方法生成并返回一个 **CallObservable** 对象，在这个 **CallObservable** 内部通调用 **OKHttpCall** 的 **execute()** 方法进行网络访问，并将获取到的数据发送到下一级的观察者observer中。

到此为止，Retrofit的源码分析终于结束了，其实它并不难，但想要完全理解整个网络访问流程，除了要明白Retrofit，还需要了解OKHttp甚至是RxJava，对后两者不太熟悉的小伙伴们可以看我的另外两篇源码分析：[OkHttp源码分析](https://github.com/BlackFlagBin/blackflagbin.github.blog/blob/master/OkHttp%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/OkHttp%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.md) ， [RxJava2源码分析](https://github.com/BlackFlagBin/blackflagbin.github.blog/blob/master/RxJava2%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/RxJava2%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.md) ，在熟悉了OKHttp和RxJava2的基础上再回过头来看Retrofit，相信你会有更深的理解。












