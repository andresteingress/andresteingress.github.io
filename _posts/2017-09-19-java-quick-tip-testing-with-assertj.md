---
layout: post
title: AssertJ
categories:
- java
- spring
tags: []
status: publish
type: post
published: true
---

As one of my current projects is heavily Spring Boot based, I naturally came across the [boot-starter-test](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-testing.html) package very early.

The starter-test module comes with a couple of pre-configured dependencies:

* JUnit
* Spring Test
* AssertJ
* Hamcrest
* Mockito
* JSONassert
* JsonPath

I knew nearly all those libraries except AssertJ, so I had a closer look at it.

### The AssertJ Library

[AssertJ](http://joel-costigliola.github.io/assertj/) is a modular Java library providing additional components for various popular libraries like Guava, JodaTime, Neo4J and others. 

The core of the library is [AssertJ Core](http://joel-costigliola.github.io/assertj/assertj-core.html). Depending on your Java version, you have to select the appropriate version branch. We are currently using the 3.x branch which has adapted APIs for Java 8 and therefore makes use of lamdas.

The main entry point into AssertJ core is the class `org.assertj.core.api.Assertions` ([JavaDoc](http://joel-costigliola.github.io/assertj/core-8/api/org/assertj/core/api/Assertions.html)). You can define a static import for its methods in your JUnit test:

{% highlight java %}

import static org.assertj.core.api.Assertions.*;

import org.junit.Test;

public class MyAssertJTests {
    
    @Test
    public void someTest() {
        // ...
    }

}
{% endhighlight %}

Once you have statically imported all `Assertions` methods, you can start by having a look at its `assertThat*` methods. The most prominent one being the `assertThat()` method.

`assertThat()` comes as overloaded variant for a whole range of different parameter types. So there is an `assertThat(String)`, `assertThat(double)`, etc. for a very large range of input types.

`assertThat()` does not return `void` but a result of type `Assert`. On this class there are various methods for defining the contract/the assertions which should be done on the given argument value. Or in other words, a plain call to `assertThat()` wont execute any assertion immediately but you can use the returned `Assert` instance to define the contract that should be met.

Let's say we wanted to validate some `String` return value for its correctness. What you can do is to chain the AssertJ method calls like that:

{% highlight java %}
import static org.assertj.core.api.Assertions.*;

import org.junit.Test;

public class MyAssertJTests {
    
    SomeStringGenerator generator = // ...;

    @Test
    public void someTest() {
        assertThat(generator.generate(42)).isEqualTo("test");
    }
}
{% endhighlight %}

It's not restricted to having only one assertion. You can add more than one assertions too:

{% highlight java %}
import static org.assertj.core.api.Assertions.*;

import org.junit.Test;

public class MyAssertJTests {
    
    SomeStringGenerator generator = // ...;

    @Test
    public void someTest() {
        assertThat(generator.generate(42)).isNotNull().isEqualTo("test");
    }
}
{% endhighlight %}

There are also more advanced APIs (in version 3.x) that use lambdas in its method interfaces. 

One of my favorites is `assertThrownBy` which takes a code block as lamda and executes assertions based on the throwable that was thrown by this code block:

{% highlight java %}
import static org.assertj.core.api.Assertions.*;

import org.junit.Test;

public class MyAssertJTests {
    
    SomeStringGenerator generator = // ...;

    @Test
    public void someTest() {
        assertThatThrownBy(() -> { generator.generate(); }) // execute some code block in a Java 8 lambda
                .hasCauseExactlyInstanceOf(IllegalStateException.class)
                .hasMessageContaining("generator failed");
    }
}
{% endhighlight %}

Besides the core module, AssertJ also comes with an assertion generator that can be used from the command-line, Maven or Gradle to generate custom assertion classes based on some domain classes. We do not use that by now but it sounds like a good idea to generate custom assertion classes with some sort of domain specific language, instead of relying on the pre-defined asssertions.

### Summary

All in all, I can highly recommend AssertJ so far. Once being a bit familiar with its Api, it is far easier to use as Hamcrest or plain JUnit assertions with a much more readable DSL. All that comes with a good typing system making it easy for IDEs like Eclipse and IntelliJ to provide proper auto-completion. If you have never heard of that library, you should definitly have a look!
