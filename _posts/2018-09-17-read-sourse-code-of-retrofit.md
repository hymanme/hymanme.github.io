---
layout: post
title: Retrofit 源码浅析
summary: 软件开发中必不可少的会牵扯到网络请求，紧接着网络请求框架应用而生。他们高度封装，方便了开发者进行网络请求，完完全全当个伸手党就可以了。而本文着重讲一下网络请求封装库--Retrofit 的使用以及从源码角度分析其实现原理。
date: 2018-09-17 11:30:20
categories: Android
tags: [Retrofit, Source Code]
featured-img: beach
---

软件开发中必不可少的会牵扯到网络请求，紧接着网络请求框架应用而生，诸如`Volley`、`Android-Async-Http`、`OkHttp`、`Retrofit`等。他们高度封装，方便了开发者进行网络请求，完完全全当个伸手党就可以了。而本文着重讲一下网络请求封装库--Retrofit 的使用以及从源码角度分析其实现原理。

> 本文介绍的 Retrofit 为 Retrofit2.x 版

## Retrofit 是什么？

官方介绍：

> Type-safe HTTP client for Android and Java by Square, Inc.

一款 Square 公司开发的专为 Android 和 Java 提供类型安全的HTTP客户端的封装库。简单地说就是一个基于 OkHttp 的 Restfull 友好的高度封装的 API 请求封装库，它以其简易的接口配置、强大的扩展支持、优雅的代码结构等优点迅速被众多开发者锁追捧，Retrofit 使得调用每一个 API 接口请求都变得极其简单快捷。从上面的解释你可能了解到，其实真正的网络请求并不是 Retrofit 来处理的，而是交给了底层的 OkHttp，二者分工协作。

```sequence
title: 
Application -> Retrofit: call Api
Retrofit -> OkHttp: adapter
OkHttp --> Server: http request
Server --> OkHttp: http response
OkHttp -> Retrofit: resouse
Retrofit -> Application: callback
```

从上面的图可以直观的看体会到 Retrofit 和 OkHttp 的分工协作，更能体会到 **Retrofitting OkHttp** 的含义了，也许 Square 公司开发 Retrofit 就是用来改造 OkHttp，使用请求更加方便而已。

> Retrofit 译为 改造

## 如何使用？

Retrofit 使用起来也很简单，面向注解的接口编写，首先添加依赖

```java
implementation 'com.squareup.retrofit2:retrofit:2.4.0'
```

定义一个接口，如 GitHub 接口

```java
public interface GithubService {
  @GET("users/{user}/repos")
  Call<List<Repo>> getUserRepos(@Path("user") String user);
}
```

这里使用的 Restfull 风格的 API 接口请求，当我们调用这个接口时，会去请求以下接口

```java
https://api.github.com/users/{user}/repos
```

其中`{user}`会被替换为 path 路径传入的 user 变量，使用的是注解 `@Path`来指定。

接口已经有了，接下来就需要构建我们的 Retrofit 了

```java
Retrofit mRetrofit = new Retrofit.Builder()
    .client(getOkHttpClient())
    .baseUrl(baseUrl)
    .addConverterFactory(GsonConverterFactory.create())//添加数据转换器
    .addCallAdapterFactory(RxJava2CallAdapterFactory.create())//添加请求执行Adapter
    .build();
```

开始使用

```java
GithubService api = retrofit.create(GithubService.class);
Call<List<Repo>> repos = api.getUserRepos("hymanme");
```

这里的 repos 并不是最终的数据集，而是一个 `Call` 接口，后面我们会发现，这个 `Call` 其实就是 OkHttp 中的 `Call` 接口，我们可以调用其方法来进行发起请求

```java
// 同步调用
List<Repo> data = repos.execute(); 

// 异步调用
repos.enqueue(new Callback<List<Repo>>() {
    @Override
    public void onResponse(Call<List<Repo>> call, Response<List<Repo>> response) {
    	List<Repo> data = response.body();
    }

    @Override
    public void onFailure(Call<List<Repo>> call, Throwable t) {
    	t.printStackTrace();
    }
});
```

