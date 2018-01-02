# 创建自定义指令
这是一篇译文，来自angular开发者说明的指令。主要面向已经熟悉angular开发基础的开发者。这篇文档解释了什么情况下需要创建自己的指令，和如何去创建指令。
***
## 什么是指令
从一个高的层面来讲，指令是angular $compile服务的说明，当特定的标签(属性，元素名，或者注释) 出现在DOM中的时候，它让编译器附加指定的行为到DOM上。

这个过程是很简单的。angular内部有很用这样自带的指令，比如说ngBind, ngView,就像你创建控制器和服务一样，你也可以创建自己的指令。当angular启动的时候，Angular的编译器分析html去匹配指令，这允许指令注册行为或者改变DOM。

### 匹配指令
在写指令之前，我们首先需要知道的是angular是如何匹配到一个指令的，在以下的例子我们说input元素匹配到ngModel指令.
```javascript
<input ng-model="foo">
```
下面这种方法同样也会匹配到ngModel：
```javascript
<input data-ng:model="foo">
```
Angular会规范化一个元素的标签名和属性名，以便确定哪一个元素匹配到哪一个指令。我们在js中是通过使用规范化后的驼峰式的名字来引用指令(比如ngModel)。在HTML中常常使用'-'划定的属性名字来调用指令(比如说ng-model).

规范化的处理过程：
-去掉元素或属性上面的x-和data-的前缀
-转化':','-'和‘_-’形式的命名为驼峰式拼写

以下例子展示了用不同的方式匹配到ngBind指令
```javascript
<span ng-bind="name"></span> <br/>
<span ng:bind="name"></span> <br/>
<span ng_bind="name"></span> <br/>
<span data-ng-bind="name"></span> <br/>
<span x-ng-bind="name"></span> <br/>
```
Best Practice: 优先使用'-'格式的命名(比如说ng-bind匹配ngBind)。如果你想在HTML验证工具中通过，你可以用'data-'前缀的方式(比如data-ng-bind)。其他格式的命名是因为历史遗留的原因存在，避免使用它们。

$compile服务可以基于元素的名字，属性名，类名，和注释来匹配指令

所有Angular内部提供的指令都匹配属性名，标签名，注释，或者类名。以下不同的方式都可以被解析到
```javascript
<my-dir></my-dir>
<span my-dir="exp"></span>
<!-- directive: my-dir exp -->
<span class="my-dir: exp;"></span>
```
Best Practice: 优先利用标签名和属性名的方式使用指令。这样子更容易理解指定的元素匹配到了哪个元素。

Best Practice: 注释的方式通常被用在DOM API限制创建跨越多个元素的指令，比如说table元素，限制重复嵌套，这样就要用注释的方式。在AngularJS 1.2版本中，通过使用ng-repeat-start 和 ng-repeat-end 作为一个更好的方案来解决这个问题。在可能的情况下，推荐使用这种方式。

#### 文本和属性的绑定
在编译过程中，编译器会使用$interpolate服务来检测匹配到的文本和属性值是否包含内嵌表达式。这些表达式被注册为watches，可以在digest循环时被更新。
```javascript
<a ng-href="img/{{username}}.jpg">Hello {{username}}!</a>
```
#### ngAttr属性的绑定
浏览器有些时候会对它认为合法的属性值非常的挑剔(就是某些元素的属性是不可以任意赋值的，否则会报错)。

比如：
```javascript
<svg>
  <circle cx="{{cx}}"></circle>
</svg>
```
使用这样的写法时，我们会发现控制台中报错<code>Error: Invalid value for attribute cx="{{cx}}". </code>.这是由于SVG DOM API的限制，你不能简单的写为<code>cx="{{cx}}"</code>.

使用ng-attr-cx 可以解决这个问题

如果一个绑定的属性使用ngAttr前缀(或者ng-attr), 那么在绑定的时候将会被应用到相应的未前缀化的属性，这种方式允许你绑定到需要马上被浏览器处理的属性上面(比如SVG元素的circle[cx]属性)。

所以，我们可以这样写来修复以上的问题:
```javascript
<svg>
  <circle ng-attr-cx="{{cx}}"></circle>
</svg>
```

