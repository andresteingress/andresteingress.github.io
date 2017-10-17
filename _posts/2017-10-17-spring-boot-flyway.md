---
layout: post
title: Spring Boot - Flyway
categories:
- java
- spring
- springboot
tags: []
status: publish
type: post
published: true
---

This article is about Spring Boot's [Flyway](https://flywaydb.org/) integration. Flyay is a tool for automating DB schema migration. It is open-source and favours simplicity and convention over configuration principles. 

Flyway solves the issue of keeping track and executing database schema migrations. If you are using Hibernate, you might have heard or you might be using the `hibernate.hbm2ddl.auto` setting with `update`. This setting will automatically export updates to the database when the Hibernate `SessionFactory` is created. Relying on this setting is not favourable for production environments though as it might suffer from inflexibility in certain use-cases. Consider you need to run an SQL statement for pre-filling a newly created table. That's not something that can be done with Hibernate schema migrations. 

Flyway is simple and flexible enough to provide us ways how to handle more advanced db migrations. On top of that, with the introduction of such a schema migration tool you automatically pre-define a way and format how db migration steps need to be specified and possibly committed into your version control system of choice. 

Compared to [Liquibase](http://www.liquibase.org/) we prefer Flyway as it offers a more "down-to-SQL" approach which feels super simple, instead of providing a meta-data DSL for specifying the migration steps. 

### Adding Flyway

Adding Flyway to Spring Boot is just a matter of adding another dependency in your `build.gradle` file:

{% highlight groovy %}
dependencies {
    // ...
    
    // Flyway support
    compile 'org.flywaydb:flyway-core:4.2.0'
}
{% endhighlight %}

The most current version of Flyway is 4.2.0. Once added in the `build.gradle` file, Spring Boot will automatically detect the Flyway classes on its classpath and will auto configure it. In Spring Boot, there is a class called `FlywayAutoConfiguration` in package `org.springframework.boot.autoconfigure.flyway` being part of the `spring-boot-autoconfigure` dependency/JAR file which does this job. 

For certain Flyway commands you will also need to add [Flyway Gradle support](https://flywaydb.org/documentation/gradle/) to your `build.gradle` or you need to install the [Flyway command-line client](https://flywaydb.org/documentation/commandline/).

Per default, the newly added dependency will auto configure Flyway with the following settings ([documented here](https://docs.spring.io/spring-boot/docs/1.5.8.RELEASE/reference/htmlsingle/#common-application-properties)):

{% highlight java %}
# FLYWAY (FlywayProperties)
flyway.baseline-description= #
flyway.baseline-version=1 # version to start migration
flyway.baseline-on-migrate= #
flyway.check-location=false # Check that migration scripts location exists.
flyway.clean-on-validation-error= #
flyway.enabled=true # Enable flyway.
flyway.encoding= #
flyway.ignore-failed-future-migration= #
flyway.init-sqls= # SQL statements to execute to initialize a connection immediately after obtaining it.
flyway.locations=classpath:db/migration # locations of migrations scripts
flyway.out-of-order= #
flyway.password= # JDBC password if you want Flyway to create its own DataSource
flyway.placeholder-prefix= #
flyway.placeholder-replacement= #
flyway.placeholder-suffix= #
flyway.placeholders.*= #
flyway.schemas= # schemas to update
flyway.sql-migration-prefix=V #
flyway.sql-migration-separator= #
flyway.sql-migration-suffix=.sql #
flyway.table= #
flyway.url= # JDBC url of the database to migrate. If not set, the primary configured data source is used.
flyway.user= # Login user of the database to migrate.
flyway.validate-on-migrate= #
{% endhighlight %}

It is important to note that per default, Flyway will be executing migrations against the default data source. If you have more data sources configured, you need to add the `@FlywayDataSource` annotation as detailed in the [Spring Boot documentation](https://docs.spring.io/spring-boot/docs/current/reference/html/howto-database-initialization.html#howto-execute-flyway-database-migrations-on-startup). 

As you can see, the only two properties being preconfigured are `flyway.baseline-version` and `flyway.locations`. In Flyway the database migrations are specified in SQL files. The default location for those migration files is `src/main/resources/db/migration`. In this directory you can place files following this naming scheme: `V<version number>__<description>.sql`. The `version number` will basically be a positive number which will have to be incremented by the database migration script author with every database migration. `flyway.baseline-version` configures the initial version number to be `1`, so your first database migration file will start with `V2__<description>.sql`.

On startup of the Spring Boot application, Flyway will look into `db/migration` and executed all database migration files which have not yet been executed. In order to keep track of the executions, it will initially create a table called `schema_version` in the target database. 

This initial creation is called "baselining". It will create the `schema_version` table and will assume that the target database is in the most current state, it will not execute any database migration files around found in `db/migration`. However, baselining is something that we have to trigger, it is not done automatically.

As mentioned above, `baseline` is such a command that we have to do via the Gradle integration (and therefore a Gradle tasks) or the Flyway command-line client. In our case, we use the Gradle integration:

{% highlight groovy %}
plugins {
    id "org.flywaydb.flyway" version "4.2.0"
}
{% endhighlight %}


Besides activating the Flyway Gradle plugin, we also have to specify the database connection properties in our `build.gradle` file:

{% highlight groovy %}
flyway {
    url = 'jdbc:h2:mem:mydb'
    user = 'myUsr'
    password = 'mySecretPwd'
    schemas = ['schema1', 'schema2', 'schema3']
    placeholders = [
        'keyABC': 'valueXYZ',
        'otherplaceholder': 'value123'
    ]
}
{% endhighlight %}

More information on configuring Flyway with Gradle can be found [here](https://flywaydb.org/documentation/gradle/).

Once we have configured our Flyway integration, we can run `baseline` against our target database. With Gradle, this is done by executing the `flywayBaseline` task. After it was successfully executed, we will have a new `schema_version` table in our database that will actually hold only one row of type `BASELINE`. This is the starting point for database migrations. 

__Hint__: do not forget to switch your `hibernate.hbm2ddl.auto` setting to `none` instead of `update` or any other option which would cause the target database to be modified.

### Adding Migrations

Let's say we have a class `Car` and we want to add a version column for activating optimistic locking for this JPA entity. We would simply add another instance variable called `version`:

{% highlight java %}
import javax.persistence.Entity;
import javax.persistence.Version;

import lombok.Getter;
import lombok.Setter;

@Getter
@Setter
@Entity
public class Car { 

    // ...
    
    @Version
    private Integer version;

    // ...
}

{% endhighlight %}

However, as we have turned off Hibernate's schema migration with `hibernate.hbm2ddl.auto=none`, we need to add the `VERSION` column ourselves. First step is to create a new database migration file and place it in `db/migration`. As it is our first database migration and the baseline version starts with 1 (according to the default config values), we need to call it `V2__add_version_to_car.sql`. The `add_version_to_car` can be completely arbitrary text, important is the version number after `V`. Inside this SQL file, we can use arbitrary statements for our target database. Flyway supports a wide range of databases, a full list can be found [here](https://flywaydb.org/documentation/).

For the matter of simplicity, we use H2 in our example above. To add the `VERSION` column we can simply say in `V2__add_version_to_car.sql`:

{% highlight sql %}
ALTER TABLE CAR ADD COLUMN VERSION INT;
UPDATE CAR SET VERSION = 0;
{% endhighlight %}

The 2nd SQL statement is important. If we already have entries in our `CAR` table we need to make sure that the new column will get prefilled with 0. 

That's it for now. When we start our Spring Boot application again, Flyway will - based on the `schema_version` table - detect there is one database migration pending and will automatically executed it during startup. Appropriate logs will appear in the console or log file of choice. 

There is one important thing to mention: once a database migration has been executed, it can not be changed and executed again without manual changes on the `schema_version` table. When you have a look at the `schema_version` table, you will see that it contains a column for keeping track of the checksum of the executed statement. When the statement is changed, Flyway will complain that the checksum has changed and your application will not start up. Best practice is to revert the change in another database migration step. 

### More Configuration

So far, we were running with the default settings provided by Spring Boot's auto configuration mechanism. It is important to note though, that those configuration settings can of course be overridden by the application. For example, the `flyway.locations` configuration value supports a special variable `{vendor}` which can be used in the directory location for the specific database type. In our case, the database type is `H2` so we could rewrite `flyway.locations` to

{% highlight java %}
flyway.locations=classpath:db/migration/{vendor}/
{% endhighlight %}

and Flyway would look in `db/migration/h2/` for database migration scripts when running with H2. If you are using multiple Spring environments/profiles, you could of course set the `flyway.locations` configuration setting depending on the currently active profile.

### More Commands

In this article we used basically two Flyway commands: `migrate` and `baseline`. However, there are four more commands which might come in handy in certain situations (`clean`, `info`, `validate` and `repair`). For a more detailed look at them, please go through the [Flyway documentation](https://flywaydb.org/documentation/).