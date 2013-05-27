---
layout: post
title: "Easy stubbing of HTTP requests in AngularJS for backend-less frontend development"
date: 2013-05-27 23:53
comments: true
categories: angularjs javascript 
---

Writing single page applications requires HTTP calls to server in order to either get data from or post data to server. In production environment there is real backend system with its own business logic, persistence etc. But what if we could leave all the backend infrastructure aside when in development mode of client part of application? In fact the only thing we are interested in (when working on client app) is particular response for particular request, not if server logic is implemented correctly. Sure, you can implement corresponding request handler on server and return static data for development needs, but there must be a better way to do that. What if we could stub HTTP requests on client side and leave our full-blown backend server aside completely, even without firing it up?

If you write your browser client using (excellent) [AngularJS](http://angularjs.org) framework it's pretty easy to do that. First of all Angular has all the low level AJAX-related details encapsulated into `$http` service (or level up - `$resource` to deal with world the RESTful way), so it means you usually call server like this:

``` javascript
    $http({method: 'GET', url: '/endpoint'})
```

and it handles all the underlying HTTP magic for you. It simply calls your server's `/endpoint` address and `GET`s data from it. 

[AngularJS](http://angularjs.org) is very serious on testing and it already has `$httpBackend` to simulate requests in unit tests etc., so why not use similar approach in development? Angular provides separate module called `ngMockE2E` dedicated for end-to-end testing which contains fake `$http` service definition and it turns out it can be easily used outside tests and serve as backend for your development environment. Angular guys recommend to use it the following way (from their documentation):

``` javascript
    myAppDev = angular.module('myAppDev', ['myApp', 'ngMockE2E']);
    myAppDev.run(function($httpBackend) {
        // define responses for requests here
    });
```

That's all fine, but there is no easy way to make it work out of the box without tweaking existing code every time you want to enable this stub mode. You still need to modify your `ng-app` attribute in your `index.html` file and point it to `myAppDev` module to be able to bootstrap your application with stubbed backend. 

Although there is a way you can make use of this fake backend without modifying your app bootstrap HTML code and this is what we did in [Codebrag](http://codebrag.com). We needed a way to fire up application without backend infrastructure e.g. to let our HTML/CSS wizard to work on application files without setting up all this server stuff etc. or just to work on some frontend without having backend done. So how we did that? That's quick'n'easy!

``` javascript httpBackendStub.js
    // 'codebrag' is our main application module
    angular.module('codebrag')
        .config(function($provide) {
            $provide.decorator('$httpBackend', angular.mock.e2e.$httpBackendDecorator);
        })
        .run(function($httpBackend) {
            // define responses for requests here as usual
        });
```

Angular lets you define multiple `run` and `config` functions for modules so we could use that ability to provide additional configuration and setup of our fake backend besides our regular configuration of `codebrag` module. Include this file (as well as `angular-mocks.js` from Angular guys) in your `index.html` and it will magically make your application using with stubbed backend. 

To make it even less obtrusive (at this stage you still need to add/remove this file reference in order to enable/disable stubbed backend) we did the following:

``` javascript httpBackendStub.js
    (function(ng) {
    
        if(!document.URL.match(/\?nobackend$/)) {
            return; // do nothing special - this app is not gonna use stubbed backend
        }

        console.log('======== ACHTUNG!!! USING STUBBED BACKEND ========');
        initializeStubbedBackend();

        function initializeStubbedBackend() {
            // 'codebrag' is our main application module
            ng.module('codebrag')
                .config(function($provide) {
                    $provide.decorator('$httpBackend', angular.mock.e2e.$httpBackendDecorator);
                })
                .run(function($httpBackend) {
                    // define responses for requests here as usual
                });
        }
    })(angular);
```

Here we used some Angular internals to tweak our main module configuration by providing stubbed `$http` service version. Now you can have `httpBackendStub.js` file included in your index.html every time when in development (no matter if you want to talk to real server via real HTTP calls or stubbed one) and it's just a matter of passing `?nobackend` parameter when hitting application in your browser's address bar when you want to access with "no server" mode. 

The only thing is you still need to serve your files (html partials, css styles and js scripts) via HTTP server somehow. In production one option is that it may be served from your backend server. Now when you don't have backend server up and running you still need simple web server to give you access to your files (due to CORS and security policies in browsers you may not do http requests from `file://` protocol). But fear not - this is as simple as firing up for example

``` bash
    python -m SimpleHTTPServer
```

in the main directory of your application's client part. So for example if you are java-being you probably have your web application client stuff in `src/main/webapp` directory. Just go there and fire the command below. It defaults to port 8000 so you can reach your app under [http://localhost:8080/app_url_here](http://localhost:8080/app_url_here). See? No full blown backend server there and you are still able to work on client side!

One final note - when preparing prod. distribution you can safely exclude this file containing fake backend configuration and definition from the process you use (concat/min/whatever) and your app will happily talk to real server.



