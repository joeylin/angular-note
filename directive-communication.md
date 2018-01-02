# 指令通信

***
## 指令与控制器的通信
通过收集整理，总结出了以下三种指令与控制器的通信方式：共享对象，服务，事件；每种方法都有自己的优缺点跟使用范围，大家可以斟酌使用。

### 共享对象
最简单的方法实现指令跟控制的双边绑定是通过使用"="进行作用域绑定，从而在两者之间设置共享对象。这样之后，控制器跟指令都可以通过watch函数在监听共享对象的改变，任何一处值的改变，都会引起另一处值的改变。

### 服务
另外一种方法是通过服务，跟共享对象的方法相比最大的区别是，服务是在一个模块内的单例对象，所有注入该服务的控制器实例和指令都会共享到这个单例对象。服务一个非常好的用处就是在应用的范围内实现所有指令实例的数据共享。以下例子中，创建一个服务对象，然后分别在控制器和指令中依赖注入，达到数据的共享。
```javascript
var mod = angular.module('MyModule', []);
mod.service('SettingsService', function() {
    return {
        'currency' : 'INR',
        'precision' : 2
    };
});
mod.controller('SettingsController', ['$scope', 'SettingsService', function($scope, settings) {
    $scope.settings = settings;
    
}]);
mod.directive('currencyDisplay', ['SettingsService', function(settings) {
    return {
        'restrict' : 'A',
        'scope' : {
            'amount' : '=currencyDisplay'
        },
        'template' : '<span>{{ settings.currency }} {{ amount | number:settings.precision }}</span>',
        'replace' : true,
        'link' : function(scope, element, attrs) {
            scope.settings = settings;
        }
    }
}]);
```
### 事件
```javascript
```
## 指令间的通信
require可以用在同一个元素上面，也可以用在父子元素上面，但是不可以用在兄弟之间。<foo bar></foo>  <foo><bar></bar></foo>


## 实例
指令间的通信
```javascript
app.directive("foobar", function() {
  return {
    restrict: "A",
    controller: function($scope) {
      $scope.trigger = function() {
        // do stuff
      };
    },
    link: function(scope, element) {
     // do more stuff
    }
  };
});
app.directive("bazqux", function() {
  return {
    restrict: "A",
    require: "foobar",
    link: function(scope, element, attrs, fooBarCtrl) {
        fooBarCtrl.trigger();
    }
  };
});
```
指令配合事件
```javascript
app.directive('directiveA', function($rootScope){
    return function(scope, element, attrs){
        $rootScope.$on('someEvent', function(){
            alert('Directive responds to a global event');
        });
    };
});
//html
<button ng-click="$emit('someEvent')">Click me!</button>
```
