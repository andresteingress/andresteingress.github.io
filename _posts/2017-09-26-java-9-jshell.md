---
layout: post
title: Java 9 - JShell
categories:
- java
- java9
- jshell
tags: []
status: publish
type: post
published: true
---

As I have been developing in Groovy for quite some time (and still enjoy it whenever I get a chance to use it), I always was quite a fan of using the [Groovy Console](http://www.groovy-lang.org/groovyconsole.html) for testing small code snippets and doing API experiments.

With Java 9, there finally comes a nearly equivalent command line tool (it's command-line, not a Swing GUI) bundled with the JDK: [jshell](http://openjdk.java.net/jeps/222). 

### JShell Command

The `jshell` command can be found alongside the `java`, `javac` commands and the other usual suspects in the `bin` directory of your JDK installation. On my mac, it can be found in `/Library/Java/JavaVirtualMachines/jdk-9.jdk/Contents/Home`. Note the JDK 9 directory name does not contain the obligatory "1." prefix we had for all the pevious Java versions (which was actually already announced for Java 5).

Once you find the `jshell` command or if you've added the directory in your operating system's environment path, you can simply run `jshell` in the command-line. 

JShell implements a so-called "read-eval-print" loop (REPL). Nowadays, a vast majority of modern programming languages come with REPLs. So it has only been a matter of time since Java gets an offical REPL. 

A REPL basically provides a command-line/shell interface where certain expressions and statements can be specified in a particular programming language. Once you press `ENTER` the current line gets read and evaluated and results are usually stored in REPL-local variables. It's possible to explicitly define local variables too though. 

In JShell, according to [JEP 222](http://openjdk.java.net/jeps/222), those code pieces are referred to as _snippets_. A snippet may be an expression, a statement, class/interface declaration, field/method implementation or and import declaration. Notice that a snippet might be a full class declaration, so truth be told it's actually possible to have multiple lines of code belonging to a single snippet, snippets must not be one-liners.

### The Java REPL

Let's have a look at an example. With `jshell` started, you can do stuff like that:

{% highlight shell %}

jshell> int a = 42
a ==> 42

jshell> a + 3
$2 ==> 45

{% endhighlight %}

As you can see above, `jshell` allows either explicit or implicit variable declarations. Implicitly generated variables have the prefix `$X` where `X` is the so-called snippet identifier. 

Variables are not wrapped in a user-visible class, although internally synthetic code is generated. Variales can be seen as global inside the current `jshell` session. Internally, variables, methods and classes declared in JShell are compiled to static members of a synthetic (generated) class. Expressions and statements are compiled to static methods, again in a synthetic class.

As for creating collections inside JShell, it's convenient to know and use the new static factory methods in the JDK 9 collection classes, this is, `java.util.List.of` or `java.util.Map.of`:

{% highlight shell %}

jshell> List<String> items = List.of("a", "b", "c")
items ==> [a, b, c]

jshell> Map<Integer, String> enties = Map.of(1, "a", 2, "b", 3, "c")
entries ==> {3=c, 2=b, 1=a}

{% endhighlight %}

Of course another convenient features in this context is the Java 8 lamda syntax which allows use to declare code-blocks in a very compact syntax:

{% highlight shell %}

jshell> List<String> items = List.of("a", "b", "c")
items ==> [a, b, c]

jshell> items.stream().anyMatch(elem -> "a".equals(elem))
$15 ==> true

{% endhighlight %}

As you might have noticed, it's unnecessary to execute an import statement for the referenced collection classes. `jshell` imports a default package list. The imported packages can be shown with the `/imports` command, being one of the special `/*` commands (besides `/exit` btw ;-)):

{% highlight shell %}
jshell> /imports
|    import java.io.*
|    import java.math.*
|    import java.net.*
|    import java.nio.file.*
|    import java.util.*
|    import java.util.concurrent.*
|    import java.util.function.*
|    import java.util.prefs.*
|    import java.util.regex.*
|    import java.util.stream.*
{% endhighlight %}

`/help` shows additional commands specific to the JShell command line tool. 

{% highlight shell %}
jshell> /help
|  Type a Java language expression, statement, or declaration.
|  Or type one of the following commands:
|  /list [<name or id>|-all|-start]
|   list the source you have typed
|  /edit <name or id>
|   edit a source entry referenced by name or id
|  /drop <name or id>
|   delete a source entry referenced by name or id
|  /save [-all|-history|-start] <file>
|   Save snippet source to a file.
|  /open <file>
|   open a file as source input
|  /vars [<name or id>|-all|-start]
|   list the declared variables and their values
|  /methods [<name or id>|-all|-start]
|   list the declared methods and their signatures
|  /types [<name or id>|-all|-start]
|   list the declared types
|  /imports
|   list the imported items
|  /exit
|   exit jshell
|  /env [-class-path <path>] [-module-path <path>] [-add-modules <modules>] ...
|   view or change the evaluation context
|  /reset [-class-path <path>] [-module-path <path>] [-add-modules <modules>]...
|   reset jshell
|  /reload [-restore] [-quiet] [-class-path <path>] [-module-path <path>]...
|   reset and replay relevant history -- current or previous (-restore)
|  /history
|   history of what you have typed
|  /help [<command>|<subject>]
|   get information about jshell
|  /set editor|start|feedback|mode|prompt|truncation|format ...
|   set jshell configuration information
|  /? [<command>|<subject>]
|   get information about jshell
|  /!
|   re-run last snippet
|  /<id>
|   re-run snippet by id
|  /-<n>
|   re-run n-th previous snippet
|
|  For more information type '/help' followed by the name of a
|  command or a subject.
|  For example '/help /list' or '/help intro'.
|
|  Subjects:
|
|  intro
|   an introduction to the jshell tool
|  shortcuts
|   a description of keystrokes for snippet and command completion,
|   information access, and automatic code generation
|  context
|   the evaluation context options for /env /reload and /reset
{% endhighlight %}


There are commands for listing the created snippets, variables, methods, types etc. It's even possible to modify the source of existing snippets used previously with `/edit X` (X is the snipped identifier) and store those snippets in distinct files via `/store X <file>`. Re-running snippets is done via `/X`. I guess there is some more cool stuff to discover there.

### JShell API

Not only is JShell a command-line tool. It even comes with an [API](http://download.java.net/java/jdk9/docs/api/jdk.jshell-summary.html) for integrating JShell into JVM-based applications. 

Module `jdk.shell` comes with four packages whereas the first package is interesting for integration requirements: 

* `jdk.shell`: contains an API for programtically creating and evaluating snippets
* `jdk.shell.spi`: contains service provider interfaces for custom execution engine implementations. The default implementation for `jshell` is found in the `jdk.shell.execution` package, being the third package
* `jdk.shell.tool`: this package has the `JavaShellToolBuilder` for launching the shell programtically

Let's have a brief look at the `jdk.shell` package. 

The central class there is `JShell`. It is the central container for snippets together with the execution state those snippets have produced. Every source code  snippet is represented as an instance of [Snippet](http://download.java.net/java/jdk9/docs/api/jdk/jshell/Snippet.html) at runtime. 

[SnippetEvent](http://download.java.net/java/jdk9/docs/api/jdk/jshell/SnippetEvent.html) instances are fired once new snippets are added/removed or existing snippets have been updated. It gets particularly interesting when snippets belong to each other and one of the snippets gets dropped. JShell might end up with an unresolved reference to some entity (for example a variable), its the event listeners to task to react in such a scenario (or to end up with an unresolved reference in such a scenario).

### Summary

Finally, starting with Java 9, JDK comes bundled with a Java REPL (read-eval-print loop) called `jshell`. In this article we give a first overview of JShell and JShell API.