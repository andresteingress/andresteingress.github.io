---
layout: post
title: Grails Quick Tip - Cleaning Up
categories:
- grails
- groovy
tags: []
status: publish
type: post
published: true
---
Mr. Haki, once again, wrote [a really cool blog post](http://mrhaki.blogspot.co.at/2014/02/grails-goodness-cleaning-up.html) about `grails clean` and how it can be configured to delete all or only certain generated files in the working directory.

When reading the blog post a trick came to my mind that was once mentioned by Burt Beckwith in his excellent book [Programming Grails](http://shop.oreilly.com/product/0636920024750.do).

Burt said he often replaces the standard configuration in `BuildConfig.groovy`

{% highlight java %}
grails.project.class.dir = "target/classes"
grails.project.test.class.dir = "target/test-classes"
grails.project.test.reports.dir = "target/test-reports"
{% endhighlight %}

with

{% highlight java %}
grails.project.work.dir = "target"
{% endhighlight %}
This has the advantage that all generated classes, including plugin-classes, are saved in a single directory. This is really cool as installed plugins would otherwise be saved and cached in the `~/.grails/` directory.

![Grails Work Dir Target](/assets/grails-clean.png)

Once things mess up and a clean state is needed, a simple `rm -rf target` gives us a completely zero state in terms of generated artefacts.