好了，至此一个接口就一件请求完毕，是不是觉得特别简单？当然 Retrofit 还有很多可定制的内容，以及很多细节方面的设置等，比如 Url 配置、请求参数设置、各种注解的使用、Converter、CallAdapter 等等，这里暂时不详细介绍了，详情请移步 [Retrofit Document](https://square.github.io/retrofit/) 。

## 源码剖析

先不急的一梭子分析，我们先提几个问题，然后带着问题去看源码，通过源码找出问题的答案，这样才更加深刻。

那么问题来了

* `GithubService api = retrofit.create(GithubService.class)`都做了些什么？
* Retrofit 是如何委托 okhttp 来处理网络请求的？
* Converter 和 CallAdapter 是如何做到像插件一样即插即用？
* Retrofit 是如何工作的？
* 。。。

### 创建代理对象

这里是用户创建一个接口代理对象，实际实现交给代理来处理，调用者只关心调用接口可以得到请求结果。

```java
@SuppressWarnings("unchecked") // Single-interface proxy creation guarded by parameter safety.
public <T> T create(final Class<T> service) {
  Utils.validateServiceInterface(service);
  if (validateEagerly) {
    eagerlyValidateMethods(service);
  }
  return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
      new InvocationHandler() {
        private final Platform platform = Platform.get();

        @Override public Object invoke(Object proxy, Method method, @Nullable Object[] args)
            throws Throwable {
          // If the method is a method from Object then defer to normal invocation.
          if (method.getDeclaringClass() == Object.class) {
            return method.invoke(this, args);
          }
          if (platform.isDefaultMethod(method)) {
            return platform.invokeDefaultMethod(method, service, proxy, args);
          }
          ServiceMethod<Object, Object> serviceMethod =
              (ServiceMethod<Object, Object>) loadServiceMethod(method);
          OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
          return serviceMethod.callAdapter.adapt(okHttpCall);
        }
      });
}
```

整个`create()`方法 30行代码不到，使用了动态代理模式，我们知道`GithubService`是一个接口，我们创建的肯定不是一个接口，因为我们根本没实现里面的方法，是无法完成我们想要的功能的。那么创建的这个`api`对象究竟是什么呢？

实际上 Retrofit 创建的并不是 Service 接口的实现类对象，而是一个 Service 接口的代理对象，当我们创建好了 Service 对象调用其方法时，会被 `InvocationHandler`的`invoke()`方法拦截，也就是最终会执行到`invoke()`方法。关于动态代理详解请移步[动态代理](http://www.cnblogs.com/xiaoluo501395377/p/3383130.html)

为了直观了解流程，我们先精简一下代码，因为上面的代码有些是做验证性的代码，比如参数验证、是否提前初始化`ServiceMethod`等等，精简后：

```java
public <T> T create(final Class<T> service) {
  return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
      new InvocationHandler() {
        @Override 
        public Object invoke(Object proxy, Method method, @Nullable Object[] args)
            throws Throwable {
          ServiceMethod<Object, Object> serviceMethod =
              (ServiceMethod<Object, Object>) loadServiceMethod(method);
          OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
          return serviceMethod.callAdapter.adapt(okHttpCall);
        }
      });
}
```

当我们调用`api.getUserRepos("hymanme");`时，实际上会执行到`invoke`方法。会传递三个参数

* proxy：api
* method：getUserRepos() 方法
* args：["hymane"]数组

Retrofit 需要用到了 method 和 args 两个参数，其中 method 主要用来解析注解，然后根据注解去组装一个完整的 Request 请求，args 则是填充请求参数。

#### 1. 加载 ServiceMethod

这里是获取（从缓存取或者创建）一个 ServiceMethod 对象，封装了一个请求的所有必要信息，如请求 url、参数、请求头、请求方式等等，方便后面通过 adapter 交给 OkHttp 来处理。

首先调用`loadServiceMethod()`方法通过传入的`method`参数获取到一个`ServiceMethod`对象，我们暂且不去细究它是用来干嘛的，只需要知道，这个`ServiceMethod`用来组装一个请求用的。

```java
ServiceMethod<Object, Object> serviceMethod =
              (ServiceMethod<Object, Object>) loadServiceMethod(method);
```

进 `ServiceMethod` 类，看看这里`loadServiceMethod()`方法都做了些什么？

```java
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

serviceMethodCache 是一个线程安全的Map ConcurrentHashMap。 

```java
private final Map<Method, ServiceMethod<?, ?>> serviceMethodCache = new ConcurrentHashMap<>();
```

这里用 serviceMethodCache 做了一个缓存，为了提高加载速度，避免了每次接口调用都去生成 ServiceMethod 对象，这是一个较为耗时的操作。

首先从 cache 中查找对应 method 的封装对象 ServiceMethod ，找到直接返回，没找到就创建。这里用到了双重锁定法确保了再多线程下`result`的原子性，类似双重锁定单例模式。创建好了`ServiceMethod`对象之后，将其添加到缓存中方便下次快速访问，然后方法返回。

ServiceMethod 是如何创建的？

```java
new ServiceMethod.Builder<>(this, method).build()
```

构造方法

```java
Builder(Retrofit retrofit, Method method) {
  this.retrofit = retrofit;
  this.method = method;
  this.methodAnnotations = method.getAnnotations();
  this.parameterTypes = method.getGenericParameterTypes();
  this.parameterAnnotationsArray = method.getParameterAnnotations();
}
```

build() 方法

```java
public ServiceMethod build() {
  //①创建 CallAdapter
  callAdapter = createCallAdapter();
  responseType = callAdapter.responseType();
  //返回类型校验
  if (responseType == Response.class || responseType == okhttp3.Response.class) {
    throw methodError("'"
        + Utils.getRawType(responseType).getName()
        + "' is not a valid response body type. Did you mean ResponseBody?");
  }
  //②创建数据转换器
  responseConverter = createResponseConverter();
  //③解析接口注解，比如 header、url、parameter、method 等等
  for (Annotation annotation : methodAnnotations) {
    parseMethodAnnotation(annotation);
  }
  int parameterCount = parameterAnnotationsArray.length;
    parameterHandlers = new ParameterHandler<?>[parameterCount];
    for (int p = 0; p < parameterCount; p++) {
        Type parameterType = parameterTypes[p];
        if (Utils.hasUnresolvableType(parameterType)) {
            throw parameterError(p, "Parameter type must not include a type variable or wildcard: %s",parameterType);
        }
        Annotation[] parameterAnnotations = parameterAnnotationsArray[p];
        if (parameterAnnotations == null) {
            throw parameterError(p, "No Retrofit annotation found.");
        }
        parameterHandlers[p] = parseParameter(p, parameterType, parameterAnnotations);
  }
  //以下省略一些参数验证代码，主要验证接口注解是否合法，比如是否指代了 url 等
  //。。。
  return new ServiceMethod<>(this);//将Build作为参数创建ServiceMethod对象
}
```