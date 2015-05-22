---
layout: post
title: This AngularJS modules/dependencies thing is a lie
date: 2015-05-21 21:52
comments: true
categories: angularjs javascript
---

Bold statement in the title above, huh? Just stay with me, I'll try to make it clear in a minute. This is short story about AngularJS modules that define components and dependencies between these modules. Let's write some code first:

``` javascript

// third-party modules
angular.module('thirdParty1', []).factory('hello', function () {
  return 'hello world';
});
angular.module('thirdParty2', []).factory('hello', function () {
  return 'hi there';
});

// "own" modules
angular.module('own1', ['thirdParty1']).controller('Own1Ctrl', function(hello) {
  this.greet = hello;
});
angular.module('own2', ['thirdParty2']).controller('Own2Ctrl', function(hello) {
  this.greet = hello;
});

angular.module('app', ['own1', 'own2']);
```

And some view for that:

{% raw %}
``` html
<body ng-app="app">
  <div ng-controller="Own1Ctrl as own1">
    OwnCtrl1: {{ own1.greet }}
  </div>
  <div ng-controller="Own2Ctrl as own2">
    OwnCtrl2: {{ own2.greet }}
  </div>
</body>
```
{% endraw %}

This above is your application: you own (control) modules `own1` and `own2` and obviously the `app` one. There are also two other modules you depend on `thirdParty1` and `thirdParty2`. It happens that both third party modules define factories with the same name (but they both do different things). Your modules depend on these third party ones: `own1` depends on `thirdParty1` and `own2` depends on `thirdParty2`. Finally, your `app` depends on both of your modules. Simple. Now the fun part and question to you: 

## What do you think will be displayed?

Come on, this can't be hard. `Own1Ctrl` is defined in `own1` module that depends on `thirdParty1` which in turn defines `hello` factory as returning "hello world". Same with `Own2Ctrl`, depends on `thirdParty2` with "hi there". So I'd bet like this:

```
OwnCtrl1: hello world
OwnCtrl2: hi there
```

If you are like me and the above looks pretty logical to you, you've just failed. The answer is

```
OwnCtrl1: hi there
OwnCtrl2: hi there
```

## WAT?

It turns out AngularJS doesn't really care where it takes things to inject from as long as the name matches. It doesn't really matter that `own1` depends on `thirdParty1` and wants `hello` from __exactly this module__ (as it, well... depends on it). Angular may as well give you the other `hello` - all in all names match so why not? There is more, your module may have completely no dependencies at all (literally `[]`) and you'll be just fine injecting `hello` to this module's components. Even more - application will run fine, but your tests may break because there is no `hello` defined. And the other way round: your tests may pass because in tests you selectively instantiate modules, but your production app may fail (because it'll have the other module too). How weird is this?

## What "module" means to me

I think of module as a box with tools inside. When I declare that I need given box (read: depend on it), it means I want a tool from __exactly this box__, not the other one that is around and happens to have similar tool inside (think: both have hammers inside but one has ball-pen hammer and the other one has framing hammer). What AngularJS does is: when you are about to do some construction work it takes all the tools from all the boxes you have around (doesn't matter if you declare that you depend on them) throws them into one giant box and simply gives you one of these hammers (doesn't matter which one, and doesn't matter you asked for specific one by depending on specific box).

We're bitten by that today when part of our application suddenly stopped working even though there were no changes in that part of codebase. Namely we had `$modal` service from one library and in the other part we included dependency to `$modal` from different library (with different syntax). These two parts were completely unrelated and had almost nothing in common but it managed to broke the app. I know, two `$modal`s is not the best thing, in fact we are removing one of them, but anyway it just made the issue to pop up.

## What now?

As far as I know there is no way to make it "right" (in my understanding, shown above). One thing you can do is to uniquely name your components, e.g. use `$com.example.modal` instead of just `$modal`. But it has one drawback, it forces you to use this so-called "extended" way of defining components (with explicit string dependencies array) as dots are not allowed in parameters' names. You can also use camelCase notation etc. Unfortunately it looks like most people care more about unique name of their __modules__ and forget about unique naming for __components__ (especially in libraries) maybe because they are not aware of this behavior. There is one more way suggested on [Stack Overflow](http://stackoverflow.com/questions/30374934/angularjs-module-dependencies-naming-clash/30376123#30376123) where I asked this question to double-check my understanding. This one uses `$injector` to cherry-pick correct components and create application-wide wrappers for them.


## Summary

In AngularJS there is no such thing as module you depend on. Even when you ask for a thing, you cannot be sure you'll get the one you expect just because you declare dependency on module that has exactly this thing. This is all one giant toolbox. 

I thought I understood what's up with this modules system, why they invented their own one, but now I'm not sure. It helps in testing - fine, but my tests may pass while the app doesn't work because I get different dependencies in both cases! To me depending on module means I want tools from __exactly__ this module, not from the other one I'm even not interested in and I'm really surprised it's not the case here. If it happens in code you fully control, it's ok - you can rename stuff, but what if  In case you want to play with this example and see it live, here it is on [JSBin](http://jsbin.com/vapuye/3/edit?html,js,output).




