---
layout: post
title: Gradle Daemon on Jenkins
categories:
- gradle
tags: []
status: publish
type: post
published: true
---

### Gradle Daemons on CI Servers

In one of my current projects we're migrating an existing monolith towards microservices. Of course our build is based on the excellent [Gradle build tool](https://gradle.org). As this implies separate Jenkins pipelines for all our services, we wanted to have the [Gradle daemon](https://docs.gradle.org/current/userguide/gradle_daemon.html) activated. 

The first fallacy I came across on various blog posts and StackOverflow answers is to turn off the Gradle daemon on CI servers like Jenkins. However, when you look up the most recent [Gradle Daemon documentation](https://docs.gradle.org/current/userguide/gradle_daemon.html) you will find the following paragraph:

> Since Gradle 3.0, we enable Daemon by default and recommend using it for both developers' machines and Continuous Integration servers.

As a matter of fact, as René Gröschke told us in a comment on our [latest podcast episode](https://dtr.fm/dtr164-kostenpflichtiges-java-und-gradle-plugins-entwickeln/#comments), it actually is emphazied to use the Gradle daemon _especially_ on CI servers. So how can this be done on Jenkins when using the Pipeline DSL Jenksfile?

### Gradle Daemon on Jenkins

As Jenkins is our CI server of choice, we needed to find a way to spawn the Gradle daemon from our pipeline builds without the Gradle daemon getting killed after the build was done. As it turned out there is separate section on [spawning processes from within the build](https://wiki.jenkins.io/display/JENKINS/Spawning+processes+from+build) in the Jenkins wiki.

In our pipeline DSL we had to set the following environment variable:

```groovy
env.JENKINS_NODE_COOKIE = 'dontKillMe' // this is necessary for the Gradle daemon to be kept alive
```

Jenkins uses the `BUILD_ID` or `JENKINS_NODE_COOKIE` environment variables to detect which child processes to kill after build completion. As we explicitly set the environment variable to something completely generic (`dontKillMe`), the child process won't be killed by Jenkins as the parent process can't link to it anymore. Be aware you need to set `JENKINS_NODE_COOKIE` in pipeline DSL scripts and **not** `BUILD_ID`.

After we did the changes described above we ran into an issue with the [incremental Gradle build](https://docs.gradle.org/current/userguide/more_about_tasks.html#sec:up_to_date_checks) not updating the generated JUnit test XML files. As Gradle will only regenerate JUnit XML files whenever code changes have been detected, it wouldn't generate new files on unmodified builds which causes the Jenkins JUnit plugin to fail, see [JENKINS-6268](https://issues.jenkins-ci.org/browse/JENKINS-6268).

We added the following `script` step to our Jenkinsfile in order to fix that annoyance:

```groovy
steps {
    sh './gradlew test'

    script {
        def testResults = findFiles(glob: 'build/test-results/**/*.xml')
        testResults.each { xml ->
            touch xml.path
        }
    }
}
```

And that's it. With these changes in place, our build now spawns Gradle daemons if necessary and takes full power of Gradle's incremental build and caching capabilities.

### Summary

Running the Gradle daemon on CI servers makes sense in the most cases as your build gains [performance improvements](https://guides.gradle.org/performance/) through it. In addition, incremental builds are especially interesting with microservice architectures where there is a need to build many different modules (or even the root project including it's submodules) quite often (e.g. on every commit).