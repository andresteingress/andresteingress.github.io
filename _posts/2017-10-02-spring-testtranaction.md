---
layout: post
title: Spring - TestTransaction
categories:
- java
- java9
- testing
tags: []
status: publish
type: post
published: true
---

Spring comes with great support for [writing unit and integration tests](https://docs.spring.io/spring/docs/current/spring-framework-reference/testing.html#testing-introduction). However there is one detail I wanted to point out today which makes the life of developers easier in certain situations. 

Spring comes with the so-called _TestContext_  framework, location in package `org.springframework.test.context`. It provides generic support for writing unit and integration tests, not being bound towards a particular test library or framework. 

In addition to the generic test support, Spring comes with explicit support for JUnit 4/5 and TestNG. In our projects, we are basically using JUnit, so we choose to run all Spring-related tests with Spring's JUnit runner `SpringRunner` ([JavaDoc](https://docs.spring.io/spring/docs/4.2.5.RELEASE_to_4.3.0.RC1/Spring%20Framework%204.3.0.RC1/org/springframework/test/context/junit4/SpringRunner.html):

{% highlight java %}
import org.junit.runner.RunWith;
import org.springframework.test.context.junit4.SpringRunner;

@RunWith(SpringRunner.class)
public class MyTests { ... }
{% endhighlight %}

Once the `SpringRunner` is used, we can make use the TestContext functionality provided by Spring. `TestContext` is basically an abstract encapsulating the context in which a test is executed. It may also create a Spring `ApplicationContext` instance for testing, this is done on-demand. 

The `TestContextManager` is the central class managing a single `TestContext`. It allows `TextExecutionListener` instances to be registered, which will be trigged on certain events like before/after a test class is executed, or before/after a test method is executed etc. Spring comes with multiple implementations for this interface, for this article only one of them is important: `org.springframework.test.context.transaction.TransactionalTestExecutionListener`.

### Transactional Test-Cases

`TransactionalTestExecutionListener` is a `TestExecutionListener` which will create a transaction for test methods annotated with `org.springframework.transaction.annotation.Transactional`. By default, it will rollback the transaction when the test method has executed.

So to enable the functionality by this listener you have to configure a `PlatformTransactionManager` bean within the test application context (loaded e.g. via `@ContextConfiguration`) _and_ you must use the `@Transactional` annotation on either class- or method-level. By the way, the mentioned annotations can all be applied in a base class and will be found/inherited in/to concrete test classes:

{% highlight java %}
@RunWith(SpringRunner.class)
@ContextConfiguration(classes = { PersistenceConfiguration.class })
@Transactional
public abstract class IntegrationTestSupport {
    ...
}
{% endhighlight %}

The `IntegrationTest` base class can be further used in concrete implementations:

{% highlight java %}
public class ConcreteTests extends IntegrationTestSupport {
    
    @Autowired
    private EntityManager em;
    
    @Test
    public void persistCar() {
        
        Car car = new Car();
        car.setName("Fiat");
        
        em.persist(car);
        
        Assert.assertTrue(car.getId() != null);
    }
}
{% endhighlight %}

In the above test-case, the `Car` instance would have been inserted in the current transaction opened by the declarations in our `IntegrationTest` parent class. As the `TransactionalTestExecutionListener` is automatically configured by the TestContext framework, the transaction will be rolled back when `persistCar` completes.

For a vast majority of tests, this default behaviour is good enough. However, there are cases where the transactional boundaries themselves are subject to test. A typical example might be to test the flushing behaviour of your JPA persistence provider (e.g. Hibernate), our the automatic indexing done on transaction commit by our search framework (e.g. Hibernate Search). 

Prior to Spring 4.1 it has been a bit more difficult to handle such cases where someone wanted to do trigger a commit or rollback in the middle of the test-case and not just in the end. Starting with Spring 4.1, there comes a nice class being part of the TestContext framework: `TestTransaction` ([JavaDoc](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/test/context/transaction/TestTransaction.html)).

### TestTransaction

`org.springframework.test.context.transaction.TestTransaction` provides static methods for programmatic transactions. The methods might be used within a test method, before or after a test method. Support fo this class is automatically available once `TransactionalTestExecutionListener` is used, which is the case by default as we have seen above. Let's assume we wanted to test certain aspects of our JPA persistence provider. With persistence provider it is indeed the case that some functionality is only really executed on transaction commits. We want to write a test-case hopefully fulfilling our assumptions:

{% highlight java %}
@Test
public void persistCar() throws Exception {

    Car car = new Car();
    car.setName("Fiat");
    
    em.persist(car);
    
    TestTransaction.flagForCommit();
    TestTransaction.end();
    
    em.clear();
    
    TestTransaction.start();
    
    car = em.find(Car.class, car.getId());

    Assert.assertNotNull(car);
    
    CriteriaBuilder cb = em.getCriteriaBuilder();
    CriteriaQuery<Long> q = cb.createQuery(Long.class);
	q.select(cb.count(q.from(Car.class)));
	
    long carsTotal = (long) em.createQuery(q).getSingleResult();
    
    Assert.assertEquals(1l, carsTotal);
}
{% endhighlight %}

In the example above, we use the static methods from `TestTransaction` to flag the current transaction for a commit and end it afterwards (resulting in a commit). Then we open a new transaction and try load the `Car` instance again in this transaction to see whether the instance has been really persisted/saved or not. A `TestTransaction.start()` at the beginning of the test case is not necessary because Spring's `TestExecutionListener` opens automatially a transaction. 

`TestTransaction` is really cool and easy to use, however, we now have the problem that we are executing real commits against our test database. However, there is a trick we apply. As we are running on [H2](http://h2database.com/html/main.html) (an in-memory DB), we can use the `@DirtiesContext` ([JavaDoc](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/test/annotation/DirtiesContext.html)) annotation to throw the entire ApplicationContext away after every test-method. As a side-effect, our in-memory database will be removed too (as the data-source will be removed too) and the next test-method can execute its code on a brand new in-memory database. So let's modify `IntegrationTestSupport` to reflect this change:

{% highlight java %}
@RunWith(SpringRunner.class)
@ContextConfiguration(classes = { PersistenceConfiguration.class })
@DirtiesContext(classMode = ClassMode.AFTER_EACH_TEST_METHOD)
@Transactional
public abstract class IntegrationTestSupport {
    ...
}
{% endhighlight %}
  
This this change we finally get the behaviour we want for our test methods. We can do programmtic commits/rollbacks in our test methods and every test-method can be executed upon a new state of the embedded database.

### Summary

The article shows how to use Spring's `TestTransaction` for tests focusing on transaction boundaries. 

