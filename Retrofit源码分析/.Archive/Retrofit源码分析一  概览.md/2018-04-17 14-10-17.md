# Retrofit源码分析一  概览
* Retrofit的本质和与Okhttp的关系
​	说到Retrofit，免不了要提起Okhttp，因为二者通常是绑定到一起使用的。那么我们首先要明确一点Retrofit并不是一个网络请求框架，而是一个对网络请求框架（也就是Okhttp）的封装。二者都是Squire公司的开源框架，Retrofit并不能脱离OKhttp，因为底层的网络访问是由Okhttp来实现的。说到对网络请求的封装，这里小伙伴们可能会有一些疑问，什么叫做**对网络请求的封装**？这里我们先简单的贴一些代码来看一下
```
    //登录
    @FormUrlEncoded
    @POST("${BuildConfig.EXTRA_URL}account/login.do")
    fun login(@Field("username") userName: String, @Field("password") pwd: String, @Field("clientType") clientType: Int): Observable<HttpResultEntity<UserEntity>>
```
在上面这个Kotlin编写的的网络请求方法中，**@FormUrlEncoded**、**@POST("${BuildConfig.EXTRA_URL}account/login")**、 **@Field("username")**、**@Field( "password") **、**@Field("clientType")**、**Observable<HttpResultEntity<UserEntity>>** 这些都是对网络请求的封装。这里需要知道的是，对网络请求的封装包括两个方面：**1. 对请求参数的封装**；**2. 对网络返回结果的封装**。上面列出来的几项中除了**Observable<HttpResultEntity<UserEntity>>** 之外都是对请求参数的封装，即使是对Retrofit不太了解的同学应该也是可以很轻松的看懂一些参数代表的意义，比如**@POST**代表这个网络请求采用post方式，**@Field("username")**代表post请求域中要包含一个**username**的请求参数。而与之相对应的**Observable<HttpResultEntity<UserEntity>>**就是对网络返回结果的封装，对Rxjava了解的同学应该明白，Retrofit把网络返回的原始数据包装成了一个Observable，便于我们的开发。
* Retrofit流程分析
    Retrofit的流程图如下所示
![Retrofit流程图](https://github.com/BlackFlagBin/MarkDownPicture/blob/master/retrofitpic/Retrofit%E6%B5%81%E7%A8%8B.jpg?raw=true)
关于更加详细的流程我们之后的章节会进行分析，大家先看个大概，心里有个印象就行。等我们对整个Retrofit的源码分析结束之后，相信大家对这幅图会有更加深入的认知。

    

