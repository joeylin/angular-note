# $parse/$eval和$observe/$watch如何区分

大家在看angular的时候，有时候偶尔会看到$parse,$eval和$observe,$watch这两对语法，随着深入使用angular，就不可避免使用到它。
文章从内部运行机制跟实际需求的角度来解释这两对语法的区别。

## 原理
###$parse和$eval
首先，$parse跟$eval都是用来解析表达式的， 但是$parse是作为一个单独的服务存在的。$eval是作为scope的方法来使用的。
$parse典型的使用是放在设置字符串表达式映射在真实对象上的值。也可以从$parse上直接获取到表达式对应的值。

```javascript
var getter = $parse('user.name'); 
var setter = getter.assign; 
setter(scope, 'new name');
getter(context, locals) // 传入作用域，返回值
setter(scope,'new name') // 修改映射在scope上的属性的值为‘new value’
```
$eval 即scope.$eval，是执行当前作用域下的表达式，如：scope.$eval('a+b'); 而这个里的a,b是来自 scope = {a: 2, b:3};
看看源码它的实现是
```javascript
$eval: function(expr, locals) {
    return $parse(expr)(this, locals);
},
```
可以找到它也是基于$parse，不过它的参数已经被固定为this，就是当前的scope，所以$eval只是在$parse基础上的封装而已，是一种$parse快捷的API。
###$observe和$watch

$observe和$watch都可以用来对属性进行监控的，先来看下$observe源码：
```javascript
$observe: function(key, fn) {
	var attrs = this,
	    $$observers = (attrs.$$observers || (attrs.$$observers = {})),
	    listeners = ($$observers[key] || ($$observers[key] = []));

	listeners.push(fn);
	$rootScope.$evalAsync(function() {
	  if (!listeners.$$inter) {
	    // no one registered attribute interpolation function, so lets call it manually
	    fn(attrs[key]);
	  }
	});
	return fn;
}
```
从这里可以看出来它是使用了$rootScope.$evalAsync()方法来监控的。什么是$evalAsync呢？是一个异步解析的操作，是在其他表达式都已经解析之后再解析，这样使它拥有了处理像{{}}插值字符串的机会。

$observe是属性对象上的方法，因此它是用来监控DOM属性上的值的变化，它仅用在指令内部，当你需要在指令内部监控包含有插值表达式的DOM属性的时候，就要用到这个方法，

比如，<code>attr1="Name:{{name}}"</code>,然后在指令里面：<code>attrs.$observe('attr1', ....)</code>,但是假如你只用<code>scope.$watch(attrs.attr1,...)</code>,这种情况下是无效的，因为<code>{{}}</code>无法被解析，所以你得到的是<code>undefined</code>, 在其他情况下用$watch。

$watch更复杂一点，它可以监视表达式，这个表达式可以是函数或者字符串，假如表达式是字符串的话，会被封装成一个函数，然后在digest循环的时候被调用。 这个字符串表达式不能包含<code>{{}}</code>,$watch是一个scope对象上的方法，所以它可以在任何你可以访问到作用域的地方被调用。比如，控制器中或者link函数中。因为字符串是被当做angular的表达式解析的，所以$watch经常被用在当你想要监控一个模型或者作用域对象的时候，

比如:<code>attr1="myModel.some_prop"</code>,然后在控制器中或者link函数中<code>scope.$watch('myModel.some_prop',...)</code>或者<code>scope.$watch(attrs.attr1,...)</code>或者<code>scope.$watch(attrs['attr1'],...)</code>。假如你使用<code>attrs.$observe('attr1')</code>你就只能得到<code>myModel.some_prop</code>字符串的值。

$observe和$watch都会在每个digest阶段被执行。

带有隔离作用域的指令的情况会复杂点。假如'@'语法被使用的时候，你可以使用$observe或者$watch一个包含插值<code>{{}}</code>的DOM属性，之所以$watch在这里也能用，是因为'@'语法为我们处理了{{}}，因此传给$watch是的时候是没有{{}}的字符串。这样子我们使用起来就更容易了。但是这种情况还是推荐使用$observe。

想要更好的理解，大家可以看这个例子<a href="http://plnkr.co/edit/HBha8sVdeCqhJtQghGxw?p=preview">Plunker</a>。

注意到，当link函数被执行的时候，任何包含{{}}的DOM属性都还没被解析，所以此时假如你检查这种方式定义的属性值的话，就是得到undefined, 唯一的方法可以得到{{}}内部的真实值的是使用$observe，或者在包含带有@语法的隔离作用域中使用$watch.因此获取这些属性的值是异步操作(因为要{{}}内部的值是链接到其他地方，所以要等这些值稳定之后，才能得到属性的正确值，而在上文的源码中你可以看到使用了$rootScope.$evalAsync())。

有时候我们并不需要$observe和$watch，比如你的属性一个boolen类型或者数字类型的值，只要解析他们一次就够了,你可以这么做：<code>attr1="22"</code>, 同时你的链接函数: <code>var count = scope.$eval(attrs.attr1)</code>,假如只是包含字符串，<code>attr1="my string"</code>,你就可以直接使用<code>attrs.attr1</code>而不需要用$eval方法。

##使用场景

想要在沒有isolate scope的directive中取出foo的属性值，存在以下几种情况：

1. foo属性值是固定的字串值，例如想要傳 class name，id 等。
```javascript
<div foo="fadeOut"></div>
```
因为这种情况是直接給定固定字串值，可以直接在 foo directive 中的 link function 直接取出属性值，attrs.foo。


2. foo 属性值是非字串的值，例如：boolean, number 等。
```javascript
<div foo="true"></div>, <div foo="123"></div>
```
如果通过 attrs.foo 直接取值的话，就只能取到字符串值，但是我想要取值后就是正确的类型的話，可以使用 scope.$eval(attrs.foo)。

 
3. foo 属性是一個 scope property。
```javascript
angular.controller('BarCtrl', function($scope) {
  $scope.bar = {
    test: ''
  };
});
// html
<div ng-controller="BarCtrl">
  <div foo="bar.test"></div>
</div>
```
如果想要在 foo directive 中去改变bar.test的值的話，可以使用$parse來取值，$parse(attrs.foo)，$parse后会一個 function，之後你可以通过这个function的assign property 去设置 bar.test 的值：
```javascript
var model = $parse(attrs.foo);
model.assign(scope, 'Hello world');    // scope 为 link function 的 scope
```

4. foo 属性值是 interpolated attribute。
```javascript
<div foo="{{1+1}}"></div>, <div foo="{{bar}}"></div>
```
如果直接在 link function 直接使用 attrs.foo 取值的話，得到的是 undefined，因为此时 interpolated attribute 还没被解析，所以我們可以通过 $observe 來取值，attrs.$observe。
