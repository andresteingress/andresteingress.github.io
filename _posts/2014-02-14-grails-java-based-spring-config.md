---
layout: post
title: Grails - Java-Based Configuration
categories:
- grails
- groovy
- spring
tags: []
status: publish
type: post
published: true
---
Spring 3.0 introduced an interesting alternative to XML file-based Spring container configuration. After Spring 2.5 already introduced the _annotation-based_ configuration, it added a new way to configure the Spring bean container: via _java-based_ configuration (the SpringConfig project was taken included in the Spring framework).

### Java-Based Configuration

As its name implies, Java-based Spring container configuration is about using Java code instead of XML or annotations to configure Spring beans. This way reminds of [Google Guice](https://code.google.com/p/google-guice/), a dependency injection framework solely using Java code to setup the dependency injection components. In Spring, the `org.springframework.context.annotation` package holds the following annotations that can be used as an alternative to XML files to annotate certain classes as Spring context configuration classes:

* `@Configuration`

  The `@Configuration` class indicates that a class is defining multiple methods creating Spring-managed beans. The `@Configuration` class needs to be found by the annotation processing configuration. We will have a look how that can be configured in a Grails application.

* `@Bean`

  The `@Bean` annotation is mainly applied on methods (although it can be applied on annotation types too). It indicates that a method returns a bean manager by the Spring application container. The default strategy for the bean name is to use the name of the bean generating method.

* `@ComponentScan`

  The `@ComponentScan` annotation can be applied on `@Configuation` classes to configure component scanning for certain Java packages or classes.

* `@DependsOn`

  The `@DependsOn` annotation can be used to indicate that a bean depends on the creation of other beans. It defines a happens-before relationship in terms of the container-managed order of bean creations.

* `@Import`

  The `@Import` annotation can be used to import `@Configuration` classes into the annotated class. It resembles the `<beans:import/>` XML tag.

* `@ImportResource`

  The `@ImportResource` annotation can be used to import bean definitions into the current `@Configuration` class.

* `@Lazy`

  The `@Lazy` anntotation indicates a bean has to be created by the Spring container in a lazy manner.

* `@Profile`

  The `@Profile` annotation can be used to annotate `@Configuration` classes for certain profiles. The current profile can be set by using a VM argument or programatically by calling the `ConfigurableEnvironment.setActiveProfiles(String ...)` method.

* `@PropertySource`

  The `@PropertySource` adds property files to the current Spring environment.

* `@Scope`

  The `@Scope` annotation indicates the bean scope the bean is created in.


Now let's have a look at how Java-based Spring configuration can be used in a Grails application.

### Enabling Java-Based Configs

Let's assume we have a package `my.app.config` that holds all configuration classes. In a Grails application we need to add this packacke in `Config.groovy` to the `grails.spring.bean.packages` bean scanning path list. 

```groovy
grails.spring.bean.packages = ['my.app.config']
```

Voilà, now we can use the annotations described above inside the `my.app.config` package. The classes can either be implemented as Groovy or Java classes.

The Java-based Spring configuration can even be used in a mixed-mode: there can still be a `resources.groovy` or a `resources.xml` or more Spring configuration files in the `grails-app/conf/spring` directory, the Spring bean container will merge and resolve the bean definitions from the various sources.

### Configuration Example

The [Spring Reactor](https://github.com/reactor/reactor) project comes with a pretty neat `AsyncTaskExecutor` implementation that uses the [LMAX Disruptor library](https://github.com/LMAX-Exchange/disruptor) under the hoods. If we wanted to use the `reactor.spring.core.task.WorkQueueAsyncTaskExecutor` from Reactors Spring module, we can do so by defining a Java-based bean configuration.

In the case of the Reactor project, we're already provided with a pre-defined configuration annotation from the Reactor project that we have to use additionally on our `@Configuration` class: the `@EnableReactor` annotation.

The `@EnableRector` annotation uses the `@Import` annotation to import a default configuration into our `@Configuration` class:

```groovy
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Import(ReactorBeanDefinitionRegistrar.class)
public @interface EnableReactor {
    // ...
}
```

The `ReactorBeanDefinitionRegistrar` uses another cool feature of the Java-based Spring configuration. It implements the `o.s.context.annotation.ImportBeanDefinitionRegistrar` interface to register beans in the current application context if not already defined by the user configuration.

```groovy
public class ReactorBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {

  private static final String DEFAULT_ENV_NAME = "reactorEnv";

  @Override
  public void registerBeanDefinitions(AnnotationMetadata meta, BeanDefinitionRegistry registry) {
    Map<String, Object> attrs = meta.getAnnotationAttributes(EnableReactor.class.getName());

    // Create a root Enivronment
    if (!registry.containsBeanDefinition(DEFAULT_ENV_NAME)) {
      ...
      registry.registerBeanDefinition(DEFAULT_ENV_NAME, envBeanDef.getBeanDefinition());
    }
  }
}
```

The `ImportBeanDefinitionRegistrar` implementation can be used along `@Configuration` classes as annotation parameters for the `@Import` annotation in our configuration class:

```groovy
@Configuration
@EnableReactor
class ReactorConfiguration {

}
```

To add a `WorkQueueAsyncTaskExecutor` bean to this configuration we simply have to provide a `@Bean` annotated method:

```groovy
@Configuration
@EnableReactor
class ReactorConfiguration {

  @Bean
  AsyncTaskExecutor taskExecutor(Environment env) {
    return new WorkQueueAsyncTaskExecutor(env)
      .setName("workQueueAsyncTaskExecutor")
      .setBacklog(2048)
      .setProducerType(ProducerType.MULTI)
      .setWaitStrategy(new SleepingWaitStrategy()) 
  }
}
```

Since the method is named `taskExecutor` our bean has the same name. And of course, we could use arbitrary Java code to initialize our bean. Once defined, we can refer to `taskExecutor` in every configuration artefact of our application, let it be `resources.groovy`, `resources.xml` or any other configuration files.

Please don't confuse the `Environment` parameter with the `org.springframework.core.env.Environment` interface, the parameter is of type `reactor.core.Environment` and is injected automatically, after being created by the `@EnableRector` annotation.

If we have a look at our code example above, we could decided to move all the builder method arguments to our `app.properties` file. We can use the `@PropertySource` configuration annotation to include this file in our current configuration:

```groovy
@Configuration
@PropertySource('classpath:/my/app/config/app.properties')
@EnableReactor
class ReactorConfiguration {

  @Autowired
  Environment environment

  @Bean
  AsyncTaskExecutor taskExecutor(reactor.core.Environment env) {
    return new WorkQueueAsyncTaskExecutor(env)
      .setName(environment.getProperty("queue.name"))
      .setBacklog(environment.getProperty("queue.backlog", Integer.class))
      .setProducerType(ProducerType.MULTI)
      .setWaitStrategy(new SleepingWaitStrategy()) 
  }
}
```

As can be seen above, the `queue.name` and `queue.backlog` settings in the `app.properties` file can be accessed via the `org.springframework.core.env.Environment` bean. This bean needs to be autowired in our configuration to access the message resources. This shows another impressive feature of Spring's Java-based configuration: `@Configuration` classes are themselves treated as beans. This means we can use for example `@Autowired` to inject beans into our `@Configuration` class. The [Spring documentation](http://docs.spring.io/spring/docs/4.0.2.BUILD-SNAPSHOT/spring-framework-reference/htmlsingle/#beans-annotation-config) shows an even better example:

```java
@Configuration
public class ServiceConfig {

  @Autowired
  private AccountRepository accountRepository;

  @Bean
  public TransferService transferService() {
    return new TransferServiceImpl(accountRepository);
  }
}

@Configuration
public class RepositoryConfig {

  @Autowired
  private DataSource dataSource;

  @Bean
  public AccountRepository accountRepository() {
    return new JdbcAccountRepository(dataSource);
  }
}

@Configuration
@Import({ServiceConfig.class, RepositoryConfig.class})
public class SystemTestConfig {

  @Bean
  public DataSource dataSource() {
    // return new DataSource
  }
}

public static void main(String[] args) {
  ApplicationContext ctx = new AnnotationConfigApplicationContext(SystemTestConfig.class);
  // everything wires up across configuration classes...
  TransferService transferService = ctx.getBean(TransferService.class);
  transferService.transfer(100.00, "A123", "C456");
}
```

We can even auto-wire `@Configuration` classes themselves:

```java
@Configuration
public class ServiceConfig {

  @Autowired
  private RepositoryConfig repositoryConfig;

  @Bean
  public TransferService transferService() {
    // navigate through the config class to the @Bean method!
    return new TransferServiceImpl(repositoryConfig.accountRepository());
  }
}
```

These examples show that the Java-based configuration approach can provide us with much more modularity and reusability than using (the already mighty) Spring bean Groovy DSL which is, by the way, [part of Spring 4](http://docs.spring.io/spring/docs/4.0.2.BUILD-SNAPSHOT/spring-framework-reference/htmlsingle/#_groovy_bean_definition_dsl). As every Grails application is a Spring application, we can use this approach to better structure application provided beans.

### Conclusion

Spring 3 introduced another configuration approach besides the XML configuration files and annotation-based configuration: the so-called _Java-based configuration_. Spring provides a buch of annotations enabling to define beans in pure Java/Groovy code with even better modularity and reusability as can be found in Grails Spring DSL files. This article is an introduction to Java-based configuration and shows how to enable the mechanism in Grails applications.
