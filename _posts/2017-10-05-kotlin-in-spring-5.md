---
layout: post
title: Kotlin in Spring 5
categories:
- kotlin
- spring
- springboot
tags: []
status: publish
type: post
published: true
---

Spring 5 introduces first-class support for Kotlin 1.1+. Besides Groovy, Kotlin is now another JVM programming language besides Java getting tightly integrated into Spring. Given the latest push from [Google](https://android-developers.googleblog.com/2017/05/android-announces-support-for-kotlin.html) and [Gradle](https://blog.gradle.org/kotlin-meets-gradle) this seems especially interesting. 

### Kotlin Extension API

With version 5 Pivatal releases a [Kotlin extension API](https://docs.spring.io/spring-framework/docs/5.0.0.RELEASE/kdoc-api/spring-framework/). Currently, those extensions are available for the following Spring framework packages:

* org.springframework.beans.factory
* org.springframework.context.annotation
* org.springframework.context.support
* org.springframework.core.env
* org.springframework.jdbc.core
* org.springframework.jdbc.core.namedparam
* org.springframework.test.web.reactive.server
* org.springframework.ui
* org.springframework.web.client
* org.springframework.web.reactive.function.client
* org.springframework.web.reactive.function.server

Let's have a look at some of the newly added extensions. Many extensions basically add function extensions to already exisiting Spring APIs, making those classes easier to use from within Kotlin code and using specific language features of Kotlin to make the code more concise. 

For example, in package `org.springframework.beans.factory`, the extension `fun <T : Any> BeanFactory.getBean(): T` adds support in Springs `org.springframework.beans.factory.BeanFactory` for querying a bean by specifying the bean type as [reified type parameter](https://kotlinlang.org/docs/reference/inline-functions.html#reified-type-parameters) instead of a `Class` argument:

{% highlight kotlin %}

@Autowired
lateinit var beanFactory : BeanFactory

@PostConstruct
fun init() {
    val visitRepository = beanFactory.getBean<VisitRepository>()
    // ...
}

{% endhighlight %}

Another interesting extension example can be found in `org.springframework.ui` where operator overloading is used to add an array-like setter and getter to the `Model` interface ([JavaDoc](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/ui/Model.html)):

{% highlight kotlin %}

model["lastName"] = "Mustermann"

{% endhighlight %}

Besides Spring, other Pivotal related libraries like Spring Data or Reactor will or are already providing Kotlin extensions.

### Null-Safety

Null-Safety and integration with Java frameworks has been a weak point with Kotlin for some time. With version 5, Spring adds null-safety indicating annotations (found in `org.springframework.lang`) in the whole Spring framework API. Spring uses `@NonNull`, `@Nullable`, `@NonNullApi` and `@NonNullFields` in the `org.springframework.lang` package. Those annotations are meta-annotated with JSR-305 annotations:

{% highlight java %}

@Target({ElementType.METHOD, ElementType.PARAMETER, ElementType.TYPE_PARAMETER, ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Nonnull(
    when = When.MAYBE
)
@TypeQualifierNickname
public @interface Nullable {}

{% endhighlight %}

These annotations are used through the Spring API but they can be leveraged by application programmers to to declare nullable or non-nullable fields that may also be inspected by tools or IDEs.

In the case of the Kotlin integration those (meta-) annotations are used to guarantee null-safety. Interestingly, Spring uses null-safety also in other parts, for example, when binding HTTP request parameters. When the request param is declared as `@RequestParam name: String?` Spring handles the `name` request parameter as optional and not as a required parameter. The same goes for `lateinit`bean injections.

### GenericApplicationContext Bean DSL

`GenericApplicationContext` is another component getting a Kotlin extension. In the case of `GenericApplicationContext`, it is adding a new capability for defining application contexts. With Spring 5, besides Java and XML based configurations, a new [Kotlin BeanDefinition DSL](https://docs.spring.io/spring-framework/docs/5.0.0.RELEASE/kdoc-api/spring-framework/org.springframework.context.support/-bean-definition-dsl/) has been added. Here is an example from the [Spring documentation](https://docs.spring.io/spring/docs/current/spring-framework-reference/kotlin.html#bean-definition-dsl):

{% highlight kotlin %}

fun beans() = beans {
  bean<UserHandler>()
  bean<Routes>()
  bean<WebHandler>("webHandler") {
    RouterFunctions.toWebHandler(
      ref<Routes>().router(),
      HandlerStrategies.builder().viewResolver(ref()).build()
    )
  }

  bean("messageSource") {
    ReloadableResourceBundleMessageSource().apply {
      setBasename("messages")
      setDefaultEncoding("UTF-8")
    }
  }

  bean {
    val prefix = "classpath:/templates/"
    val suffix = ".mustache"
    val loader = MustacheResourceTemplateLoader(prefix, suffix)
    MustacheViewResolver(Mustache.compiler().withLoader(loader)).apply {
      setPrefix(prefix)
      setSuffix(suffix)
    }
  }

  profile("foo") {
    bean<Foo>()
  }
}

{% endhighlight %}

### Sealed Classes

When integrating a framework like Spring with Kotlin, an interesting aspect is the Kotlin default behavior when defining classes. Except a class is using the `open` modifier, the class is basically `final` and can not be extended. As Spring makes heavy use of proxies, the `open` modifier would be necessary in order for proxies to be generated. For this to solve, Kotling comes with its own compiler-plugin [kotlin-spring](https://kotlinlang.org/docs/reference/compiler-plugins.html#kotlin-spring-compiler-plugin) which automatically configues the all-open plugin for classes annotated with `@Component`, `@Controller`, `@Service` and other stereotype annotations. 

### Configuration Properties

Another thing to be aware of when writing Kotlin Spring components together with `@Value` is Kotlin string interpolation. As `$` is a special character in string interpolation it needs escaping when used in Kotlin classes: 

{% highlight kotlin %}

@Value("\${property}")

{% endhighlight %}

### Summary

This article shows some aspects of the new Kotlin integration new Spring Framework 5. Spring adds custom Kotlin extensions to Spring framework API classes and comes with a new Spring DSL builder. 
