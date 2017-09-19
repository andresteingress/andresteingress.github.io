---
layout: post
title: "Grails, GPars and Hibernate"
categories:
- grails
- groovy
- java
- gpars
tags: []
status: publish
type: post
published: true
---
The [GPars library](https://github.com/GPars/GPars) is a great way to add concurrency supporting constructs to your Groovy code. It equips Groovy projects with powerful concurrency concepts like _parallel collections_, _map reduce_ operations, _actors_  and _dataflow variables_.

### Adding the GPars dependency

GPars can also be added to Grails applications (in our case, we have a Grails 2.2.5 application, so this article was not tested with any versions up or below). Simply add GPars as a dependency in the `BuildConfig.groovy`:

{% highlight java %}
dependecies {
  compile 'org.codehaus.gpars:gpars:some_version_number' // we use 1.1.0 for our code
}
{% endhighlight %}

That is actually enough to have the GPars features enabled, once included, you can access the _parallel collection_ methods in the targeted collection classes, like:

{% highlight java %}
GParsPool.withPool {
  [1, 2, 3, 4].findParallel { it == 3 }
}
{% endhighlight %}

You can see in the above example that the GPars convention is to use the original method name with a `Parallel` appended. The `GParsPool.withPool` method has to be called in order to initialise the GPars thread pool and equip collection classes with the parallel methods. An overview of all the available methods can be found in [GParsExecutorsPoolEnhancer](https://github.com/GPars/GPars/blob/master/src/main/groovy/groovyx/gpars/GParsExecutorsPoolEnhancer.groovy).

### GPars and Hibernate

Our requirement was to speed up a Quartz job iterating over all our customers - which are all Hibernate entities. As the customers could be easily grouped into different sets, the precondition for jork-join processing were met. But the question was how to enable Hibernate read-only processing in the `eachParallel` GPars method as we wanted to do something like:

{% highlight java %}
differentCustomerGroups.eachParallel { CustomerGroup customerGroup ->
  // process all customers of the given customer group
  // IN THE CURRENT HIBERNATE SESSION
}
{% endhighlight %}

As every Hibernate session is bound to the current thread, it was necessary to create a new Session for the current thread created by GPars, attach it and close it once processing was done.

The key to enabling this (in Grails 2.2.5 at least), was to use the `persistenceInterceptor` bean that can be injected into any Grails artefact:

{% highlight java %}
def persistenceInterceptor
{% endhighlight %}

It implements the `PersistenceContextInterceptor` interface which can be used to initialise and destroy the current persistence context. In the case of its Hibernate implementation, `HibernatePersistenceContextInterceptor`, the persistence context is the current Hibernate session. Thus, the persistence context interceptor bean can be used to initialise and destroy the session in our closure. The `eachParallel` uses the bean like that (this pattern can also be found in other places in Grails btw):

{% highlight java %}
differentCustomerGroups.eachParallel { CustomerGroup customerGroup ->
  // init the persistence context
  persistenceInterceptor.init()

  try {
    int offset = 0
    def customers = Customer.executeQuery("select c from customer c where c.customerGroup = ?", customerGroup, [readOnly: true, max: 100, offset: offset])

    // loop over customers till all are processed ...

    // flush the context
    persistenceInterceptor.flush()
  } finally {
    // destroy the context and release resources
		persistenceInterceptor.destroy()
  }
}
{% endhighlight %}

This is effectively enough to create and destroy a new Hibernate session to be used by the GPars code. Note, that we also had to disable the automatic Hibernate session creation done by Quartz, specifying the `def sessionRequired = false` property in the job class:

{% highlight java %}
class CustomerJob {
  def concurrent = false
  def sessionRequired = false // do not create a Hibernate session on job startup

  def execute() {
    // ...
  }
}
{% endhighlight %}

Another thing to note is maybe the `readOnly` option that was given to the `executeQuery` method. It disables snap-shotting of entities, which in our case was possible (since the customer instances were not modified themselves).

### Conclusion

GPars is a library that adds concurrency constructs to your class and also provides concurrency concepts like actors, agents and dataflow variables. This article shows how to handle the Hibernate session in code that is concurrently executed by GPars. The persistence context interceptor is a class provided by Grails that enables setting up and destroying the persistence context in arbitrary places. Note that this article is based on Grails 2.2.5.
