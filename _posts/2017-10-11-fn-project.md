---
layout: post
title: Fn Project
categories:
- java
- functional
tags: []
status: publish
type: post
published: true
---

At JavaOne 2017 this year, Oracle announced and open-sourced its own serverless paltform called [Fn Project](http://fnproject.io). In this article, we will have a first look on Fn Project and what it offers to developers.

### What's That?

According to the [project homepage](http://fnproject.io), Fn is a _container native serverless platform_ for running functions. Maybe it is a bit too far fetched to call it a platform, however, it is a suite of tools, programs and libraries that allow for writing, testing and deploying functions on a FaaS (function-as-a-serivce} platform like AWS Lamda. You're not forced to do so, you can also run the functions locally or in any other public or private cloud (so you can run them basically everywhere).

Although coming from Oracle, Fn does not only support Java but basically every programming language according to the documentation. The `fn init` command however hows the following platforms/file extensions: `.js, .rb, .py, .java, .go, .php, .rs, .cs, .fs`. 

It's important to understand that the basic building block of Fn are containers. Every function defined is written and tested as a separate function and packaged in its own container. To be more precise, currently Docker is the primary technology for packaging and distribution (to Docker registries). However, according to Fn's github [FAQ](https://github.com/fnproject/fn/blob/master/docs/faq.md#which-languages-are-supported), the tool is build in a way so that it abstracts the actual container technology.

### The First Function

Let's have a look how functions can be defined. One part of Projec Fn is the command line tool (CLI). In order to install it locally you can run

{% highlight shell %}
curl -LSs https://raw.githubusercontent.com/fnproject/cli/master/install | sh
{% endhighlight %}

or you might instead want to download [the binary distribution](https://github.com/fnproject/cli/releases)

Once the script or binary is installed, you can run 

{% highlight shell %}
fn start
{% endhighlight %}

This will fire up a local server including an embedded database and message queue. Imagine this server is acting as a function container. It provides a function API so that deployed functions can be called and served via this server. The server is again a Docker container and it uses the image "fnproject/functions" for running the function container. Once started, you can access Fn functions under `localhost:8080`:

{% highlight shell %}
$ curl http://localhost:8080
{"goto":"https://github.com/fnproject/fn","hello":"world!"}
{% endhighlight %}

There is also a Docker image providing a graphical user interface for Fn, you can launch it with

{% highlight shell %}
docker run --rm -it --link functions:api -p 4000:4000 -e "API_URL=http://api:8080" fnproject/ui
{% endhighlight %}

It provides a dashboard and a link to the Swagger [Fn Function API documentation](http://petstore.swagger.io/?url=https://raw.githubusercontent.com/fnproject/fn/master/docs/swagger.yml).

Now we add our first function in Java and create the initial project structure with `fn init`:

{% highlight shell %}
$ ~/Development/projectfn » fn init --runtime java

        ______
       / ____/___
      / /_  / __ \
     / __/ / / / /
    /_/   /_/ /_/

Runtime: java
Function boilerplate generated.
func.yaml created.

{% endhighlight %}

We specify the runtime type directly here, we could have also created our function first to let Fn determine the runtime type automatically, however, for runtime Java it is better to specify the runtime directly.

The resulting project structure is

{% highlight shell %}
$ ~/Development/projectfn » tree
.
├── func.yaml
├── pom.xml
└── src
    ├── main
    │   └── java
    │       └── com
    │           └── example
    │               └── fn
    │                   └── HelloFunction.java
    └── test
        └── java
            └── com
                └── example
                    └── fn
                        └── HelloFunctionTest.java

11 directories, 4 files

{% endhighlight %}

Fn uses Maven in the background for packaging the function, so that's why the directory layout should look familiar for readers of this blog.

That's how the generated `HelloFunction.java` looks like:

{% highlight java %}

package com.example.fn;

public class HelloFunction {

    public String handleRequest(String input) {
        String name = (input == null || input.isEmpty()) ? "world"  : input;

        return "Hello, " + name + "!";
    }
}

{% endhighlight %}

When we run this function with `fn run`, we get the following output

{% highlight shell %}

$ ~/Development/projectfn » fn run
Building image projectfn:0.0.1
Sending build context to Docker daemon  13.82kB
Step 1/11 : FROM fnproject/fn-java-fdk-build:jdk9-latest as build-stage
 ---> f320c1ada1f8
Step 2/11 : WORKDIR /function
 ---> Using cache
 ---> 3a853726d2d9
Step 3/11 : ENV MAVEN_OPTS -Dhttp.proxyHost= -Dhttp.proxyPort= -Dhttps.proxyHost= -Dhttps.proxyPort= -Dhttp.nonProxyHosts= -Dmaven.repo.local=/usr/share/maven/ref/repository
 ---> Using cache
 ---> 45c3658e71cd
Step 4/11 : ADD pom.xml /function/pom.xml
 ---> Using cache
 ---> 3389274c6650
Step 5/11 : RUN mvn package dependency:copy-dependencies -DincludeScope=runtime -DskipTests=true -Dmdep.prependGroupId=true -DoutputDirectory=target --fail-never
 ---> Using cache
 ---> e9cdfef68cad
Step 6/11 : ADD src /function/src
 ---> Using cache
 ---> f7e306d5a79c
Step 7/11 : RUN mvn package
 ---> Using cache
 ---> 5f8ff54771e7
Step 8/11 : FROM fnproject/fn-java-fdk:jdk9-latest
 ---> d7674a04217c
Step 9/11 : WORKDIR /function
 ---> Using cache
 ---> 261da1bb0790
Step 10/11 : COPY --from=build-stage /function/target/*.jar /function/app/
 ---> Using cache
 ---> 65045dafbc2c
Step 11/11 : CMD com.example.fn.HelloFunction::handleRequest
 ---> Using cache
 ---> f9281f4a7626
Successfully built f9281f4a7626
Successfully tagged projectfn:0.0.1
Hello, world!
{% endhighlight %}

As you can see, it builds a Docker image from our function and directly runs it as a container, resulting in the `Hello, world!` output in the last line. What we just saw is the basic concept of Fn. Package a function into a container and whenever this function is requested, run the container to execute the function. As we said in the beginning, the container is really the basic building block.  

`func.yaml` contains the function configuration. You can see that `HelloFunction::handleRequest` is defined as an entry point, together with the version string and the runtime type:

{% highlight shell %}
$ ~/Development/projectfn » cat func.yaml
version: 0.0.1
runtime: java
cmd: com.example.fn.HelloFunction::handleRequest
{% endhighlight %}

There is more to configure than that, but for our first example this should be okay.

`fn run` just executed our function, however, that was only for testing the function output. Let us deploy the function to our function container running on `localhost:8080`. Deploying a function register the function in the function container and creates a so-called route to the newly registered function. After this route is created, the function container news about the function and it its execution can be requested via a REST call.

Before we can do that, we have to set the `FN_REGISTRY` environment variable. It is needed because Fn will push the created image to the Docker registry (as a default, to the Docker hub). If you didn't do a `docker login` before, you might want to do that before proceeding with the next commands:

{% highlight shell %}
# If you haven't, login at Docker hub
# $ docker login

$ ~/Development/projectfn » export FN_REGISTRY=<your docker_id>

$ ~/Development/projectfn » fn deploy --app projectfn

Deploying projectfn to app: my_function at path: /projectfn
Bumped to version 0.0.4
Building image appcaregmbh/projectfn:0.0.4
Sending build context to Docker daemon  84.99kB
Step 1/11 : FROM fnproject/fn-java-fdk-build:jdk9-latest as build-stage
 ---> f320c1ada1f8
Step 2/11 : WORKDIR /function
 ---> Using cache
 ---> 3a853726d2d9
Step 3/11 : ENV MAVEN_OPTS -Dhttp.proxyHost= -Dhttp.proxyPort= -Dhttps.proxyHost= -Dhttps.proxyPort= -Dhttp.nonProxyHosts= -Dmaven.repo.local=/usr/share/maven/ref/repository
 ---> Using cache
 ---> 45c3658e71cd
Step 4/11 : ADD pom.xml /function/pom.xml
 ---> Using cache
 ---> 3389274c6650
Step 5/11 : RUN mvn package dependency:copy-dependencies -DincludeScope=runtime -DskipTests=true -Dmdep.prependGroupId=true -DoutputDirectory=target --fail-never
 ---> Using cache
 ---> e9cdfef68cad
Step 6/11 : ADD src /function/src
 ---> Using cache
 ---> f7e306d5a79c
Step 7/11 : RUN mvn package
 ---> Using cache
 ---> 5f8ff54771e7
Step 8/11 : FROM fnproject/fn-java-fdk:jdk9-latest
 ---> d7674a04217c
Step 9/11 : WORKDIR /function
 ---> Using cache
 ---> 261da1bb0790
Step 10/11 : COPY --from=build-stage /function/target/*.jar /function/app/
 ---> Using cache
 ---> 65045dafbc2c
Step 11/11 : CMD com.example.fn.HelloFunction::handleRequest
 ---> Using cache
 ---> f9281f4a7626
...
0.0.4: digest: sha256:5a6a8f349db2d96205cbd376a13a81d4db624b0eba8daebd6e7dac930ed1c7c0 size: 1997
Updating route /projectfn using image appcaregmbh/projectfn:0.0.4...

{% endhighlight %}

When we deploy our function, we have to add the function to an application (here `projectfn`). In Fn, applications basically bundle a set of functions.

After our newly created application has been created and the function has been deployed, we can list our routes to see the actual REST URL for our function:

{% highlight shell %}
$ ~/Development/projectfn » fn routes list projectfn                                                             
path        image               endpoint
/projectfn  appcaregmbh/projectfn:0.0.7 localhost:8080/r/projectfn/projectfn
{% endhighlight %}

New we can call our function with `curl` or `fn call`

{% highlight shell %}
$ ~/Development/projectfn » curl localhost:8080/r/projectfn/projectfn
Hello, world!

$ ~/Development/projectfn » fn call projectfn /projectfn                                                         
Hello, world!
{% endhighlight %}

Maybe you saw that our example function `HelloFunction` also took an input argument. We can `echo` this to our `curl` call:

{% highlight shell %}
~/Development/projectfn » echo 'test' | fn call projectfn /projectfn
Hello, test!
{% endhighlight %}

As stated above, the function container comes with a function API so that REST calls can also be used to gather meta-data and execute operations about/on the available applications and functions. 

### Fn Java FDK

When using Fn together with Java, you can make use of the [Function Development Kit (FDK)](https://github.com/fnproject/fdk-java). As a matter of fact, we already made use of the FDK in our previous example. 

In our function `HelloFunction` we had a method `handleRequest(String)` which got a `String` argument. Normally, incoming data is coming from STDIN. With the FDK however, we got a nice argument without having to read with a buffered input, this functionality is called [simple data binding](https://github.com/fnproject/fdk-java/blob/master/docs/DataBinding.md). FDK does not only support `String` or `byte[]` input parameters, it supports binding JSON data types too, implemented using Jackson API.

Together with `HelloFunction` a template for JUnit tests had been generated too, `HelloFunctionTest`:

{% highlight java %}
package com.example.fn;

import com.fnproject.fn.testing.*;
import org.junit.*;

import static org.junit.Assert.*;

public class HelloFunctionTest {

    @Rule
    public final FnTestingRule testing = FnTestingRule.createDefault();

    @Test
    public void shouldReturnGreeting() {
        testing.givenEvent().enqueue();
        testing.thenRun(HelloFunction.class, "handleRequest");

        FnResult result = testing.getOnlyResult();
        assertEquals("Hello, world!", result.getBodyAsString());
    }

}
{% endhighlight %}

As you can see, it comes with a special `FnTestingRule` JUnit rule which allows for calling the function. `shouldReturnGreeeting` first enqueues an empty event (without a body) and then calls the `HelloFunction` acting on behalf of the enqueued event. 

To execute our function JUnit tests, you can run `fn build`. The underlying Maven build should run the function tests is part of the build.

### More to come

In this article we only saw the tip of the iceberg. For example, the Fn platform also comes with [database](https://github.com/fnproject/fn/blob/master/docs/operating/databases/README.md) and [message queues](https://github.com/fnproject/fn/blob/master/docs/operating/mqs/README.md) support, allowing for persistent storage and messaging between (asynchronous) function calls. 

There is another very interesting Fn feature: Flows. Flows allow you to orchestrate multiple function calls and implement asynchronous functions which will be consumed by a long-running server component (the completer) in the end. For an example of function orchestration, we will refer to the [async thumbnail generation example](https://github.com/fnproject/fdk-java/tree/master/examples/async-thumbnails) on GitHub.

As last example, we will show how to use this so-called completer together with Fn Flows to get an impression on about this topic.

Start our function container on port 8080 again:

{% highlight shell %}
$ ~/Development/projectfn » fn start
{% endhighlight %}

Start the completer container:

{% highlight shell %}
{% raw %}
$ ~ » DOCKER_LOCALHOST=$(docker inspect --type container -f '{{.NetworkSettings.Gateway}}' functions)
{% endraw %}

$ docker run --rm  \ 
       -p 8081:8081 \
       -d \
       -e API_URL="http://$DOCKER_LOCALHOST:8080/r" \
       -e no_proxy=$DOCKER_LOCALHOST \
       --name completer \
       fnproject/completer:latest
{% endhighlight %}

Do a `docker ps` to see if our two containers are running:

{% highlight shell %}
~ » docker ps
CONTAINER ID        IMAGE                        COMMAND                  CREATED              STATUS              PORTS                              NAMES
7126f034ccfa        fnproject/completer:latest   "/fnproject/completer"   38 seconds ago       Up 42 seconds       0.0.0.0:8081->8081/tcp             completer
7ac2c2fb71b6        fnproject/functions          "preentry.sh ./fun..."   About a minute ago   Up About a minute   2375/tcp, 0.0.0.0:8080->8080/tcp   functions
{% endhighlight %}

Create new function `primes`:

{% highlight shell %}
$ ~/Development/projectfn » mkdir primes
------------------------------------------------------------
$ ~/Development/projectfn » cd primes
------------------------------------------------------------
~/Development/projectfn/primes » fn init --runtime java appcaregmbh/primes
Creating function at: /appcaregmbh/primes

        ______
       / ____/___
      / /_  / __ \
     / __/ / / / /
    /_/   /_/ /_/

Runtime: java
Function boilerplate generated.
func.yaml created.
------------------------------------------------------------
{% endhighlight %}

Remove the generated Java files and create a new function called `PrimesFunction.java`:

{% highlight java %}
package com.example.fn;

import com.fnproject.fn.api.flow.Flow;
import com.fnproject.fn.api.flow.Flows;

public class PrimesFunction {

    public String handleRequest(int nth) {

        Flow fl = Flows.currentFlow();

        return fl.supply(
                () -> {
                    int num = 1, count = 0, i = 0;

                    while (count < nth) {
                        num = num + 1;
                        for (i = 2; i <= num; i++) {
                            if (num % i == 0) {
                                break;
                            }
                        }
                        if (i == num) {
                            count = count + 1;
                        }
                    }
                    return num;
                })

                .thenApply(i -> "The " + nth + "th prime number is " + i)
                .get();
    }
}
{% endhighlight %}

Now change the `func.yaml` function configuration to:

{% highlight yaml %}
version: 0.0.1
runtime: java
cmd: com.example.fn.PrimesFunction::handleRequest
path: /primes
{% endhighlight %}

and create a new Fn app and run the function:

{% highlight shell %}
$ ~/Development/projectfn/primes/appcaregmbh/primes » fn apps create primes-examplefn apps create primes-example
Successfully created app:  primes-example

$ ~/Development/projectfn/primes/appcaregmbh/primes » fn deploy --app primes-example
{% endhighlight %}

Now we have to configure our Flow function to talk to the completer we started just before:

{% highlight shell %}
{% raw %}
$ ~/Development/projectfn/primes/appcaregmbh/primes »  DOCKER_LOCALHOST=$(docker inspect --type container -f '{{.NetworkSettings.Gateway}}' functions)
{% endraw %}
 
$ ~/Development/projectfn/primes/appcaregmbh/primes »  fn apps config set primes-example COMPLETER_BASE_URL "http://$DOCKER_LOCALHOST:8081"
{% endhighlight %}

Once we have set the `COMPLETER_BASE` configuration property, we have set up the Flow correctly. We can now run our function `/primes`:

{% highlight shell %}
$ ~/Development/projectfn/primes/appcaregmbh/primes » echo 42 | fn call primes-example /primes
The 42th prime number is 181
{% endhighlight %}

Flows provide a meta-level on top of the functions and there are many more concepts and asynchronous programming patterns supported by Fn. If you are interested you can start with the [Flow User Guide](https://github.com/fnproject/fdk-java/blob/master/docs/FnFlowsUserGuide.md) for more information.

### Summary

The Fn Project has been announced and open-sourced by Oracle at JavaOne 2017 this year. Fn adds support for creating, testing, running and deploying functions in serverless architectures or in public or private clouds (so basically everywhere), utilising FaaS, functions-as-a-service. This article has a first look on Fn and how to use its Java support together with FDK to write serverless functions.