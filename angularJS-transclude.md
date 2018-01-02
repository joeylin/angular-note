# angular中的transclude

***

Transclude - 在Angular的指令中，大家会看到有一个这样的一个配置属性，这个单词在英文字典里面也查询不到真实的意思，所以就用英文来标示它吧。如果你深入的使用angular的话，你就花很大一部分时间来创建自定义指令，那么就不可避免的要深入理解transclude。简单的讲，transclude主要完成以下工作，取出自定义指令中的内容(就是写在指令里面的子元素)，以正确的作用域解析它,然后再放回指令模板中标记的位置(通常是ng-transclude标记的地方)，虽然使用内建的ngTransclude对于基本的transclude操作已经足够简单，但是在文档中对这个transclude的解释还是有存在很多疑惑，比如说：
* 在compile函数中接收到了一个叫transclude的参数是什么东西呢？有什么用呢？
* 在控制器中也有个叫$transclude的可以通过依赖注入的服务，这又是什么呢？
* 隔离作用域跟transclude有什么关系？
* 属性的transclude操作

接下来我们将一个个的解释:

## 基本的transclude

我们通过一个基本的transclude例子来讲解吧，我们现在要创建的是一个叫buttonBar的指令，用户可以通过它来添加一组button到页面上，这个指令会对不同的button进行位置的排列。以下例子css样式是使用Bootstrap框架。

<a href="http://jsfiddle.net/ospatil/A969Z/157/">在fiddle中查看例子</a>

```javascript
<div ng-controller="parentController">    
    <button-bar>
        <button class="primary" ng-click="onPrimary1Click()">{{primary1Label}}</button>
        <button class="primary">Primary2</button>
    </button-bar>
</div>
// JS
var testapp = angular.module('testapp', []);
testapp.controller('parentController', ['$scope', '$window', function($scope, $window) {
    console.log('parentController scope id = ', $scope.$id);
    $scope.primary1Label = 'Prime1';
    
    $scope.onPrimary1Click = function() {
        $window.alert('Primary1 clicked');    
    };
}]);
testapp.directive('primary', function() {
    return {
        restrict: 'C',
        link: function(scope, element, attrs) {
            element.addClass('btn btn-primary');
        }
    }
});
testapp.directive('buttonBar', function() {
    return {
        restrict: 'EA',
        template: '<div class="span4 well clearfix"><div class="pull-right" ng-transclude></div></div>',
        replace: true,
        transclude: true
    };
});
```

我们先看下HTML标签,buttonBar指令包裹着几个button元素。而button元素也被链接上了基于class的primary指令，不要太在意这个primary指令的功能它只不过为button元素添加一些css的样式而已。现在我们来看buttonBar指令，它提供了一个<code>transclude:true</code>属性，同时在它的模板里面使用<code>ng-transclude</code>指令。在运行的过程中，Angular获取到自定义指令的内容，处理完了之后把结果放到了模板中链接上<code>ng-transclude</code>的div。

## transclude到多个位置

现在我们来增强下我们的buttonBar指令的功能，我们增加了两种按钮，primary和secondary,其中primary按钮是排右边，secondary是排左边。所以要做到这个功能，它必须能够取出指令的内容，然后把它们分别添加到不同的div中，一个用来放primary按钮, 一个用来放secondary按钮。

这样的话，默认的机制已经满足不了我们的要求，于是我们有了另外一种方法：
* 设置transclude为true  
* 手工移动button元素到合适的div
* 最后，在指令的编译或链接函数中移除原始的用来transclude操作的元素

这种方法就是先把所有的内容插入到<code>ng-transclude</code>标记的元素中，然后在link函数中再找出元素的插入的元素，重新放到元素的其他地方，最后删除原来暂存内容的元素。

<a href="http://jsfiddle.net/ospatil/A969Z/158/">在fiddle中查看例子</a>

