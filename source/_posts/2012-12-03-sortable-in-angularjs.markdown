---
layout: post
title: Multiple jQuery sortables in AngularJS
date: 2012-12-03 10:46
comments: true
categories: javascript angularjs
---

Recently for one of my side-projects I was looking for solution that'd allow user to drag elements from one logical set to another and to reorder them. I knew there was [sortable plugin from jQuery UI ](http://jqueryui.com/sortable/) that would do the trick. As my application is powered by awesome [AngularJS](http://angularjs.org/) I was searching for something similar in AngularJS world (possibly kind of sortable wrapper). It turns out that there is such extension in [angular-ui](https://github.com/angular-ui/angular-ui) project that wraps jQuery sortable plugin in AngularJS directive. After several quick&dirty shots and source code inspection I noticed that it has functionality limited to only handling single collection. I could sort/reorder items within one set, but couldn't link multiple sets together.

## Here comes ui-multi-sortable directive ##
 
So I decided to try to create something that satisfies those requirements. As a result I came up with `ui-multi-sortable` directive that you can find [here](https://github.com/mostr/angular-ui-multi-sortable). It is partially based on angular-ui directive mentioned above. I'll describe how to use this new directive and what were problems I faced during implementation. 

Let's start with model definition for this example. Suppose we are building simple taskboard that has two columns: "To do" and "Done". Our data model (defined in AngularJS controller) would look like below:

``` javascript
    $scope.items = {
      todo: [
        {name: 'todo 1'},
        {name: 'todo 2'},
        {name: 'todo 3'},
        {name: 'todo 4'}
      ],
      done: [
        {name: 'done 1'},
        {name: 'done 2'},
      ]
    };
```
	
We want our users to be able to grab items from "To do" column and move them to "Done" column. Obviously we want our underlying model to be in sync with what user does.
Let's define our UI part as simple two-columns layout:

``` html    
    <body ng-controller="TaskboardController">
        <div ui-multi-sortable ng-model="items" model-subset="todo" class="column">
            <div class="item" ng-repeat="item in items.todo">{{ item.name }}</div>
        </div>        
        <div ui-multi-sortable ng-model="items" model-subset="done" class="column">
            <div class="item" ng-repeat="item in items.done">{{ item.name }}</div>
        </div>
    </body>
```
    
As you can see there is `ui-multi-sortable` directive used instead of `ui-sortable` one available in angular-ui. As our model isn't just plain array of objects (it's object containing several arrays of items instead) we need to tell Angular which subset of items should be linked with which sortable element. That's what additional `model-subset` attribute does. It is simply defined as key name under model root. Here `items` is model root and `todo` is what contains array of elements to be used in first column.

For each sortable element we need to explicitly tell jQuery what are the others sortables we want to link with. In this case we'd like to have all elements with class `column` to be used together. This may be done by providing `ui-options` attribute for every single column as follows:

``` xml
	ui-options="{connectWith: '.column'}"
```
This is set of options that are passed to underlying jQuery plugin. For more information and available options see jQuery UI sortable API. But that's not "DRY" enough, right? Instead we can provide global options for sortable in AngularJS like this:

``` javascript
    angular.module('ui.config', []).value('ui.config', {
      sortable: {
        connectWith: '.column', 
      }
    });
```

And that's all. You can now move items between those two columns and your underlying model will always stay in sync with UI changes. 

## What is the magic behind all that? ##

Regarding implementation, the key is in correct underlying model manipulation (either just reorder items if in single sortable, or move items between different sortables). All that is not too difficult, just clever usage of `splice` on array. The trick is how to hook up all this into right jQuery sortable callbacks sequence.

Standard `ui-sortable` directive defines `start` callback that grabs original position of dragged item and `update` callback to execute actual model manipulation. Unfortunately in case of several related sortables this approach fails because when dragging item between different sortables sequence of events emitted is as follows: 

- `start` when sorting starts
- `update` on original sortable
- `remove` remove item from original sortable
- `receive` put item being moved in target sortable
- `update` on target sortable
- `stop` when sorting is done

The problem here is in two `update` callbacks being fired. 
First one fired on source sortable and looks as if there was operation of reordering items inside this sortable only (which is clearly not true here as we don't want to mess up with model this way). There is no way to get data about target sortable here. The second one is on target sortable, but this one on the other side knows nothing about source sortable and both source and target positions of item. I'm not sure if this was done that way intentionally, as I can't imagine good use case for it. All in all I could see no good way of using `update` for related sortables. So I decided to split that as below:

- use `stop` to change model when sorting happens within one sortable. Here it is fired only once, on source sortable and has all the data I need just for sorting elements.  
- use `receive` for cross-sortable interactions. It is fired on target sortable and also has all the data to update model.

Finally I came up with `ModelSynchronizer` object which handles all the model updates and encapsulates data required for this update. This is the only place where all the heavy lifting happens. The directive itself only defines set of callbacks which simply invoke corresponding operations on instances of `ModelSynchronizer` + additional callbacks defined by user.

In case you find it useful, go and grab all the code from [here](https://github.com/mostr/angular-ui-multi-sortable). Pull requests are also welcome.
