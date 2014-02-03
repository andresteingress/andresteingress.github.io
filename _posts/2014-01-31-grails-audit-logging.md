---
layout: post
title: Grails Audit Logging
categories:
- grails
- groovy
tags: []
status: publish
type: post
published: true
---
One of my projects needed a way to perform audit logging for Grails Hibernate domain classes. The audit log should contain at least the time and the property name and values that have been updated/inserted for a specific domain class.

I came across the [Grails Audit-Loggin Plugin](http://grails.org/plugin/audit-logging) but unfortunately there were various reasons why the plugin wasn't a good fit for us:

- configuration of the included fields was needed (instead of the excluded ones)
- usage of Hibernate stateless sessions for writing the AuditLog domain objects (better performance)
- some things seemed to be broken after writing the first integration tests, e.g. keeping the correct version number in the audit log
- more object-oriented structure to enable encapsulated unit tests and easier writing of integration tests
- last but not least: we wanted full control over the ongoing development of the plugin because the audit logging will be an important functionality of the application

In a nutshell: we started a new plugin that was initially based on the Grails Audit-Plugin to get some core components (like the `AuditLogListener`) without starting completely from scratch.

### Grails Hibernate Audit Log Plugin

That's how we named the new plugin. Right now the audit log mechanism registers itself only by the `hibernateDatastore`, no other datastores are supported. Besides completely changing the interal structure of the plugin, we threw some features over board we hadn't any use for. We thought it would be better to keep it simple instead of preparing for complex scenarios we didn't see any use for in our application. 

As the time of writing, the plugin supports the following configuration variables:

```groovy
// All Hibernate audit log configuration variables
auditLog.disabled = false        // globally disable audit logging

// Defines how to retrieve the actor name - the name of the principal that will be persisted
auditLog.sessionAttribute = ""   // the session attribute under which the actor name is found
auditLog.actorKey = ""           // the request attribute key under which the actor name is found
auditLog.actorClosure = null     // a closure for getting the current actor name

auditLog.defaultInclude = []     // optional list of globally included properties
auditLog.defaultIgnore = []      // optional list of globally excluded properties

// audit log persistence settings
auditLog.tablename = null        // custom AuditLog table name, defaults to "audit_log"
auditLog.truncateLength = null   // maximum length for property values after String conversion
```

The `auditLog.disabled` switch is global one to turn off audit logging for the entire application. If it is `false` an additional step is needed to enable audit logging for a particular domain class:

```groovy
class Person {

  static auditable = true

  String name
  String surName

  static constraints = {
  }
}
```

The domain class needs to have a static property `auditable = true`. Alternatively, the static property might be referring to a map to provide local configuration attributes:

```groovy
class Person {

  static auditable = [include: ['name']]

  String name
  String surName

  static constraints = {
  }
}
```

As a default setting all domain class properties are enabled for audit logging as long as `include` (or `defaultInclude` in the `Config.groovy`) or `ignore` (or `defaultIgnore` in `Config.groovy`) aren't specified in the `auditable` map. In the case of `include` only the specified properties are logged, in the case of `exclude` all other persistent properties are logged.

### At runtime

Once the audit log is enabled for a domain class all insert/update/delete operations will be kept track of by the plugin in the `audit_log` table. Actually the table comes as GORM domain class `AuditLogEvent` that is supplied by the plugin. For every property values that changes a new `AuditLogEvent` is created. Here are the domain class properties:

```groovy
class AuditLogEvent {

  Date dateCreated

  String actor
  String uri
  String className
  String persistedObjectId

  String eventName // one of ['INSERT', 'UDPATE', 'DELETE']
  String propertyName
  String oldValue
  String newValue
}
```

All properties should be self speaking except for the `actor` and the `uri` properties (I guess). The `actor` property has been introduced to keep track of the principal (name) that triggered the event. In the configuration options above the `actorClosure` was shown that can be specified to retrieve the principal name from the HTTP request or session. The `uri` contains the web URI that was used to initially trigger the domain class change.

Here is a testcase method that shows how the `AuditLogEvent` will be filled when a new `Person` object is saved to the DB:

```groovy
@Test
void testInsertEvent() {
  def p = new Person(name: "Andre", surName: "Steingress").save(flush: true)

  def auditLogEvent = AuditLogEvent.findByClassName('Person')
  assert auditLogEvent != null

  assert auditLogEvent.actor == 'system'
  assert auditLogEvent.className == 'Person'
  assert auditLogEvent.dateCreated != null
  assert auditLogEvent.eventName == AuditLogEventRepository.EVENT_NAME_INSERT
  assert auditLogEvent.newValue == 'Andre'
  assert auditLogEvent.oldValue == null
  assert auditLogEvent.persistedObjectId == p.id as String
  assert auditLogEvent.propertyName == 'name'
}
```

As the `AuditLogEvent` class is a pure Grails domain class its dynamic methods can be used for querying the audit log table.

Be aware that the `oldValue` (on upates) and the `newValue` will be stored for every property (as per configuration) of the auditable domain class. One way to truncate these strings is by using the `auditLog.truncateLength` option. In the most current version all values are persisted as type `String` by using the default Groovy type coercion to `String`. This will definitely change in future versions as needed.

### Installation

The plugin is available at [Github](https://github.com/andresteingress/grails-hibernate-auditlog). A plugin zip can be created with `grails package-plugin` from the root project directory. Right now the plugin is not stable enough to be published via the Grails plugin portal but as time goes by a stable version will definitely be published in the portal.

### Conclusion

The Grails _Hibernate Audit Log Plugin_ can be used for audit logging changes in Grails domain classes. Every domain class under target needs to specify a static `auditable` property that is either `true` or contains a `Map` of local settings. The plugin is currently only accessible [via Github](https://github.com/andresteingress/grails-hibernate-auditlog) but a stable version is planned to be released at the Grails plugin portal.


