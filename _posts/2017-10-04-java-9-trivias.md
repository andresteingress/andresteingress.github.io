---
layout: post
title: Java 9 - Trivias
categories:
- java
- java9
tags: []
status: publish
type: post
published: true
---

This article summarises a couple of Java 9 features we did not talk about in our previous articles. As most of them are really rather small, we will go quickly through them - section by section.

Let's start with enhancements to Project Coin, initially introduced in Java 7.

### Project Coin Enhancements

Project Coin introduced rather small language changes in JDK/SE 7. Java 9 addresses some rough edges of those changes:

* underscore `_` as an identifier creates an error now in Java 9 (previously a warning only).
* `@SafeVarargs` ([JavaDoc](http://docs.oracle.com/javase/7/docs/api/java/lang/SafeVarargs.html)) is now allowed on private instance methods.
* allow effectively-final variables to be used in try-resources (more about that in the next sections).
* usage of the diamond operator with anonymous classes is allowed if the type is denotable.
* support for private methods in interfaces (more about that in the next section).

#### try-with-resources

The try-with-resources language construct, introduced with Java 7, gains another extension in Java 7. You do not have to declare `java.io.Closeable` resources within the `try` instead you can use an effectively-final variable from within the `try` expression:

{% highlight java %}

java.io.Closeable myCloseable = // ...

try (myCloseable) {
    // ... do stuff
}

{% endhighlight %}

You can do that for multiple resources too:

{% highlight java %}

java.io.Closeable myCloseable = // ...
java.io.Closeable anotherCloseable = // ...

try (myCloseable, anotherCloseable) {
    // ... do even more stuff
}

{% endhighlight %}

#### Private methods in interfaces

Private methods are now allowed in interfaces to share common functionality between either static or default method implementations in the interface:

{% highlight java %}

public interface SomeInterface {

    default String saySomething(String text) {
        return buildString(text);
    }

    default String shoutSomething(String text) {
        return buildString(text);
    }

    // private method is allowed since Java 9:
    private String buildString(String text)  {
        StringBuilder stringBuilder = new StringBuilder();
        stringBuilder.append("This is your text: ");
        stringBuilder.append(text);
        return stringBuilder.toString();
    }
}

{% endhighlight %}

The following features are independent of Project Coin changes addressing other areas in JDK/SE.

### `Flow` API

Java 9 comes with a nice API addition supporting the implementation of flow-controlled components: the Flow API, found in package `java.util.concurrent`. 

The new interfaces from the Flow API implement the [reactive streams specification](http://www.reactive-streams.org/). The following interfaces come with Java 9:

* `Flow.Publisher` - is a producer of items
* `Flow.Subscriber` - a receiver of items
* `Flow.Subscription` - message control linking between `Flow.Publisher` and `Flow.Subscriber`
* `Flow.Processor` - a component which acts as both, as publisher and subscriber

So those classes are basically interfaces only. In JDK 9 itself, concrete interface implementations can be found in the `java.util.concurrent` package and the `jdk.incubator.http` package. For the new [Java 9 HTTP client](http://download.java.net/java/jdk9/docs/api/jdk/incubator/http/HttpClient.html), there is for example a `HttpResponse.BodyProcessor` interface which allows for the subscriber to work on the incoming byte-stream. 

### HTTP 2 Client

As mentioned in the last section, Java 9 comes with a [new HTTP 2 client](http://docs.oracle.com/javase/9/docs/api/jdk.incubator.httpclient-summary.html) found in package `jdk.incubator.httpclient` in module `jdk.incubator.httpclient`. The classes therein define high-level HTTP and WebSocket APIs.

The `HttpClient.Builder` can be used to create an instance of `HttpClient`. Those instances are immutable and can be used for multiple `HttpRequest`s. Besides synchronous sending with `send`, `HttpClient` also has an asynchronous `sendAsync` method returning a `CompletableFuture<HttpResponse>`.

### JavaDoc Search

[JavaDoc Search](http://openjdk.java.net/jeps/225) is a nice new addition to the generated JavaDoc documentation files. It adds a search box in the top right corner for the user to lookup program elements or tagged words/phrases within the documentation. This is all done with client-side JavaScript. When you navigate to official the [Java 9 JavaDoc documentation](http://download.java.net/java/jdk9/docs/api/overview-summary.html), you can have a look at the search box and play with it.

Besides providing a new search box, JavaDoc has also been enhanced to generate a nicer [HTML 5 markup](http://openjdk.java.net/jeps/224).

### Enhanced Depreciation

[Enhanced Depreciation](http://openjdk.java.net/jeps/277) is another small feature, being quite useful though. It is about extending the `@Deprecated` annotation with new attributes for providing better and more detailed information about the status and the intended disposition of APIs.

`@Deprecated(since = "9")` allows for specifying the version in which the annotated element became deprecated. The given version string should be in the same format as the version used in the JavaDoc `@since` tag. `@Deprecated(forRemoval = true)` indicates whether the annotated element is subject to removal in a future version. 

Several Java SE APIs have been refined with more detailed `@Deprecated` annotations. 

### `ProcessHandle`

Class `java.lang.ProcessHandle` ([JavaDoc](http://download.java.net/java/jdk9/docs/api/java/lang/ProcessHandle.html)) has been introduced in Java 9 to get more information about the currently running JVM process and its children. It provides methods for querying the operating system process that the JVM is currently running in.

For example, querying the current process ID (PID) is now as easy as

{% highlight java %}

ProcessHandle.current().pid()

{% endhighlight %}

`ProcessHandle::inf()` returns a `ProcessHandle.Info` instance containing various meta-data about the current process, like the command, command-line arguments, process user, etc.

### UTF-8 Property Files

The default encoding for property files changes from ISO-8859-1 to UTF-8 in Java 9. Applications therefore no longer need to convert characters whose code points are over U+00FF to escaped characters. If, for some reason, this change needs to be overridden, there is a new system property `java.util.PropertyResourceBundle.encoding` which allows for that.

### jlink - Java Linker

`jlink` is a new tool distributed with the JDK that can assemble and optimise the set of modules into a custom run-time image. We will cover `jlink` in a separate blog post, for now you can gather more information [here](http://openjdk.java.net/jeps/282) or [here](https://docs.oracle.com/javase/9/tools/jlink.htm).

### Summary

This article summarises Java 9 features not being covered [in our previous articles](https://www.google.at/search?q=inurl%3Ablog.andresteingress.com+intitle%3Ajava+9&oq=inurl%3Ablog.andresteingress.com+intitle%3Ajava+9). Java 9 comes with a couple of nice though smaller features, enhancing Project Coin or adding new Flow reactive-stream APIs. This article gives an overview about these features.