```javascript
<div ng-controller="parentController">    
    <button-bar>
        <button class="primary" ng-click="onPrimary1Click()">{{primary1Label}}</button>
        <button class="primary">Primary2</button>
        <button class="secondary">Secondary1</button>
    </button-bar>
</div>

// JS
var testapp = angular.module('testapp', []);
testapp.controller('parentController', ['$scope', '$window',function($scope, $window) {
    $scope.primary1Label = 'Prime1';
    $scope.onPrimary1Click = function() {
        $window.alert('Primary 1 clicked');
    }        
}]);
testapp.directive('primary', function() {
    return {
        restrict: 'C',
        link: function(scope, element, attrs) {
            element.addClass('btn btn-primary');
        }
    }
});  
testapp.directive('secondary', function() {
    return {
        restrict: 'C',
        link: function(scope, element, attrs) {
            element.addClass('btn');
        }
    }
});    
testapp.directive('buttonBar', function() {
    return {
        restrict: 'EA',
        template: '<div class="span4 well clearfix"><div class="primary-block pull-right"></div><div class="secondary-block"></div><div class="transcluded" ng-transclude></div></div>',
        replace: true,
        transclude: true,
        link: function(scope, element, attrs) {
            var primaryBlock = element.find('div.primary-block');
            var secondaryBlock = element.find('div.secondary-block');
            var transcludedBlock = element.find('div.transcluded');
            var transcludedButtons = transcludedBlock.children().filter(':button');
            angular.forEach(transcludedButtons, function(elem) {
                if (angular.element(elem).hasClass('primary')) {
                    primaryBlock.append(elem);
                } else if (angular.element(elem).hasClass('secondary')) {
                    secondaryBlock.append(elem);
                }
            });
            transcludedBlock.remove();
        }
    };
});
```
虽然这种方法达到了我们的目的，但是允许默认的transclude操作，然后再人工的从DOM元素中移出不是非常有效率的。因此，我们有了compile函数中的transclude参数和控制器中的$transclude服务

##编译函数参数中的transclude

开发者指南中给了我们以下的关于指令中编译函数的形式：
<code>function compile(tElement, tAttrs, transclude) { ... }</code>
其中关于第三个参数transclude的解释是:
<code>transclude - A transclude linking function: function(scope, cloneLinkingFn).</code>

好的，现在我们利用这个函数来实现我们刚才讲到的功能，从而不需要再先暂存内容，然后再插入到其他地方。

<a href="http://jsfiddle.net/ospatil/A969Z/161/">在fiddle中查看例子</a>

```javascript
<div ng-controller="parentController">    
    <button-bar>
        <button class="primary" ng-click="onPrimary1Click()">{{primary1Label}}</button>
        <button class="primary">Primary2</button>
        <button class="secondary">Secondary1</button>
    </button-bar>
</div>

// JS
var testapp = angular.module('testapp', []);
testapp.controller('parentController', ['$scope', '$window', function($scope, $window) {
    $scope.primary1Label = 'Prime1';   
    $scope.onPrimary1Click = function() {
        $window.alert('Primary 1 clicked');                
    }
}]);
testapp.directive('primary', function() {
    return {
        restrict: 'C',
        link: function(scope, element, attrs) {
            element.addClass('btn btn-primary');
        }
    }
});
testapp.directive('secondary', function() {
    return {
        restrict: 'C',
        link: function(scope, element, attrs) {
            element.addClass('btn');
        }
    }
});
testapp.directive('buttonBar', function() {
    return {
        restrict: 'EA',
        template: '<div class="span4 well clearfix"><div class="primary-block pull-right"></div><div class="secondary-block"></div></div>',
        replace: true,
        transclude: true,
        compile: function(elem, attrs, transcludeFn) {
            return function (scope, element, attrs) {
                transcludeFn(scope, function(clone) {
                    var primaryBlock = elem.find('div.primary-block');
                    var secondaryBlock = elem.find('div.secondary-block');
                    var transcludedButtons = clone.filter(':button'); 
                    angular.forEach(transcludedButtons, function(e) {
                        if (angular.element(e).hasClass('primary')) {
                            primaryBlock.append(e);
                        } else if (angular.element(e).hasClass('secondary')) {
                            secondaryBlock.append(e);
                        }
                    });
                });
            };
        }
    };
});
```
注意到，transcludeFn函数需要一个可用的scope作为第一个参数，但是编译函数中没有可用的scope，所以这里需要在链接函数中执行transcludeFn。这种方法实际上是在link函数中同时操作编译后的DOM元素和模板元素(主要是因为transcludeFn函数中保存着指令的内容)。

