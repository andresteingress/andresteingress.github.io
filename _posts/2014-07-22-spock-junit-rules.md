---
layout: post
title: JUnit Rules and Spock
categories:
- groovy
- spock
- testing
tags: []
status: publish
type: post
published: true
---
I recently had to implement a file upload for a Grails application. In this application we have quite a bunch of JUnit tests but we do want to utilize Spock for all the newly added tests. 

As it is in the nature of file uploads, the Spock specification needs to create quite a few temporary files and folders.

One option I've seen quite a few times is to either use `File#createTemp` or use JDK 7's `Files#createTempDirectory/Files#createTempFile` methods. Both have the disadvantage of the intial setup and cleanup, which results in helper code that is added to the test and distracts from the real test code.

### JUnit Rules

JUnit provides a way to intercept the test suite and test method execution by providing the concept of rules. A rule implementation can intercept test method execution and alter the behaviour of these tests, or add cleanup work as it was done in `@Before`, `@After`, `@BeforeClass` and `@AfterClass` or Spock's `setup`, `cleanup`, `setupSpec` and `cleanupSpec` methods. For a particular test case, an instance of the rule implementation must be available via a public instance field and it must be annotated with `@Rule`.

For example, the `TestName` rule implementation can be used to have access to the test case name in a test method at runtime.

```java
public class NameRuleTest {
  @Rule
  public TestName name = new TestName();

  @Test
  public void testA() {
    assertEquals("testA", name.getMethodName());
  }

  @Test
  public void testB() {
    assertEquals("testB", name.getMethodName());
  }
}
```

### Spock and JUnit Rules

As it turns out, Spock has support for applying JUnit rules in specifications. This be done by providing a Groovy property with the type of the rule implementation, annotated by `@Rule`. This was really good news for my file upload tests as this allowed me to use one of my favorite JUnit rules: the `org.junit.rules.TemporaryFolder` rule.

As its name implies, the `TemporaryFolder` rule gives a convenient way to create temporary folders and files in test methods. The rule concept is used to intercept before and after each test method execution to do all the setup work.

This makes testing my `AttachmentService` very slick:

```groovy
@TestFor(AttachmentService)
@Mock([Attachment])
class AttachmentServiceSpec extends Specification {

    @Rule
    TemporaryFolder temporaryFolder = new TemporaryFolder()

    def "load the persistent file for a given attachment"() {
        setup:
          def tempFile = temporaryFolder.newFile('test.txt')
          def attachment = new Attachment(
                               uploadId: '123', 
                               originalFilename: 'test.txt', 
                               location: tempFile.toURL()).save()

        when: "an attachment with a URL reference is loaded"
          def file = service.loadFile(attachment)

        then: "the underyling File must be returned"
          file == tempFile
    }
}
```

As you can see in the code above, the `TemporaryFolder` can be used to create a new file with the `newFile` method. If we wanted to create a new folder, there is also a `newFolder` method available. We do not have to specify any temporary folder or do any cleanup work, this is all done by the rule implementation itself.

There is a good overview for the base rules provided in JUnit at [Github](https://github.com/junit-team/junit/wiki/Rules). 

### Conclusion

Spock comes with support for JUnit rules. A rule can intercept test method execution, do setup or cleanup work and might even change the test results. The `TemporaryFolder` rule is a useful rule that allows to create temporary files and folders in test cases while keeping track of these files and cleaning them up after the test execution.