## 创建指令

首先我们来谈论下注册指令的API，跟controller一样，指令是注册在module上，不同的是，指令是通过module.directive API来注册的。module.directive接受的是一个规范化的名字和工厂函数，这个工厂函数返回一个包含不同配置的对象，这个对象用来告诉$compile服务如何进行下一步处理。

工厂函数仅在编译器第一次匹配到指令的时候调用一次。通常在工厂函数中执行初始化的工作。该函数使用$injector.invoke调用，所以它可以像controller一样进行依赖注入。

Best Practice: 优先返回一个定义好的对象，而不是返回一个函数。

接下来，我们先会了解一些常见的例子，然后再深入了解不同的配置项的原理和编译过程。

Best Practice: 为了避免与某些未来的标准命名冲突，最好前缀化你自己的指令，比如你创建一个<carousel>指令，它可能会产生冲突，加入HTML7引入相同的元素。推荐使用两三个单词的前缀(比如btfCarousel)，同样不能使用ng或者其他可能与angular未来版本起冲突的前缀。

以下的例子中，我们统一使用my前缀。

### 模板扩展的指令
当你有大量代表客户信息的模板。这个模板在你的代码中重复了很多次，当你改变一个地方的时候，你不得不在其他地方同时改动，这时候，你就要使用指令来简化你的模板。

我们来创建一个指令，简单的时候静态模板来替换它的内容。

```javascript
<div ng-controller="Ctrl">
     <div my-customer></div>
 </div>
// JS
angular.module('docsSimpleDirective', [])
  .controller('Ctrl', function($scope) {
    $scope.customer = {
      name: 'Naomi',
      address: '1600 Amphitheatre'
    };
  })
  .directive('myCustomer', function() {
    return {
      template: 'Name: {{customer.name}} Address: {{customer.address}}'
    };
  });
```
注意我们这里做了一些绑定，$compile编译很链接<div my-customer> </div>之后，它将会匹配子元素的指令。这意味着你可以组合一些指令。以下例子中你会看到如何做到这一点。

这个例子中，我们直接在template配置项里写上模板，但是随着模板大小的增加，这样非常不优雅。

Best Practice: 除非你的模板非常小，否则更好的是分割成单独的hmtl文件，然后使用templateUrl选项来加载。

加入你熟悉ngInclude，templateUrl跟它非常类似。现在我们使用templateUrl方式重写上面的例子:

 ```javascript
<div ng-controller="Ctrl">
     <div my-customer></div>
 </div>
// JS
angular.module('docsTemplateUrlDirective', [])
  .controller('Ctrl', function($scope) {
    $scope.customer = {
      name: 'Naomi',
      address: '1600 Amphitheatre'
    };
  })
  .directive('myCustomer', function() {
    return {
      templateUrl: 'my-customer.html'
    };
  });
// my-customer.html
Name: {{customer.name}} Address: {{customer.address}}
```
非常好，但是如果我们想让我们的指令匹配标签名<my-customer>？ 如果我们只是简单的把<my-customer>元素放在hmtl上面，会发现没有效果。

Note: 创建指令的时候，默认仅使用属性的方式。为了创建一个能由元素名字触发的指令，你需要用到restrict配置。

restrict配置可以按如下方式设置:
-'A' 仅匹配属性名字
-'E' 仅匹配元素名字
-'AE' 可以匹配到属性名字或者元素名

所以，我们可以使用 <code>restrict: 'E'</code>配置我们指令。

 ```javascript
<div ng-controller="Ctrl">
     <div my-customer></div>
 </div>
// JS
angular.module('docsTemplateUrlDirective', [])
  .controller('Ctrl', function($scope) {
    $scope.customer = {
      name: 'Naomi',
      address: '1600 Amphitheatre'
    };
  })
  .directive('myCustomer', function() {
    return {
      restrict: 'E',
      templateUrl: 'my-customer.html'
    };
  });
// my-customer.html
Name: {{customer.name}} Address: {{customer.address}}
```
点击查看更多关于<a>restrictJ</a>信息

