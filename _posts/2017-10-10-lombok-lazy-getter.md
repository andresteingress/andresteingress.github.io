---
layout: post
title: Lombok - Lazy Getters
categories:
- java
- lombok
tags: []
status: publish
type: post
published: true
---

This article shows how to use lazy getters in [Lombok](https://projectlombok.org/features/GetterLazy). Lazy getters are a way to define getters with initialisation logic, only being executed once and when the getts is really accessed.

### @Getter Annotation

Lombok's `@Getter` annotation is used on private fields to indicate the creation of a Java bean getter method.

{% highlight java %}

public class MyClass {
    

    @Getter
    private String name;

}

{% endhighlight %}

will be generated as

{% highlight java %}

public class MyClass {
    

    private String name;

    public String getName() {
        return this.name;
    }

}

{% endhighlight %}

Let's imagine we have a more advanced getter where we want to cache the output from the first method call. For example, we have to do a heavy computation and do not want to trigger that computation on every getter-method call. Exactly for this use-case Lombok privates the `lazy` attribute in `@Getter`. Note that `@Getter(lazy = true)` can be applied on `private final` fields only.

If it is set to `true` the generated class will look something like that:

{% highlight java %}

public class MyClass {
    

    @Getter(lazy = true)
    private final String name = heavyCalculation();

    private String heavyCalculation() {
        // ... do heavy stuff
        return someString;
    }

}

{% endhighlight %}

becomes

{% highlight java %}

public class MyClass {
    private final java.util.concurrent.AtomicReference<java.lang.Object> name = new java.util.concurrent.AtomicReference<java.lang.Object>();
  
    public String getName() {
        java.lang.Object value = this.name.get();
        if (value == null) {
            synchronized(this.name) {
                value = this.name.get();
                if (value == null) {
                    final String actualValue = heavyCalculation();
                    value = actualValue == null ? this.name : actualValue;
                    this.name.set(value);
                }
            }
        }
        return (double[])(value == this.name ? null : value);
    }
  
    private String heavyCalculation() {
        // ... do heavy stuff
        return someString;
    }
}

{% endhighlight %}

As you can see in the code above, the generated code is a bit heavier than without `lazy=true`. It basically wraps our `heavyCalculation` method in a `synchronized` block (synchronisation is done on the generated `AtomicReference`) and stores the result in an `java.util.concurrent.atomic.AtomicReference` ([JavaDoc](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/AtomicReference.html)). Starting from the next getter method call, it will return the calculated result.

### Summary

This article showed how to implement calculated and cached getter methods with Lombok. The library provides a `lazy` attribute on its `@Getter` annotation which allows for generating caching getter methods.

