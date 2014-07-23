---
layout: post
title: "Scalaval - dead simple Scala validation micro library"
date: 2014-07-23 08:31
comments: true
categories: scala tools oss mirco-libraries
---
I'm big fan of what I call "lego approach" in software development. It means building project stack using many but small and focused libraries or frameworks where I can tweak every aspect of this project stack. I prefer this over full blown "platforms" which give you everything you *may* need together with tons of complexity and time required to learn stuff. This "micro libraries" approach is strong in JavaScript, especially node.js community as there are tons of npm modules that are really tiny and do one specific thing so you can compose your stack with number of such modules. The same thing is emerging in Ruby ecosystem, see [microrb.com](http://microrb.com/).

I'm not sure yet how it is in Scala, but it just got one more, new micro library authored by me. This one is called [ScalaVal](http://github.com/mostr/scalaval) and aims to help you writing validation in your application. Project's README answers "Why" question pretty well, I think:

> Because every project bigger than HelloWorldApp requires data validation at some point. Reinventing the structure and validation handling boilerplate every time you need one is definitely not something developers enjoy. I too had this pain when hacking on <a href="http://codebrag.com">Codebrag</a>.

> This small utility aims to provide minimal framework to write your validation logic, act as a guard between data and action and collect results in unified way. That's all it does. And it's really, really tiny. No external dependencies and no magic included. Just boilerplate. 

ScalaVal was just released to Maven Central in 0.1 version and works with Scala 2.10+. For usage examples please consult [projects README](http://github.com/mostr/scalaval), or (even better) see [project's specs](https://github.com/mostr/scalaval/tree/master/src/test/scala/com/softwaremill/scalaval). 

Entire source code is one, quite small source file (there is way more test code there). It may look silly to release something like this as full blown open source project, but as I said - I'm in love with micro-libraries approach and I really appreciate that I can take small library like this and it solves me exactly one issue in my project - specifically data validation in this case.

It's my first scala open source thing so I'm pretty excited even if it's just few lines of code. In case you spot any issue PRs are welcome, as well as comments and other ideas.
