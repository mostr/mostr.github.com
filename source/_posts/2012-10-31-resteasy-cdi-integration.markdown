---
layout: post
title: RESTEasy resources are CDI-aware in JBoss AS by default
date: 2012-10-31 13:29
comments: true
categories: java javaee cdi jboss jax-rs
---

JAX-RS and CDI have slightly different scopes so that by default there is no way to mix them and use CDI bean as JAX-RS resource, so `@Inject` and other CDI magic tools are not available. To get it working you normally need to annotate your class with `@RequestScoped` like below

``` java BookResource.java
    @Path("/book")
    @RequestScoped
    public class BookResource {

        // this will work only when @RequestScoped is present on a class level
        @Inject
        private MyService service;

        @GET
        @Produces("text/plain")
        public String doSomething() {
            return service.doSomething();
        }

    }
```

So every time you create endpoint you need to write that extra line to switch this CDI magic on.
Recently when implementing I forgot to put this `@RequestScoped` thing on my class and guess what? It actually worked fine! I spotted lack of that annotation accidentally some time later when I had all my tests passing and the whole feature working correctly.

## JBoss extras to the rescue

So I started digging deeper into that. I tried it on two 7.x versions of JBoss AS and on Glassfish 3.1.2.2. It turned out that on Glassfish there is `NullPointerException` when accessing injected field. So this is what expected. But on both JBoss servers it was all fine with no exception thrown and with work done.


It turns out that JBoss has its own extra module called `resteasy-cdi` for integration of RESTEasy and CDI that you can read about [here](http://docs.jboss.org/resteasy/2.0.0.GA/userguide/html/CDI.html). TL;DR (from the doc):

> The resteasy-cdi module is a bridge that allows RESTEasy to work with class instances obtained from the CDI container.
During a web service invocation, resteasy-cdi asks the CDI container for the managed instance of a JAX-RS component. Then, this instance is passed to RESTEasy (...)
As a result, CDI services like injection, lifecycle management, events, decoration and interceptor bindings can be used in JAX-RS components.

This is a tiny trick that can save you from writing some boilerplate code. It is actually only one line, but shouldn't we apply DRY where possible? Just remember, this is something extra from JBoss and is available only when using RESTEasy with CDI, and that's why Glassfish can't do that.