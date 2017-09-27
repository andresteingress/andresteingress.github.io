---
layout: post
title: Java 9 - Optional Additions
categories:
- java
- java9
tags: []
status: publish
type: post
published: true
---

In Java 8, the `java.util.Optional` class was introduced ([JavaDoc](http://download.java.net/java/jdk9/docs/api/java/util/Optional.html)) as a way to define method return types where there is a need to return no result, and where `null` is not a good fit as it needs to be explicitly mentioned that an empty return value is indeed possible.

In this article we will have a look at the `Optional` methods which were added with Java 9.

_Hint_: in case you wonder about the method references in this article, it is using the [method reference syntax](https://docs.oracle.com/javase/tutorial/java/javaOO/methodreferences.html) from Java 8. In the code examples, we use _jshell_, [a new command-line tool](https://blog.andresteingress.com/2017-09-26-java-9-jshell.html) being part of Java 9 JDK. 

### `Optional::ifPresentOrElse`

`Optional::ifPresentOrElse` is used on `Optional` instances to execute a lambda being a `java.util.function.Consumer` if the value exists, or a `Runnable` - or so-called empty action - if the value does not exist. 

{% highlight java %}

jshell> Optional<String> optString = Optional.of("test")
jshell> optString.ifPresentOrElse(s -> System.out.println(s), () -> System.out.println("not there"))
test

{% endhighlight %}

In the code snippet from above, the first lamda defines the `Consumer` implementation which prints out the given `String` argument to stdout. The second lamda defines the `Runnable` implementation without an argument.

### `Optional::or`

The use-case for `Optional::or` should be relatively self-explanatory. If called on an `Optional`, it returns the optional itself if its value exists or the `Optional` created by the given `java.util.function.Supplier` function ([JavaDoc](http://download.java.net/java/jdk9/docs/api/java/util/function/Supplier.html)).

{% highlight java %}

jshell> Optional<String> optString = Optional.of("test")
jshell> optString.or(() -> Optional.of("42"))
$3 ==> Optional[test]

{% endhighlight %}

The code snippet above in fact does not use the given `Supplier` function which would return `Optional.of("42")`.

As the return type of `or` is an `Optional`, it's perfectly valid to chain multipe `or` calls:

{% highlight java %}

jshell> Optional.empty().
   ...> or(() -> Optional.empty()).
   ...> or(() -> Optional.of("42"))
$6 ==> Optional[42]

{% endhighlight %}

### `Optional::stream`

The `stream` method is the most useful addition (imho). If a value is present, it returns a `java.util.stream.Stream` instance containing only the optional value.

{% highlight java %}

jshell> Optional<String> optString = Optional.of("test")
jshell> optString.stream().forEach(elem -> System.out.println(elem));
test

{% endhighlight %}

Maybe its usefulness might not be visible at first sight, however, when dealing with `Stream`-typed return values it indeed makes sense to return a `Stream` either based on an `Optional` or on any other `Stream` implementation.

### Summary

In this article we showed the latest additions to `java.util.Optional` in JDK 9, the most useful addition being the new `Optional::stream` method. 