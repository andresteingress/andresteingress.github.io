---
layout: post
title: Quick Tip - Spring Boot with Kotlin
categories:
- kotlin
- spring
- springboot
tags: []
status: publish
type: post
published: true
---

This article shows how to run your Gradle based Spring Boot application with Kotlin. 

### Changes in build.gradle

First of all, you need to make some additions to your Gradle `build.gradle` file. First step is to add the [kotlin-gradle plugin](https://kotlinlang.org/docs/reference/using-gradle.html):

{% highlight groovy %}

plugins {
    // ...
    id "org.jetbrains.kotlin.jvm" version "1.1.51"
}

{% endhighlight %}

The plugin version corresponds to the language runtime version, in our case it is the lastest version as of beginning October 2017 (though Kotlin 1.2 [is on its way](https://blog.jetbrains.com/kotlin/2017/09/kotlin-1-2-beta-is-out/)). 

The Gradle plugin adds a custom Kotlin Gradle source set and various Gradle targets for compiling Kotlin code in your Spring Boot project. Once you've added the plugin, you can put Kotlin files in `src/main/kotlin` or `src/test/kotlin`. 

Besides enabling the plugin, you need to add dependencies to the Kotlin standard library:

{% highlight groovy %}

dependencies {
    // ...
    compile "org.jetbrains.kotlin:kotlin-stdlib-jre8"
    compile "org.jetbrains.kotlin:kotlin-reflect"

    testCompile "org.jetbrains.kotlin:kotlin-test"
    testCompile "org.jetbrains.kotlin:kotlin-test-junit"
}

{% endhighlight %}

As we also want to add Kotlin's test support for JUnit, we need to import the testing dependencies together with the Kotlin reflect library. Note that you do not have to specify the version number as the kotlin-gradle plugin automatically applies the dependency management plugin and configures it with the plugin version.

### Summary

In this short article we showed how to configure Spring Boot with Kotlin. As Spring Framework 5 [has just been released](https://spring.io/blog/2017/09/28/spring-framework-5-0-goes-ga) last week with [dedicated support for Kotlin](https://docs.spring.io/spring/docs/current/spring-framework-reference/kotlin.html#kotlin), you might want to wait a little for Spring Boot 2.0 which is just around the corner.



