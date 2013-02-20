---
layout: post
title: "Cookbook dependencies in Chef Solo with Librarian"
date: 2013-02-20 10:12
comments: true
categories: automation chef-solo tools
---
*Disclaimer: I'm just chef-solo newbie and those are my findings after few hours with chef*


When cooking something non-trivial with chef-solo your cookbooks (or community cookbooks you use) may depend on other cookbooks. For example to use [this](https://github.com/edelight/chef-mongodb) `mongodb` cookbook you need `apt` and `yum` cookbooks too. So you need to be sure you have all the dependencies available in place so chef can reach them. It may be real PITA and you know that if you are e.g. java developer and had a chance to work without dependencies management tool and you had to figure out and download all the dependendent jars by hand. But fear not, fortunately "there is a gem for that" that helps you manage cookbooks' dependencies. It is called [Librarian](https://github.com/applicationsonline/librarian) and is works like [Bundler](http://gembundler.com/) for ruby projects, or like [Maven](http://maven.apache.org/) (its dependencies management part) for java projects. It uses its own `Cheffile` to declare required dependencies. For example to use `mongodb` cookbook mentioned above you may write the following:

``` ruby Cheffile
site 'http://community.opscode.com/api/v1'
cookbook 'mongodb', :git => https://github.com/edelight/chef-mongodb.git
```

Now run `librarian-chef install` and it will download all the declared cookbooks together with their dependencies to your cookbooks location. In case of the `mongodb` cookbook it will download this `mongodb` cookbook itself and `apt` and `yum` cookbooks as well.

I recommend you to take a look at [Librarian](https://github.com/applicationsonline/librarian) if you haven't used it before as well as get familiar with this whole topic of automated provisioning (no matter what tool you will choose, it's worth spending some time to learn it). I enjoyed it all a lot.

