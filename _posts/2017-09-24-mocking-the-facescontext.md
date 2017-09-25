---
layout: post
title: Mocking JSF's FacesContext
categories:
- java
- jsf
tags: []
status: publish
type: post
published: true
---

When you are in a JSF application and you want to write tests for your view beans, you will very likely come into the situation of having to mock JSF's [FacesContext](http://docs.oracle.com/javaee/7/api/javax/faces/context/FacesContext.html). The `FacesContext` is a central class in JSF, providing methods for doing redirects, adding messages shown to the user and giving acceess to the "external context", which in turn provides access to HTTP related functionality.

In one of my projects we are using [Mockito](http://site.mockito.org/) as our mocking library of choice. If you do not know Mockito, a good introduction can be found on the [project homepage](http://site.mockito.org/). 

### Mocking FacesContext

Once you get to the point to mock `FacesContext`, you get into a dilemma. Mockito [does not allow to mock static methods](https://stackoverflow.com/questions/4482315/why-does-mockito-not-mock-static-methods). But instead of using an extension library like [PowerMock](https://github.com/powermock/powermock/wiki/Mockito) you can also go another route.

For JSF projects, it is nearly impossible to go without [OmniFaces](http://omnifaces.org/). OmniFaces is a libary offering various adaptions and extensions commonly needed in JSF projects. You can almost certainly throw OmniFaces on your JSF project's classpath, it will be needed at some point.

One of the classes provided by OmniFaces is `org.omnifaces.util.Faces` ([JavaDoc](http://omnifaces.org/docs/javadoc/2.6/org/omnifaces/util/Faces.html)). It is basically a collection of utility methods for conveniently accessing certain functionality from JSF's `FacesContext`. For example, you can do a redirect in your view bean like in the following example

{% highlight java %}
// Send a redirect with parameters UTF-8 encoded in query string.
Faces.redirect("product.xhtml?id=%d&name=%s", product.getId(), product.getName());
{% endhighlight %}

or get some value from the underlying HTTP session 

{% highlight java %}
Faces.getSessionAttribute("user");
{% endhighlight %}

I think you get the gist of it. 

Coming back to our mocking problem, one of the methods provided by `Faces` is the `setContext(FacesContext context)` method. This method was introduced to coveniently replace the current `FacesContext` instance with [some wrapper implementation](https://docs.oracle.com/javaee/7/api/javax/faces/context/FacesContextWrapper.html). 

But this means when we use OmniFaces's `Faces` class instead of the JSF `FacesContext` in our code base, it gets easy to mock those places in our unit tests.

### Example

Let's assume we have this method in one of our view classes

{% highlight java %}
public class MyView {

    public void doSomeRedirect() throws IOException {
        Faces.redirect("somewhere.xhtml");
    }
}
{% endhighlight %}

As we use OmniFaces's `Faces` class we can conveniently set the mocked `FacesContext` as current instance in our test-case:

{% highlight java %}
import org.mockito.ArgumentCaptor;
import org.mockito.Mock;
import org.mockito.Mockito;
import org.mockito.runners.MockitoJUnitRunner;
import org.omnifaces.util.Faces;

import sample.view.AutoBearbeitenView;

import static org.assertj.core.api.Assertions.assertThat;

import java.io.IOException;

import javax.faces.context.ExternalContext;
import javax.faces.context.FacesContext;
import javax.faces.context.Flash;
import javax.servlet.http.HttpServletRequest;

import org.junit.Before;
import org.junit.Test;
import org.junit.runner.RunWith;

@RunWith(MockitoJUnitRunner.class)
public class MyViewTests {

	@Mock FacesContext facesContextMock;
	@Mock ExternalContext externalContextMock;
	@Mock Flash flashMock;
	@Mock HttpServletRequest requestMock;
	
	@Before
	public void setup() {
		// do some wiring
		Faces.setContext(facesContextMock);
		
		Mockito.when(facesContextMock.getExternalContext()).thenReturn(externalContextMock);
		Mockito.when(externalContextMock.getFlash()).thenReturn(flashMock);
		Mockito.when(externalContextMock.getRequest()).thenReturn(requestMock);
	}
	
	@Test
	public void doSomeRedirect() throws IOException {
		
		Mockito.when(requestMock.getContextPath()).thenReturn("admin");
		
		ArgumentCaptor<String> redirectUrl = ArgumentCaptor.forClass(String.class);
		Mockito.doNothing().when(externalContextMock).redirect(redirectUrl.capture());
		
		AutoBearbeitenView view = new AutoBearbeitenView();
		view.doSomeRedirect();
		 
		assertThat(redirectUrl.getValue()).isEqualTo("admin/somewhere.xhtml");
	}
}
{% endhighlight %}

As you can see, in JUnit's `@Before` method we do some wiring to mock out all the relevant parts. Starting from `FacesContext` down to the actual `HttpServletRequest`. The test-case above uses some nice Mockito features like it's JUnit runner supporting the `@Mock` annotation to do automatic mocking.

Another useful element is Mockito's [ArgumentCaptor](https://static.javadoc.io/org.mockito/mockito-core/2.10.0/org/mockito/ArgumentCaptor.html). It is used to capture the redirect URL we are getting as an argument when `ExternalContext.redirect(String)` is called.

However, the key element is this line 

{% highlight java %}
Faces.setContext(facesContextMock);
{% endhighlight %}

If OmniFaces' `Faces` class is used throught your code base, only this line is needed to start going with your mocking library/approach of choice.

### Summary

In this article we showed how the usage of OmniFaces' `Faces` class can make mocking of JSF's `FacesContext` very easy. We use Mockito in the test examples to do the actual mocking work, however, the groundwork is laid by OmniFaces.