Note: 什么时候使用属性名或元素名呢？ 当创建一个含有自己模板的组件的时候，需要使用元素名，如果仅仅是为已有的元素添加功能的话，使用属性名。

使用元素名做为myCustomer指令是非常正确的决定，因为你不是用一些'customer'行为来点缀元素，而是定义一个具有自己行为的元素作为customer组件。

###隔离指令的作用域
上面我们的myCustomer指令已经非常好了，但是它有个致命的缺陷，我们在给定的作用域内仅能使用一次。

它现在的实现是，我们每次重用该指令的时候都要为它新创一个控制器。

 ```javascript
<div ng-controller="NaomiCtrl">
  <my-customer></my-customer>
</div>
<hr>
<div ng-controller="IgorCtrl">
   <my-customer></my-customer>
</div>
// JS
angular.module('docsScopeProblemExample', [])
  .controller('NaomiCtrl', function($scope) {
    $scope.customer = {
      name: 'Naomi',
      address: '1600 Amphitheatre'
    };
  })
  .controller('IgorCtrl', function($scope) {
    $scope.customer = {
      name: 'Igor',
      address: '123 Somewhere'
    };
  })
  .directive('myCustomer', function() {
    return {
      restrict: 'E',
      templateUrl: 'my-customer.html'
    };
  });
// my-customer.html
Name: {{customer.name}} Address: {{customer.address}}
```

这很明显不是一个好的解决方案。

我们想要做的是能够把指令的作用域与外部的作用域隔离开来，然后映射外部的作用域到指令内部的作用域。可以通过创建isolate scope来完成这个目的。这样的话，我们使用指令的scope配置。

 ```javascript
<div ng-controller="Ctrl">
   <my-customer customer="naomi"></my-customer>
   <hr>
   <my-customer customer="igor"></my-customer>
</div>
// JS
angular.module('docsIsolateScopeDirective', [])
  .controller('Ctrl', function($scope) {
    $scope.naomi = { name: 'Naomi', address: '1600 Amphitheatre' };
    $scope.igor = { name: 'Igor', address: '123 Somewhere' };
  })
  .directive('myCustomer', function() {
    return {
      restrict: 'E',
      scope: {
        customer: '=customer'
      },
      templateUrl: 'my-customer.html'
    };
  });
// my-customer.html
Name: {{customer.name}} Address: {{customer.address}}
```
首先看hmtl，第一个<my-customer>绑定内部作用域的customer到naomi。这个naomi我们在控制器中已经定义好了。第二个是绑定customer到igor。

现在看看scope是如何配置的。
 ```javascript
//...
scope: {
  customer: '=customer'
},
//...
```
属性名(customer)是myCustomer指令上isolated scope的变量名。它的值(=customer)告诉编译器绑定到customer属性。

Note: 指令作用域配置中的'=attr'属性名是被规范化过后的名字，比如要绑定<div bind-to-this="thing">中的属性，你就要使用'=bindToThis'的绑定。

对于属性名和你想要绑定的值的名字一样，你可以使用这样的快捷语法:
 ```javascript
//...
scope: {
  // same as '=customer'
  customer: '='
},
//...
```
使用isolated scope还有另外一个用处，那就是可以绑定不同的数据到指令内部的作用域。
在我们的例子中，我们可以添加另外一个属性vojta到我们的作用域，然后在我们的指令模板中访问它。

 ```javascript
<div ng-controller="Ctrl">
      <my-customer customer="naomi"></my-customer>
</div>
//JS
angular.module('docsIsolationExample', [])
  .controller('Ctrl', function($scope) {
    $scope.naomi = { name: 'Naomi', address: '1600 Amphitheatre' };
 
    $scope.vojta = { name: 'Vojta', address: '3456 Somewhere Else' };
  })
  .directive('myCustomer', function() {
    return {
      restrict: 'E',
      scope: {
        customer: '=customer'
      },
      templateUrl: 'my-customer-plus-vojta.html'
    };
  });
//my-customer-plus-vojta.html
Name: {{customer.name}} Address: {{customer.address}}
<hr>
Name: {{vojta.name}} Address: {{vojta.address}}
```
注意到，{{vojta.name}}和{{vojta.address}}都是空的，意味着他们是undefined, 虽然我们在控制器中定义了vojta，但是在指令内部访问不到。

