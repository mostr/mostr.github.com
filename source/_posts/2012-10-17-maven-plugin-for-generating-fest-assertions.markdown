---
layout: post
title: Even easier FEST assertion generation? Here comes maven plugin for that
date: 2012-10-17 09:18
comments: true
categories: java test tools maven plugin
---

Two posts back, [here](http://localhost:4000/blog/2012/10/07/generate-custom-fest-assertion-classes-with-one-shot/) I've described very neat tool for getting your tests (specifically assertions) even more readable. Unfortunately original version this tool is a command-line script and can't be easily linked with build/development process itself.  

As I found this tool very useful, I decided to try to give it even more power and spend some time to wrap it in maven plugin. That would allow to link it with regular maven project's lifecycle and have assertions generated on the fly during build process and ready to use immediately. This plugin is now available in Maven Central repo so you can grab it using

``` xml 
    <groupId>org.easytesting</groupId>
    <artifactId>maven-fest-assertion-generator-plugin</artifactId>
    <version>1.0</version>
```
    
You can also find its sources [here](https://github.com/joel-costigliola/maven-fest-assertion-generator-plugin) on github. The repo is hosted on Joel Costigliola's (the author of original generator tool) account in order to have all generator-related stuff in one place, easier to maintain and release. 

It's worth saying that there are two important improvements (I also mentioned in previous post) if compared to original previous version of this tool. First of all it generates assertion files in correct package directories structure, so they are ready to use immediately with no tweaks required. The second is that source classes can contain references to other classes from any maven dependency defined in project and they will still have assertions genearated properly. This plugin takes care of loading classes from maven dependencies as well as classes from the project itself. It means you can also generate assertions for classes from your maven dependencies, if required.

###Want to use it?

First of all it requires FEST assertions in 2.x version, so add the following to your `pom.xml`:

``` xml 
    <dependency>
        <groupId>org.easytesting</groupId>
        <artifactId>fest-assert-core</artifactId>
        <version>2.0M8</version>
        <scope>test</scope>
    </dependency>
```

Then bind and configure the plugin itself in the `<build>` section:

``` xml 
    <plugin>
        <groupId>org.easytesting</groupId>
        <artifactId>maven-fest-assertion-generator-plugin</artifactId>
        <version>1.0</version>
        <executions>
            <execution>
                <goals>
                    <goal>generate-assertions</goal>
                </goals>
            </execution>
        </executions>
        <configuration>
            <packages>
                <param>your.first.package</param>
                <param>your.second.package</param>
                ...
            </packages>
        </configuration>
    </plugin>
```

The plugin can be launched with command `mvn generate-test-sources` (or simply `mvn test` as it will be fired before tests are executed) or with any IDE that supports maven. By default, it generates the assertions source files in `target/generated-test-sources/assertions` as per maven convention (but this can be changed by defining `targetDir` configuration element).

This is basic version of this plugin. In future versions defining own templates and assertions generation for single classes will probably be supported. 

If you spot any issue, feel free to report it or even better - submit a pull request with fix or improvement.

Have a good assertions!