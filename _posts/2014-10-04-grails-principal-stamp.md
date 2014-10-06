---
layout: post
title: Grails - Tracking Principals
categories:
- grails
- spring
- groovy
- java
tags: []
status: publish
type: post
published: true
---
We use the Grails [auto timestamp feature](http://grails.org/doc/latest/ref/Database%20Mapping/autoTimestamp.html) in nearly all of our domain classes. It basically allows the definition of two special domain class properties `dateCreated` and `lastUpdated` and automatically 
sets the creation and modification date whenever a domain object is inserted or updated. 

In addition to `dateCreated` and `lastUpdated` we wanted to have a way to define two additional properties *userCreated* and *userUpdated* to save the principal who created, updated or 
deleted a domain class (deletion because we have audit log tables that track all table changes, so when an entry is deleted and the principal is set before, we can see who deleted an 
entry).

### `PersistenceEventListener`

Grails provides the concept of GORM events, so we thought its implementation might be a good hint on how to implement our requirement for having `userCreated` and `userUpdated`. And indeed, 
we found `DomainEventListener`, a descendant class of `AbstractPersistenceEventListener`. It turns out that `DomainEventListener` is responsible for executing the GORM event hooks on domain 
object inserts, updates and deletes.

The event listener is registered at the application context as the `PersistenceListener` interface (which is implemented by `AbstractPersistenceListener`) extends from Spring's `ApplicationListener` 
and therefore actually uses the Spring event system.

In order to create a custom persistence listener, we just have to extend `AbstractPersistenceEventListener` and listen for the GORM events which are useful to us. Here is the implementation we ended 
up with:


```groovy
@Log4j
class PrincipalPersistenceListener extends AbstractPersistenceEventListener {

    public static final String PROPERTY_PRINCIPAL_UPDATED = 'userUpdated'
    public static final String PROPERTY_PRINCIPAL_CREATED = 'userCreated'

    SpringSecurityService springSecurityService

    PrincipalPersistenceListener(Datastore datastore) {
        super(datastore)
    }

    @Override
    protected void onPersistenceEvent(AbstractPersistenceEvent event) {

        def entityObject = event.entityObject

        if (hasPrincipalProperty(entityObject)) {
            switch (event.eventType) {
                case EventType.PreInsert:
                    setPrincipalProperties(entityObject, true)
                    break

                case EventType.Validation:
                    setPrincipalProperties(entityObject, entityObject.id == null)
                    break

                case EventType.PreUpdate:
                    setPrincipalProperties(entityObject, false)
                    break

                case EventType.PreDelete:
                    setPrincipalProperties(entityObject, false)
                    break
            }
        }
    }

    protected boolean hasPrincipalProperty(def entityObject) {
        return entityObject.metaClass.hasProperty(entityObject, PROPERTY_PRINCIPAL_UPDATED) || entityObject.metaClass.hasProperty(entityObject, PROPERTY_PRINCIPAL_CREATED)
    }

    protected void setPrincipalProperties(def entityObject, boolean insert)  {
        def currentUser = springSecurityService.currentUser

        if (currentUser instanceof User) {
            def propertyUpdated = entityObject.metaClass.getMetaProperty(PROPERTY_PRINCIPAL_UPDATED)
            if (propertyUpdated != null)  {
                propertyUpdated.setProperty(entityObject, currentUser.uuid)
            }

            if (insert)  {
                def propertyCreated = entityObject.metaClass.getMetaProperty(PROPERTY_PRINCIPAL_CREATED)
                if (propertyCreated != null)  {
                    propertyCreated.setProperty(entityObject, currentUser.uuid)
                }
            }
        }
    }

    @Override
    boolean supportsEventType(Class<? extends ApplicationEvent> eventType) {
        return eventType.isAssignableFrom(PreInsertEvent) ||
                eventType.isAssignableFrom(PreUpdateEvent) ||
                eventType.isAssignableFrom(PreDeleteEvent) ||
                eventType.isAssignableFrom(ValidationEvent)
    }
}
```

As you can see in the code above, the implementation intercepts the `PreInsert`, `PreUpdate` and `PreDelete` events. If any of these event types is thrown, the code checks the affected 
domain object for the existence of either the `userCreated` or `userUpdated` property. If available, it uses the `springSecurityService` to access the currently logged-in principal and 
uses its `uuid` property, as this is the unique identifier of our users in this application.

To register the `PrincipalPersistenceListener` and attach it to a Grails datastore, we need to add the following code to `BootStrap.groovy`:

```groovy
def ctx = grailsApplication.mainContext
ctx.eventTriggeringInterceptor.datastores.each { key, datastore ->

    def listener = new PrincipalPersistenceListener(datastore)
    listener.springSecurityService = springSecurityService

    ctx.addApplicationListener(listener)
}
```

To make this work, the `springSecurityService` needs to be injected, the same is true for `grailsApplication`. 

But that's all we have to do to support our new domain class properties `userCreated` and `userUpdated`. The last step is to add both properties to the domain class(es) we want to track.

### Conclusion

Grails integrates with Spring's event mechanism and provides the `AbstractPersistenceEventListener` base class to listen to certain GORM events. Grails uses this mechanism internally for example for 
the GORM event hooks but it can of course be used by the application logic too. This article showed how to introduce support for `userCreated` and `userUpdated` which are similar to `dateCreated` and 
`lastUpdated` but store the principal how created, updated or deleted a domain object.
