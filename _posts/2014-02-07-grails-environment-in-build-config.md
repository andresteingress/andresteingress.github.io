---
layout: post
title: Grails - Environments in BuildConfig
categories:
- grails
- groovy
tags: []
status: publish
type: post
published: true
---
As of Grails 2.3 it is possible to use the `Environment` class everywhere in `BuildConfig.groovy`. This can be useful for environment-specific settings regarding all the configuration that can be done in `BuildConfig.groovy` including dependencies and plugins.

### What's the deal?

Recently I came across an issue with the UI performance plugin (like mentioned by someone [on the Grails user list](http://grails.1312388.n4.nabble.com/UI-Performance-NoSuchMethodError-td1385924.html)) where we needed to exclude dependencies to be only used in the `test` environment. Before Grails 2.3.x it was only possible to use the `Environment` class inside the `dependencies` or `plugins` closure:

{% highlight java %}

dependencies {
	
  	// ...

	if (Environment.current != Environment.PRODUCTION)  {
    	test "org.gebish:geb-junit4:0.9.2"
        test "org.seleniumhq.selenium:selenium-support:2.39.0"
        test "org.seleniumhq.selenium:selenium-firefox-driver:2.39.0"
        test "org.seleniumhq.selenium:selenium-chrome-driver:2.39.0"
    }
}

plugins {
	
	// ...

	test ":spock:0.7"
    test(":cucumber:0.10.0") {
    	exclude "spock-core"
    }

    test ":code-coverage:1.2.7"
    test ":geb:0.9.2"
}
{% endhighlight %}

This little trick was possible because the closure was evaluated at a point of time where the `Environment` already had been initialized. Something along the lines of

{% highlight java %}
grails.server.port.http = 8080

Environment.executeForEnvironment(Environment.DEVELOPMENT)  {
    grails.server.port.http = 8082	
}
{% endhighlight %}

wouldn't have been successful caused by the `Environment` class not being initialized at runtime.

This behaviour has been [fixed in Grails 2.3](http://jira.grails.org/browse/GRAILS-4260), the BuildConfig DSL has even been extended to support the _environments block_ feature. We can use it to specify environment-specific dependencies blocks or other BuildConfig configuration settings:

{% highlight java %}
environments {
	test {
		dependencies {
			test "org.gebish:geb-junit4:0.9.2"
        	test "org.seleniumhq.selenium:selenium-support:2.39.0"
        	test "org.seleniumhq.selenium:selenium-firefox-driver:2.39.0"
        	test "org.seleniumhq.selenium:selenium-chrome-driver:2.39.0"
		}
	}
}
{% endhighlight %}

### Conclusion

Before 2.3.x the `Environment` could only be used restrictively in `BuildConfig.groovy`. With 2.3.x this issue has been fixed and support for the `environments` block has been added to the BuildConfig DSL. 
