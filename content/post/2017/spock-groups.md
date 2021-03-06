---
title: "Spock tests grouping using @Category from junit"
Tags: ["groovy", "spock", "tdd"]
Categories: ["groovy"]
description: "Example on how to configure test groups using junit @Category annotation"
date: "2017-01-14"
---

When the project gets bigger and bigger there should be more and more tests. In the perfect world,
all tests should be executed really fast, but life is far from perfect and sometimes some tests are
slow.  When using Gradle + Spock combination we have few ways of deciding how to group tests. I'm
going to explore junit @Category in combination with Spock and grade.

<!--more-->

# Why

When project hits certain size there is a lot of tests. Usually, some tests are faster than others.
For example database tests will be slower than domain tests. When developing small feature
completely unrelated to database code we don’t really need to run DB tests on our machine CI tool
will do it for us. So to improve development experience we can categorize tests into groups.

When using Gradle + Spock combination it isn’t obvious how it should be configured. [Spock
documentation](http://spockframework.org/spock/docs/1.1-rc-3/index.html) has nothing on groups
configuration. Gradle documentation doesn’t have much either. There is a simple reason to it. Groups
configuration depends on the testing framework. It will be different for TestNG and JUnit.

# How

The good news is that Spock uses JUnit test runner to execute tests. There is nice feature in JUnit
- [@Category](http://junit.org/junit4/javadoc/4.12/org/junit/experimental/categories/Categories.html)
(it is experimental and it's [@Tag in
junit5](http://junit.org/junit5/docs/current/user-guide/#migrating-from-junit4-tips)). Now all we
need to do is [configure JUnit
runner](https://docs.gradle.org/current/userguide/java_plugin.html#test_grouping) in Gradle It’s all
sounds good but full class names for categories configuration hard coded in build.gradle is not
really useful but with a bit of Gradle magic we can do something nice with it.

The idea is to create some marker interfaces - test categories. Then write helper class (maybe it
should be a plugin, but I’m not that good with Gradle - see below) which will parse command line
parameters and decide which groups should be executed.

# Offtopic

Nice Gradle's feature I’ve discovered while preparing is [buildSrc
directory](https://docs.gradle.org/current/userguide/organizing_build_logic.html#sec:build_sources).
<blockquote>Gradle then automatically compiles and tests this code and puts it in the classpath of
your build script. You don't need to provide any further instruction. This can be a good place to
add your custom tasks and plugins.</blockquote> Which is awesome and I’m ashamed that I’ve
discovered it so late.

# Code

<small>All code is on [github](https://github.com/pchudzik/blog-example-spock-groups) and it
compiles ;)</small>

Let’s define test categories:

{{<highlight groovy>}}
package com.pchudzik.blog.spock.groups

@interface Fast{}
@interface Integration {}
@interface Slow {}
@interface Database {}
{{</highlight>}}

Now create some tests which simulate “big project”:

{{<highlight groovy>}}
package com.pchudzik.blog.spock

class NotAnnotatedTest extends Specification {
  def "should execute not categorized test"() {
    given: println "running not categorized test"
    expect: true
  }
}

@Category(Fast)
class FastTest extends Specification {
  def "fast test example"() {
    given: println "running fast test"
    expect: true
  }
}

@Category(Integration)
class IntegrationTest extends Specification {
  def "integration test example"() {
    given: println "running integration test"
    expect: true
  }
}

@Category(Slow)
class SlowTest extends Specification {
  def "slow test example"() {
    given: println "running slow test"
    expect: true
  }
}

@Category(Database)
abstract class DatabaseSpecification extends Specification {
  def setup() {
    println("database setup")
  }
}

class RepositoryTest extends DatabaseSpecification {
  def "inherited test"() {
    given: println "running inherited test"
    expect: true
  }
}
{{</highlight>}}

With those in place let's write helper class which will parse groups:

{{<highlight groovy>}}
class TestGroups {
  private static final groupDefinitions = [
      "fast" : "com.pchudzik.blog.spock.groups.Fast",
      "slow" : "com.pchudzik.blog.spock.groups.Slow",
      "db"   : "com.pchudzik.blog.spock.groups.Database",
      "it"   : "com.pchudzik.blog.spock.groups.Integration"
  ]

  private final Collection<String> groupsParam

  TestGroups(String groupsString) {
    groupsParam = (groupsString ?: "")
        .split(",")
        .toList()
        .findAll { !it.isAllWhitespace() }
  }

  String[] excludedGroups() {
    resolveGroups(excludes())
  }

  String[] includedGroups() {
    resolveGroups(includes())
  }

  private String[] resolveGroups(Collection<String> groups) {
    groups
        .collect { groupDefinitions[it] }
        .toArray(new String[groups.size()])
  }

  private Collection<String> includes() {
    groupsParam.findAll { !isExcluded(it) }
  }

  private Collection<String> excludes() {
    groupsParam
        .findAll { isExcluded(it) }
        .collect { it.replaceFirst("-", "") }
  }

  private boolean isExcluded(String group) {
    group.startsWith("-")
  }
}
{{</highlight>}}

Using buildSrc directory I can write
[tests](https://github.com/pchudzik/blog-example-spock-groups/blob/master/buildSrc/src/test/groovy/TestGroupsTest.groovy)
which will verify if my code works as expected, which is actually faster than running all tests from
“big project” ;)

The last step is to configure test groups in build.gradle:

{{<highlight groovy>}}
tasks.withType(Test) {
  def testGroups = new TestGroups(System.getProperty("groups"))

  useJUnit {
    includeCategories testGroups.includedGroups()
    excludeCategories testGroups.excludedGroups()
  }
}
{{</highlight>}}

When everything is ready it's time to play a little and see how it's working. Let's start with
something easy:

{{<highlight text>}}
./gradlew test

com.pchudzik.blog.spock.NotAnnotatedTest > should execute not categorized test STANDARD_OUT 
    running not categorized test
com.pchudzik.blog.spock.IntegrationTest > integration test example STANDARD_OUT
    running integration test
com.pchudzik.blog.spock.SlowTest > slow test example STANDARD_OUT
    running slow test
com.pchudzik.blog.spock.SlowTest > mixed test example STANDARD_OUT
    database test example
com.pchudzik.blog.spock.FastTest > fast test example STANDARD_OUT
    running fast test
com.pchudzik.blog.spock.database.OtherDatabaseReleatedTest > uncategorized test from db package STANDARD_OUT
    running uncategorized test from database package
com.pchudzik.blog.spock.database.RepositoryTest > inherited test STANDARD_OUT
    database setup
    running inherited test
com.pchudzik.blog.spock.database.IntegrationDbTest > integration test in database package STANDARD_OUT
    running integration test from database package
BUILD SUCCESSFUL
{{</highlight>}}

Ok. Now something more tricky:

{{<highlight text>}}
./gradlew test -Dgroups=-db --rerun-tasks

com.pchudzik.blog.spock.NotAnnotatedTest > should execute not categorized test STANDARD_OUT
    running not categorized test
com.pchudzik.blog.spock.IntegrationTest > integration test example STANDARD_OUT
    running integration test
com.pchudzik.blog.spock.SlowTest > slow test example STANDARD_OUT
    running slow test
com.pchudzik.blog.spock.FastTest > fast test example STANDARD_OUT
    running fast test
com.pchudzik.blog.spock.database.OtherDatabaseReleatedTest > uncategorized test from db package STANDARD_OUT
    running uncategorized test from database package
com.pchudzik.blog.spock.database.IntegrationDbTest > integration test in database package STANDARD_OUT
    running integration test from database package
BUILD SUCCESSFUL
{{</highlight>}}

Awesome, all DB tests are skipped.

What about something crazy:

{{<highlight text>}}
./gradlew test --tests com.pchudzik.blog.spock.database.* --rerun-tasks -Dgroups=-db

com.pchudzik.blog.spock.database.OtherDatabaseReleatedTest > uncategorized test from db package STANDARD_OUT
    running uncategorized test from database package
com.pchudzik.blog.spock.database.IntegrationDbTest > integration test in database package STANDARD_OUT
    running integration test from database package
BUILD SUCCESSFUL
{{</highlight>}}

And it works!

So to wrap it up. To execute tests only from specific group: -Dgroups=group1,group2<br/> To exclude
group from all tests: -Dgroups=-group1,-group2<br/> Even standard gradle [test
filtering](https://docs.gradle.org/current/userguide/java_plugin.html#test_filtering) mechanism will
work

# Summary

Using JUnit categories works exactly as expected ;). It's not yet perfect because it's impossible to
run only uncategorized tests (which is actually really easy with Spock RunnerConfiguration), but
with a bit of effort, it should be possible to exclude all groups and run only uncategorized tests.

It’s worth mentioning that it’s not the only way of grouping Spock tests in Gradle. You can use
[RunnerConfiguration](http://spockframework.org/spock/javadoc/1.0/spock/config/RunnerConfiguration.html)
to achieve exactly the same behavior (not sure if --tests will work). Examples on how to use
RunnerConfiguration for test grouping:
http://wordpress.transentia.com.au/wordpress/2014/04/23/using-spock-configuration-in-grails/ <br/>
http://mrhaki.rblogspot.com/2015/08/spocklight-including-or-excluding.html