就像它的名字暗示的一样， 指令的isolate scope隔离了除了你添加到作用域:{}对象中的数据模型外的一切东西。这对于你要建立一个可重复使用的组件是非常有用的，因为它阻止了除了你想要传入的数据模型外其他东西改变你数据模型的状态。

Note: 正常情况下，作用域是原型继承自父作用域。但是isolate scope没有这样的继承。

Best Practice: 当你想要使你的组件在应用范围内可重用，那么使用scope配置去创建一个isolate scopes

###创建一个操作DOM的指令
在这个例子中，我们会创建一个显示当前时间的指令，每秒一次更新DOM以正确的显示当前的时间。

指令修改DOM通常是在link配置中，link选项接受一个带有如下标签的函数<code>function link(scope,element,attrs) {...}</code>其中：
-<code>scope</code>是angular scope对象
-<code>element</code>指令匹配的jqLite封装的元素(angular内部实现的类jquery的库)
-<code>attrs</code>是一个带有规范化后属性名字和相应值的对象

在我们的link函数中，我们更新显示时间每秒一次，或者当用户改变指定绑定的时间格式字符串的时候。我们也要移除定时器，当指令被删除的时候，以避免引入内存泄露。

```javascript
<div ng-controller="Ctrl2">
    Date format: <input ng-model="format"> <hr/>
    Current time is: <span my-current-time="format"></span>
</div>
//JS
angular.module('docsTimeDirective', [])
  .controller('Ctrl2', function($scope) {
    $scope.format = 'M/d/yy h:mm:ss a';
  })
  .directive('myCurrentTime', function($timeout, dateFilter) {
 
    function link(scope, element, attrs) {
      var format,
          timeoutId;
 
      function updateTime() {
        element.text(dateFilter(new Date(), format));
      }
 
      scope.$watch(attrs.myCurrentTime, function(value) {
        format = value;
        updateTime();
      });
 
      function scheduleUpdate() {
        // save the timeoutId for canceling
        timeoutId = $timeout(function() {
          updateTime(); // update DOM
          scheduleUpdate(); // schedule the next update
        }, 1000);
      }
 
      element.on('$destroy', function() {
        $timeout.cancel(timeoutId);
      });
 
      // start the UI update process.
      scheduleUpdate();
    }
 
    return {
      link: link
    };
  });
```

这里有很多东西值得注意的，就像module.controller API, module.directive中函数参数是依赖注入，因此，我们可以在Link函数内部使用$timeout和dataFilter服务。

我们注册了一个事件<code>element.on('$destroy', ...)</code>, 是什么触发了这个<code>$destory</code>事件呢？

AngularJS会触发一些特定的事件，当一个被angular编译过的DOM元素被移除的时候，它会触发一个<code>$destory</code>事件，同样的，当一个angular作用域被移除的时候，它会向下广播<code>$destory</code>事件到所有监听的作用域。

通过监听事件，你可以移除可能引起内存泄露的事件监听器，注册在元素和作用域上的监听器在它们被移除的时候，会自动会清理掉，但是假如注册一个事件在服务或者没有被删除的DOM节点上，你就必须手工清理，否则会有内存泄露的风险。

Best Practice:执行被移除的时候应该做一些清理的操作， 可以使用<code>element.on('$destroy', ...)</code>或者<code>scope.on('$destroy', ...)</code>来运行解除绑定的函数，

### 创建包裹其他元素的指令
我们现在已经实现了，使用isolate scopes传递数据模型到指令里面。但是有时候我们需要能够传递一整个模板而不是字符串或者对象。让我们通过创建'dialog box'组件来说明。这个'dialog box'组件应该能够包裹任意内容。

