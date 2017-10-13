---
layout: post
title: Spring Boot - Multiple Data-Sources
categories:
- java
- spring
- springboot
tags: []
status: publish
type: post
published: true
---

This article is about configuring multiple data-sources in Spring Boot applications. 

The convention over configuration in Spring Boot is to configure a single data-source. This is done via the `spring.datasource.*` properties and the configuraton classes from the Spring Boot package `org.springframework.boot.autoconfigure.jdbc.*`. 

When having a look at class `org.springframework.boot.autoconfigure.jdbc.DataSourceConfiguration` you will see that it defines multiple `@Configuration` classes for the various environments where data-sources can be defined (e.g. with Tomcat, in a DBCP2 connection pool etc.). Each of the `@Bean` methods configure a data-source. The data-source properties are set with class `org.springframework.boot.autoconfigure.jdbc.DataSourceProperties` which is basically a `@ConfigurationProperties` container for the property values:

{% highlight java %}
@ConfigurationProperties(prefix = "spring.datasource")
public class DataSourceProperties
        implements BeanClassLoaderAware, EnvironmentAware, InitializingBean {

    private ClassLoader classLoader;

    private Environment environment;

    /**
     * Name of the datasource.
     */
    private String name = "testdb";

    /**
     * Generate a random datasource name.
     */
    private boolean generateUniqueName;

    /**
     * Fully qualified name of the connection pool implementation to use. By default, it
     * is auto-detected from the classpath.
     */
    private Class<? extends DataSource> type;

    /**
     * Fully qualified name of the JDBC driver. Auto-detected based on the URL by default.
     */
    private String driverClassName;

    /**
     * JDBC url of the database.
     */
    private String url;

    /**
     * Login user of the database.
     */
    private String username;

    /**
     * Login password of the database.
     */
    private String password;

    /**
     * JNDI location of the datasource. Class, url, username & password are ignored when
     * set.
     */
    private String jndiName;
    
    // ...
{% endhighlight %}

As you can see above, all the properties found with prefix `spring.datasource` are bound against an instance of this class.

Now let's say we want to introduce a second data-source besides the Spring Boot configured one. In order to do so, we have to overwrite the default `DataSource` bean and mark it as primary one to be used by Spring and we have to define a second `DataSource` bean.

When configuring the beans, we can reuse Spring Boot's `DataSourceProperties` in that we use it in a `@Bean` method we annotate with `@ConfigurationProperties`. In such a way we can have exactly the same property names for configuring the data-source bean as Spring Boot, but mapped to another prefix:

{% highlight java %}
@Configuration
public class DataSourceConfiguration {
    
    @Bean
    @Primary
    @ConfigurationProperties("app.datasource")
    public DataSourceProperties appDataSourceProperties() {
        return new DataSourceProperties();
    }
    
    @Bean
    @Primary
    public DataSource appDataSource() {
        return appDataSourceProperties().initializeDataSourceBuilder().build();
    }
}
{% endhighlight %}

The custom `DataSourceConfiguration` from above defines a new primary `java.sql.DataSource` bean. The `@Primary` annotation is important so that this data-source is treated as primary data-source that has precedence for Spring Boot's auto-configuration mechanism e.g. when creating an `EntityManagerFactoryBean` etc.

Let's add the second data-source:

{% highlight java %}
@Configuration
public class DataSourceConfiguration {
    
    @Bean
    @Primary
    @ConfigurationProperties("app.datasource")
    public DataSourceProperties appDataSourceProperties() {
        return new DataSourceProperties();
    }
    
    @Bean
    @Primary
    public DataSource appDataSource() {
        return appDataSourceProperties().initializeDataSourceBuilder().build();
    }
    
    @Configuration
    @ConditionalOnProperty(name = "app.opt.datasource.active")
    public static class SomeOtherConfiguration {
        
        @Bean
        public DataSourceProperties appOptDataSourceProperties() {
            return new DataSourceProperties();
        }
        
        @Bean
        public DataSource appOptDataSource() {
            return appOptDataSourceProperties().initializeDataSourceBuilder().build();
        }
    }
}
{% endhighlight %}

We do so by adding a nested `@Configuration` being only active if the property value `app.opt.datasource.active` is set to `true`. That's basically what `@ConditionalOnProperty(name = "app.optdatasource.active")` does. It checks the `app.opt.datasource.active` property and if it is present and `true`, the `@Configuration` will get executed and the second data-source bean will be defined.

If course, you could even add more data-sources, for example, you could add data-sources depending on the current Spring environment/profile currently active. But the general scheme would be the same.

### Summary 

This article showed how to configure multiple data-sources in a Spring Boot application. Besides the default data-source, we add an optional data-source that can be used from within the application code for special purposes.
