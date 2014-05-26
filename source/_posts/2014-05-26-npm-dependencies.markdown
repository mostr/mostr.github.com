---
layout: post
title: "On managing dependencies in javascript build system"
date: 2014-05-26 09:06
comments: true
categories: javascript npm tools
---

Every serious project depends on libraries or modules written by other developers. We don't want to reinvent the wheel and prefer to use available tried and tested solutions to do our stuff. And here comes dependency management, a way to maintain versions of libraries we use in our projects. It applies to both libraries we use in production application as well as libraries/tools we use to support our development and automation of our workflow. In Javascript there are two most popular ways of handling dependencies: for frontend components there is [Bower](http://bower.io) and there is [npm - node packaged modules](http://npmjs.org) which is, well... package manager for node.js. Here is my take on managing dependencies with npm but it describes only developer's workflow support. I'm not gonna talk about Bower or any other way of managing frontend-only components, as well as node.js-based production deployments, maybe some time soon.

In [Codebrag](http://codebrag.com) we have completely separated frontend application and use [GruntJS](http://gruntjs.com) as a build tool for that. As this is node.js-based tool we also make use of npm to manage all the dependencies (grunt plugins etc). There are basically two approaches I've found so far to handle that in node.js based projects, both of which with their pros and cons:

- commit all dependencies into VCS
- rely on `package.json` file and `npm install`

## node_modules - commit or not

Why would you want to commit all your dependencies to your version control system? It'd significantly increase your repo size, right? That's true, but the most important advantage here is that your application is ready to go immediately after cloning repo. It means you don't have to rely on any external repository (be it official npm registry or your local network cache), you just clone your repo and boot up your application - no more `npm install` required. Looks like pretty cool thing, huh? 

Well, I'm not convinced that much and let me tell you why. I come from java development world where you have tools like Maven, gradle etc. with both central and your local repositories. No dependency is commited to your repository. Sure, it used to be the case years ago when people were not using any standard way of managing dependencies - all libraries your project required were checked into VCS. Also for me it is natural that my project repository contains only this project's code without any external dependencies. Another thing is that in case of dependencies being commited to VCS every single change in `package.json` must be manually handled in repository - clean up unused packages, add new etc. One more reason I don't feel this approach is my fav one is that you sometimes depend on packages that are platform-dependent, like phantom-js which has different binaries for different platforms. How would you handle that when your dev platform is different from production one (e.g. Mac and Linux)? Would you commit all possible versions? So as you can see, there is a lot of things to consider when choosing your way. There are discussions around all this "commit or not" all over the internet so you can read other people's opinions before choosing your own way.

## npm install 

Now let's move to the other option which sounds better to me personally. It's something I'm used to and it doesn't clutter your project's repo with tons of dependencies. You have them all defined in `packages.json` file and doing simple `npm install` fetches them for you. After that you are theoretically ready to boot up your application. But that's not that easy. While defining version of your dependency you can either put strict version, like `1.2.3`, or you can say `>1.2` or even `~1.2` (which is "reasonably close to 1.2") etc. It means you can define ranges of applicable version. And this is where  things get way more complicated. Say, you depend on module `foo` in `1.0.2` version and it in turn depends on module `bar` in version `>2.1.0`. You have no control over `bar` version as it is transitive dependency coming with `foo`. Whenever `bar` gets released in new version that satisfies `>2.1.0` you get this new version, even though you don't want that and you haven't changed anything in your dependencies. This is where your build may go unstable and where real WTF happens. There are many issues that can bite you there, in [Codebrag](http://codebrag.com) we had problems with non existing versions, npm servers returning 403 and so on. All that with no single change in our dependencies. So as you can see, this approach doesn't force you to maintain the entire jungle when you only want one banana, but it has its own drawbacks. 

## npm shrinkwrap

So is there any other way to handle that? Yes, there is and it's kind of variation of the one above. It's called [`npm shrinkwrap`](https://www.npmjs.org/doc/cli/npm-shrinkwrap.html). This tiny feature of `npm` effectively resolves all the dependencies tree from what is currently installed in `node_modules` and locks all the versions down. It creates file `npm-shrinkwrap.json` next to `package.json` which is used to determine versions while doing subsequent `npm install` calls. Now (going back to the foo bar example from above) even if new 'bar' gets released you will not be impacted by this, because you have current version locked and your dependencies tree is always the same. Again this looks like perfect solution - no deps checked into VCS and deterministic versions all the time, yay! Again, not so fast cowboy. Generated file doesn't care whether given dependency is production- or dev-related. By default `npm shrinkwrap` ignores dev dependencies, but you can force it to use them using `--dev` switch. But this means that doing subsequent `npm install` your dev dependencies will be also installed on your production environment, so be aware of that.

### Summary

The above drawback applies only to node.js applications which is not our case. In [Codebrag](http://codebrag.com) we use node.js + npm + grunt to automate our workflow so every dependency in our `package.json` is dev dependency. It means we can safely use `npm shrinkwrap --dev` to lock down versions of entire dependencies tree. So whenever you choose not to commit your npm dependencies into your VCS, be aware of this versioning gotcha and consider shrinkwrapping your deps. When the only node.js-based code in your application is the one supporting your frontend build system, `npm shrinkwrap --dev` seems to be perfect fit for you. Just remember that whenever you change anything dependencies-related in `package.json`, you need to shrinkwrap it again after new package lands in `node_modules` so new dependency version gets locked down.




