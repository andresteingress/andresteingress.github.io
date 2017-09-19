---
layout: post
title: Spring Bean Removal
categories:
- java
- spring
tags: []
status: publish
type: post
published: true
---

In one of my current projects we had the requirement to remove Spring beans during application startup. The actual context being that batch programs shared the same application code then the web application. In such a situation, there is a need to exclude certain beans from starting up in the batch code and vice versa.

### BeanFactoryPostProcessor

So we had the requirement for removing Spring beans during the application container startup. For the batch code it ment that we needed to remove all beans having some sort of HTTP-based scope. Spring comes with `@RequestScope`, `@SessionScope` ([you can find more about Spring scopes here](https://docs.spring.io/spring/docs/3.0.0.M3/reference/html/ch04s04.html)) and we also had two custom scopes for tab and view scopes. 

All those scopes, being based on Spring‘s `@Scope` annotation and [CustomScopeConfigurer](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/beans/factory/config/CustomScopeConfigurer.html) at some point or another use the [Servlet API](http://docs.oracle.com/javaee/6/api/index.html?javax/servlet/package-summary.html) in order to store the corresponding state either in the HTTP request or the HTTP session. 

It was about exactly those beans to remove them and start batch applications without them. 

After some digging around, we decided to implement our requirement with a custom [BeanFactoryPostProcessor](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/beans/factory/config/BeanFactoryPostProcessor.html). This interface allows for the modification of so-called bean registration objects. Note that it is bean registration objects and not beans. BeanFactoryPostProcessors are triggered by Spring after the bean meta-data is available and just before the actual bean instances are about to be created.

So this seemed like the exact hook in order to do our modifications. 

### The Implementation

In order to find beans implementing one of `@RequestScope`, `@SessionScope` or our own `@TabScope` and `@ViewScope` scopes, we need to go through all the registered bean names:

<pre><code class="language-groovy">
@Bean
public static BeanFactoryPostProcessor registerPostProcessor() {
	return (ConfigurableListableBeanFactory beanFactory) -> {
		BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
		for (String beanDefinitionName : registry.getBeanDefinitionNames()) {
			// do some work ...
		}
		// ...
	};
}
</code></pre>

Don't get confused by the implementation of the `BeanFactoryPostProcessor` here. It uses a Java 8 lamda to implement this [SAM interface](https://stackoverflow.com/questions/17913409/what-is-a-sam-type-in-java).

For every [BeanDefinition](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/beans/factory/config/BeanDefinition.html) we try to determine its current scope. Luckily, there is a `getScope()` method on every bean definition that can be used for that.

Once the scope is one of our to-be-excluded scopes, we can simply call `removeBeanDefinition(String)` from the [BeanDefinitionRegistry](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/beans/factory/support/BeanDefinitionRegistry.html) interface, which we get with `ConfigurableListableBeanFactory` as an argument to `postProcessBeanFactory` our single method we have to implement with the `BeanFactoryPostProcessor` interface:

<pre><code class="language-groovy">
if (beanDefinition != null && isWebScope(beanDefinition.getScope())) {
	if (registry.containsBeanDefinition(beanDefinitionName)) {
    	registry.removeBeanDefinition(beanDefinitionName);
    }
}
</code></pre>

The actual work to determine whether the scope is a web scope or the standard Spring scope, is done in `isWebScope`:

<pre><code class="language-groovy">
private static boolean isWebScope(String scope) {
	return !(StringUtils.isBlank(scope) || 
    	ConfigurableBeanFactory.SCOPE_PROTOTYPE.equals(scope) || 
        ConfigurableBeanFactory.SCOPE_SINGLETON.equals(scope));
}
</code></pre>

This implementation is kind of an all-in variant. It will simply remove all beans that are not singleton or protoype scoped, so the implementation could also have been to check only the HTTP-related scopes and excludes only those. We decided for this variant in order to automatically support new scopes as there is a very high possibility new scopes will be based on the HTTP request or session in some way. 

Let us now assume there is a class `IndexView` in our code base:

<pre><code class="language-groovy">
@Component
@RequestScope
public class RequestInformation { ... }
</code></pre>

As `RequestInformation` is marked as request-scoped it needs to be instantiated on every request by the Spring DI container. This is usually done via proxies. Let‘s assume `RequestInformation` is used from another component that is indeed a singleton:

<pre><code class="language-groovy">
@Service
public class UserService { 
    
    @Autowired
    private RequestInformation requestInformation;
    
    ...
}
</code></pre>

Spring needs to replace the logical `RequestInformation` reference on every request. This is done via AOP proxies. The proxy is injected like a singleton bean but implements the logic to fallback to the correct scope to delegate to the actual reference. 

In order to find more about the HTTP scope bean proxing provided by Spring, [ScopedProxyUtils](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/aop/scope/ScopedProxyUtils.html) is a good starting point. This is Spring's utility class to create scoped proxies. When having a look at the method `createScopedProxy` we see that Spring will create actually two beans for our `requestInformation` bean. The proxy bean would get the name `requestInformation`, the target bean gets the name `scopedTarget.requestInformation`. 

All of that is necessary to understand that there are actually two bean definitions in the bean registry for HTTP scoped beans.

For our `UserService` we would find a bean definition for `userService`and for `targetScope.userService`.

So for our `BeanPostFactoryProcessor` to work, we need to take into account both bean definitions as removing either one of them would lead to errors during the container startup. So as a matter of fact, we end up with this implementation:

<pre><code class="language-groovy">
@Configuration
public class BatchConfiguration {
    
    private static final String SCOPE_BEAN_NAME_PREFIX = "scopedTarget";
    
    @Bean
    public static BeanFactoryPostProcessor registerPostProcessor() {
        return (ConfigurableListableBeanFactory beanFactory) -> {
            
            BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
                
            for (String beanDefinitionName : registry.getBeanDefinitionNames()) {
                BeanDefinition beanDefinition = registry.containsBeanDefinition(beanDefinitionName) ? registry.getBeanDefinition(beanDefinitionName) : null;
                if (beanDefinition != null && isWebScope(beanDefinition.getScope())) {
                    
                    if (registry.containsBeanDefinition(beanDefinitionName)) {
                        log.info("Removing bean definition for bean {} as it should not be used outside a web application context", beanDefinitionName);
                        
                        registry.removeBeanDefinition(beanDefinitionName);
                    }
                    
                    // see ScopedProxyUtils: Spring actually creates 2 bean definitions 
                    // the proxy bean gets the original bean name, the target bean gets the name "scopedTarget.${originalBeanName}"
                    if (beanDefinitionName.startsWith(SCOPE_BEAN_NAME_PREFIX + ".")) {
                        String beanWithoutScope = beanDefinitionName.replace(SCOPE_BEAN_NAME_PREFIX + ".", "");
                        
                        if (registry.containsBeanDefinition(beanWithoutScope))  {
                            log.info("Removing bean definition for bean {} as it should not be used outside a web application context", beanWithoutScope);
                            
                            registry.removeBeanDefinition(beanWithoutScope);
                        }
                    }
                }
            }
        };
    }
    
    private static boolean isWebScope(String scope) {
        return !(StringUtils.isBlank(scope) || 
                ConfigurableBeanFactory.SCOPE_PROTOTYPE.equals(scope) || 
                ConfigurableBeanFactory.SCOPE_SINGLETON.equals(scope));
    }
}
</code></pre>

### Summary

Spring provides various hooks to influence the way how the DI container actually creates beans. In this article we showed a way how to adapt Spring with a `BeanFactoryPostProcessor` in order to remove certain beans when running in batch classes.




