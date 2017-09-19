---
layout: post
title: Spring Boot's @Conditional Annotations
categories:
- java
- spring
- spring boot
tags: []
status: publish
type: post
published: false
---

One of the outstanding features of Spring Boot is without doubt its auto-configuration capability. However, auto-configuration is implemented upon another great Spring feature: conditional annotations. 

### @Conditional Annotations

Spring 3.1 came with support for the environment abstraction. Along came the `@Profile` annotation that we talked about in one of the [last blog posts](http://blog.andresteingress.com/2017/09/18/environment-meta-annotations).

However, Spring 4.0 introduced a new mechanism on which since then `@Profile` is based: the `@Conditional` annotation and the `Condition` interface.

[Conditional](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/context/annotation/Conditional.html) can be used as (meta-)annotation on components in order to define rules that determine wether the annotated component/bean/factory method is eligible for registration in the DI container. The rule part is done via implementing the [Condition](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/context/annotation/Condition.html) interface. 

When we have a look at `@Profile` again, we can see that it is nothing more then an `@Conditional` annotation (when putting the usual retention annotation et. al. aside):

{% highlight java %}
...
@Conditional(ProfileCondition.class)
public @interface Profile { .. }
{% endhighlight %}

The `ProfileCondition` implementation is rather easy, so we can show the entire source code:

{% highlight java %}
class ProfileCondition implements Condition {

	@Override
	public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
		if (context.getEnvironment() != null) {
			MultiValueMap<String, Object> attrs = metadata.getAllAnnotationAttributes(Profile.class.getName());
			if (attrs != null) {
				for (Object value : attrs.get("value")) {
					if (context.getEnvironment().acceptsProfiles(((String[]) value))) {
						return true;
					}
				}
				return false;
			}
		}
		return true;
	}
}
{% endhighlight %}

As you can see, it basically has a look at the current environment and sees if the environment names specified in the annotation are accepted. If so, the bean will be registered.

The `AnnotatedTypeMetadata` argument holds meta-data about the annotated type. This includes information about all the annotations. Interesting part here is that Spring retrieves this information without necessarily requiring class-loading. 

### Conditional Annotations in Spring Boot

Spring Boot takes conditional annotations to a whole new level. It comes with various `@Conditional*` annotations being very useful for library/framework but also application developers. Let’s go through our favourite three annotations, keep in mind that the list is by far not complete. So whenever you are looking for some conditional exclusion or inclusion of beans, have a look in the `org.springframework.boot.autoconfigure.condition` package:

`@ConditionalOnClass`

[ConditionalOnClass](https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/autoconfigure/condition/ConditionalOnClass.html) can be used to register a bean only when the given class(es) can be found on the class-path. You can define the fully qualified class name either via a String or via a `Class<?>` reference. Sounds a bit like a chicken and egg problem because you are defining and importing classes that might not be available at compile-time. But as Spring uses [ASM](http://asm.ow2.org) to gather the annotation meta-data it is possible to refer to classes not being on the current class-path without running into linkage errors. Especially when providing library/framework classes to other developers, this annotation can be used to active certain beans only based on the existing of specific classes, e.g. driver-classes etc.

`@ConditionalOnWebApplication`

[ConditionalOnWebApplication](https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/autoconfigure/condition/ConditionalOnWebApplication.html) can be used to define beans that only exist if the application is executed in a JEE-based servlet-context/web application context. For example, when you are developing a Spring Boot based JSF application, you could think about defining a custom annotation `@Web` which can use `@ConditionalOnWebApplication` as meta-annotation, which works without problems by the way:

{% highlighting java %}
...
@ConditionalOnWebApplication
public @interface Web { .. }
{% endhighlight %}

`@ConditionalOnProperty`

[ConditionalOnProperty](https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/autoconfigure/condition/ConditionalOnProperty.html) can be used to register beans only if certain configuration properties are being found with the specified values. This is useful if you have the case where you want to activate certain beans based on a value in the application.properties file(s). In that case, it is even enough to simply specify the configuration property, if it is `false` or not given at all, it will not register the affected beans:

{% highlighting java %}
@Configuration
@ConditionalOnProperty("my.feature.active")
public class MyFeatureConfiguration { ... }
{% endhighlight %}

### Summary

Spring Boot has a very cool set of conditional annotations. Conditional annotations are based upon Spring 4.0‘s `Condition` interface and its `@Conditional` annotation. If you are using Spring Boot, there is a very high chance one of the `@Conditional*` annotations are becoming interesting at some point or another.