---
layout: post
title: Query by Example in Spring Data
categories:
- java
- jsf
tags: []
status: publish
type: post
published: true
---
We are currently using Spring Data JPA in a project and really love it. Lately we came across a feature we sort of oversaw so far in Spring Data's excellent documentation and we definitely wanted to sure with you. 

Before we start, we first have to create some context. We use Spring Data in a Spring Boot based JSF application. We often have the requirement to show data in a Data-Table (for that we use [PrimeFaces](http://primefaces.org/)) based on the field values of a search form. In our code we follow the approach to create an empty domain object in the view class acting as our backing object that provides us with the search parameters. 

Here is an example. This is our view class:

{% highlight java %}
@Component
@ViewScope
public class AnimalSearchView {

    @Autowired
    @Getter
    private AnimalSearchForm form;
    
    @Autowired
    private AnimalRepository animalRepository;
    
    @Getter
    private AinmalDataModel dataModel;
    
    @PostConstruct
    public void init() {
        dataModel = new AinmalDataModel(animalRepository, form);
    }
    
    public void search() {
        info("Search executed sucessfully!");
    }
}
{% endhighlight %}

As you might notice, the `@ViewScope` is a custom component that has been developed on our side. It basically works the same as `@ViewScoped`, only difference is it was implemented based on Spring. The `AnimalRepository` is a Spring Data JPA repository and `AnimalDataModel` is a PrimeFaces [LazyDataModel](https://www.primefaces.org/showcase/ui/data/datatable/lazy.xhtml).

The relation between the `AnimalDataModel` and the `AnimalRepository` is especially important to notice. The `AnimalRepository` being the mediator between the model and the data mapping layer, that is, it creates and executes the DB queries. The `AinmalDataModel` has to use the `AnimalRepository` for making paged queries. Let's look at the `AnimalDataModel` implementation (teaser: it's not complete yet):
 
{% highlight java %}
import java.util.List;
import java.util.Map;

import org.primefaces.model.LazyDataModel;
import org.primefaces.model.SortOrder;

import sample.model.Animal;
import sample.repository.AnimalRepository;

public class AnimalDataModel extends LazyDataModel<Animal> {

    private AnimalRepository animalRepository;
    private AnimalSearchForm form;
    
    private List<Animal> dataSet;
    
    public AnimalDataModel(AnimalRepository animalRepository, AnimalSearchForm form) {
        this.animalRepository = animalRepository;
        this.form = form;
    }
    
    @Override
    public Integer getRowKey(Animal animal) {
        return animal.getId();
    }
    
    @Override
    public Animal getRowData(String rowKey) {
        return animalRepository.findOne(Integer.parseInt(rowKey));
    }
    
    @Override
    public List<Animal> load(int first, int pageSize, String sortField, SortOrder sortOrder, Map<String,Object> filters) {
        // magic happens here
    }
}
{% endhighlight %}

It's not necessary for this article to be familar with PrimeFaces `LazyDataModel`. It is basically an abstraction allowing for lazy-loaded data table paging. Usually, the paging logic is found in the `load` method. 

In the `load` implementation we need to retrieve the page of animals to be displayed and the total number of elements which is necessary in the view to show the correct paging indicators in our GUI. PrimeFaces hands over the offset (`first` parameter) and the `pageSize` (representing whatever is set in the XHTML/JSP for the `p:dataTable`).

So what we would need to do is to make in fact two database queries. One query for the data of the current page, and another COUNT query resulting in the total number of elements found (the overall number, independent on the current page).

Spring Data has a nice abstraction that is perfect for this use-case: the [Page](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/domain/Page.html) class.

A `Page` represents a sub-set of the total result _and_ it has a `getTotalElements()` method returning the number of total elements. What's even better, the `Page` class is directly supported in Spring Data repositories. That means you can use `Page` as return type in your self-defined query methods. [PagingAndSortingRepository](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/repository/PagingAndSortingRepository.html) comes with a `findAll` variant returning a `Page` instance.

Let's see how we could implement our `load` method with that knowledge:

{% highlight java %}
@Override
public List<Animal> load(int first, int pageSize, String sortField, SortOrder sortOrder, Map<String,Object> filters) {
    
    Page<Animal> animals = animalRepository.findBySpecies(form.getAnimal().getSpecies(), 
                                new PageRequest((first / pageSize), pageSize, sortOrder == SortOrder.ASCENDING ? Direction.ASC : Direction.DESC, sortField));
    
    this.dataSet = animals.getContent();
    setRowCount((int) animals.getTotalElements());
        
    return this.dataSet;
}
{% endhighlight %}

As you can see, the `Page` abstraction is perfect for this use-case. We get a sub-list of animals from our repository and can also access the total number of elements. 

*Hint:* The `PageRequest` gets the _page number_, not the 0-based offset, as first argument in the constructor. We can't simply pass the `first` argument given by PrimeFaces but have to calculate the current 0-based page number.

There is one down-side with the above approach. We need to create the query based on the data that we get from the `AnimalSearchForm` object. We cheated a bit in our example because we queried the `species` property only. However, when you have to build more complex queries because you need to incorporate more fields from the search form, we would need to dynamically construct a criteriaq query.

Before we go further, let us have a look at the `AnimalSearchForm` class:

{% highlight java %}
@Component
@TabScope
public class AnimalSearchForm implements Serializable {

    @Getter @Setter
    private Animal animal = new Animal();
}
{% endhighlight %}

The `AnimalSearchForm` has scope `@TabScope`, again a custom Spring-scope of our project. Besides that, the class only has a single property with an empty `Animal` instance. In the UI, the data from the search form is bound against that object:

{% highlight xml %}
<p:panelGrid columns="2" layout="grid">
    <p:outputLabel for="species" value="Species"/>
    <p:inputText id="species" value="#{animalSearchForm.animal.species}">
        <f:validateLength maximum="50" />
    </p:inputText>
    
    <p:commandButton value="Search" action="#{animalSearchView.search()}" icon="fa fa-search" update="@all"/>
</p:panelGrid>
{% endhighlight %}

Long story short, for such a use-case what we really want is to define a query based on domain object properties. We do not want to dig into complex dynamic criteria construction in our `load()` method implementation. But guess what, Spring Data JPA comes with a solution for that.

### Query by Example API

Query by Example (QBE) is a pattern targeting to query the database without ever even constructing a query. The query is enitrely generated based on a given _example_ instance. Let's say I wanted to query for `Animal` instances, I could provide a probe `Animal` instance that has some properties set. Those properties will be take into account as query parameters during the automatic query generation that is done by Spring Data in the background.

Besides the [Example](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/domain/Example.html), there is another abstraction that goes hand in hand with it: the [ExampleMatcher](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/domain/ExampleMatcher.html). The `ExampleMatcher` can be utilised to detail certain aspects of the query to be generated. 

If a Spring Data repository wants to support QBE, it needs to extend the [QueryByExampleExecutor](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/repository/query/QueryByExampleExecutor.html) interface. Once the repository descends from this interface, it can use common query methods like `findOne`, `findAll`, `count` etc. all taking an `Example` instance as an argument. 

With this knowledge we can rewrite our example from above to

{% highlight java %}
private AnimalRepository animalRepository;
private AnimalSearchForm form;

private List<Animal> dataSet;

private ExampleMatcher exampleMatcher;

public AnimalDataModel(AnimalRepository animalRepository, AnimalSearchForm form) {
    this.animalRepository = animalRepository;
    this.form = form;
    
    this.exampleMatcher = ExampleMatcher.matching()
            .withIgnoreNullValues()
            .withIgnoreCase()
            .withMatcher("species", match -> match.startsWith());            
}

@Override
public List<Animal> load(int first, int pageSize, String sortField, SortOrder sortOrder, Map<String,Object> filters) {
    
    Example<Animal> example = Example.of(form.getAnimal(), exampleMatcher);
        
    Page<Animal> animals = animalrepository.findAll(example, 
                new PageRequest((first / pageSize), pageSize, sortOrder == SortOrder.ASCENDING ? Direction.ASC : Direction.DESC, sortField));
    
    this.dataSet = animals.getContent();
    setRowCount((int) animals.getTotalElements());
    
    return this.dataSet;
}
{% endhighlight %}

We removed the part that directly accessed the `form.getAnimal().getSpecies()` property with the creation of the `Example` instance. With this instance, the `findAll()` method (from `QueryByExampleExecutor`) is called together with a `PageRequest` to find the approriate animals. As you see, when we add another field to our search form in the UI, there are nearly no changes necessary in the `load()` method. 

The only thing to consider is the `ExampleMatcher` instance that is used to create the `Example` instance. In our case, we wanted to ignore null properties from our `Animal` instance. We also wanted to support a case-insensitive search. All those query-related requirements need to be specified when constructing the `ExampleMatcher`, that's exactly its purpose. 

For the `species` property we can use a more advanced matcher configuration, all queries will automatically be appended with '%'. We chose the Java 8 lambda syntax for that, an alternative way is to use the methods defined in [ExampleMatcher.GenericPropertyMatchers](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/domain/ExampleMatcher.html) to do exactly that:

{% highlight java %}
public AnimalDataModel(AnimalRepository animalRepository, AnimalSearchForm form) {
    this.animalRepository = animalRepository;
    this.form = form;
    
    this.exampleMatcher = ExampleMatcher.matching()
            .withIgnoreNullValues()
            .withIgnoreCase()
            .withMatcher("species", ExampleMatcher.GenericPropertyMatchers.startsWith());            
}
{% endhighlight %}

### Summary

This article showed how to use the Query By Example (QBE) API in Spring Data. The QBE pattern can come in handy for certain use-cases that require to run queries with query parameters based on a given domain object. In this article we showed a real-world use-case based on PrimeFaces LazyDataModels.