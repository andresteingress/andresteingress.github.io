---
layout: post
title: "Spock Quick-Tip: Grails Integration Tests"
categories:
- grails
- groovy
- java
tags: []
status: publish
type: post
published: true
---
I'm currently working in project with a nearly six year old Grails application. Over time, we changed from having plain JUnit 4/`GroovyTestCase` unit and integration tests to [Spock](https://github.com/spockframework) tests. Spock is such a great library to write both, unit and integration tests. Normally, we tend to write unit tests whenever possible. But there are cases (and code parts) where it is more practical to rely on a fully initialized application context and (Hibernate) data-store.

### Integration Tests with Spock

The [Spock Grails plugin](http://grails.org/plugin/spock) comes with a distinct base class for Grails integration tests: `IntegrationSpec`. 

[IntegrationSpec](https://github.com/grails/grails-core/blob/master/grails-test/src/main/groovy/grails/test/spock/IntegrationSpec.groovy) initialises the application context, sets up an autowirer that autowires bean properties from the specification class and create a transactional boundary around every feature method in the specification.

All our Spock integration tests extend `IntegrationSpec`. 

### Mocking with Groovy's meta-classes

One thing I love about Spock is that it comes out-of-the-box with [great support for mocking and stubbing](http://blog.andresteingress.com/2013/10/29/spock-mocking-and-stubbing). But there are times when you actually need to stub certain parts of the Grails artifact that is currently under test by your integration test.

We do this with the help of the Groovy Meta-Object protocol (MOP), that is, by altering the underlying meta-class. The next example shows how `getCurrentUser` is overwritten, as we do want to stub out the [Spring Security](http://grails.org/plugin/spring-security-core) part from the `StatisticsService`.

<pre><code class="language-groovy">
StatisticsServiceIntegrationTest extends IntegrationSpec {
    
    StatisticsService statisticsService

    void "count login to private area"() {

        setup:
            def user = new User(statistics: new UserStatistics())
            statisticsService.metaClass.getCurrentUser = { -> user }

        when:
            statisticsService.countLoginPA()

        then:
            user.statistics.logins == 1

    }    
}
</code></pre>

Altering classes at runtime is a nice feature, but it can also become confusing when you don't know about the side-effects it may cause. For integration tests, changes to the meta-class won't be resetted, so once you do changes to a meta-class (we are working with [per-instance meta-class](http://groovy.codehaus.org/Per-Instance+MetaClass) changes, the same is even more true for global meta-class changes) those will be persistent through the entire test.

To solve that, we added a helper method that allows to revoke meta-class changes inbetween test runs:

<pre><code class="language-groovy">
public static void revokeMetaClassChanges(Class type, def instance = null)  {
    GroovySystem.metaClassRegistry.removeMetaClass(type)
    if (instance != null)  {
        instance.metaClass = null
    }
}
</code></pre>

And applied it like this:

<pre><code class="language-groovy">
StatisticsServiceIntegrationTest extends IntegrationSpec {
    
    StatisticsService statisticsService

    void "count login to private area"() {

        setup:
            def user = new User(statistics: new UserStatistics())
            statisticsService.metaClass.getCurrentUser = { -> user }

        when:
            statisticsService.countLoginPA()

        then:
            user.statistics.logins == 1

        cleanup:
            revokeMetaClassChanges(StatisticsService, statisticsService)

    }    
}
</code></pre>

This actually sets back the meta-class code and the service class is again un-altered when executing the next feature method. 

Be warned. 

Meta-class overriding can become tricky. One thing we came across multiple times is that you can't replace methods of super classes being called from super class methods. Here is a simplified example:

<pre><code class="language-groovy">
class A {
    def a(){
       a2()

    }
    def a2(){   
         println 'In class A'
    }
}

class B extends A{
    def b(){
        a()
    }
}

B b = new B();

b.metaClass.a2 = {
    println 'In class B'
}

b.b(); // still prints 'In class A'
</code></pre>

If we wanted to stub the implementation of `b` inside our test code, this wouldn't work, as `a` and `a2` are implemented in the same class `A` and therefore the method call won't be intercepted by a per-instance change to instance `b`. This now might seem obvious, but we had a hard time tracking this down.

If you start to experience weird issues of tests failing when you run the entire test suite, but being green when executed separately, it almost certainly has to do with meta-class rewriting that isn't undone between feature methods or even specifications. Just be aware of that. 

### `@ConfineMetaClassChanges`

Lately I became aware that our `revokeMetaClassChanges` is actually "part" of Spock with the `@ConfineMetaClassChanges` extension annotation. 

The code behind it works [a bit differently](https://github.com/spockframework/spock/blob/391f3d8c5c557ce0de1e024676fb70ef71cc0d3f/spock-core/src/main/java/org/spockframework/runtime/extension/builtin/ConfineMetaClassChangesInterceptor.java) but the meaning is the same; it can be used on methods or classes to rollback meta-class changes declaratively:

<pre><code class="language-groovy">
@ConfineMetaClassChanges([StatisticsService])
StatisticsServiceIntegrationTest extends IntegrationSpec {
    
    StatisticsService statisticsService

    void "count login to private area"() {

        setup:
            def user = new User(statistics: new UserStatistics())
            statisticsService.metaClass.getCurrentUser = { -> user }

        when:
            statisticsService.countLoginPA()

        then:
            user.statistics.logins == 1

    }    
}
</code></pre>

Speaking of Spock extensions. It's definitely worth to have a look at [the chapter on Spock Extensions](http://spock-framework.readthedocs.org/en/latest/extensions.html) in the documentation. There is lots of great stuff already available (and coming in Spock 1.0).

### Conclusion

Besides Spock's great mocking and stubbing capabilities, writing Grails integration tests also involves meta-class changes. This article shows how to rollback these changes to avoid side-effects and explained the usage of `@ConfineMetaClassChanges` a Spock extension annotation.


