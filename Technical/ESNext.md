# ES2016及ESNext语法前瞻
ECMAScript（以下简称ES）是Javascript的标准，每一个版本迭代都在改变着Javascript的编写方法，目前使用广泛的最新版本是ES2015（又称ES6）。然而ECMAScript一直都在变化，只是每隔几年才会制定出一个标准被各大浏览器支持普及起来。ESNext即是下一阶段ECMAScript的形态，它可能每隔一周就会改变，但是有一些语法最终可能会保留下来，并被包含在一两年后Javascript编写的标准中。

本文将通过截取ESNext目前一些具有代表性的新语法，提前预告一下近几年Javascript语言规范、新语言特性的走向。

## ECMAScript简述
ECMA是"European Computer Manufacturers Association"的简称，最初的目标是制定一个脚本语言规范。Javascript后来成为了这个规范最知名的实现，ECMAScript也自然而然的成为Javascript的标准。

顺便提一下，当我们讨论一个前端JS框架的浏览器兼容性，实质上通常就是讨论该框架是否使用了一些老浏览器不支持的新的ECMAScript标准包含的特性。当然，polyfill可以解决部分这种问题。

### ES2016/ESNext是什么
ESNext是下一阶段ECMAScript的内容，这其中可能包含很多未确定保留的语法特性，所有可能被纳入新标准的语法特性都算在ESNext的范畴内。
ES2016，因为并没有正式发版，其实也算在ESNext里头。只是人们通常说到ES2016是指那些已经比较成熟、普遍被认可、甚至已经投入使用的语法特性。

### Proposal与Stage
这两个概念代表着一个Javascript语法功能的诞生与通过流程。每一个向标准制定委员会委员会提出的新语法被称为Proposal，然后其会经过5个Stage：

- Stage 0：Strawman，允许进入标准规范
- Stage 1：Proposal，描述解决方案，指出潜在挑战
- Stage 2：Draft，拟执行的实施方案，草稿
- Stage 3：Candidate，可行性较高，需要参考更多用户反馈
- Stage 4：Finished，已经准备纳入下一个ECMAScript标准

## 部分具有代表性语法/特性介绍

### Array.prototype.includes (Stage 4)
在很长一段时间内，javascript中判断一个元素是否在Array中的方法是：
```javascript
if(array.indexOf(element) > 0) //或者 !==-1 等等
{
  ...
}
```
该新语法在Array的原型中加入includes方法，能够为代码编写带来便利（讲真这个东西早就该有了好嘛）。
```javascript
if(array.includes(element))
{
  ...
}
```

### 求幂操作符 (Stage 3)
在其他语言中早先就有这个操作符：
- Python
  - `math.pow(x, y)`
  - `x ** y`
- CoffeeScript
  - `x ** y`
- F#
  - `x ** y`
- Ruby
  - `x ** y`
- Perl
  - `x ** y`
- Lua, Basic, MATLAB, etc.
  - `x ^ y`

现在，Javascript也可以有了：
```javascript
// x ** y

let squared = 2 ** 2;
// same as: 2 * 2

let cubed = 2 ** 3;
// same as: 2 * 2 * 2

```

```javascript
// x **= y

let a = 2;
a **= 2;
// same as: a = a * a;



let b = 3;
b **= 3;
// same as: b = b * b * b;

```

### Object.values / Object.entries (Stage 4)
目前Javascript已经有Object.keys这个方法可以返回一个object中的所有key，这个提议加入了Object.values和Object.entries分别返回一个object中的所有value，以及entry（即key-value pair）。

### Observable构造器（Stage 1）
这个提议引入Observable类型进入ECMAScript标准库。该类型可以用于对push-based数据源进行建模，比如DOM事件，计时器interval，以及socket操作。可以实现事件监听代码的重用、组件化、模块化。

例：

针对DOM元素和事件创建了一个observable事件流：
```Javascript
function listen(element, eventName) {
    return new Observable(observer => {
        // Create an event handler which sends data to the sink
        let handler = event => observer.next(event);

        // Attach the event handler
        element.addEventListener(eventName, handler, true);

        // Return a cleanup function which will cancel the event stream
        return () => {
            // Detach the event handler from the element
            element.removeEventListener(eventName, handler, true);
        };
    });
}
```
之后可以使用标准组合器（也就是一个函数）过滤和映射事件，就像对array操作一样：
```Javascript
// Return an observable of special key down commands
function commandKeys(element) {
    let keyCommands = { "38": "up", "40": "down" };

    return listen(element, "keydown")
        .filter(event => event.keyCode in keyCommands)
        .map(event => keyCommands[event.keyCode])
}
```
最后，当我们在任何地方需要监听以上定义的事件流时，我们直接订阅observer就可以了：
```Javascript
let subscription = commandKeys(inputElement).subscribe({
    next(val) { console.log("Received key command: " + val) },
    error(err) { console.log("Received an error: " + err) },
    complete() { console.log("Stream complete") },
});
```
我们的inputElement此时会监测到所有上箭头键或者下箭头键的按键操作。以及以后所有需要监听这两个键的DOM元素都只需要调用前面的方法subscribe一下，就完成了。

当然，有subscribe就有unsubscribe。就像email订阅一样：
```Javascript
// After calling this function, no more events will be sent
subscription.unsubscribe();
```

### Rest/Spread属性（Stage 3）
- Rest属性：该属性会自己选择object内还没有占用的key。
```Javascript
let { x, y, ...z } = { x: 1, y: 2, a: 3, b: 4 };
x; // 1
y; // 2
z; // { a: 3, b: 4 }
```

- Spread属性：从提供的object拷贝枚举属性进入新建的object。
```Javascript
let n = { x, y, ...z };
n; // { x: 1, y: 2, a: 3, b: 4 }
```

### 类和属性装饰器（Decorators）（Stage 2）
知名state管理工具MobX正在使用该特性：
```Javascript
@frozen class Foo {
  @configurable(false) @enumerable(true) method() {}
}
```
装饰器可以装饰类和成员属性，这是一个语法糖，可以大大简化代码，提高可阅读性。装饰器实际上就是一个函数，可以对成员变量或者类的实例执行一些操作（在函数里面定义）。

## 结语
最近两年前端技术变化飞快，半年一迭代，一年换一代。移动互联百花齐放的今天，大家都在寻找最佳实践，同时，前端作为内容展示的最直接的一部分，是对用户体验影响最直接的一块内容。

所以在对于飞速发展的前端技术，要保持不断的学习。

共勉。

有关更多ESNext的新特性的进度，可以参考[这个Repo](https://github.com/tc39/ecma262)。
