---
layout: post
title: 使用泛型反射搭配 retrofit 让你一个方法搞定所有接口
summary: 本篇主要讲的是我们日常使用 retrofit 处理 restful 接口时，需要频繁定义很多 service 接口，带着这个问题进行了一次深入的思考并最终优化了传统使用方式，做到一个接口方法搞定所有 api 请求。需要我们了解到 Java 的泛型，反射等使用。
date: 2019-05-14 20:18:30
categories: Android
tags: [Android, retrofit, 反射, java泛型]
featured-img: beach
---

本篇主要讲的是我们日常使用 retrofit 处理 restful 接口时，需要频繁定义很多 service 接口，带着这个问题进行了一次深入的思考并最终优化了传统使用方式，做到一个接口方法搞定所有 api 请求。需要我们了解到 Java 的泛型，反射等使用。

## 引言

Retroft 是一个十分实用的封装库，它将我们接口请求做了一次高度的封装，使得我们可以更加方便、清晰的去处理接口请求，尤其是面对 Restful 风格的接口，Retrofit 就再适合不过了。Retroft 的详细介绍就不赘述了，有想了解源码的可以移步[Retrofit 源码浅析](https://hymane.itscoder.com/read-sourse-code-of-retrofit)。

## 我们目前是如何做的?

在使用 Retrofit 会有以下几步

#### 1. 创建 Retrofit 对象

```java
 private static Retrofit getRetrofit(String baseUrl) {
     if (mRetrofit == null) {
         synchronized (Retrofit.class) {
             if (mRetrofit == null) {
                 mRetrofit = new Retrofit.Builder()
                     .client(getOkHttpClient())
                     .baseUrl(baseUrl)
                     .addConverterFactory(GsonConverterFactory.create())
                     .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
                     .build();
             }
         }
     }
     return mRetrofit;
 }
```

#### 2. 创建 Service 代理

```java
/***
* 使用一个默认的baseUrl创建一个新的service
* @return service
*/
public static <T> T getInstance(Class<T> serviceClazz) {
    return getInstance(Emmm.getBaseUrl(), serviceClazz);
}
```

#### 3. 创建 Service 接口以及接口方法

```java
public interface IUserService {
    @FormUrlEncoded
    @POST("api/user")
    Call<Response<User>> getUser(@Filed("id") int id);
}
```

#### 4. 使用

```java
IUserService service = ServiceFactory.getInstance(IUserService.class);
Call<Response<User>> call = service.getUser(1);
```

通过以上几步我们就可以使用 retrofit 来获取到 call 对象，然后即可请求接口，所有接口参数、返回值类型、调用方式都定义的很明确，清晰明了。其实在书写一两个接口方法我们并没有感觉到任何不对的地方，但是如果当你的项目足够大，接口方法多到一定程度，你就会发现需要按模块分很多 Service 接口，每个 Service 有有更多的接口方法(单个 api 请求方法)。除了接口方法众多外，没写一个接口都要去关系接口参数、返回值、url格式等等细节问题，稍不留意可能就会写错。

### 问题在哪里？

通过以上，可以发现，传统的写法，故然简单明了，但是会随着时间和项目的推移出现很多问题。

1. 接口数目随着项目推进呈现剧增趋势
2. 接口定义十分繁琐
3. 接口无法扩展(一个接口只能返回特定的返回值类型)
4. 为了区分模块需要维护多个不同模块的 Service

主要问题就是上面列出的这些，总结出来就是传统写法不好维护，或者说维护起来十分的麻烦、繁琐。带着这些问题，深入探索一下，是否有可以优化地方？

### 优化与重构

#### 1. 解决接口定义繁琐的问题

我们已经觉得定义一个个接口是否的令人头疼，那我们有没有什么方法可以缩减一下接口定义个数呢？

目前我们定义的接口方法都是比较明确的，比如`getUser()`,`login()`等接口，是一个具体的接口。如果我们要缩减接口个数，肯定要这些具体的接口转化成抽象的接口，因为具体的接口不可减，但是可以将其抽象化。**这里我们可以将所有接口抽象成通用的几个接口：Get，Post，Put，Delete，Patch等** Restful 接口，使用 restful 规范我们可以将所有接口分到对应的接口下。如`getUser`就可以用**Get**，`login`就可以用**Post**接口。如下：

```java
/**
 * Author   :hymane
 * Email    :hymanmee@gmail.com
 * Create at 2017-08-09
 * Description:统一的公共Service接口
 */
public interface ApiService {
    @GET
    Observable<Response<ResponseBody>> get(
            @Url String url,
            @QueryMap Map<String, Object> fields
    );

    @FormUrlEncoded
    @POST
    Observable<Response<ResponseBody>> post(
            @Url String url,
            @FieldMap Map<String, Object> fields
    );

    @FormUrlEncoded
    @PUT
    Observable<Response<ResponseBody>> put(
            @Url String url,
            @FieldMap Map<String, Object> fields
    );

    @FormUrlEncoded
    @DELETE
    Observable<Response<ResponseBody>> delete(
            @Url String url,
            @FieldMap Map<String, Object> fields
    );

    @FormUrlEncoded
    @PATCH
    Observable<Response<ResponseBody>> patch(
            @Url String url,
            @FieldMap Map<String, Object> fields
    );

    @Streaming
    @GET
    Observable<Response<ResponseBody>> download(
            @Url String url
    );
}
```

那么接下来我们如何使用呢？

```java
mApiService = ServiceFactory.getInstance(ApiService.class);
String url = "api/user";
Map<String, Object> parameters=new HashMap<>();
parameters.put("id",1);
mApiService.get(url,parameters);
```

这样我们原本需要按模块创建的 XXXService 已经变成现在的仅有的一个 ApiService。接口方法也从原本 N 个变成现在的 6 个接口。是不是瞬间轻松的好多？

#### 2. 引出新的问题

虽然我们完成了一个质的改版，但是却引入了一个问题，那就是返回值类型不明确的问题，之前的写法，我们在每个接口处都明确指定了返回值类型。比如`geUser`方法返回的就是`Response<User>`对象，但是改版之后，接口方法以及抽象出来，我们已经不能得知请求接口的返回值是什么类型的了，准确的说我们是知道的，只不过是通用的`Response<ResponseBody>`类型，如果我们想在调用`mApiService.get("api/user",parameters);`时能明确的返回想要的`User`类型可不可以呢？

#### 3. 抽象得到具体对象？

抽象的方法获得具体的实例，我们首先想到的就是泛型了，我们将`ApiService.java`接口返回值都加上泛型试试！！！

```java
 @GET
 <T> Observable<Response<T>> get(
     @Url String url,
     @QueryMap Map<String, Object> fields
 );
```

不幸的是这样做是不行的，retrofit 返回值类型中并不支持泛型类型，因为泛型在 runtime 会被擦除，retrofit 也不知道你究竟想要哪种类型的返回值。既然这种方案不可行，那还有其他解决办法吗？

#### 4. 手动解析响应

我们已经知道了现在 Retrofit 会返回一个`Response<ResponseBody>`对象，而这里面有我们想要的响应体，我们可以自己手动来解析里面的实体，比如 Json 响应，我们可以用 Gson 或 FastJson 等库来处理。

使用 Rxjava 配合 Retrofit 使用

```java
/***
* 简单的单次get接口
* @param url 请求网址
* @param parameters 请求参数
* @param observer 回调
* @param <T> 回调数据类型
* Server.java
*/
public <T> void get(final String url, Map<String, Object> parameters, SimpleObserver<T> observer) {
    mApiService.get(url, parameters)
    .map(new NetworkFunction<>(observer))
    .compose(RxSchedulers.<T>applyObservableAsync())
    .subscribe(observer);
}
```

手动转换响应体类型

```java
 @Override
    public T apply(@NonNull Response<ResponseBody> response) throws Exception {
        if (response.isSuccessful()) {
            ResponseBody body = response.body();
            if (body != null) {
                Class<T> clazz = getClassFromInterface(mObserver);//重点
                if (clazz == null) {
                    throw new Exception("获取接口泛型参数异常");
                }
                String result = body.string();
                return (T)JSON.parseObject(result,clazz);
            } else {
                throw new UnknownHostException();
            }
        } else {
            throw new UnknownHostException();
        }
    }
```

使用反射从带有泛型的参数中获取具体类型

```java
    /**
     * 获取泛型
     *
     * @param observer
     * @return
     * NetworkFunction.java
     */
    @SuppressWarnings("unchecked")
    private Class<T> getClassFromInterface(Observer<T> observer) {
        try {
            Type[] interfaceTypes = observer.getClass().getGenericInterfaces();
            Type type;
            if (interfaceTypes.length == 0) {
                //非接口
                type = observer.getClass().getGenericSuperclass();
            } else {
                type = interfaceTypes[0];
            }
            if (type == null) {
                return null;
            }
            if (ParameterizedType.class.isAssignableFrom(type.getClass())) {
                Type item = (((ParameterizedType) type).getActualTypeArguments())[0];
                return ((Class<T>) item);
            }
        } catch (Exception e) {
            return null;
        }
        return null;
    }
```

至此，我们使用了通用的 ApiService 接口以及手动解析响应体，使用 Rxjava map操作符将`ResponseBody`转换成想要的`<T>`类型。已经基本达到我们想要的目的了，目前新增接口，我们不再需要定义 Service 以及 Service 接口方法了，只需要调用`mApiService.get("newUrl",newParams,observer)`就 ok 了，是不是突然得到了解脱？

#### 5. 继续优化

上面我们已经完成了这次大改造，并成功解放了生产力，但是还有一个问题，需要再考虑一下。因为泛型擦除以及数组列表泛型特殊性的问题，导致我们的 observer 泛型局限性很大，比如不支持嵌套泛型，最外层泛型不支持使用 List，数组。也就意味着泛型 T 只可以是我们定义的某个类。虽然在生成环境大多数是使用自定义类的，但是也不排除有直接返回 List 的可能性。那么现在我们着手将 List 支持进去。

首先有两个方法`getGenericInterfaces`,`getGenericSuperclass`。这两个方法是获取泛型类型用的，前者适用于接口上，后者使用于类上。然后通过`getActualTypeArguments()`方法具体的类型。我们再获取到具体类型之后，先判断一下是否是 List，如果是 List，我们要剥开 List，继续往下取下一层泛型的具体类型。所以我们要一个判断是否是 List 的方法。

```java
  /***
     * 判断泛型 type 是否是 List 类型
     * @param type 泛型类型
     * @return
     */
    private boolean isTypeOfList(Type type) {
        if (ParameterizedType.class.isAssignableFrom(type.getClass())) {
            Type rawType = ((ParameterizedType) type).getRawType();
            return rawType == java.util.List.class;
        } else {
            return false;
        }
    }

	/**
     * 获取泛型
     *
     * @param observer
     * @return
     * NetworkFunction.java
     */
    @SuppressWarnings("unchecked")
    private <T> Type getTypeFromInterface(Observer<T> observer) {
        try {
            Type[] interfaceTypes = observer.getClass().getGenericInterfaces();
            Type type;
            if (interfaceTypes.length == 0) {
                //非接口
                type = observer.getClass().getGenericSuperclass();
            } else {
                type = interfaceTypes[0];
            }
            if (type == null) {
                return null;
            }
            if (ParameterizedType.class.isAssignableFrom(type.getClass())) {
                Type item = (((ParameterizedType) type).getActualTypeArguments())[0];
                return item;
            }
        } catch (Exception e) {
            return null;
        }
        return null;
    }
```

修一下我们的解析方法

```java
 	@Override
    public T apply(@NonNull Response<ResponseBody> response) throws Exception {
        if (response.isSuccessful()) {
            ResponseBody body = response.body();
            if (body != null) {
                Type clazz = getTypeFromInterface(mObserver);
                if (clazz == null) {
                    throw new Exception("获取接口泛型参数异常");
                }
                String result = body.string();
                if (isTypeOfList(clazz)) {//如果是list
                    Type[] actualTypeArguments =((ParameterizedType)clazz).getActualTypeArguments();
                    Type argument = actualTypeArguments[0];
                    return (T) JSON.parseArray(arg.toString(), (Class<?>) argument);
                }else{
                    return (T)JSON.parseObject(result,clazz); 
                }
            } else {
                throw new UnknownHostException();
            }
        } else {
            throw new UnknownHostException();
        }
    }
```

这里在手动解析响应体的时候先判断了一次外层泛型是否是 List，如果是 List 就再深挖一层泛型，再解析，并包装成 List 返回。如果不是 List 就直接常规解析即可。

### 总结

经过改造之后，将原本繁琐的新增接口任务变得异常简单，原本每一个新接口都需要去定义一个 service 接口方法，包括参数、返回值、url等，现在不需要定义任何接口，直接调用就可以。封装完毕之后，新增新的接口只需要：

```java
HashMap<String, Object> params = newParams();
params.put("start", start);
params.put("count", count);
Server.instance().get(ApiConstant.Movie.TOP250,params,observer);
```

唯独需要的是接口 url 和返回值类型`<T>`，仅此而已。