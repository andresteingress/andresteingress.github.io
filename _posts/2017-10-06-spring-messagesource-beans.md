---
layout: post
title: Spring - Multiple MessageSources
categories:
- java
- spring
- springboot
tags: []
status: publish
type: post
published: true
---

This article shows how to configure multiple message sources in a Spring Boot application for different environments.

### Configuring `MessageSource` Beans

The problem is that we want to actually define two Spring beans of type `org.springframework.context.MessageSource`: one for the local development environment and the other one for all the other environments supported. During development, we want to use Spring's `org.springframework.context.support.ReloadableResourceBundleMessageSource` to have automatic reloading of the message source. In the other environments though, we want to use Spring's static `ResourceBundleMessageSource`. Both beans should be named `messageSource` as this is the conventional name for the `MessageSource`. Also Spring Boot predefines a bean with such a name and type.

As we want to have two `MessageSource` beans with the same name, we decided to use two `@Configuration` classes, one for each environment, each one define a `messageSource` bean. We already have a configuration called `WebConfiguration` which is executed only in a web-environment, that is the place to add the new bean definitions:

{% highlight java %}

@Configuration
@ConditionalOnWebApplication
public class WebConfiguration {
    
    // ... other @Bean configurations for the web application 
    
    @Configuration
	@Profile("development")
	public class LocalMessageSourceConfiguration {
		
		@Bean
		public ReloadableResourceBundleMessageSource messageSource() {
			ReloadableResourceBundleMessageSource source = new ReloadableResourceBundleMessageSource();
			source.setBasename("classpath:messages");
			source.setCacheSeconds(0); 
			source.setDefaultEncoding("UTF-8");
			return source;
		} 
	}
	
	@Configuration
	@Profile("!development")
	public class MessageSourceConfiguration {
		
		@Bean
		public ResourceBundleMessageSource messageSource() {
			ResourceBundleMessageSource source = new ResourceBundleMessageSource();
			source.setBasename("messages");
			source.setDefaultEncoding("UTF-8");
			return source;
		}
	}
    
}

{% endhighlight %}

The trick we use here to add two `@Configuration` annotated inner classes which both define the bean named `messageSource` of some `MessageSource` interface implementation.

In addition, we use the `@Profile("development")` annotation for the configuration that should be active in the development environment. For the other configuration, we use the NOT operator in the profile name given to `@Profile` to have this configuration only being executed in all environments except `development`: `@Profile("!development")`.

The default encoding for `java.util.ResourceBundle` is ISO-8859-1, therefore this is the default encoding Spring's `MessageSource` implementations assume. Starting with Java 9, the default encoding of `ResourceBundle` [will be UTF-8](https://blog.andresteingress.com/2017/10/04/java-9-trivias.html) so we chose to use UTF-8 right from the start.

Notice that `source.setCacheSeconds(0);` in the case of the `ReloadableResourceBundleMessageSource` configures the message source to reload the message source based on the modification date of the message source file(s). This check is done with every message source access, so it is strongly discouraged to use such a configured `MessageSource` in production. 

### Summary

In this article we showed how to configure different types of `MessageSource` beans for different environments in Spring Boot. 

