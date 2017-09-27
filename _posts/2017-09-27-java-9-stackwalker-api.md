---
layout: post
title: Java 9 - StackWalker API
categories:
- java
- java9
tags: []
status: publish
type: post
published: true
---

Java 9 introduces a new efficient API for walking stack traces besides the already existing, and for Java developers well-known, `Throwable::getStackTrace` and `Thread::getStackTrace` methods.

### The `getStackTrace` Situation

`getStackTrace` has several disadvantages from various point of views. The API requires the JVM to eagerly capture the entire stack at the point where an exception is thrown. There is for example no way to capture only the first line of the stack trace, without loading the entire stack trace. In addition, the stack trace does not contain actual references to `Class<?>` instances and as mentioned in [JEP 259](http://openjdk.java.net/jeps/259) the VM implementation is allowed to skip frames for performance reasons, so developers might end up with an incomplete stack-trace and therefore missing information.

### Stack-Walking API

In Java 9, the new stack-walking API strives for eliminating the disadvantages mentioned in the last paragraph. It allows for laziness and frame filtering and even accessing the class information via access to actual `Class` instances. 

The stack-walking API manifests itself in the [java.lang.StackWalker](http://download.java.net/java/jdk9/docs/api/java/lang/StackWalker.html) class. `StackWalker` is thread-safe, multiple threads can share a single `StackWalker` object. To obtain an instance, it provides several `getInstance()` methods returning the stack trace for the current thread. 

The method for walking the stack trace, is `StackWalker::walk`. It takes a `Function` which in turn gets a `Stream<StackWalker.StackFrame>` as argument, so a stream of stack frames. A stream of `StackFrame` instances has the main advantage of supporting filtering and all the other `Stream` operations of course.

To obtain a `StackWalker` instance, just call its `getInstance()` method:

{% highlight java %}

StackWalker walker = StackWalker.getInstance();

{% endhighlight %}

The `getInstance()` method without arguments defaults to skip all hidden frames and class references. The JVM may hide implementation specifc frames together with reflection specific frames, those are refered to as _hidden_ frames. Class references are not retained automatically in the default `getInstance()` call, but might be optionally included if necessary. 

There are overloaded `getInstance(..)` methods which take one or more `StackWalker.Option` instances ([JavaDoc](http://download.java.net/java/jdk9/docs/api/java/lang/StackWalker.Option.html)) for overruling the defaults.

{% highlight java %}

StackWalker walker = StackWalker.getInstance(StackWalker.Option.RETAIN_CLASS_REFERENCE);

{% endhighlight %}

_Hint:_ Be aware that `RETAIN_CLASS_REFERENCE` will check with the current `SecurityManager` whether there is the runtime permission "getStackWalkerWithClassReference".

Once the `StackWalker` instance is retrieved, either `StackWalker::walk` or `StackWalker::forEach` can be used to traverse the stack trace from top to bottom. Remember, `walk` actually gets a lambda function as argument which in turn will be called with a `Stream` of `StackWalker.StackFrame` instances. `StackFrame` itself comes with additional meta-data on the captured method invocation, like the byte-code index, declaring class, line number, method name and a reference to the `StackTraceElement` as returned by `Throwable::getStackTrace()`:

{% highlight java %}

StackWalker walker = StackWalker.getInstance();
List<StackWalker.StackFrame> frames = walker.walk(s -> s.dropWhile(f -> f.getClassName().startsWith("com.foo"))
	 .limit(10)
	 .collect(Collectors.toList())
);

{% endhighlight %}

The example above basically ignores/drops all stack frames from classes with a package name starting with `com.foo`. All the other classes in the stack frame will be collected, however, only the first 10 matches will be collected into a list. This list is returned as a result. As you can see, having a `Stream` to navigate through all the stack traces in a performant way is a really nice solution, allowing for using all the methods defined in the Stream API.

### Summary

Java 9 comes with a new API for navigating and filtering stack traces: `StackWalker`. Based on the Stream API, it provides a way to move through a stream of stack frames, including on-demand loading of stack frames and retaining of the referred to `Class` objects.


