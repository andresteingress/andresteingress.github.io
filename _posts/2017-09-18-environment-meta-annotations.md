---
layout: post
title: „Environment Meta-Annotations“
categories:
- java
- spring
tags: []
status: publish
type: post
published: true
---

Starting with Spring 3.1, the framework introduced a new `@Profile` [annotation](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/annotation/Profile.html) - along with the introduction of the new environment abstraction.

The `@Profile` annotation can be applied on type or method level. It indicates that a certain component or bean factory result is only eligible for bean registration once the specified profile is currently active. 

It‘s especially interesting that even `@Configuration` classes can be annotated with `@Profile`. This allows for certain configurations only being valid in certain profiles or (as they are often called) environments. Be its also perfectly valid to apply the annotation on custom components annotated with Spring‘s stereotype annotations (e.g. `@Component`, `@Service` et. al.). 

### Meta-Annotations

There is even another way how to apply the `@Profile` annotation. It can be used in meta-annotations for the purpose of composing custom annotations. 

This is a common property that is quite often found in Spring: it‘s possible to take annotations provided by Spring and combine them in custom annotations, so-called meta-annotations.

In the case of the `@Profile` annotation it means that you - as a library or framework developer - can encapsulate the environments behind custom annotations. 

### Example

Let‘s assume we have a „development stage“ where we have certain beans only eligible in the developers local environment. 

One way to define a bean only valid in development phase, is to define the `@Profile` annotation on that component:

<pre><code class="language-groovy">
@Component
@Profile(„development“)
public class ApplicationBootstrap { ... }
</code></pre>

The `AppplicationBootstrap` bean would therefore be only registered once the development environment/profile is active.

The same can also be achieved with a custom meta-annotation:

<pre><code class="language-groovy">
@Target(value={TYPE})
@Retention(value=RUNTIME)
@Documented
@Profile(„development“)
public @interface Development {}
</code></pre>

Note that the `@Development` annotation has the `@Profile(„development“)` annotation applied. With this custom annotation we could refactor the previous code sample just like that:

<pre><code class="language-groovy">
@Component
@Development
public class ApplicationBootstrap { ... }
</code></pre>

We could even further reduce the code in case we also decided to add even the `@Component` annotation to `@Development`. However, there is also a lot for keeping the meaning of `@Development` simple and clear, dedicated only to the purpose of defining the development environment. 

The advantage of the custom annotation is the replacement of the profile string (`development`). We could even go further and define all the available environments as constants in an `ApplicationEnvironments` class. So we would then have a single point of pre-defined profile/environment names available to the libraries/frameworks clients.
