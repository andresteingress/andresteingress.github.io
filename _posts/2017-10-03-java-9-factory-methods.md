---
layout: post
title: Java 9 - Factory Methods
categories:
- java
- java9
- testing
tags: []
status: publish
type: post
published: true
---

After covering modules in [our last Java 9 article](https://blog.andresteingress.com/2017/09/29/java-9-modules.html), we have a look at another nice, although smaller, feature in this article: convenience factory methods ([JEP 269](http://openjdk.java.net/jeps/269)).

### Motivation

The goal of providing factory methods is to make it convenient to create instances of various collection types with a very small number of syntactical footprint. In JVM based languages like [Groovy](http://www.groovy-lang.org/), collection literals have been one of our favorite syntactical features ever since:

{% highlight java %}

def list = [1, 2, 3, 4, 5] // instantiates a java.util.ArrayList
def map = [myKey: 42, anotherKey: 71] // instantiates a java.util.LinkedHashMap 

{% endhighlight %}

Creating a list or a map like in the example above is syntactically very consice and short compared to the Java equivalent:

{% highlight java %}

ArrayList<Integer> list = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5));

Map<String, Integer> map = new LinkedHashMap<>();
map.put("myKey", 42);
map.put("anotherKey", 71);

{% endhighlight %}

For all collection types except maps, Java comes with a work-around to reduce the syntactical clutter with `Arrays.asList` ([JavaDoc](https://docs.oracle.com/javase/7/docs/api/java/util/Arrays.html#asList(T...))), however, instantiating a concrete `Map` is still syntactically a bit of a pain. Another option was to use the instance initializer and an anonymous sub-class to call `add` or `put` in the initializer:

{% highlight java %}

ArrayList<Integer> list = new ArrayList<>() {
    {
        add(1);
        add(2);
        add(3);
        add(4);
        add(5);
    }
};

{% endhighlight %}

However, besides this approach being rather weird to most developers, it can cause memory-leaks and it creates a new sub-class for every usage. That's not quite optimal too.

Java 9 provides subtile API additions to make those ugly code pieces go away.

### Collection API Additions

In Java 9, the `java.util.List`, `java.util.Set` and `java.util.Map` interfaces got extended with new `*.of` factory methods:

{% highlight java %}

List<Integer> list = List.of(1, 2, 3, 4, 5);
Map<String, Integer> map = Map.of("myKey", 42, "anotherKey", 71);

{% endhighlight %}

It's important to understand that `of` factory methods guarantee to return immutable collections, so you can't use the returned collections to add more elements or remove existing ones. The concrete return collection type is considered to be an implementation detail, so there shouldn't be any explicit checks on those types.  

The new factory methods come with overloaded `@SafeVarargs` methods allowing for an unlimited number of arguments. Besides the varargs method interface, the interfaces come with fixed argument method interfaces for up to 10 arguments. The overloaded method variants have been introduced to avoid array allocation and garbage collection overhead that would be introduced by having variable argument method interfaces only. Code implementing an `of` method might choose to return a collection implementation optimised based on the number of given arguments. 

### The `Map` Factory Method

`Map.of` is special in regard of variable args as it can not provide a variable list of arguments in its `of` method implementation due to the possibility of having completely different key/value types. It does, however, provide overloaded `of` method implementations for up to ten map entries, alternating between key and value arguments. To make this more clear, here is the method interface for ten map entries:

{% highlight java %}

static <K, V> Map<K, V> of(K k1, V v1, K k2, V v2, K k3, V v3, K k4, V v4, K k5, V v5,
                               K k6, V v6, K k7, V v7, K k8, V v8, K k9, V v9, K k10, V v10);

{% endhighlight %}

In addition, the `java.util.Map` interface does provide an alternative method, `ofEntries` expecting pairs of `Map.Entry<K, V>` instances. The new `Map.entry(K, V)` convenience method introduced in Java 9 can be used to create entry pairs:

{% highlight java %}

Map<String, Integer> map = Map.ofEntries(
    entry("someKey", 42),
    entry("anotherKey", 72)
);

{% endhighlight %}

The last `of` implementation not shown so far is `Set.of` which uses basically the same syntax as `List.of`:

{% highlight java %}

Set<Integer> set = Set.of(1, 2, 3, 4, 5);

{% endhighlight %}

### Summary

In this article we have a look at yet another Java 9 feature: convenience factory methods. Java 9 comes with new ways to instantiate immutable collection types with a more convenient syntax at which we will have a look in this article.