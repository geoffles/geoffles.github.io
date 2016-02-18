---
layout:	post
title:	"Modal Dialogs and Angular and Visual Studio"
categories:	Web
date:	2016-02-18 15:03:17+0200
tags:	[Angular, Visual Studio]
---

***Abstract:** The barebones and an explanation thereof for getting a modal dialog in Angular using Visual Studio.*

* TOC
{:toc}

## Introduction

I've been playing around with Angular a lot lately and found that some concepts were difficult to get a blow by blow account of what is needed.

My specific problem was using the Bootstrap UI modal dialogs in Angular from Visual Studio. Angular is quite extensive and I'd reccommend using one of the many package managers out there to help you out. In the case of Visual Studio, that means Nuget.

## Angular and Bootstrap Packages

So lets start with what packages you need to have a modal dialog:

1. Angular
2. Bootstrap
3. Angular.UI.Bootstrap

Search for these in the nuget package manager, or from the PM console:

{% highlight powershell %}
Install-Package Angular
Install-Package Bootstrap
Install-Package Angular.UI.Bootstrap
{% endhighlight %}

You should now have all the dependencies you need to use modal dialogs :)

## Implementing the Modal Dialogs

### The JavaScript

You need to declare a module for your app, and need to include a 2 other modules: `ngAnimate`, `ui.bootstrap`

{% highlight javascript %}
angular.module('testApp1', ['testApp1.controllers', 'ngAnimate', 'ui.bootstrap']);
{% endhighlight %}

Now you can declare your controllers:

```javascript
angular.module('testApp1', ['ngAnimate', 'ui.bootstrap']);
angular.module('testApp1').controller('testController', function ($scope, $uibModal) {
    
});

angular.module('testApp1').controller('testModalController', function ($scope, $uibModalInstance, someArg) {
   
});
```

There is a controller for the page, and another for the Modal Dialog itself. Note that you are *injecting* `$uibModal` into the main controller, and `$uibModalInstance` into the Modal controller. These services will be used to control the dialog itself. These services are provided by the module dependency `ui.bootstrap`. I'll come back to `someArg`.

The next step is to provide a function on your `$scope` to bring up the dialog. This is as simple as calling `$uibModal.open` like this:

```javascript
angular.module('testApp1').controller('testController', function ($scope, $uibModal) {
    //Open a dialog
    $scope.showMyDialog = function (someArg) {
        var modalInstance = $uibModal.open({
            animation: true,
            templateUrl: 'mytemplate.html',
            controller: 'testModalController',
            resolve: {
                someArg: function () {
                    return someArg;
                }
            }
        });

        //Promise on close
        modalInstance.result.then(function (result) {
            alert(result);
        }, function () {
            alert('aborted');
        });
    }
});
```

Most of the properties for the object passed to the `open` function are self explanatory:

-  `animation` controls the animation on the dialog (you will need ngAnimate for animations)
-  `templateUrl` refers to the template to use
-  `controller` is the controller that the dialog should use
-  `resolve` is a mechanism for injecting arguments into the dialogs controller. This is where `someArg` comes in.

We also have a promise (also sometimes referred to as a *future*) for controlling what happens when we close the dialog. As will most promises in Angular, the first function is executed on "success" and the second is executed on "failure" - basically, clicking "ok" and "cancel".

Next we need to define a couple of things on the Modal Controller, namely adding our injected data onto the `$scope`, and some functions for clicking ok and cancel:

{% highlight javascript %}
angular.module('testApp1').controller('testModalController', function ($scope, $uibModalInstance, someArg) {
    $scope.stuff = someArg;

    $scope.ok = function () {
        $uibModalInstance.close($scope.stuff);
    }

    $scope.cancel = function () {
        $uibModalInstance.dismiss('cancel');
    }
});
{% endhighlight %}

Note that we use the `$uibModalInstance` to actually close the dialog. Internally this will also correctly route back to the promise in the main controller.

### The HTML

Of course, we need some HTML:

```xml
<html>
<head>
    <link href="/Content/bootstrap.min.css" rel="stylesheet" type="text/css" />
    <link href="/Content/ui-bootstrap-csp.css" rel="stylesheet" type="text/css" />
</head>
<body ng-app="testApp1" ng-controller="testController">
    <script type="text/ng-template" id="mytemplate.html">
        <div class="modal-header">
            <h3 class="modal-title">My Dialog!</h3>
        </div>
        <div class="modal-body">
            SomeArg: <b>{{stuff}}</b>
        </div>
        <div class="modal-footer">
            <button class="btn btn-primary" type="button" ng-click="ok()">OK</button>
            <button class="btn btn-warning" type="button" ng-click="cancel()">Cancel</button>
        </div>
    </script>
    <script src="/Scripts/jquery-1.9.1.min.js"></script>
    <script src="/Scripts/bootstrap.min.js"></script>
    <script src="/Scripts/angular.min.js"></script>
    <script src="/Scripts/angular-animate.js"></script>
    <script src="/Scripts/angular-ui/ui-bootstrap-tpls.js"></script>
    <script type="text/javascript">
         //JS Goes here
    </script>

    <div>
        <button class="btn btn-default" type="button" ng-click="showMyDialog('foo')">Open Dialog</button>
    </div>
</body>
</html>

```

You can copy the above into an HTML page in a Visual Studio Web Project (after also adding the nuget packages), sub in the JS and you should be A-for-away. Alternatively you can pick up a sample HTML from [here](/res/web/ng_dialog_sample.html) (right-click->save as).

## Results and Discussion

When you "run" the page from Visual Studio the page it should look like this:

![Open me](/img/web/ng_dialog.png)

And you can then of course click the button to show:

![Open me](/img/web/ng_dialog_open.png)

Something important to note is that `foo` was injected into the Modal Dialog Controller. I've just injected a string, but it could be anything! Also of note is that the Modal Dialog can **return** data back to the main controller (Note the contents of the alert) - so you can of course use this technique to provide a pop-up editor. Super cool.

## Conclusion

So, that brings us to conclusion! You should now be able to add interactive dialogs to your angular app :)

Note that this may not conform to an Angular "Best Practice", the purpose here is to distill the essence of getting a Modal Dialog working! Don't neglect best practices!  

