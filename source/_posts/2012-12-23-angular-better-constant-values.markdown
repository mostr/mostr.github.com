---
layout: post
title: Use functions (IIFE) to define AngularJS constants and values
date: 2012-12-23 13:23
comments: true
categories: javascript angularjs
---

In AngularJS to provide some constant values to your application (e.g. some config values used across modules your app) you can use either `value` or `constant` when defining module and get them injected where required, in services, controllers etc.

Basically creating `value` and `constant` look as follows:


``` javascript
    var application = angular.module('application', []);
    application.constant('foo', 'fooConstant');
    application.value('bar', 'barValue');
    
    // inject them and use
    application.controller('FoobarCtrl', function(foo, bar) {
        $scope.foobar = foo + " " + bar;
    });
```

Both `value` and `constant` values can be also literal objects as below:


``` javascript
    application.constant('foobar', {
        foo: 'foo',
        bar: 'bar'
    });

```    
    
But suppose we'd like to have set of configuration parameters where one depends on another (e.g. one is build using the other's value), but we don't want to repeat this value again and again (remember DRY, huh?). For example we could have configuration as below:


``` javascript    
    protocol = 'http://'
    fooService = protocol + 'fooservice.com'
    barService = protocol + 'barservice.com'
```    
    
Unfortunately we can't do this using javascript object literals as there is no way to reference `protocol` property of the same object. We could define `fooService` and `barService` as functions but we'd like to have configuration as properties not  functions. Defining separate `constant` or `value` for protocol doesn't help much as we can't get it injected when defining others. Theoretically we could achieve desired result by defining function as follows:


``` javascript
    function() {
        var protocol = 'http://';
        return {
            fooService: protocol + 'fooservice.com',
            barService: protocol + 'barservice.com',
        }
    }
```    
    
Unfortunately still no luck here because functions are not accepted as values for AngularJS `value` and `constant` definitions.  But fear not. Immediately Invoked Function Expression (IIFE) to the rescue. Such expression allows you to define function and invoke it immediately, so this whole expression evaluates to the return value of this function. So we can grab the function definition above and make use of it as follows:


``` javascript
    application.constant('servicesConfig', (function() {
        var protocol = 'http://';
        return {
            fooService: protocol + 'fooservice.com',
            barService: protocol + 'barservice.com',
        }        
    })());
```

Doing this we can reference `protocol` definition without duplicating its value while still being able to define full-blown AngularJS `constant` or `value` with only properties.