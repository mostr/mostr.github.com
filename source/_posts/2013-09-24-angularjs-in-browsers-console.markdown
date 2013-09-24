---
layout: post
title: "Access and manipulate AngularJS $scope from browser's console"
date: 2013-09-24 12:28
comments: true
categories: angularjs javascript
---

This one's gonna be quick. Have you ever wanted to trigger some action in your AngularJS application from browser's console? I mean, broadcast event (just for development/debug purposes), call functions on `$scope` or do other stuff? Let's see what we can do with it.

Every AngularJS application is bound to exactly one DOM element e.g.

``` html
<!DOCTYPE html>
<html lang="en">
<head>
  // regular stuff here
</head>
<body ng-app="myApplication">
  // AngularJS stuff here
</body>
</html>
```

It also means that every single DOM element within `body` (in example above) is bound to some instance of `Scope`. And that's all we need - scope we can call functions on, change properties, access `$rootScope`, broadcast events etc. So let's grab this scope to play with it. AngularJS adds some extras to existing jQuery (if loaded) or provides its own jQLite with those extras built-in. One of those extra features is ability to get reference to scope bound to given DOM element. Try it:

``` javascript
// in browser's console
$('body').scope();

>> Scope {$id: "002", $$childTail: Child, $$childHead: Scope, $$prevSibling: null, …}
```

Whoaa! Looks like we got scope reference. From now on you can do whatever you need with this scope. Obviously you can grab any DOM element (within AngularJS application), target it with jQuery and get its scope to play with. If you don't have full blown jQuery loaded you can still do tricks as above but with using `angular.element` instead

``` javascript
var sidebar = document.getElementsById('sidebar');
var scope = angular.element(sidebar).scope();

>> Scope {$id: "00J", $$childTail: null, $$childHead: null, $$prevSibling: null, …}
```

One thing you need to remember about is if you want to fire an action on `$scope` from outside of AngularJS "world", you need to wrap your stuff in `$scope.$apply` to let AngularJS know about your changes. So just to recap, fire an event using `$rootScope` from console would look like below:

``` javascript
// in browser's console
var sidebar = document.getElementsById('sidebar');
var scope = angular.element(sidebar).scope();
var rootScope = scope.$root;
scope.$apply(function() {
  rootScope.$broadcast('myEvent', {data: myData});
});
```

And you should now see your application reacting on this freshly baked event, raised from browser's console.
