# 控制器之间的通信

***

## 利用作用域的继承方式

由于作用域的继承是基于js的原型继承方式，所以这里分为两种情况，当作用域上面的值为基本类型的时候，修改父作用域上面的值会
影响到子作用域，反之，修改子作用域只会影响子作用域的值，不会影响父作用域上面的值；如果需要父作用域与子作用域共享一个值
的话，就需要用到后面一种，即作用域上的值为对象，任何一方的修改都能影响另一方，这是因为在js中对象都是引用类型。
    

基本类型
```javascript
function Sandcrawler($scope) {
    $scope.location = "Mos Eisley North";
    $scope.move = function(newLocation) {
        $scope.location = newLocation;
    }
}
function Droid($scope) {
    $scope.sell = function(newLocation) {
        $scope.location = newLocation;
    }
}
// html
<div ng-controller="Sandcrawler">
    <p>Location: {{location}}</p>
    <button ng-click="move('Mos Eisley South')">Move</button>
    <div ng-controller="Droid">
        <p>Location: {{location}}</p>
        <button ng-click="sell('Owen Farm')">Sell</button>
    </div>
</div>

```
对象
```javascript
function Sandcrawler($scope) {
    $scope.obj = {location:"Mos Eisley North"};
}
function Droid($scope) {
    $scope.summon = function(newLocation) {
        $scope.obj.location = newLocation;
    }
}
// html
<div ng-controller="Sandcrawler">
    <p>Sandcrawler Location: {{location}}</p>
    <div ng-controller="Droid">
        <button ng-click="summon('Owen Farm')">
            Summon Sandcrawler
        </button>
    </div>
</div>
```

## 基于事件的方式

在一般情况下基于继承的方式已经足够满足大部分情况了，但是这种方式没有实现兄弟控制器之间的通信方式，所以引出了事件的方式
。基于事件的方式中我们可以里面作用的$on,$emit,$boardcast这几个方式来实现，其中$on表示事件监听，$emit表示向父级以上的
作用域触发事件， $boardcast表示向子级以下的作用域广播事件。参照以下代码：

向上传播事件
```javascript
function Sandcrawler($scope) {
    $scope.location = "Mos Eisley North";
    $scope.$on('summon', function(e, newLocation) {
        $scope.location = newLocation;
    });
}
function Droid($scope) {
    $scope.location = "Owen Farm";
    $scope.summon = function() {
        $scope.$emit('summon', $scope.location);
    }
}
// html
<div ng-controller="Sandcrawler">
    <p>Sandcrawler Location: {{location}}</p>
    <div ng-controller="Droid">
        <p>Droid Location: {{location}}</p>
        <button ng-click="summon()">Summon Sandcrawler</button>
    </div>
</div>
```
向下广播事件
```javascript
function Sandcrawler($scope) {
    $scope.location = "Mos Eisley North";
    $scope.recall = function() {
        $scope.$broadcast('recall', $scope.location);
    }
}
function Droid($scope) {
    $scope.location = "Owen Farm";
    $scope.$on('recall', function(e, newLocation) {
        $scope.location = newLocation;
    });
}
//html
<div ng-controller="Sandcrawler">
    <p>Sandcrawler Location: {{location}}</p>
    <button ng-click="recall()">Recall Droids</button>
    <div ng-controller="Droid">
        <p>Droid Location: {{location}}</p>
    </div>
</div>

```

从这个用法我们可以引申出一种用于兄弟控制间进行通信的方法，首先我们一个兄弟控制中向父作用域触发一个事件，然后在父作用域
中监听事件，再广播给子作用域，这样通过事件携带的参数，实现了数据经过父作用域，在兄弟作用域之间传播。这里要注意的是，通过父元素作为中介进行传递的话，兄弟元素用的事件名不能一样，否则会进入死循环。请看代码：

-兄弟作用域之间传播
```javascript
function Sandcrawler($scope) {
    $scope.$on('requestDroidRecall', function(e) {
        $scope.$broadcast('executeDroidRecall');
    });
}
function Droid($scope) {
    $scope.location = "Owen Farm";
    $scope.recallAllDroids = function() {
        $scope.$emit('requestDroidRecall');
    }
    $scope.$on('executeDroidRecall', function() { 
        $scope.location = "Sandcrawler"
    });
}
// html
<div ng-controller="Sandcrawler">
    <div ng-controller="Droid">
        <h2>R2-D2</h2>
        <p>Droid Location: {{location}}</p>
        <button ng-click="recallAddDroids()">Recall All Droids</button>
    </div>
    <div ng-controller="Droid">
        <h2>C-3PO</h2>
        <p>Droid Location: {{status}}</p>
        <button ng-click="recallAddDroids()">Recall All Droids</button>
    </div>
</div>
```

## angular服务的方式
在ng中服务是一个单例，所以在服务中生成一个对象，该对象就可以利用依赖注入的方式在所有的控制器中共享。参照以下例子，在一个控制器修改了服务对象的值，在另一个控制器中获取到修改后的值：
```javascript
var app = angular.module('myApp', []);
app.factory('instance', function(){
    return {};
});
app.controller('MainCtrl', function($scope, instance) {
  $scope.change = function() {
       instance.name = $scope.test;
  };
});
app.controller('sideCtrl', function($scope, instance) {
    $scope.add = function() {
        $scope.name = instance.name;
    };
});
//html
<div ng-controller="MainCtrl">
     <input type="text" ng-model="test" />
     <div ng-click="change()">click me</div>
</div>
<div ng-controller="sideCtrl">
    <div ng-click="add()">my name {{name}}</div>
</div>
```