要实现这个，我们使用transclude配置

 ```javascript
<div ng-controller="Ctrl">
    <my-dialog>Check out the contents, {{name}}!</my-dialog>
</div>
//JS
angular.module('docsTransclusionDirective', [])
  .controller('Ctrl', function($scope) {
    $scope.name = 'Tobias';
  })
  .directive('myDialog', function() {
    return {
      restrict: 'E',
      transclude: true,
      templateUrl: 'my-dialog.html'
    };
  });
//my-dialog.html
<div class="alert" ng-transclude>
</div>
```
这个transclude配置用来干嘛呢？ transclude使带有这个配置的指令的内容能够访问指令外部的作用域。

参照以下例子，注意到我们增加了一个link函数，在这个link函数内部我们重定义了name属性的值为Jeff，那么现在这个{{name}}会被解析成哪个值呢？

 ```javascript
<div ng-controller="Ctrl">
    <my-dialog>Check out the contents, {{name}}!</my-dialog>
</div>
//JS
angular.module('docsTransclusionDirective', [])
  .controller('Ctrl', function($scope) {
    $scope.name = 'Tobias';
  })
  .directive('myDialog', function() {
    return {
      restrict: 'E',
      transclude: true,
      templateUrl: 'my-dialog.html',
      link: function (element, scope) {
        scope.name = 'Jeff';
      }
    };
  });
//my-dialog.html
<div class="alert" ng-transclude>
</div>
``` 
一般，我们会认为{{name}}会被解析为Jeff，然而这里，我们看到这个例子中的{{name}}还是被解析成了Tobias.

transclude配置改变了指令相互嵌套的方式，他使指令的内容拥有任何指令外部的作用域，而不是内部的作用域。为了实现这个，它给指令内容一次访问外部作用域的机会。

这样的行为对于包裹内容的指令是非常有意义的。因为如果不这样的话，你就必须分别传入每个你需要使用的数据模型。如果你需要传入每个要使用的数据模型，那么你就无法做到适应各种不同内容的情况。

Best Practice: 仅当你要创建一个包裹任意内容的指令的时候使用transclude:true。

下一步，我们增加一个按钮到'dialog box'组件里面，允许用户使用指令绑定自己定义的行为。

```javascript
<div ng-controller="Ctrl">
      <my-dialog ng-hide="dialogIsHidden" on-close="dialogIsHidden = true">
        Check out the contents, {{name}}!
      </my-dialog>
</div>
//JS
angular.module('docsIsoFnBindExample', [])
  .controller('Ctrl', function($scope, $timeout) {
    $scope.name = 'Tobias';
    $scope.hideDialog = function () {
      $scope.dialogIsHidden = true;
      $timeout(function () {
        $scope.dialogIsHidden = false;
      }, 2000);
    };
  })
  .directive('myDialog', function() {
    return {
      restrict: 'E',
      transclude: true,
      scope: {
        'close': '&onClose'
      },
      templateUrl: 'my-dialog-close.html'
    };
  });
//my-dialog-close.html
<div class="alert">
  <a href class="close" ng-click="close()">&times;</a>
  <div ng-transclude></div>
</div>
``` 
我们想要通过在指令的作用域上调用，来运行我们传递进去的函数，但是这个函数是运行在定义时候的上下文(js通常都是这样子的)。

先前我们看到如何scope配置使用'=prop'，但是在上文的例子中，我们使用'&prop'，'&'绑定开放了一个函数到isolated scope，允许 isolated scope调用它，同时维持原来函数的作用域(这里的作用域都是指$scope)。所以当一个用户点击x时候，就会运行Ctrl控制器的close函数。

Best Practice: 当你的指令想要开放一个API去绑定特定的行为，在scope配置中使用'&prop'.

###创建一个添加事件监听器的指令
先前，我们使用link函数创建一个操作DOM元素的指令，基于上面的例子，我们创建一个在元素上添加事件监听的指令。

举个例子，假如我们想要创建一个让用户可拖拽的元素，该怎么做呢？

