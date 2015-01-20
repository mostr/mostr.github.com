---
layout: post
title: You don't always need AngularJS DI in directives
date: 2015-01-18 22:43
comments: true
categories: javascript angularjs web
---

This post is a follow up to the discussion about Angular directives we had recently in [SoftwareMill](http://softwaremill.com) with [Marcin Kubala](https://twitter.com/marcin_kubala). Marcin was working on refreshing our Scala with AngularJS scaffolding project called [Bootzooka](https://github.com/softwaremill/bootzooka).

## Use case

We want to display some notifications to user (think of it as a Twitter Bootstrap's "alert" component). This is to let user know for example that changes he made were either successfuly saved or not, or display any other information user would be able to dismiss once read. There is usually one such place in application that displays that kind of things.

## To the code

So let's say we have Angular component to keep and manage (add new, dismiss) messages to display. It's gonna be `factory` so that it nicely encapsulates data and behavior we need and it's easily injectable to almost any other component. Here is slightly simplified version, just for this post's purposes:


``` javascript
app.factory('MessagesStore', function () {
  
  var messages = []
  
  function dismiss(msg) {
    var i = messages.indexOf(msg);
    messages.splice(i, 1);
  }
  
  function add(type, text) {
    messages.push({ type: type, text: text });
  }
  
  return {
    messages: messages,
    dismiss: dismiss,
    add: add
  }
  
})
```

Once we have place to store and access messages to be displayed, let's make it displayable with directive. At first we'll define simple template for directive to use:

{% raw %}
``` html
    <script id="messages.html" type="text/ng-template">
      <ul class="messages">
        <li ng-repeat="m in msgSource.messages">
          {{ m.type }}: {{ m.text }} <button ng-click="msgSource.dismiss(m)">dismiss</button>
        </li>
      </ul>
    </script>
```
{% endraw %}

No rocket science here, just simple `ng-repeat` to display all stored messages. Also each displayed message can be dismissed by user by clicking **dismiss** button next to it. Now, let's move on to the actual directive:

``` javascript
app.directive('messages', function (MessagesStore) {
  
  return {
    restrict: 'E',
    templateUrl: 'messages.html',
    link: function(scope) {
      scope.msgSource = MessagesStore;
    }
  }
  
})
```

As you can see to let directive know about messages to display we simply inject `MessagesStore` and make use of it (in this case simply assigning it to `msgSource` used in template). Obviously if there were other stuff to be done (like auto-dismissing messages after some time etc.) there would be more code in `link` function, but let's not complicate things.

And now... it's done, you may say. It works like a charm, displays any messages that are already stored in `MessagesStore` and lets you dismiss every single one. Sure, but I personally see one drawback here: this directive is tightly coupled to `MessagesStore` service. You cannot use it without having this service around and I don't feel this is the best solution. I like to have my directives being independent from other components as much as possible. So let's see how can we deal with it and make it a bit better.

## "Events!", you may say

And you're right... more or less. Events and publish/subscribe patterns in general give fairly good decoupling. Simply use pub/sub system built into Angular itself. Let's see how it would look like:

``` javascript
app.directive('messages', function ($rootScope) {
  
  return {
    restrict: 'E',
    scope: {},
    templateUrl: 'messages.html',
    link: function(scope, el, attrs) {
      
      var messages = [];
      
      $rootScope.$on('msg', function (e, msg) {
        messages.push({ type: msg.type, text: msg.text });  
      });
      
      function dismiss (msg) {
        var i = messages.indexOf(msg);
        messages.splice(i, 1);
      }
      
      scope.msgSource = {
        messages: messages,
        dismiss: dismiss
      };
      
    }
  }
  
})
```
To display a message you need to publish a `msg` event from the `$rootScope`, like this:

``` javascript
$rootScope.$emit('msg', { type: 'info', text: 'I am message from event'} );
```

Looks like fairly good way of getting rid of dependencies in directive. Actually it still has one `$rootScope`, but it's not that bad as "everything" in Angular is in some scope and technically you could with minimal effort use directive's `scope` instead. But there is one thing that bothers me in this approach: there is no clear concept of messages and messages store. Also there is no clear API saying how it can be used. See? No `MessagesStore` anywhere, entire well-defined part of the system got blurred somewhere. Now suppose you need two different notifiers in one application displaying different notifications. You can't do that with this evented version shown above. This is because both instances of directive will react on the same event, and will consequently display the same messages. This isn't what you want, right? There is another issue that relates to events in general: if you abuse events and pub/sub in general you'll have hard times when reasoning about the code, tracking all the events and their handlers, order of execution etc. It looks like it's better in terms of dependencies-coupling, but it has its own problems. So is there a hope? Sure it is (if not why would I write that thing anyway?)


## Make it work, then make it right...

So, both solutions work and are perfectly fine in some kind of applications. But I personally like to have my directives to be as independent and reusable as possible. Let's say I want exactly the same component in my other application: I can't easily take the first one without taking `MessagesStore` with it. Suppose `MessagesStore` depends on yet another service and you end up with whole dependency tree to satisfy while you don't really need it in your project. Taking the events-based version introduces yet another event publisher and subscriber to complicate your app a bit more. They in turn introduce events that aren't "native" to your app, they are breaking naming conventions, etc.

Let's define what we really want: reusable directive with possibly minimal set of "component-based" dependencies that need to be satisfied, that has well defined API and clearly visible domain concept it interacts with.

In both solutions shown above, directive itself comes down to `<messages></messages>` snippet in your view. But directives usage can communicate much more, it can define API, entry points. Instead of relying on directive taking service as dependency on framework level (or internally listening for specific event), provide it with `MessagesStore`-like object that it could interact with. Pass it to directive's isolated scope and make it whatever you want (just stick to `MessagesStore` API).

``` javascript
app.directive('messages', function () {

  return {
    restrict: 'E',
    scope: {
      msgSource: '=',
    },
    templateUrl: 'messages.html'
  }
  
});
```
That's all, as simple as that. Now, use it as follows:

``` html
<messages msg-source="msgStore"></messages>
```

Because every directive's instance lives in some controller's scope, just let controller provide stuff that this directive needs:

``` javascript
app.controller('AppCtrl', function($scope, MessagesStore) {
  
  $scope.store = MessagesStore;  
  
});
```

It may look like unnecessary complexity introducing kind of level of indirection, but think of it: when you need to use this directive in any other project you just take it (with no deps) and the only thing to make it working is to provide it with something that quacks like `MessagesStore`. When you need several instances of such notifier - no worries, just provide each one with separate `MessagesStore` like object. Doesn't matter whether the `MessageStore`-like thing is service, factory or just simple object created ad-hoc in controller, directive doesn't care unless you obey API contract. Another benefit for me personally is that this form of directive (taking attribute defining messages source) is way more readable. It's way more informative than cryptic `<messages></messages>` thing.

And that's all, that's how I'd code that for now. There is nothing new, nothing complicated and definitely no magic (well, maybe except directive's `=`, for some less experienced angularians). Conceptually this is still dependency injection but key point here is that you don't always need to couple stuff on framework level with its own DI, even if it's perfectly doable and easy. Think of long term project development, maintenance, about components reusability, your fellows developers, etc. I'm not saying this is silver bullet and will work in all your projects, and you should ban injecting any dependencies to directives - I'm far from that. Just think how would you like to use that directive if you were newcomer and what are potential tradeoffs of each solution (because you always have at least two, don't you?).

Regarding [YAGNI](http://c2.com/cgi/wiki?YouArentGonnaNeedIt) and not complicating stuff if not required for those who may doubt: initial discussion that triggered this post took place during **migration of some Angular components** from [Codebrag](http://codebrag.com) to [Bootzooka](https://github.com/softwaremill/bootzooka), so as you can see thinking a bit about reusability isn't that bad.