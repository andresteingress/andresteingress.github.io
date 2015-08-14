---
layout: post
title: "Grails: Reconnecting JDBC Connections"
categories:
- grails
- groovy
- java
tags: []
status: publish
type: post
published: true
---
At our company we are utilising the well-known [Quartz library and Grails plugin](https://grails.org/plugin/quartz) for our batch jobs in Grails applications. When going throgh the server log files of our production server I lately came acress this error:

<pre><code class="language-groovy">
org.apache.tomcat.jdbc.pool.ConnectionPool abandon
WARNING: Connection has been abandoned PooledConnection[com.mysql.jdbc.JDBC4Connection@650599cb]:java.lang.Exception
        at org.apache.tomcat.jdbc.pool.ConnectionPool.getThreadDump(ConnectionPool.java:967)
        at org.apache.tomcat.jdbc.pool.ConnectionPool.borrowConnection(ConnectionPool.java:721)
        at org.apache.tomcat.jdbc.pool.ConnectionPool.borrowConnection(ConnectionPool.java:579)
        at org.apache.tomcat.jdbc.pool.ConnectionPool.getConnection(ConnectionPool.java:174)
        at org.apache.tomcat.jdbc.pool.DataSourceProxy.getConnection(DataSourceProxy.java:111)
        at org.springframework.jdbc.datasource.DataSourceUtils.doGetConnection(DataSourceUtils.java:111)
        at org.springframework.jdbc.datasource.TransactionAwareDataSourceProxy$TransactionAwareInvocationHandler.invoke(TransactionAwareDataSourceProxy.java:224)
        ...
        at grails.plugins.quartz.GrailsJobFactory$GrailsJob.execute(GrailsJobFactory.java:104)
        at org.quartz.Job$execute.call(Unknown Source)
        at grails.plugins.quartz.QuartzDisplayJob.execute(QuartzDisplayJob.groovy:29)
        at org.quartz.core.JobRunShell.run(JobRunShell.java:207)
        at org.quartz.simpl.SimpleThreadPool$WorkerThread.run(SimpleThreadPool.java:560)
</code></pre>

<p>
The beginning of the stack-trace showed that the exception was thrown by a Quartz worker thread - so how come the current DB connection was abandoned?
</p>

I investigated the connection pool settings and I found the following in the Tomcat connection pool configuration:


<pre>
removeAbandoned="true"
removeAbandonedTimeout="3600"
</pre>

The abandoned timeout was set to `3600` seconds aka as one hour. The batch job took over 1 hour, therefore the connection pool abandoned the connection which led to the error above.

### A Session in a Quartz Job

As long as the property `sessionRequired` is not explicitly set to `false` in a Grails Quartz job class, the Quartz plugin will create a Hibernate session that is bound to the Quartz worker thread. This is done by the `SessionBinderJobListener` that comes with the Quartz plugin that uses the current _persistence context interceptor_ of type `HibernatePersistenceContextInterceptor`, it is only enabled when Hibernate is in use as general data store.

For long-running quartz jobs the binding to a Hibernate session is bad, as this means that the Hibernate session binds a JDBC connection from the connection pool for the live-time of the session (starting once the session has to use the connection to the database).

When skipping through the documentation for `org.hibernate.Session` the `disconnect(Connection)` method appeared to me. As the documentation mentioned:

<abbr>
Disconnect the Session from the current JDBC connection. If the connection was obtained by Hibernate close it and return it to the connection pool; otherwise, return it to the application.

This is used by applications which supply JDBC connections to Hibernate and which require long-sessions (or long-conversations)
</abbr>

<br>
And it adds:

<abbr>
Note that disconnect() called on a session where the connection was retrieved by Hibernate through its configured org.hibernate.connection.ConnectionProvider has no effect, provided ConnectionReleaseMode.ON_CLOSE is not in effect.
</abbr>

<br>

<p>
So this means that the DB connection used by the session has to be given by the application-code in order to use disconnection. In our application, this was the case, so we could go into that direction to solve this issue.
</p>

### Refreshing a Session

So the goal was to rewrite the long-running batch jobs so that they disconnect and close the JDBC connection from time to time. This would release the connection and put it back to the pool, a connection abandonment as we have seen before is not possible anymore as long as the time span for the disconnect is of course smaller than an hour.

First thing we did in the Quartz job was to disable the automatic Hibernate session creation via the `sessionRequired` property:

<pre><code class="language-groovy">
class SomeJob {

  static triggers = {
      // ...
  }

  def concurrent = false
  def sessionRequired = false

  GrailsApplication grailsApplication
  SessionFactory sessionFactory
  DataSource dataSource

  // ...
}
</code></pre>

In order to get a connection from the DB pool we injected the `javax.sql.DataSource` and rewrote the code so that it would use the created Hibernate session (instead of the Grails provided ones):

<pre><code class="language-groovy">
def session = sessionFactory.openSession(dataSource.connection)
</code></pre>

All the HQL queries and Hibernate operations would be rewritten to be running over this particular session. 

Keeping the session open for the entire life-time of the job implies flushing and clearing the session from time to time to avoid running in to performance or out-of-memory issues. As suggested in the [Hibernate Documentation](https://docs.jboss.org/hibernate/orm/3.3/reference/en/html/batch.html), the session is flushed after a certain nubmer of entities processed:

<pre><code class="language-groovy">
protected void refreshJDBCConnection(Session session) {
    session.flush()
    session.clear()

    // ...
  }
</code></pre>

Exactly at this pace in the code, we added the code that would take the current connection, close it, and return it to the database connection pool:

<pre><code class="language-groovy">
protected void refreshJDBCConnection(Session session) {
    session.flush()
    session.clear()

    // we need to disconnect and get a new DB connection here
    def connection = session.disconnect()
    connection.close()
    
    session.reconnect(dataSource.connection)
  }
</code></pre>

The code above will get the JDBC connection out of the session and uses the `dataSource` to obtain a new database connection and reconnect it with the existing session. With this we can keep the abandoned connection setting and ensure that the job is not aborted after the configured aboandonment time.

### Conclusion

Connection pools allow to abandon database connections. After a certain time, the DB connection can be forced to be put back into the connection pool, no matter if it is still in use by the application or not. With Grails Quartz jobs there can be an issue with this feature in combination with long-running jobs that exceed the configured abandonment time. This article showed how to disconnect and reconnect a Hibernate session that can be kept open during the life-time of such long-running jobs.

