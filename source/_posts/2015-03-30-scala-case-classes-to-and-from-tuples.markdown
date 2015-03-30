---
layout: post
title: Scala case classes to and from tuples
date: 2015-03-30 18:53
comments: true
categories: scala
---

This one is going to be pretty quick and it's more like note to self, but maybe some of you find if helpful too. 

Have you ever had to convert between case classes and tuples in your Scala code? It's not something you do every day, all day long but it happens to me from time to time. The worst thing here is I constantly keep forgetting how to do __exactly__ this dance and have to look for it in APIs or fallback to Google/Stack Overflow for that. So without further ado, let's go to the codes.

There is `User` case class defined as follows

``` scala
case class User(name: String, email: String, age: Int)
```

Say you have a tuple that "looks like" `User`'s parameters list. It'd be nice to be able to use `User.apply` on it, wouldn't it?

``` scala
// this is regular Tuple3
val userLikeData = ("John", "john@doe.com", 33)

val user = (User.apply _).tupled(userLikeData)
// => User("John", "john@doe.com", 33)
```

In short it takes `User.apply` as a function first. On every function (here of type `Function3`) you can invoke `tupled` to convert its parameters list to corresponding tuple type. Because it's all about `Function` the example above can also be written as:

``` scala
Function.tupled(User.apply _)(userLikeData)
```

Ok, let's get tuple back out of already created case class instance.

``` scala
// this is regular User
val user = User("John", "john@doe.com", 33)

User.unapply(ben).get
// => ("John", "john@doe.com", 33)
```

And that's it. Case classes have `unapply` which returns `Option` with tuple inside. Just `get` from this option to receive plain tuple with user's data.

## Bonus points - real classes

Is above possible with regular classes (not case classes) too? Let's see

``` scala
class Address(city: String, street: String, number: Int)
```

We don't have any `apply` here to use (it's not real function), but there is good ol' partial application that we can use:

``` scala
val addressCtorFn = new Address(_, _, _)
// => (String, String, Int) => Address
```

Now we have a function which basically works like `Address` constructor and already know what to do with it:

``` scala
Function.tupled(addressCtorFn)
// => ((String, String, Int)) => Address
// or simply
addressCtorFn.tupled
// => ((String, String, Int)) => Address
```

And done, now you can build regular `Address` classes out of tuples (if they match, obviously).

There is no way (at least at my current Scala-fu level) that would allow getting tuples back from regular classes. If it is somehow possible and you know the right lines, just drop me a comment.




