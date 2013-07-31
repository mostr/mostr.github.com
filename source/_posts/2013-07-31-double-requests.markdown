---
layout: post
title: Preventing duplicated requests in AngularJS
date: 2013-07-31 22:12
comments: true
categories: javascript angularjs
---

You know the case when you hit submit button quickly enough that it sends the form twice? Or when you hit submit button and it seems that nothing happens, so you click it once again and another request is being fired? Duplicated requests issue - you can experience it in both traditional web application as well as in modern so-called Single Page Applications. I'd like to show you how it can be handled in AngularJS and how we did it in [Codebrag](http://codebrag.com).

AngularJS itself has no built-in stuff for that. You can fire as many HTTP requests as you want and it is up to you and your application's code to handle it the way that would prevent double form submission. So let's see what weapons we are armed with to win this battle.

### Disable button after submit

Suppose you want to prevent user to double-submit the same form. You can write pretty simple AngularJS directive to disable button when clicked. It will effectively block this button from being pressed again until e.g. response comes back. I personally don't really like this one, because you need to remember to put this directive on every form in you application. It looks like violating DRY a bit. So let's look for something better.

### Asynchronous UI approach

This is not a solution to this particular problem per se but more an approach to building web applications. [Alex McCaw](http://blog.alexmaccaw.com/asynchronous-ui) has great post about it which I highly recommend reading. Basically the trick is not to wait for server's response if it is not absolutely required. Simply act as if action was performed succesfully and immediately do what should be done on success. So let's take our form submit example. When user clicks submit button you fire HTTP request as usual and close form (possibly doing something more too, like adding comment to list) without waiting for response from server. It effectively prevents user from resubmitting form as it disappears immediately after button click. Obviously there are some drawbacks too and there are also use-cases (like payments etc) where this approach isn't recommended, but in most cases it will work fine. For more details I really recommend reading [Alex's post](http://blog.alexmaccaw.com/asynchronous-ui).

### AngularJS-based solution

Ok but this approach above requires rethinking and possibly changing significant stuff in you application. What if you don't really want to do that? It turns out that solution for our initial problem can be quite easily implemented with tooling already available in AngularJS.

What we need to do is to "intercept" `$http` service calls and decide if given request should be sent or not. Angular has concept of http interceptors but only for responses, not for requests (yet). But fear not, we'll use decorators. As official doc says

> Decoration of service, allows the decorator to intercept the service instance creation. The returned instance may be the original instance, or a new instance which delegates to the original instance.

and that's exactly what we need. So this is how decorator is applied in [Codebrag](http://codebrag.com):

``` javascript decorator usage
	angular.module('codebrag').config(function($provide) {
        $provide.decorator('$http', function($delegate, $q) {
            return codebrag.uniqueRequestsAwareHttpService($delegate, $q);
        });
    })
```

I'll show you the details of implementation soon. Next issue to solve is how to determine when requests are identical and if one is already in progress? In the simplest form HTTP requests in AngularJS are done as below:
	
``` javascript simple http call	
	var config = {method: 'POST', url: 'http://api.myapp.com/comment', data: {msg: 'foo bar'}};
	$http(config);
```
	
All other methods like shortcut `get`, `post` as well as `$resource` use this form internally.
It turns out we can freely add our own properties to this `config` object. They will help us identify requests. So we can for example modify the config above as follows:

``` javascript modified config	
	var config = {
		method: 'POST', 
		url: 'http://api.myapp.com/comment', 
		data: {msg: 'foo bar'}, 
		unique: true,
		requestId: create-comment'
	};
```
	
This new config contains two new properties `unique` and `requestId`. The first one determines if for given types of requests we should check for duplicates or let all of them be sent (we probably can let all GET requests to go through without this check, and have e.g. POSTs checked). The second one, `requestId` is a property we'll match on when looking for duplicates. In this case it has constant value (create-comment) which means that only one POST request to `http://api.myapp.com/comment` should be pending at any time. This value can be dynamically calculated (e.g. using `data` for more fine-grained control).

Ok, but how can we find out which requests are currently in progress? `$http` service has one neat property called `pendingRequests` which is array of `config` objects for request that were sent. So matching duplicated requests is just a matter of searching through `pendingRequests` for request with identical `requestId` as one in our request we are about to send. So here is first part of implementation:

``` javascript first implementation	of modified service
	codebrag.uniqueRequestsAwareHttpService = function($http) {
	
	    var uniqueRequestOptionName = "unique";
	    var requestIdOptionName = 'requestId';
	
		// should we care about duplicates check
	    function checkForDuplicates(requestConfig) {
	        return !!requestConfig[uniqueRequestOptionName];
	    }
	
		// find identical request in pending requests
	    function checkIfDuplicated(requestConfig) {
	        var duplicated = $http.pendingRequests.filter(function(pendingReqConfig) {
	            return pendingReqConfig[requestIdOptionName] && pendingReqConfig[requestIdOptionName] === requestConfig[requestIdOptionName];
	        });
	        return duplicated.length > 0;
	    }
	
	    var modifiedHttpService = function(requestConfig) {
			// if we need to check for dups and pending found - return
	        if(checkForDuplicates(requestConfig) && checkIfDuplicated(requestConfig)) {
	            return;
	        }
	        // otherwise pass requeust to original $http service
	        return $http(requestConfig);
	    };
	    
	    return modifiedHttpService;
	};
```

It works fine if you try it, but has one huge drawback. `$http` service calls return promises so you can attach to them using `then` function and wait for them to be either resolved or rejected. Our current implementation is not consistent in return types. In fact it returns nothing when duplicate is detected, but returns regular promise when request is passed to original `$http`. To fix it we need to construct deferred using `$q` service as below.

``` javascript returning promise
	codebrag.uniqueRequestsAwareHttpService = function($http, $q) {
	
	    var DUPLICATED_REQUEST_STATUS_CODE = 499; // I just made it up - nothing special
	    var EMPTY_BODY = '';
	    var EMPTY_HEADERS = {};
	
		// previous stuff here

	    function buildRejectedRequestPromise(requestConfig) {
	        var dfd = $q.defer();
	        // build response for duplicated request
	        var response = {data: EMPTY_BODY, headers: EMPTY_HEADERS, status: DUPLICATED_REQUEST_STATUS_CODE, config: requestConfig};
	        console.info('Such request is already in progres, rejecting this one with', response);
	        // reject promise with response above
	        dfd.reject(response);
	        return dfd.promise;
	    }
	
	    var modifiedHttpService = function(requestConfig) {
	        if(checkForDuplicates(requestConfig) && checkIfDuplicated(requestConfig)) {
	        	// return rejected promise with response consistent with those from $http calls
	            return buildRejectedRequestPromise(requestConfig);
	        }
	        return $http(requestConfig);
	    };
	    
	    return modifiedHttpService;
	};	
```

Done, we have fully working implementation. Just one note, it works only for direct `$http` calls, if you try to fire `$http.get` or `$http.post` or even `$resource` it won't work, because those use shortcut calls defined directly on `$http`. Fix for that would be to define such functions on our modified version of `$http` service, but I'll leave it to you.

And that's all. We have fully working decorator implementation that can prevent duplicated requests from sending. You can configure it separately for every `$http` request you define in your application. Just add `unique` and `requestId` parameters to request config. 

I'm sure there are other methods for doing this kind of stuff. If you know one, let me know about it in comments below.
