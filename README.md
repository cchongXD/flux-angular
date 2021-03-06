flux-angular
==========

## Welcome to version 2 of flux-angular
There are some pretty big changes to the API in the new version. If you want to keep using the previous API, go to [flux-angular 1.x](FLUX-ANGULAR-1.md). I would like to give special thanks to @sheerun for all the discussions and code contributions.

## Features

- **Yahoo Dispatchr**
- **EventEmitter2**
- **Angular Store**
- **Scope listenTo**
- **Immutable**

## Concept
flux-angular 2 uses a more traditional flux pattern. It has the [Yahoo Dispatchr](https://github.com/yahoo/dispatchr) and [EventEmitter2](https://github.com/asyncly/EventEmitter2) for its event emitting. **Did you really monkeypatch Angular?**. Yes. Angular has a beautiful API (except directives ;-) ) and I did not want flux-angular to feel like an alien syntax invasion, but rather it being a natural part of the Angular habitat. Angular 1.x is a stable codebase and I would be very surprised if this monkeypatch would be affected in later versions.

## Create a store
```javascript
angular.module('app', ['flux'])
.store('MyStore', function () {

  return {

    // State
    comments: [],

    // Action handlers triggered by the dispatcher
    handlers: {
      'addComment': 'addComment'
    },
    addComment: function (comment) {
      this.comments.push(comment);
      this.emitChange();
    },

    // Getters
    exports: {
      getComments: function () {
        return this.comments;
      }
    }

  };

});
```
A store in flux-angular works just like the **Yahoo Dispatchr**, it IS the Yahoo Dispatchr. The only difference is an extra property called **exports**. So **exports** and **handlers** are special properties. **handlers** is an object defining what dispatched actions to listen to and what method to run when that occurs. **exports** is an object defining methods to expose to controllers. The methods in the exports object is bound to the store. Any data returned by an export method is cloned. This keeps the store immutable. If you need to use an other export method inside an export method use **this.exports.myOtherExport()** to do so. That will not cause cloning. 

## Dispatching actions and grabbing state from store
```javascript
angular.module('app', ['flux'])
.controller('MyCtrl', function ($scope, MyStore, flux) {
  
  $scope.comment = '';

  $scope.addComment = function () {
    flux.dispatch('addComment', $scope.comment);
    $scope.comment = '';
  };

  // $listenTo to listen to changes in store. Callback
  // runs on registration to update the $scope
  $scope.$listenTo(MyStore, function () {
    $scope.comments = MyStore.getComments();
  });

});
```
When a store runs the **emitChange** method any scopes listening to that store will trigger their callback, allowing them to update the $scope of the controller. You can also trigger specific events if you want to, with **emit('event')**.

### Event wildcards
Due to Angulars dirtycheck you are given more control of how controllers and directives react to changes in the store. By using wildcards you can choose to listen to any event change in a store, within a specific state or a specific event. All the following listeners will trigger when MyStore runs **this.emit('comments.add')**:

```javascript
angular.module('app', ['flux'])
.controller('MyCtrl', function ($scope, MyStore, flux) {

  $scope.$listenTo(MyStore, 'comments.add', function () {
    $scope.comments = MyStore.getComments();
  });

  $scope.$listenTo(MyStore, 'comments.*', function () {
    $scope.comments = MyStore.getComments();
  });

  $scope.$listenTo(MyStore, '*', function () {
    $scope.comments = MyStore.getComments();
  });

});
```

## Wait for other stores to complete their handlers
```javascript
angular.module('app', ['flux'])
.store('CommentsStore', function () {
  
  return {
    comments: [],
    handlers: {
      'addComment': 'addComment'
    },
    addComment: function (comment) {
      this.waitFor('NotificationStore', function () {
        this.comments.push(comment);
        this.emit('comments.add');
      });
    },
    getComments: function () {
      return this.comments;
    }
  };

})
.store('NotificationStore', function () {
  
  return {
    notifications: [],
    handlers: {
      'addComment': 'addNotification'
    },
    addNotification: function (comment) {
      this.notifications.push('Something happened');
      comment.hasNotified = true;
    },
    exports: {
      getNotifications: function () {
        return this.notifications;
      }
    }
  };

});
```
The **waitFor** method allows you to let other stores handle the action before the current store acts upon it. You can also pass an array of stores. It was decided to run this method straight off the store, as it gives more sense and now the callback is bound to the store itself.

### Get values from other stores
If your application is structured in such a manner that you need to share state between stores you can create a shared state object:

```javascript
angular.module('app', ['flux'])
.factory('AppState', function () {
  return {
    notifications: []
  };
})
.store('CommentsStore', function (AppState) {
  
  return {
    comments: [],
    handlers: {
      'addComment': 'addComment'
    },
    addComment: function (comment) {
      this.waitFor('NotificationStore', function () {
        comment.notificationId = AppState.notifications.length;
        this.comments.push(comment);
        this.emit('comments.add');
      });
    },
    getComments: function () {
      return this.comments;
    }
  };

})
.store('NotificationStore', function (AppState) {
  
  return {
    handlers: {
      'addComment': 'addNotification'
    },
    addNotification: function (comment) {
      AppState.notifications.push('Something happened');
      comment.hasNotified = true;
    },
    exports: {
      getNotifications: function () {
        return AppState.notifications;
      }
    }
  };

});
```
If you first start to depend on stores directly you quickly get into circular dependency issues. You might consider putting all your state in a common AppState object that only the stores will inject.

### Performance
Any $scopes listening to stores are removed when the $scope is destroyed. When it comes to cloning it only happens when you pull data out from a store. So an array of 10.000 items in the store is not a problem, because your application would probably not want to show all 10.000 items at any time. In this scenario your getter method probably does a filter, or a limit before returning the data.


License
-------

flux-angular is licensed under the [MIT license](LICENSE).

> The MIT License (MIT)
>
> Copyright (c) 2014 Christian Alfoni
>
> Permission is hereby granted, free of charge, to any person obtaining a copy
> of this software and associated documentation files (the "Software"), to deal
> in the Software without restriction, including without limitation the rights
> to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
> copies of the Software, and to permit persons to whom the Software is
> furnished to do so, subject to the following conditions:
>
> The above copyright notice and this permission notice shall be included in
> all copies or substantial portions of the Software.
>
> THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
> IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
> FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
> AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
> LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
> OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
> THE SOFTWARE.
