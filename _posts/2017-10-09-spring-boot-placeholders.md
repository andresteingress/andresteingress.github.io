---
layout: post
title: Quick Tip - Spring Boot Placeholders
categories:
- java
- spring
- springboot
tags: []
status: publish
type: post
published: true
---

This article shows a quick tip concerning Spring Boot `*.properties` files. As you know, `ResourceBundle` files are one of the ways besides [YAML files](https://de.wikipedia.org/wiki/YAML) in Spring Boot to configure applications. 

[As we are children of the past](https://twitter.com/iamjoyclark/status/916323971913650177), we use `*.properties` files in our current projects. Lately, we came across an issue with Hibernate Search: we wanted to configure one property, namely `spring.jpa.properties.hibernate.search.default.indexBase` with a temporary path. 

Goal was to define the index-base with a temporary directory as default-setting, to keep set up work for new developers relatively small and provide a convenient bootstrapping process for new developers.

### Externalized Configuration

There is a section in the Spring Boot documentation I must have a looked up a couple of times and which is worth noting. It is called [Externalized Configuration](https://docs.spring.io/spring-boot/docs/1.5.6.RELEASE/reference/htmlsingle/#boot-features-external-config) and describes the way how Spring Boot resolves configuration properties, using Spring's `PropertySource` mechanism ([JavaDoc](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/context/annotation/PropertySource.html)). This is the order in which a property is looked up:

    * Devtools global settings properties on your home directory (~/.spring-boot-devtools.properties when devtools is active).
    * @TestPropertySource` annotations on your tests.
    * @SpringBootTest#properties annotation attribute on your tests.
    * Command line arguments.
    * Properties from SPRING_APPLICATION_JSON (inline JSON embedded in an environment variable or system property)
    * ServletConfig init parameters.
    * ServletContext init parameters.
    * JNDI attributes from java:comp/env.
    * Java System properties (System.getProperties()).
    * OS environment variables.
    * A RandomValuePropertySource that only has properties in random.*.
    * Profile-specific application properties outside of your packaged jar (application-{profile}.properties and YAML variants)
    * Profile-specific application properties packaged inside your jar (application-{profile}.properties and YAML variants)
    * Application properties outside of your packaged jar (application.properties and YAML variants).
    * Application properties packaged inside your jar (application.properties and YAML variants).
    * @PropertySource annotations on your @Configuration classes.
    * Default properties (specified using SpringApplication.setDefaultProperties).

One thing came to our attention here. It mentions "Java System properties" as one of the `PropertySource`s to be resolved against. The available standard system property values are defined in the `System#getProperties` [JavaDoc](http://docs.oracle.com/javase/9/docs/api/java/lang/System.html#getProperties--). As Spring Boot loads property values against the `System.getProperties()` that means you can use all the mentioned property values from there in our `*.properties` files. 

### Using java.io.tmpdir

One of the system property values is `java.io.tmpdir`. It is defined to return the default temp file path. As those system properties are available via the `PropertySource` mechanism, we can simply use the temporary directory like variable in our `properties` file:

```
spring.jpa.properties.hibernate.search.default.indexBase=${java.io.tmpdir}/lucene-index/
```

This will create a `lucene-index` directory in the Java temporary file path, which is exactly what we want. 

Keep in mind, the list of `PropertySource`s from above is much then only reading from `System.getProperties()` obviously. When working on Spring Boot applications, it's a pretty good tip to internalize the available configuration sources.

### Summary

In this article we showed how to use the standard Java temporary path found in the `System` property `java.io.tmpdir` in Spring Boot `*.properties` files. 

