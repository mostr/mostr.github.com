---
layout: post
title: "Why I never really loved Angular auto-unwrapping promises in view"
date: 2013-12-05 08:23
comments: true
categories: javascript angularjs web
---

One Angular feature that was visually cool (from developer's perspective) was that you could pass promise directly to view and it work on it in HTML as if it was already resolved. I wrote "it was" because Angular folks decided to disable it by default in latest big 1.2 release - you can read more about it [here](https://github.com/angular/angular.js/commit/5dc35b527b3c99f6544b8cb52e93c6510d3ac577).

Let's recap how this auto-unwrapping used to work before 1.2. Let's have simple service that returns list of items wrapped in promise and call this service from controller.


``` javascript
  app.controller('ItemsListCtrl', function($scope, itemsService) {
    $scope.items = itemsService.loadItems();
    $scope.reload = function() {
      $scope.items = itemsService.loadItems();
    };
  });

  app.service('itemsService', function($q) {
    this.loadItems = function() {
      var items = ['item 1', 'item 2', 'item 3', Math.random()];
      return $q.when(items);
    };
  });
```

On the view let's display fetched items together with button to reload this list (it just fetches the same list again with random value added to clearly see it was reloaded).

``` html
  <ul>
    <li ng-repeat="i in items">{{ i }}</li>
  </ul>
  <button ng-click="reload()">Reload items</button>
```

## It's a list except when it's not

You can play with this example on [JSFiddle](http://jsfiddle.net/mostr/kQjKH/2/). It works like a charm, displays items, reloads list and so on, so what's the issue, you may ask?
Well, looking at the code of the controller it's not obvious, that `$scope.items` is really a promise. It looks as simple assignment - you load items and assign list of those items to `$scope.items`. Nice and clean, but try to e.g. log size of this list, like this:

``` javascript
  app.controller('ItemsListCtrl', function($scope, itemsService, $log) {
    $scope.items = itemsService.loadItems();
    $log.info($scope.items.length); // boom!
  });
```

You'll get `undefined` in console. Surprised? Maybe not in this simple case, but I expect it could bite you hard in real project with larger and more complicated codebase. To confuse you a bit more, when you try to display the same thing in view using `items.length` binding, it works fine. It is all because when dealing with `$scope.items` in controller you in fact deal with promise not the actual list of items, but on the view layer you get unwrapped result to play with.

## I disappear

There is another surprise you can spot while relying on this automatic promises unwrapping. Let's add some delay to our list loading function, so that it emulates fetching data from server (like in real HTTP call there is small delay as server needs to be called).

``` javascript
  app.service('itemsService', function($q, $timeout) {
    this.loadItems = function() {
      var dfd = $q.defer();
      $timeout(function() {
        var items = ['item 1', 'item 2', 'item 3', Math.random()];
        dfd.resolve(items);
      }, 500);
      return dfd.promise;
    };
  });
```

Again go to [this fiddle](http://jsfiddle.net/mostr/7qAbP/2/) to see it running. Now when you hit "reload" button, this list disappears for a while and then appears back. This is again due to the fact that you pass raw promise to the view. It cannot render until this promise gets resolved, so it renders nothing and appears again when promise is resolved. So the longer the delay is the more unpleasant the experience can be. How to avoid that? One solution is to delay controller instantiation until promises are resolved (using resolve property on route) but that's not one I'm gonna show here. The exact one is explicitly deal with promises like below:

``` javascript
  app.controller('ItemsListCtrl', function($scope, itemsService) {
    itemsService.loadItems().then(passItemsToScope);
    $scope.reload = function() {
      itemsService.loadItems(passItemsToScope);
    };
    function passItemsToScope(items) {
      $scope.items = items;
    }
  });
```

That's few keystrokes more but for me it's definitely worth it. Now you don't touch existing items list until promise is resolved so swapping actual list is blazingly fast - go and try it [there](http://jsfiddle.net/mostr/qG7jV/2/). The list doesn't disappear while waiting for promise to be resolved - it just stays there and gets reassigned and rerendered in (almost) one go. Another benefit is that this code clearly says what it does and aligns nicely with [Principle of least surprise](http://en.wikipedia.org/wiki/Principle_of_least_astonishment). You clearly see that what is returned from `itemsService.loadItems()` is a promise, so you can deal with it accordingly.

## Summary

I don't say that both of those points mentioned above are vailid in all cases. It can serve you well, especially the one with disappearing content (some may have such requirement). Anyway I'm personally ([really](https://twitter.com/mostruszka/status/391253436382978049)) happy, that Angular folks decided to disable that feature from version 1.2+. Even if it was visually appealing, clean and so on it could potentially introduce a lot of WTFs to your daily work. If you by any chance really miss that feature, there is a way to bring it back, but I'm sorry I'm not gonna give you a clear hint how to do that.