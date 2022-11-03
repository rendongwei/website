---
title: "Android-Retrofit"
date: "2022-11-02"
categories:
    - "Android"
tags:
    - "Android"
    - "Android框架"
toc: true
indent: false
original: true
draft: false
---

## 前言

Retrofit 是一个 RESTful 的 HTTP 网络请求框架的封装，网络请求的工作本质上是 OkHttp 完成，而 Retrofit 仅负责 网络请求接口的封装

## 使用步骤

### 1.添加Retrofit库的依赖：
``` gradle
implementation 'com.squareup.retrofit2:retrofit:2.0.2'
implementation 'com.squareup.retrofit2:converter-gson:2.0.2'
implementation 'com.google.code.gson:gson:2.8.5'
implementation 'com.squareup.retrofit2:adapter-rxjava:2.0.2'
```

>后面三个是可选的，分别是数据解析器和gson，以及rxjava支持的依赖

###  2.创建用于描述网络请求的接口

Retrofit将 Http请求 抽象成 kotlin接口：采用 注解 描述网络请求参数 和配置网络请求参数

``` kotlin
interface ApiSerivce{
    @GET("/api/call")
    fun getCall(@Query("name") name : String) : Call<String>
}
```

### 3.创建Retrofit实例

```kotlin
val retrofit  = Retrofit.Builder()
				.baseUrl("http://api.xxx.com")
				.addConverterFactory(GsonConverterFactory.create())
				.addCallAdapterFactory(RxJavaCallAdapterFactory.create())
				.build()
```

### 4.发送请求

请求分为同步请求和异步请求

```kotlin
val service = retrofit.create(ApiService::class)
val call = service.getCall("张**")

// 同步请求
try{
    val response : Response<String> = call.execute()
    val reuslt = response.body()
}catch(e:Exception){
    
}

// 异步请求
call.enqueue(object : CallBack<String>{
    @Override
    fun onResponse(call : Call<String> , response : Response<String>){
        val reuslt = response.body()
    }
    
    fun onError(call : Call<String> , throwable : Throwable){
       
    }
})
```

response.body()就是Reception对象，网络请求的完整 Url =在创建Retrofit实例时通过.baseUrl()设置 +网络请求接口的注解设置（下面称 “path“ ）
整合的规则如下：

|                  类型                   | 具体使用                                                     |
| :-------------------------------------: | ------------------------------------------------------------ |
|            path = 完整的Url             | Url = "http://host:port/api/login"  <br />path = "http://host:port/api/login"<br />baseUrl = 不设置 |
|             path = 绝对路径             | Url = "http://host:port/api/login"<br />path = "/login"<br />baseUrl = "http://host:port/api/" |
| path = 相对路径<br />baseUrl = 目录形式 | Url = "http://host:port/api/v1/login" <br />path = "login"<br />baseUrl = "http://host:port/api/v1" |
| path = 相对路径<br />baseUrl = 文件形式 | Url = "http://host:port/api/v1/login" <br />path = "login"<br />baseUrl = "http://host:port/api/v1" |

### 注解

上面我们用了@GET注解来发送Get请求，Retrofit还提供了很多其他的注解类型

#### 网络请求方法

> @GET 、 @POST 、 @PUT 、@DELETE 、@PATH 、@HEAD 、@OPTIONS 、@HTTP

#### 标记类

> @FormUrlEncoded 、@Multipart 、@Streaming

#### 网络请求参数

> @Header 、@headers 、@URL 、@Body 、@Path 、@Field 、@FieldMap 、@Part 、@PartMap 、@Query 、@QueryMap



#### 第一类 : 网络请求方法

1. @GET、@POST、@PUT、@DELETE、@HEAD分别对应 HTTP中的网络请求方式

2. @HTTP替换@GET、@POST、@PUT、@DELETE、@HEAD注解的作用 及 更多功能拓展

具体使用：通过属性`method`、`path`、`hasBody`进行设置

```kotlin
interface ApiService{
    /**
    * method : 网络请求方法(区分大小写)
    * path : 网络请求地址路径
    * hasBody : 是否有请求体
    */
    @HTTP(method = "GET" , path = "blog/{id}" , hasBody = false)
    fun getCall(@Path("id") id : String) : Call<String>
}
```



#### 第二类 : 标记

1. @FormUrlEncoded

   表示发送`form-encoded`的数据，每个键值对需要用`@Filed`来注解键名，随后的对象需要提供值。

