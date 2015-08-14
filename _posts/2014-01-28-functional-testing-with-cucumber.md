---
layout: post
title: Grails Tests with Cucumber
categories:
- grails
- groovy
- testing
- cucumber
tags: []
status: publish
type: post
published: true
---
Recently one of my customers decided to introduce functional tests. The tests should be run beside the already existing unit and integration tests, utilizing a behaviour-driven development approach (BDD). 
[Behaviour-driven development](http://en.wikipedia.org/wiki/Behavior-driven_development) is an approach to specification based testing, instead of source code based testing. [Cucumber](http://cukes.info) enables writing BDD specifications for the Ruby programming language, but there are various implementations for other environments. One of them is [Cucumber-JVM](https://github.com/cucumber/cucumber-jvm), a port of Cucumber for the Java virtual machine.

Cucumber components can be devided into three categories: 
_features_, _steps_ and _hooks_. 

Features contain the human-readable specification. A specification consists of a description and various scenarios that need to be successfully executed. A scenario has at least three parts:

- **Given** 
a textual description that describes the assumption under which the specification must hold
- **When** 
a textual description of the action that causes some behaviour
- **Then** 
a textual description of the consequence that was caused by the action

Of course, the term "textual description" is not quite true as there needs to be some sort of common agreement about the syntax/semantic of feature files. For writing feature files, Cucumber uses [Gherkin](http://cukes.info/gherkin.html), a small programming language with a convenient to read syntax. Let's have an example:

<pre><code class="language-groovy">
Feature: log in to admin application
  Administrative functions need to be guarded with a login page

  Scenario: someone opens a protected link
    Given a user opens the browser
    When she opens the "admin/customer/list" link
    Then she will get a login page
</code></pre>

The example feature description above shows the definition of a single section. Gherkin would allow multiple scenarios and slightly complex scenario definitions with multiple given/when/then blocks. As can be seen above, the feature is human-readable and definitly not as complex as actual source code.

In my case, the customers application is a Grails application. There is a [Cucumber Grails plugin](http://grails.org/plugin/cucumber) available that we chose to add to our `BuildConfig.groovy` as additional plugin dependency.

> Be aware that Groovy 2.1.x ships with Groovy 1.8.x. The current Cucumber Grails plugin (0.10.0) has a dependency on the Spock Groovy 2.0 build that needs to be excluded manually in `BuildConfig.groovy` in case you're running Grails <= 2.1.x.

Once the Grails plugin is added to `BuildConfig.groovy`, Cucumber specifications can be executed by running `test-app`. Let's assume we saved the specification above in a file called `AdminLogin.feature`. In order to let the Cucumber plugin find the feature specification in the classpath, it needs to be put in the `test/functional` folder. Afterwards we can tell Grails to run all functional Cucumber tests with the following command:

</code></pre>bash
grails test-app functional:cucumber
</code></pre>

Be aware that this restricts `test-app` to execute only _functional Cucumber_ tests. All functional tests can be executed without the selector via 

</code></pre>bash
grails test-app :functional
</code></pre>

The result of running `test-app` looks similar to this:

</code></pre>bash
| Running 1 cucumber test...

1 Scenarios (
1 undefined
)
3 Steps (
3 undefined
)
0m
0.237s


You can implement missing steps with the snippets below:

Given(~'^a user opens the browser$') { ->
    // Express the Regexp above with the code you wish you had
    throw new PendingException()
}
When(~'^she opens the "([^"]*)" link$') { String arg1 ->
    // Express the Regexp above with the code you wish you had
    throw new PendingException()
}
Then(~'^she will get a login page$') { ->
    // Express the Regexp above with the code you wish you had
    throw new PendingException()
}
| Completed 1 cucumber test, 1 failed in 605ms
| Tests FAILED
</code></pre>

Ups, we wrote a specification but there is an important piece missing: the so-called _steps_ and _hooks_. Steps are needed to programmatically interact with our application. Cucumber leaves the decision about the tool we use for the steps up to us. As the project is about a web application, we needed some sort of web testing library to implement the steps. We chose to go with [Geb](http://gebish.org), a Groovy library to automate web testing. 

The Grails plugin output shown above already gives us a hint on how the step implementations should look like. We can copy these step definition stubs as a basis for our step implementation. The steps are defined in a simple Groovy file, it's recommended to use a separate package to avoid loading issues by the Cucumber Grails plugin. The file is placed in the `test/functional/steps` folder (because we are using the `steps` package in the Groovy file) of the application:

<pre><code class="language-groovy">
package steps

import pages.admin.LoginPage
import static cucumber.api.groovy.EN.*

Given(~'^a user opens the browser$') {->

}
When(~'^she opens the "([^"]*)" link$') { String link ->
    
}
Then(~'^she will get a login page$') {->
    
}
</code></pre>

`Given`, `When` and `Then` are actually static method calls to the Cucumber-JVM Groovy API. Every method is called with two arguments: a regular expression pattern (the bitwise negator is used to generate a `java.util.regex.Pattern` from the `String`) and a closure. The closure code comes with as many parameters as the regular expression amount of groups. Because we used double quotes for the link in the feature file we triggered Cucumber to guess this was a valid argument to our closure block. Of course the regular expression can be customized if needed, Cucumber only provides a starting point with the given stub code.

Now it's time for [Geb](http://gebish.org) entering the stage. In order to add a Geb `Browser` instance to our step code, we need to add a hook to our code. Cucumber hooks are needed to execute code before and after the execution of a scenario. In our case we want the code to be global, as we use Geb in every function Cucumber test case and need to ensure it is setup correctly. We place the Groovy file containing the hooks in the `test/functional/support` folder, as the Groovy code is using the `support` package:

<pre><code class="language-groovy">
package support

import geb.Browser
import geb.binding.BindingUpdater

import static cucumber.api.groovy.Hooks.*

Before () {
    bindingUpdater = new BindingUpdater(binding, new Browser())
    bindingUpdater.initialize()
}

After () {
    bindingUpdater.remove ()
}
</code></pre>

The `BindingUpdater` ensures that every step has access to the `Browser` instance via a `browser` variable. In addition, it provides a set of methods that are automatically forwarded to this `Browser` instance. Examples include, `get`, `at`, `via` and other important methods of the [Browser](http://www.gebish.org/manual/current/browser.html#the_browser). Let's see how our steps can finally be implemented by utilizing Geb:

<pre><code class="language-groovy">
package steps

import static cucumber.api.groovy.EN.*
import pages.admin.LoginPage

Given(~'^a user opens the browser$') {->

}
When(~'^she opens the "([^"]*)" link$') { String secureLink ->
    go secureLink
}
Then(~'^she will get a login page$') {->
    at LoginPage
}
</code></pre>

As you can see, instead of writing `browser.go link` we can simply use `go link` because the `BindingUpdater` set up method forwarding for this method. After the given link is opened with `go link` a Geb page is used to validate whether the login page is now shown in front of the secured page:

<pre><code class="language-groovy">
package pages.admin

import geb.Page

/**
 * Representation of the admin login page.
 */
class LoginPage extends Page {

    static url = "admin/login"

    static at = {
        (title =~ "Login") as Boolean
    }

    static content = {
        txtLogin    { $("input[name=j_username]") }
        txtPassword { $("input[name=j_password]") }

        btnLogin { $("input.submit_login") }
    }

    def login(String name, String password)  {
        txtLogin = name
        txtPassword = password

        btnLogin.click()
    }
}
</code></pre>

Geb uses the `at` verifier to determine whether the login page is shown or not. 

We're done with our first Cucumber specification!

### Update

I created a [blank Grails 2.1.5 project](https://github.com/andresteingress/grails-cucumber-example) and added the example above including an initial Geb and Cucumber configuration that should help getting things up and running.

### Conclusion

The Grails Cucumber plugin adds Cucumber-JVM support to Grails applications. Together with Geb, Cucumber can be used to create BDD-style specifications that in turn are executed at run-time using the Groovy web testing library underneath.