## 可在控制器中注入的$transclude服务

在开发者指南中对$transclude服务是这么解释的：
<code>$transclude - A transclude linking function pre-bound to the correct transclusion scope: function(cloneLinkingFn).</code>
看看如何用在我们的例子中：

<a href="http://jsfiddle.net/ospatil/A969Z/162/">在fiddle中查看例子</a>

```javascript
<div ng-controller="parentController">    
    <button-bar>
        <button class="primary" ng-click="onPrimary1Click()">{{primary1Label}}</button>
        <button class="primary">Primary2</button>
        <button class="secondary">Secondary1</button>
    </button-bar>
</div>

// JS
var testapp = angular.module('testapp', []);
testapp.controller('parentController', ['$scope', '$window', function($scope, $window) {
    $scope.onPrimary1Click = function() {
        alert('Primary1 clicked');    
    };
    $scope.primary1Label = "Prime1"
}]);
testapp.directive('primary', function() {
    return {
        restrict: 'C',
        link: function(scope, element, attrs) {
            element.addClass('btn btn-primary');
        }
    }
});
testapp.directive('secondary', function() {
    return {
        restrict: 'C',
        link: function(scope, element, attrs) {
            element.addClass('btn');
        }
    }
});
testapp.directive('buttonBar', function() {
    return {
        restrict: 'EA',
        template: '<div class="span4 well clearfix"><div class="primary-block pull-right"></div><div class="secondary-block"></div></div>',
        replace: true,
        transclude: true,
        scope: {},
        controller: ['$scope', '$element', '$transclude', function ($scope, $element, $transclude) {
            $transclude(function(clone) {
                var primaryBlock = $element.find('div.primary-block');
                var secondaryBlock = $element.find('div.secondary-block');
                var transcludedButtons = clone.filter(':button'); 
                angular.forEach(transcludedButtons, function(e) {
                    if (angular.element(e).hasClass('primary')) {
                        primaryBlock.append(e);
                    } else if (angular.element(e).hasClass('secondary')) {
                        secondaryBlock.append(e);
                    }
                });
            });
        }],
    };
});
```
同样的意思，$transclude中接收的函数里的参数含有指令元素的内容，而$element包含编译后的DOM元素，所以就可以在控制器中同时操作DOM元素和指令内容，跟上文的compile函数的实现方式有异曲同工之处，这里有几点需要注意，这个控制器应该是指令的控制器，另一个注意到上文除了第一种方法，其他的地方都没有用到
<code>ng-transclude</code>，因为无需插入到模板中。

## Transclude 和 scope

在开发者指南中提到了<code>a directive isolated scope and transclude scope are siblings</code>,这到底是什么意思呢？假如你认真看前文的例子的话，你就会发现parentController控制器创建了一个作用域，buttonBar指令在parentController下面创建了一个孤立作用域，而根据Angular文档，transclude也创建了另外一个作用域，因此指令的隔离作用域跟transclude作用域是基于同一个父作用域的兄弟作用域。

##transclude内容放入元素的属性

实际上，你不可以这么做，但是你可以通过一种变通的方法来实现这种效果

<a href="http://stackoverflow.com/questions/11703086/how-can-i-transclude-into-an-attribute/11704489#11704489">查看完整的说明</a>

```javascript
var testapp = angular.module('testapp', [])

testapp.directive('tag', function() {
  return {
    restrict: 'E',
    template: '<h1><a href="{{transcluded_content}}">{{transcluded_content}}</a></h1>',
    replace: true,
    transclude: true,
    compile: function compile(tElement, tAttrs, transclude) {
        return {
            pre: function(scope) {
                transclude(scope, function(clone) {
                  scope.transcluded_content = clone[0].textContent;
                });
            }
        }
    }
  }
});​
```
这里没有操作DOM元素，只是把元素的文本内容复制给了作用域属性，然后在通过作用域传给属性。
另外要注意的是，这里的clone参数是jquery或angular.element封装的整个模板元素。