2. @Multipart

   表示发送 `form-encoded` 的数据（适用于 有文件 上传的场景），每个键值对需要用`@Part`来注解键名，随后的对象需要提供值。

```kotlin
interface ApiService{
    /**
    * 表明是一个表单格式的请求（Content-Type:application/x-www-form-urlencoded）
    * <code>Field("name")</code> 表示将后面的 <code>String name</code>
    */
    @POST("/form")
    @FormUrlEncoded
    fun test(@Field("name") name : String) : Call<String>
    
    /**
    * Part后面支持三种类型，RequestBody、MultipartBody.Part 、任意类型
    * 除 MultipartBody.Part以外，其它类型都必须带上表单字段 (MultipartBody.Part 中已经包含了表单字段的信息)
    */
    @POST("/form")
    @Multpart
    fun test(@Part("name") name : RequestBody) : Call<String>
}
```



#### 第三类 : 网络请求参数

| 注解名称  |                             解释                             |
| :-------: | :----------------------------------------------------------: |
| @Headers  |                          添加请求头                          |
|  @Header  |                     添加不固定值的Header                     |
|   @Body   |                       用于非表单请求体                       |
|  @Field   |                     向Post表单传入键值对                     |
| @FieldMap |                     向Post表单传入键值对                     |
|   @Part   |             用于表单字段，适用于有文件上传的情况             |
| @PartMap  |             用于表单字段，适用于有文件上传的情况             |
|  @Query   | 用于表单字段，功能同@Field@FieldMap,区别是一个体现在URL上，一个在请求体上 |
| @QueryMap | 用于表单字段，功能同@Field@FieldMap,区别是一个体现在URL上，一个在请求体上 |
|   @Path   |                          URL缺省值                           |
|   @URL    |                           URL设置                            |

##### 1.@Header & @Headers

添加请求头 &添加不固定的请求头

```kotlin
// @Header
@GET("/user")
fun getUser(@Header("Authorization") authorization : String) : Call<User>

// @Headers
@GET("/user")
@Headers("Authorization : authorization")
fun getUser() : Call<User>
```

> 以上的效果是一致的
>
> 区别在于使用场景和使用方式
>
> 1. 使用场景：@Header用于添加不固定的请求头，@Headers用于添加固定的请求头
> 2. 使用方式：@Header作用于方法的参数；@Headers作用于方法

##### 2.@Body

以 Post方式 传递 自定义数据类型 给服务器,如果提交的是一个Map，那么作用相当于 @Field,不过Map要经过 FormBody.Builder 类处理成为符合 [OkHttp](https://github.com/square/okhttp)格式的表单，如：

```kotlin
// @Body
@POST("/addUser")
fun addUser(@Body body : FormBody) : Call<String>

// body
val builder = FormBody.Builder()
builder.add("key","value")
val body = builder.build()
```

##### 3.@Field & @FieldMap

发送 Post请求 时提交请求的表单字段,与 @FormUrlEncoded 注解配合使用

```kotlin
// @Field
@POST("/addUser")
@FormUrlEncoded
fun addUser(@Field("name") name : String , @Field("age") age : int) : Call<String>

// @FieldMap
@POST("/addUser")
@FormUrlEncoded
fun addUser(@FieldMap map : Map<String , Any?>) : Call<String>
```

##### 4.@Part & @PartMap

发送 Post请求 时提交请求的表单字段,与@Field的区别：功能相同，但携带的参数类型更加丰富，包括数据流，所以适用于 有文件上传 的场景,与 @Multipart 注解配合使用

```kotlin
// @Part
@POST("/form")
@Multipart
fun uploadFile(@Part("name") name : RequestBody , @Part file : MultipartBody.Part) : Call<String>

// @PartMap
@POST("/form")
@Multipart
fun uploadFile(@PartMap map : Map<String , RequestBody?> , @Part file : MultipartBody.Part) : Call<String>
```

##### 5.@Query & QueryMap

用于 @GET 方法的查询参数（Query = Url 中 ‘?’ 后面的 key-value）

```kotlin
// @Query
@GET("/getUser")
fun getUser(@Query("id") id : String) : Call<User>

// @QueryMap
@GET("/getUser")
fun getUser(@QueryMap map : Map<String , Any?>) : Call<User>
```

##### 6.@Path

URL地址的缺省值

```kotlin
@GET("/user/info/{id}")
fun getUser(@Path("id") id : String) : Call<User>
```

##### 7.Url

直接传入一个请求的 URL变量 用于URL设置

```kotlin
@GET
fun getUser(@Url url : String , @Query("id") id : String) : Call<User>
```