```javascript
<span my-draggable>Drag ME</span>
//JS
angular.module('dragModule', []).
  directive('myDraggable', function($document) {
    return function(scope, element, attr) {
      var startX = 0, startY = 0, x = 0, y = 0;
 
      element.css({
       position: 'relative',
       border: '1px solid red',
       backgroundColor: 'lightgrey',
       cursor: 'pointer'
      });
 
      element.on('mousedown', function(event) {
        // Prevent default dragging of selected content
        event.preventDefault();
        startX = event.screenX - x;
        startY = event.screenY - y;
        $document.on('mousemove', mousemove);
        $document.on('mouseup', mouseup);
      });
 
      function mousemove(event) {
        y = event.screenY - startY;
        x = event.screenX - startX;
        element.css({
          top: y + 'px',
          left:  x + 'px'
        });
      }
 
      function mouseup() {
        $document.unbind('mousemove', mousemove);
        $document.unbind('mouseup', mouseup);
      }
    }
  });
``` 

### 创建相互通信的指令
你可以通过在模板使用指令来组合任何指令。
有时候，你想要一个指令从其他的指令上面创建
想象你想要一个带有tab的容器，容器的内容对应于激活的tab。

```javascript
 <my-tabs>
      <my-pane title="Hello">
        <h5 id="creating-custom-directives_source_hello">Hello</h5>
        <p>Lorem ipsum dolor sit amet</p>
      </my-pane>
      <my-pane title="World">
        <h5 id="creating-custom-directives_source_world">World</h5>
        <em>Mauris elementum elementum enim at suscipit.</em>
        <p><a href ng-click="i = i + 1">counter: {{i || 0}}</a></p>
      </my-pane>
 </my-tabs>
//JS
angular.module('docsTabsExample', [])
  .directive('myTabs', function() {
    return {
      restrict: 'E',
      transclude: true,
      scope: {},
      controller: function($scope) {
        var panes = $scope.panes = [];
 
        $scope.select = function(pane) {
          angular.forEach(panes, function(pane) {
            pane.selected = false;
          });
          pane.selected = true;
        };
 
        this.addPane = function(pane) {
          if (panes.length == 0) {
            $scope.select(pane);
          }
          panes.push(pane);
        };
      },
      templateUrl: 'my-tabs.html'
    };
  })
  .directive('myPane', function() {
    return {
      require: '^myTabs',
      restrict: 'E',
      transclude: true,
      scope: {
        title: '@'
      },
      link: function(scope, element, attrs, tabsCtrl) {
        tabsCtrl.addPane(scope);
      },
      templateUrl: 'my-pane.html'
    };
  });
//my-tabs.html
<div class="tabbable">
  <ul class="nav nav-tabs">
    <li ng-repeat="pane in panes" ng-class="{active:pane.selected}">
      <a href="" ng-click="select(pane)">{{pane.title}}</a>
    </li>
  </ul>
  <div class="tab-content" ng-transclude></div>
</div>
//my-pane.html
<div class="tab-pane" ng-show="selected" ng-transclude>
</div>
``` 
myPane指令有一个require:'^myTabs'的配置，当指令使用这个配置，$compile服务叫myTabs的指令并获取它的控制器实例，如果没有找到，将会抛出一个错误。'^'前缀意味着指令在它的父元素上面搜索控制器(没有'^'前缀的话，指令默认会在自身的元素上面搜索指定的指令)。

这里myTabs的控制器是来自何处呢？通过使用controller配置可以为指令指定一个控制器, 上问例子中myTab就是使用这个配置。就像ngController, 这个配置为指令的模板绑定了一个控制器。

再看我们的myPane's定义，注意到link函数的最后一个参数: tabCtrl，当一个指令包含另一个指令(通过require方式)，它会接收该指令的控制器实例作为link函数的第四个参数，利用这个，myPane可以调用myTabs的addPane函数。

精明的读者可能想知道link跟controller之间的区别，最基本的区别就是控制器开放一个API(就是这个控制器实例可以被其他实例读取到)，link函数可以通过require与控制器交互。

Best Practice: 当你要开放一个API给其他指令的时候使用控制器，否则使用link函数。

### 总结
这里我们讲解了一个些指令的主要使用案例。每一个都可以作为你创建自己指令的很好的起点。

如果你想更深入的了解编译的处理过程，可以查看<a>compiler guide</a>

<a>$compile API</a>页面有directive每个配置项的具体解释，可以参阅。


