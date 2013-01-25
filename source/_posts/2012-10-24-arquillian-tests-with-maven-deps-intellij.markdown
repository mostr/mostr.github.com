---
layout: post
title: Running Arquillian tests with Maven dependencies in IntelliJ Idea
date: 2012-10-24 09:01
comments: true
categories: test arquillian intellij maven java javaee
---
Arquillian is a big thing. For me it is a revolution in in-container testing. In real world scenarios it is quite common that your JEE components use some libraries, frameworks or other utility classes provided in thirdparty archives. When building project with maven you can make use of `pom.xml` descriptor and avoid giant amounts of boilerplate code (adding classes and packages of thirdparty libs by hand). I won't dive into details how you can set it up, but the idea is to use `MavenDependencyResolver` and configure it to read your `pom.xml`.


``` java Using MavenDependencyResolver
    MavenDependencyResolver resolver = DependencyResolvers.use(MavenDependencyResolver.class);
    resolver.loadMetadataFromPom("pom.xml");
    ...
    // use in ShrinkWrap archive
```

This code above works well when run in Eclipse or in command-line Maven, but surprisingly may fail when run from IntelliJ Idea. It complains that `pom.xml` file cannot be found and read. Some time ago I switched my IDE from Eclipse to IntelliJ Idea and I'm still brand new if it comes to its cofiguration tips and tricks. Quick ShrinkWrap debugging session and it turnes out that IntelliJ invokes tests from **project's dir** instead of **module's dir**. Fortunately I've found a switch for that in IDE. 

### There I fixed it

Go to **Edit Configurations** (under **Run** menu), select **Defaults** branch and then **JUnit**. You can see that **Working directory** is set to project's directory. That's why `pom.xml` could not be found in this location. Fortunately it can be changed either to `MAVEN_REPOSITORY` or to `MODULE_DIR` and the latter is  exactly what you want. Changing that and saving it in defaults makes the issue above disappear. 

Now you can run your tests from IntelliJ even if you have maven dependencies to be included in your `@Deployment` archive.