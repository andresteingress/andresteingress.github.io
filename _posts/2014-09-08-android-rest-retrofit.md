---
layout: post
title: REST Interfaces and Android
categories:
- android
- java
tags: []
status: publish
type: post
published: true
---
As I already told in my last blog post, I lately discovered this [very good collection of open source projects](http://square.github.io/) from [Square Inc.](https://squareup.com/), a commercial payment service. 

One project among their released open source projects that raised my interest was also [Retrofit](https://github.com/square/retrofit). Retrofit allows to access REST web interfaces via type-safe Java types and annotations.

### Configuring the `RestAdapter`

Before we can declare our interface and start making REST requests, we need to configure the so-called `RestAdapter`. It allows to change various aspects from HTTP settings to adding custom converters for the content found in HTTP responses (in our case, the REST interface returned JSON) from the accessed web service.

I put this setup code into our Dagger application module:

{% highlight java %}
@Provides
@Singleton
public AppConnector providesAppConnector() {

  OkHttpClient okHttpClient = new OkHttpClient();
  okHttpClient.setConnectTimeout(ApplicationConstants.HTTP_TIMEOUT, TimeUnit.MILLISECONDS);
  okHttpClient.setWriteTimeout(ApplicationConstants.HTTP_TIMEOUT, TimeUnit.MILLISECONDS);
  okHttpClient.setReadTimeout(ApplicationConstants.HTTP_TIMEOUT, TimeUnit.MILLISECONDS);

  RestAdapter restAdapter = new RestAdapter.Builder()
    .setEndpoint(application.getString(R.string.appEndPoint))
    .setLogLevel(RestAdapter.LogLevel.FULL)
    .setClient(new OkClient(okHttpClient))
    .setConverter(new GsonConverter(gson))
    .build();

    return restAdapter.create(AppConnector.class);
}
{% endhighlight %}

Usually the `RestAdapter` defaults to parsing JSON responses (utilizing Google's GSON library), but in our case we needed to adapt the pre-defined GSON converter slightly, this shouldn't irritate in this example. The endpoint is the base URL which is used for all the REST requests. The HTTP client in use is [OkHttpClient](http://square.github.io/okhttp/), another great project from Square, actually worth another blog post. It supports loading content from multiple IPs if single hosts aren't available for some reason and also does HTTP response caching based on the given response headers, if so configured. That's it for our Retrofit configuration.

### Declaring the Java REST interface

Once the `RestAdapter` is available, it can be used to instantiate proxy implementations for your Java REST interfaces. So let's first create a simple REST Java interface:

{% highlight java %}
public interface AppConnector {

  @Headers("Cache-Control: max-age=14400")
  @GET("/connector/contents/app-id/{app-id}")
  Contents getContents(@Path("app-id") String appId);

}
{% endhighlight %}

The example above is taken from one of our production apps (with only a little change). With this interface, calling `restAdapter.create(AppConnector.class)` returns a REST client object (proxy) that implements `AppConnector` and that does all the content parsing and conversion into Java objects for us. This works for plain Java types, collection types and custom Java classes that are used as return types and/or parameter types.

The example above actually makes a synchronous request. In fact, we do use asynchronous requests in our application. Going from synchronous to asynchronous requests only needs a little change:

{% highlight java %}
public interface AppConnector {

  @Headers("Cache-Control: max-age=14400")
  @GET("/connector/contents/app-id/{app-id}")
  void getContents(@Path("app-id") String appId, Callback<Contents> callback);

}
{% endhighlight %}

For aynchronous requests a second parameter is introduced to our interface method called `callback`. The `Callback` instance is called once the request has succeeded or failed. We can also go reactive. Retrofit integrates with [RxJava](https://github.com/ReactiveX/RxJava) and allows to return `rx.Observable` instances that enfore reactive programming in your Android code :-).

The `@Headers`, `@GET` and `@Path` annotations are all pretty self-explainatory. As you can imagine there is also `@POST`, `@PUT`, `@DELETE` or, for example, `@Query` which can be used to added request parameters to the configured URL.


### Conclusion

This article should serve as a small pointer and introduction to Retrofit, a leight-weight library for calling REST web services from Android code. Retrofit basically supports the creation of REST clients based on type-save Java interfaces and types and supports synchronous, asynchronous or reactive programming paradigms. More information on Retrofit can be found [at their GitHub project page](http://square.github.io/retrofit/).
