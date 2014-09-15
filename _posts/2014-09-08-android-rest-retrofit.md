---
layout: post
title: The Google I/O App
categories:
- android
- java
tags: []
status: publish
type: post
published: true
---
A while ago I came across the Google I/O App in one of the Android Developers [blog post](http://android-developers.blogspot.co.at/2014/07/google-io-2014-app-source-code-now.html). I decided to do a bit of analyzing this app, in order to gain some stuff I can again reuse in my apps. This blog post sort of documents my findings and what I found interesting.

### `build.gradle`



### Declaring the Java REST interface

Once the `RestAdapter` is available, it can be used to instantiate proxy implementations for your Java REST interfaces. So let's first create a simple REST Java interface:

```java
public interface AppConnector {

  @Headers("Cache-Control: max-age=14400")
  @GET("/connector/contents/app-id/{app-id}")
  Contents getContents(@Path("app-id") String appId);

}
```

The example above is taken from one of our production apps (with only a little change). With this interface, calling `restAdapter.create(AppConnector.class)` returns a REST client object (proxy) that implements `AppConnector` and that does all the content parsing and conversion into Java objects for us. This works for plain Java types, collection types and custom Java classes that are used as return types and/or parameter types.

The example above actually makes a synchronous request. In fact, we do use asynchronous requests in our application. Going from synchronous to asynchronous requests only needs a little change:

```java
public interface AppConnector {

  @Headers("Cache-Control: max-age=14400")
  @GET("/connector/contents/app-id/{app-id}")
  void getContents(@Path("app-id") String appId, Callback<Contents> callback);

}
```

For aynchronous requests a second parameter is introduced to our interface method called `callback`. The `Callback` instance is called once the request has succeeded or failed. We can also go reactive. Retrofit integrates with [RxJava](https://github.com/ReactiveX/RxJava) and allows to return `rx.Observable` instances that enfore reactive programming in your Android code :-).

The `@Headers`, `@GET` and `@Path` annotations are all pretty self-explainatory. As you can imagine there is also `@POST`, `@PUT`, `@DELETE` or, for example, `@Query` which can be used to added request parameters to the configured URL.


### Conclusion

This article should serve as a small pointer and introduction to Retrofit, a leight-weight library for calling REST web services from Android code. Retrofit basically supports the creation of REST clients based on type-save Java interfaces and types and supports synchronous, asynchronous or reactive programming paradigms. More information on Retrofit can be found [at their GitHub project page](http://square.github.io/retrofit/).
