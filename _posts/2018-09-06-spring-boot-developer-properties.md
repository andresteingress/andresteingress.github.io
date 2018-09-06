---
layout: post
title: Spring Boot - Developer Properties
categories:
- spring
- springboot
- kotlin
tags: []
status: publish
type: post
published: true
---

In many of my projects there is the requirement to provide what I call "developer properties". A developer properties file holds configuration values only being applied for a specific user, e.g. `andre.properties` would be applied when JVM would have been started by a user with name `andre`. 

As an additional requirement, the developer properties should be added with high priority. Its settings will overwrite properties from other property holders.

The way such a requirement can be implemented is by having a look at interface `org.springframework.boot.env.EnvironmentPostProcessor`. This interface allows for customization of the application's `Environment` which in turn holds the property sources (= places properties come from). Classes implementing `EnvironmentPostProcessor` have to be registered in a file called `META-INF/spring.factories`:

```
# EnvironmentPostProcessors
org.springframework.boot.env.EnvironmentPostProcessor=com.exactag.uisetup.config.DeveloperConfigurationPostProcessor
```

The information found in `spring.factories` is used by Spring in a very early phase of the DI container startup and its possible to register multiple interface implementations too (separated by ','). This file will be automatically picked up by Spring when located in `src/main/resources` (in our Gradle build).

Once the implementation class is registered, we can add the actual logic and define an implementation for the `postProcessEnvironment` method (the only method of this interface):

```java
import org.springframework.boot.SpringApplication
import org.springframework.boot.env.EnvironmentPostProcessor
import org.springframework.core.env.ConfigurableEnvironment
import org.springframework.core.io.ClassPathResource
import org.springframework.core.io.support.ResourcePropertySource

class DeveloperConfigurationPostProcessor : EnvironmentPostProcessor {

    override fun postProcessEnvironment(environment: ConfigurableEnvironment, application: SpringApplication) {

        // is the dev profile active?
        if (!environment.acceptsProfiles("dev")) return

        // get the developer user name from another configuration property
        val developer = environment.getProperty("config.developer") ?: return

    	// if the user name is available, try to load his/her properties file
        val developer = environment.getProperty("config.developer") ?: return
        val developerProperties = ClassPathResource("/config/developer/$developer.properties").takeIf { it.exists() } ?: return

        // adds this new property source with highest precedence
        val mutablePropertySources = environment.propertySources
        mutablePropertySources.addFirst(ResourcePropertySource(developer, developerProperties))
    }
}
```

First of all, we do a check whether the `dev` profile is active or not. If it is active, we will read the `config.developer` property from the configuration properties. In our case, the default is configured as:

```
config.developer=${user.name}
```

`${user.name}` refers to the system property `user.name` which holds the current user name. As Spring Boot has a `SystemPropertyPlaceholderResolver` it will resolve system properties in placeholders like in the case above.

### Summary

This article showed a way how to add so-called "developer properties" files to Spring Boot projects. A developer properties file is a file with user-specific settings which will have highest priority over any other configuration properties.