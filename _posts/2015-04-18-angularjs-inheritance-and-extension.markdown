---
layout: post
title: Angular JS inheritance and extension
description: Allow developers to extends your angular code thanks to inheritance and extension
date: 2015-04-18
tags:
    - angular
    - javascript
---

## Introduction

I am currently building a kind of factory for one page website. The editing tools have been developed with AngularJs.
(You can follow development here : https://github.com/jbouzekri/OpSiteBuilder/tree/develop)

Being a PHP developer, I had a hard time to apply the concept I know about extensions and modules in my frontend code.
I would be happy to hear any suggestions and improvements on my proposals.

## Extends the controller

In this project, the controller provides actions on the page. As it is a website factory, you can add blocks, move them in the layout, ...

I created first a controller holding all my basic logic.

``` js
angular.module('MyApp').controller('MyController', function () {
    $scope.firstAction = function () {
        console.log('implement the first action');
    };

    $scope.secondAction = function () {
        console.log('implement the second action');
    };
});
```

This controller is not mean to be instantiated. It only provides basic action I want in my website and that are mandatory.
Another empty controller is provided extending this one.

``` js
angular.module('MyApp').controller('MyControllerImpl', function () {
    angular.extend(this, $controller('MyController', {$scope: $scope}));

    $scope.secondAction = function () {
        console.log('Override the second action provided by MyController controller');
    };
});
```

This implementation is empty and extends the base controller. I use it in my view.

As it is empty, anybody can replace it with their own implementation. The idea here is to allow anybody to create a custom implementation extending MyController.
Developers can add custom method to scope matching their needs or even override the ones provided by the parent controller.

The only code modification needed is of course to edit the ng-controller attribute to match the new controller.

## Directives inheritance

Previously, I proposed an inheritance mechanism for controllers. This chapter will focus on directives.

In my project, I had content blocks providing edit features. Each blocks add custom actions like drag and drop the block, change the title, ... So
I needed a way to customize the actions provided by each block.

As each block was separated from all others, I chose directives to provide actions and logic and keep separated scope. Each block has a type so the html looks like this :

``` html
<div my-global-block-logic block-type></div>
```

This HTML proposes 2 directive. The first one is a global one I applied to all block, the second one depends on the type of the block. It can add custom scope methods or
override the ones provided by the global "my-global-block-logic" directive.

Let's look at the "my-global-block-logic" directive :

``` js
angular.module('MyApp').directive('myGlobalBlockLogic', function() {
    return {
        controller: function ($scope, $element) {
            $scope.firstAction = function () {
                console.log('implement the first action provided by the directive');
            };

            $scope.secondAction = function () {
                console.log('implement the second action provided by the directive');
            };
        }
    }
});
```

Then the "block-type-directive" :

``` js
angular.module('MyApp').directive('blockType', function() {
    return {
        require: '^myGlobalBlockLogic',
        controller: function ($scope, $element) {
            $scope.secondAction = function () {
                console.log('Override the second action provided by the parent directive');
            };
        }
    }
});
```

Here the blockType directive inherits the firstAction from the parent and override the secondAction with a custom method.

## Conclusion

This code inheritance pattern allows developers to extend easily the base JS code I provided in my project :

* For controller, developers only need to replace the ng-controller with the one extending the base controller they provided
* For blocks in the page, they only need to replace the "block-type" directive with their custom block directive and keep the parent "my-global-block-logic" directive,
thus allowing to change the implementation for each block of this type
