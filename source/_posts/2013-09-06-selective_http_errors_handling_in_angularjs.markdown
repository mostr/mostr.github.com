---
layout: post
title: Fine-tune HTTP errors handling in AngularJS using custom request configurations
date: 2013-09-06 14:48
comments: true
categories: angularjs javascript
---

Let's stay close to HTTP request/response cycle in AngularJS for a while. Remember [my previous post](http://michalostruszka.pl/blog/2013/07/31/double-requests/) about throttling duplicated requests? I showed how to use `config` object populated with custom properties to control which HTTP requests are invoked and which are not. This time I'll show you how to use the same `config` object but for playing with HTTP response.

Sometimes you may want for example to process different responses in different ways. Let's take HTTP errors for example. When you receive response with HTTP code different that 2xx in most applications you probably want to let user know that something wrong happened, so you may decide to display some popup with error message. This can be easily achieved with response interceptors in AngularJS. Simply write and enable interceptor that handles all failed responses and does something (e.g. broadcasts an event with error details which is then handled by specialized directive that displays message to user). In the simplest form such interceptor may look as follows:

``` javascript Simple HTTP errors handling interceptor
    angular.module('app').factory('httpErrorsInterceptor', function ($q, $rootScope, EventsDict) {

        function successHandler(response) {
            return response;
        }

        function errorHandler(response) {
            $rootScope.$broadcast(EventsDict.httpError, response.data.cause);
            return $q.reject(response);
        }

        return function(httpPromise) {
            return httpPromise.then(success, error);
        };

    });
```

It simply passes successful responses through and for error responses it broadcasts custom event (`EventsDict.httpError`) and then rejects it as it would be done normally. That's cool, but suppose you have a feature in your application that has its own way of handling errors? In [Codebrag](http://codebrag.com) we have "Invite others" form that after sending should display to user whether his invitation was sent or whether there was something wrong with it on the server side. Besides that we have general "error bubble" that is displayed when any error (including HTTP ones) occurs. As long as our "Invite others" form handles this case on its own we don't need to display this standard bubble anymore for those requests.

Ideally we'd like to have a way to tell which requests should be even considered by our interceptor when processing response and which should be left alone and just passed through. So here is what we did. In Angular every HTTP response has `config` object attached to it and this is exactly the same `config` object as the one from corresponding request. Let's make use of it again. When sending HTTP request let's define custom property in its `config` object:

``` javascript POST request with custom config property
  var data  = {
      invitationMsg: 'Hey! Come and join Codebrag',
      newUserEmail: 'foo@bar.com'
  };
  $http.post('/invitation', data, {bypassErrorsInterceptor: true});

```

And here it is. Custom `bypassErrorsInterceptor` property will be set for each requests of this type. Now the core stuff - let's modify our simple interceptor to leave responses for those requests alone. Just let them be rejected without any other work, but keep watching and reacting on any other requests that don't have this property set.

``` javascript Complete interceptor with custom property handling
    angular.module('app').factory('httpErrorsInterceptor', function ($q, $rootScope, EventsDict) {

        function successHandler(response) {
            return response;
        }

        function errorHandler(response) {
            var config = response.config;
            if(config.bypassErrorInterceptor) {
                return $q.reject(response);
            }
            $rootScope.$broadcast(EventsDict.httpError, response.data.cause);
            return $q.reject(response);
        }

        return function (promise) {
            return promise.then(success, error);
        };

    });
```

Done. Now every HTTP request with `bypassErrorsInterceptor` property set will be passed through and the rest will result with event broadcasting.

If you want to treat different HTTP error codes in different ways you can as well match responses by response code, send different events and display different messages to user - it's up to you, this can be also done in interceptors.