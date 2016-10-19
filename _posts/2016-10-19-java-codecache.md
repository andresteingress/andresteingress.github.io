---
layout: post
title: "Java Codecache"
categories:
- java
tags: []
status: publish
type: post
published: true
---

When a Java program is run, it executes the code in a tiered manner. In the 
first tier, it uses the so-called client compiler (C1 compiler mode) in order 
to compile the code with instrumentation. The profiling data is used in the second tier (C2 compiler mode) 
for the server compiler, to compile that code in an optimised and high-performant way.

Tiered compilation is not enabled per default in Java 7 (however, it might be 
enabled by the application server that your are running), but is enabled in 
Java 8.

The JIT-compiler stores the compiled code in an area called _code-cache_. This 
area is flushed if its size exceeds a certain treshold. The memory size and 
certain other parameters like the flush treshold can be set via JVM parameters.

### When the codecache fills up

In one of our projects, we were running into a seemingly weird problem that 
actually could not be solved by the programmers for quite some time. It would
show in a way that the JEE web application running on Apache Tomcat would 
slow down after some days. The slowdown was caused by an increased load
being visible in the monitoring tools. However, there was no hint at 
performance problems neither on the database layer (in MySQL slow query 
logs) or on the application layer. It was just that the application got 
slower and slower over time. 

It was only with jconsole, a tool that comes bundled with the JDK,  
an issue got visibile. In the memory section of this tool we saw an increased
use of the "Code Cache" section. The application was running on Sun Oracle 
1.7 update 79 and therefore had a maximum code cache size of 48 MB (the 
default for the 64-bit version). As it turned out after monitoring the code cache with jconsole, 
once the code cache was filled after a couple of days, the performance
degration began. 

It was this blog article in the [Oracle knowledge base](https://blogs.oracle.com/poonam/entry/why_do_i_get_message) that pointed out 
something interesting: starting from 1.7 Update 4 until the end of the 
Java 7 branch, there are a couple of known bugs that have to do with 
filled up code caches. The most important problems in Java 7 when the code 
cache fills up are:

- The compiler may not get restarted even after the CodeCache occupancy drops down to almost half after the emergency flushing.
- The emergency flushing may cause high CPU usage by the compiler threads leading to overall performance degradation. 

There are two suggestions on how to fix this issue for Java 7:

- turn off the code-cache flushing entirely
- increase the size of the code cache up to a point never being reached

We decided to go with the 2nd option. We increased the code cache and kept watching
with jconsole. As it turned out, this led to a much better overall load of 
the system, even after two weeks, the issue previously seen did not happen anymore. 

So the outcome is that, when running still with Java 7, you should have a look
at the size of the projects code-cache in order to avoid these problems or 
to run into other bugs that have to do with flushing the code-cache.

- [Oracle Blog Post](https://blogs.oracle.com/poonam/entry/why_do_i_get_message)
- [Java 7 Performance](http://docs.oracle.com/javase/7/docs/technotes/guides/vm/
performance-enhancements-7.html)
- [Java Tiered Compilation](http://www.oracle.com/technetwork/articles/java/architect-evans-pt1-2266278.html)
