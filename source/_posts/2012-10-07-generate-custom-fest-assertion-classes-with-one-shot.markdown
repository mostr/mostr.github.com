---
layout: post
title: Generate custom FEST-assertion classes with one shot
date: 2012-10-07 21:13
comments: true
categories: java test tools
---

When writing tests there is a need to write assertions in order to find out whether test should pass or fail. Standard JUnit and TestNG assertions are ok, but they are not very readable, especially when creating more complex conditions. There are two most popular tools that allow write really easy to read assertions in our tests. Those are [Hamcrest](https://github.com/hamcrest) and [FEST Assert](https://github.com/alexruiz/fest-assert-2.x). They have slightly different style and I personally prefer the latter and if you haven't used it before, I advise you to grab it and give it a try. I bet you'll fall in love with it if only you take readablity of code you write seriously.
In short, the main strength of FEST assertions lies in its ability to deliver contextual assertion methods, so for example you can get assertions like `isEmpty`, `hasSize`, `includes` when doing assertion on Collection, and `contains`, `endsWith`, `matches` for String. Using FEST assertions we can write assertions like that:

``` java sample FEST assertions usage
    // String
    assertThat(employee.getName()).isEqualTo("John Doe");

    // Boolean
    assertThat(employee.isActive()).isTrue();

    // Collections
    assertThat(department.getEmployees()).containsOnly(manager);

```

But it can be done even better. Say hello to custom assertions:

``` java custom FEST assertions

assertThat(employee).hasName("John Doe");
assertThat(employee).isActive();

```

Whoaaa! Highly contextual assertions that you can read in almost plain english. The only drawback is that in order to get this you need to write custom assertion class for each of your classes. This is not a rocket science, just write class that extends `AbstractAssert`, provide assertion methods you want to have, and done. But what if you have several classes to create assertions for? No luck here - still need to write custom assertion class for each of them. Boring, huh? And we, developers are lazy beasts, right? Luckily

## There is a tool for that

Ruby community has a phrase: *"there is a gem for that"*. Same here - there is a tool for that. It's called [fest-assertion-generator](https://github.com/joel-costigliola/fest-assertion-generator) and can take most of this boring stuff from us. This is command-line tool that you can get as either maven artifact or windows/unix zip archive. It contains executable scripts that fire up generator plus `lib/` and `templates/` directories. In order to do the magic you need to put jar file containing classes you want to generate assertions for into `lib/` directory and fire up this tool with either fully qualified class names (separated by space) or package name as a parameter e.g :

``` text run fest-assertion-generator
    ./generate.sh pl.michalostruszka.app.Employee pl.michalostruszka.Department

    ./generate.sh pl.michalostruszka.app
```

When done, your custom assertion source files will be waiting for you in the **main directory** of generator. By default they are named like

``` text
	<YOUR_CLASS_NAME>Assert
```
Just grab them, put into test sources in your project and make use of them. There is also a `templates/` directory in case you need more control on how it generates those java files. It contains template files with tokens that are replaced with actual names during generation process. However in most cases templates provided should be sufficient as they generate assertions for every public getter your class contain. In addition `has elements` and `has no elements` assertions for arrays and collections are generated.

There are two gotchas however. First of all this tool loads every single class with `Class.forName` and it must be able to do that with no issues. If your classes use classes from other jars or in general classes that were not provided to this tool it will fail with ugly `java.lang.NoClassDefFoundError`. So remember to feed this tool with complete classes set. Second thing is that no matter how organized your packages tree is, all generated files will be put into the main directory, so you need to manually rearrange them before use.
The good news for Eclipse users is that there is a plan to wrap this tool in Eclipse plugin - the project is under active development now.

Personally I find this tool really helpful as it can do the boring stuff for you by generating assertions for classes with public getters like JPA entities or just simple POJOs. If you need specific assertion e.g. provide assertion on multiple fields and give it meaningful name, feel free to add new methods to those classes.
There are some things that could be improved in future versions, like way it stores generated sources and missing packages structure, but it is certainly "good-to-have" tool in developer's toolbox in my opinion.

**Caution**:
At the time of writing there is 1.0M5 version of this tool which contains bug that prevents you from running this tool. Fully qualified name of main class that fires the process is wrong and results with `Could not find or load main class`. My tiny pull request with fix for that was already submitted and should be merged soon. For now you can simply [clone my fork](https://github.com/mostr/fest-assertion-generator) which has this fix applied, build it with maven and you are good to go.
