---
layout: post
title: Selectively run specs or suites in Jasmine
date: 2013-11-06 22:38
comments: true
categories: javascript test tools
---

I'm sure you write tests for your code. There is no excuse for that, even for Javascript - language which used to be treated as a toy one some time ago. Now Javascript ecosystem changed and its toolbox is full of development tools also those for testing purposes.

One of the most popular testing tools for Javascript is [Jasmine](http://pivotal.github.io/jasmine). It can be run in browser via HTML runner or as a part of automated workflow (which I highly recommend to build for every serious JS app). Sometimes when working on some particular feature, or fixing some stuff you may not want to have all your specs in all suites run every time you touch source file. Suppose you have tons of tests failing (e.g. due to upgrade of some core library) with a lot of errors but you want to focus on just few of them and fix them one by one. It'd be great if there was an ability to let Jasmine know to run only specs you wanted. So how to to it? Looks like there is not much about it in Jasmine documentation. The only thing mentioned is you can disable given suite by changing `describe` to `xdescribe` or changing `it` to `xit` for single spec. But it'd be silly to disable hundreds of tests that way just to be able to run few others, right?

It turns out that there is much easier way to do that. Let's look at the code below:

``` javascript running single spec
describe('Your awesome feature' function() {

  it('should do something', function() {
    // stuff here
  });

  // only this spec will be run
  iit('should not do something other', function() {
    // stuff here
  });

});
```

In the code above changing regular `it` in second spec definition to `iit` disables all other tests (in all suites that are about to run) except this one. To run whole suite with several specs in it just change `describe` to `ddescribe` and only this suite's specs will be run. There is no need to add `iit` in every single spec inside. See the code below:

``` javascript running single suite
// only specs in this suite will be run
ddescribe('Your awesome feature' function() {

  it('should do something', function() {
    // stuff here
  });

  it('should not do something other', function() {
    // stuff here
  });

});
```

You can mark several specs/suites that way and all of them will be run exclusively. This behavior although not well documented is Jasmine feature and is independent on runner you use to run your tests. I personally use [Karma](http://karma-runner.github.io) and it works like a charm.
