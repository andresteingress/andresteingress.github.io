---
layout: post
title: Grails Tests with Cucumber Cont'd
categories:
- grails
- groovy
- geb
- cucumber
tags: []
status: publish
type: post
published: true
---
[My previous blog post](http://blog.andresteingress.com/2014/01/28/functional-testing-with-cucumber/) has been about introducing Cucumber in a Grails application. This post is a follow up on a particular requirement that came up in the course of writing the first Cucumber specifications.

One point of criticism was the lack of auto-complete support for [Geb](http://gebish.org) methods and variables which are available in our Cucumber step implementations. [As mentioned](http://blog.andresteingress.com/2014/01/28/functional-testing-with-cucumber/), the Geb [BindingUpdater](http://www.gebish.org/manual/0.7.0/api/geb-core/geb/binding/BindingUpdater.html) adds a `browser` variable and multiple methods to be used in the Cucumber step implementation code. 
For example, calling `browser.go link` can be shortened to:

<pre><code class="language-groovy">
package steps

import pages.admin.LoginPage
import static cucumber.api.groovy.EN.*

Given(~'^a user opens the browser$') {->

}
When(~'^she opens the "([^"]*)" link$') { String link ->
   go link       // instead of browser.go link    
}
Then(~'^she will get a login page$') {->
    
}
</code></pre>

As the project team is entirely using [IntelliJ](http://www.jetbrains.com/idea/), I decided to write a [GDSL](http://confluence.jetbrains.com/display/GRVY/Scripting+IDE+for+DSL+awareness) file. GDSLs can be used to add auto-completion support to Groovy DSLs in IntelliJ - exactly what we needed.

Thanks to Peter Gromov from JetBrains I could come up with a GDSL that even has support for generic method type parameters:

<pre><code class="language-groovy">
/**
 * Enables code completion for Geb inside Cucumber step implementations.
 */
def forwardedMethods = [
  "go", "to", "via", "at", "waitFor",
  "withAlert", "withNoAlert", "withConfirm", "withNoConfirm",
  "download", "downloadStream", "downloadText", "downloadBytes", "downloadContent", "report", "reportGroup", "cleanReportGroupDir"]

def scriptContext = context(
  filetypes: ['.groovy'], 
  pathRegexp: ".*/test/cucumber/.*", 
  scope: closureScope())

contributor(scriptContext) {
    property name: 'browser', type: 'geb.Browser'

    def methods = findClass('geb.Browser').methods
    methods.findAll { it.name in forwardedMethods }.each { def browserMethod ->

        def params = [:]
        browserMethod.parameterList.parameters.each { param ->
            params.put(
              param.name, 
              com.intellij.psi.util.TypeConversionUtil.erasure(param.type).canonicalText)
        }

        method name: browserMethod.name, type: com.intellij.psi.util.TypeConversionUtil.erasure(browserMethod.returnType).canonicalText, params: params
    }
}
</code></pre>

The GDSL registers all methods which are bound by the `BindingUpdater` plus it adds the `browser` variable of type `geb.Browser` for all Groovy script files in the `test/cucumber/` directory. `TypeConversionUtil` is used to type erase the method result and parameter types, so for example `<T extends Page> T at(Class<T> pageClass)` will be erased to `Page at(Class pageClass)`. One little attack against the DRY principle is the hard-coded list of forwarded methods, that's something that could be improved with a little IntelliJ PSI kung-fu.

After throwing this as `CucumberGebSupport.gdsl` in the `test/cucumber` folder (which is being registered as test-source folder), IntelliJ will recognize the file and enable the auto-completion support as specified.

![IntelliJ Auto-Completion for Geb in Cucumber](/assets/auto-complete-geb.png)

### Conclusion

IntelliJ comes with a neat mechanism that allows to add auto-completion support for custom Groovy DSLs. So-called GDSL files can be used to specify delegation classes, introduce variables and do other fancy stuff that needs to be done to have auto-completion for Groovy DSLs.



