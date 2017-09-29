---
layout: post
title: Java 9 - Modules
categories:
- java
- java9
tags: []
status: publish
type: post
published: true
---

Our previous Java 9 articles ([0](https://blog.andresteingress.com/2017/09/26/java-9-jshell.html), [1](https://blog.andresteingress.com/2017/09/27/java-9-optional-additions.html), [2](https://blog.andresteingress.com/2017/09/28/java-9-stackwalker-api.html)) dealt with rather small Java 9 features. Today we will have a look at the big elephant in the room: the Java Platform Module System (JPMS) specified by [JSR 376](https://www.jcp.org/en/jsr/detail?id=376).

### Motivation

What is the motivation behind introducing a new module system to the JDK, influencing the entire Java stack from the virtual machine, the Java compiler, linker, packing tools like `jar` and others and of course the Java standard library components themselves?

The primary goals of introducing modularisation have been mentioned as follows:

* Better maintenance of large applications and libraries/frameworks due to enforcement of strong encapsulation and explicit module dependency declaration
* Security improvements
* Improved application performance as types in modules can be detected much faster based on their package
* Enable the JDK and SE to scale down for use in small computing devices and cloud deployments

### Class Loaders

The _class loader_ is the central component loading Java classes at run-time. Once a class is loaded, it is basically known to the runtime and the code on behalf of this class can be further executed.

As experienced Java developers know, in every java application - and particularly in web applications - there is a more or less complex hierarchy of class loaders, each class loader can have either none or exactly one parent. 

When a class is looked up for the first time, the current class loader will look for this class in its class path. Class loaders are normally using a top-down approach for class lookup. So starting at the bootstrap class loader, the class loader hierarchy will be processed in reverse order. 

Starting with Java 9 and the introduction of modules, class loaders are enriched with a new capability: looking up _modules_ and retrieving _classes_ from therein. But let's have a look at the module concept first.

### Module Concept

The _module_ is a completely new Java program component. Before Java 9, there were classes in packages (with corresponding folders in the file system) and those classes have been found by class loaders. Starting with Java 9, modules are introduced directly in-between packages and class loaders. It encapsulates packages and the classes therein. It is not restricted to include only classes, it may also contain other data like resources and static information.

In modules, only explicitly exported `public` classes can be used from other modules. As a matter of fact, this is a semantical change to the `public` visibilty class modifier.

To declare an _explicit_ module, a module declaration needs to be specified. This is done in a file called `module-info.java`. As you can see from the file extension, this is a `.java` file that indeed gets compiled into a `.class` file by the java compiler.

{% highlight java %}
module com.ast.app {
    requires java.logging;

    exports java.logging.ast.app;
}
{% endhighlight %}

Don't get confused by the module-name being the same as the package name. `module com.ast.app` defines the modul name, `exports com.ast.app` exports all `public` classes found in the `com.ast.app` package inside the `com.ast.app` module. If we wouldn't specify the line containing the `exports`, we would not allow access to any types, even though they might be `public`, to other modules.

The `module-info.java` is located in the source file root folder:

{% highlight shell %}
src/com.ast.app/module-info.java
src/com.ast.app/com/ast/app/SomeClass.java
{% endhighlight %}

Directly under the `src` folder is a folder for the `com.ast.app` module. Inside that folder, you can find the `module-info.java` together with the usual package directory structure. The `javac` compiler will transform the `module-info.java` into a `module-info.class` file containing the information from the source file. The class file might have additional information besides the source code information: IDEs/tools might decide to include custom class-file attributes (e.g. module version, title, license etc.). We will have a look how new `javac` and `jar` command-line parameters can do exactly that in the next section.

However, the `module-info.class` is treated by `jar` like a normal class file and will be packaged into the generated `*.jar` file. A JAR file with a module declaration is called a _modular JAR file_. 

A modular JAR file can be used as a regular JAR file, it is compatible with Java versions prior to Java 9. For modularizing the Java SE platform, an additional artefact format has been introduced: JMOD. Whether this format should be standardized is an open question, [according to this source](http://openjdk.java.net/projects/jigsaw/spec/sotms/2016-03-08).

### Module Usage

Let's say we have configured our module like in the example above. The class loader has been extended in Java 9 to search the so-called _module path_ whenver doing a class lookup. The module path contains a list of either directories containing modules or locations directly pointing to modules. 

In contrast to the good-old class path, the module path is used to detect modules and those modules export packages. Thats an important simplification. If based on the module declarations a type of a particular package can not be found, an error will be thrown by the JVM or Java compiler. 

It is now even the case with modules that the same package must not exist in more than one module in the module path. Every module name must be unique in the module path. 

When running the JVM, the module path is declared via the `--module-path` (or short `-p`) command-line parameter:

{% highlight shell %}
$ java --module-path ./mods -m com.ast.app/com.ast.app.Main
{% endhighlight %}

In the example above, the `mods` directory contains all the application-specific modules. The `-m` command-line parameter defines the entry point to the class implementing the `main` method. Before we can execute this command, we need to create the `mods` directory and use `javac` to compile our module:

{% highlight shell %}
$ mkdir ./mods
$ javac --module-source-path ./src -d ./mods $(find src -name '*.java')
$ tree

.
├── mods
│   └── com.ast.app
│       ├── com
│       │   └── ast
│       │       └── app
│       │           └── SomeClass.class
│       └── module-info.class
└── src
    └── com.ast.app
        ├── com
        │   └── ast
        │       └── app
        │           └── SomeClass.java
        └── module-info.java

10 directories, 4 files

{% endhighlight %}

If we use `javac` to compile our module like in the example above, the result is a so-called _exploded module_, an unpacked module. If we wanted to have a modular JAR file instead, we needed to run:

{% highlight shell %}
$ jar --create --file=mlib/com.ast.app@1.0.jar --module-version=1.0 -C mods/com.ast.app .
$ tree

.
├── mlib
│   └── com.ast.app@1.0.jar
├── mods
│   └── com.ast.app
│       ├── com
│       │   └── ast
│       │       └── app
│       │           └── SomeClass.class
│       └── module-info.class
└── src
    └── com.ast.app
        ├── com
        │   └── ast
        │       └── app
        │           └── SomeClass.java
        └── module-info.java

11 directories, 5 files

{% endhighlight %}

The `module-version` command-line parameter enriched the `module-info.class` with additional meta-data about the module's version. By the way, another interesting `jar` variant is 

{% highlight shell %}
jar --update --file some.jar --module-version 1.0
{% endhighlight %}

which updates a regular JAR file to beome a module JAR file with a module declaration. However, this is not the only migration path, we have a look at various options in the next section.

As mentioned above, besides modules JARs another format has been introduced called JMOD. JDK's JMODs are found in the JDK installation directory and are included on demand, based on the derived module tree. 

As you can see in the module declaration, we have a direct dependency to `java.logging`:

{% highlight java %}
module com.ast.app {
    requires java.logging;

    exports com.ast.app;
}
{% endhighlight %}

Maybe you noticed, we do not have any dependency on the Java core classes defined in JDK's `java.base` module. That is intentional as `java.base` is the only dependency every module has per default. `java.base` itself exports all the platform's core packages:

{% highlight java %}
module java.base {
    exports java.io;
    exports java.lang;
    exports java.lang.annotation;
    exports java.lang.invoke;
    exports java.lang.module;
    exports java.lang.ref;
    exports java.lang.reflect;
    exports java.math;
    exports java.net;
    ...
}
{% endhighlight %}

So as a matter of fact, our `com.ast.app` modules directly depends on `java.logging` and `java.base`.

In JSR-speak, our module `com.ast.app` _reads_ module `java.logging` (and implicitly `java.base`) and `java.logging` (and implicitly `java.base`) is readable by `com.ast.app` - that's the so-called _readability_ relationship. 

If `java.logging` would have a dependency on a module other than `java.base`, `com.ast.app` would not have a readability relation with this transitive dependency per default. 

Depending on the API, this might be a show blocker. Just imagine you have a public type in a module that returns a public type of another module. In our example, let's say our `SomeClass` would have a public method returning a `java.util.logging.Logger`.

{% highlight java %}
package com.ast.app;

public class SomeClass {
    
        // ...
    
        public java.util.logging.Logger getLogger() { ... }
        
        // ...
}

{% endhighlight %}

With our current module declaration, this would not work for modules reading our module. 

In order to overcome this issue, we can state such an _implied readability_ in the module declaration:

{% highlight java %}
module com.ast.app {
    requires transitive java.logging;

    exports com.ast.app;
}
{% endhighlight %}

Once we created the module again with implied readability for the `java.logging` module, we can inspect our `com.ast.app@1.0.jar` with the `jdeps` tool:

{% highlight shell %}
$ jdeps mlib/com.ast.app@1.0.jar                                                                                                          -- INSERT --
com.ast.app
 [file:///project/directory/mlib/com.ast.app@1.0.jar]
   requires mandated java.base (@9)
   requires transitive java.logging (@9)
com.ast.app -> java.base
   com.ast.app                                        -> java.lang                                          java.base
{% endhighlight %}

`jdeps` is a utility tool which can be used to inspect modules or search for packages within modules. For a more detailed look at its functionality, have a look at the output of `jdeps --help`.

### The Unnamed Module

Most of the Java code existing nowadays was of course written before the introduction of the module concept. As the old class path resolving functionality has not been dropped in Java 9, it is still possible to find classes via "the old way". However, starting with Java 9, every class has exactly one module. For Java code prior to Java 9, a special module has been introduced to satisfy this rule: the _unnamed module_.

The unnamed module has a readability relationship with every other module in order for the old code to archive compatibility. Also, the unnamed module exports all of its packages. If a package is found in the unnamed module and in an explicitly named module, the package from the unnamed module is ignored. Named modules, however, can not have a dependency on the special unnamed module, it only works in the other direction, from the unnamed to the named module.

This restriction might be obvious on first sight, but let's assume the following scenario: your MVC framework has migrated towards providing Java 9 modules only, however, your persistence framework is still around in plain-old JAR files, being handled by the unnamed module. In such a situation, your MVC framework could not reference the persistence framework anymore, because it can't have a dependency on the unnamed module. Due to that fact, another mechanism has been introduced: _automatic modules_.

### Automatic Modules

With the automatic modules mechanism, it is supported to place plain-old JAR files into the module-path. Those JAR files will automatically be detected as modules, the module name will be derived from the JAR file name based [on these rules](http://openjdk.java.net/projects/jigsaw/spec/sotms/#automatic-modules).

An automatic module is said to be an _implicit module_ because it naturally does not come with a module declaration. As it is impossible for the JVM to create the exports automatically, it will export all packages per default. As it's impossible to determine dependencies to other modules, the rule with automatic modules is that every automatic can read and is readable by every other automatic module. These rules should give a convenient way to allow for step-by-step migration of existing Java code.

### Reflection

Reflection creates another interesting special case for modules. Frameworks such as Spring and others rely heavily on reflection. Loading a class via the class loader and instantiating it is a very common operation. 

{% highlight java %}
Class clazz = Class.forName("com.ast.app.SomeClass", false, Thread.getContextClassLoader());

Object ob = clazz.newInstance();
{% endhighlight %}

For a framework to load and instantiate `com.ast.app.SomeClass` it would either be necessary to have a readability permission on the unnamed module, which is not possible, or to have a dependency on the named module where `com.ast.app.SomeClass` is defined. 

In order to still support these reflection-based scenarios, the reflection API has been revised in order to assume module readability for classes taking part in reflection. Those classes must still be `public`, however, there is no need to define an explicit readability in those cases. 

### Summary

One of the big additions to Java 9 is the new module abstraction which has deep impacts on nearly every part in the JDK/SE stack. Not only it is a big change to the JDK/SE, it's definitly a big change for framework/library providers but also application developers. The article will have a first look into Java 9 modules, we show how modules are declared, how they can be packaged and used. We will touch a couple of more complex topics too, but in general it is supposed to be seen as a first introduction to this topic. 

 


