# Javscrtpt Async/Await 秒杀 Promise 的6个理由（教学向）

原著：[6 Reasons Why JavaScript's Async/Await Blows Promises Away (Tutorial)](https://hackernoon.com/6-reasons-why-javascripts-async-await-blows-Promises-away-tutorial-c7ec10518dd9)

作者：Mostafa Gaafar

译者：Tingrui Li

先说一件事，Node自从7.6版本之后已经原生支持async/await语法。如果你还未尝试过它，这里有一些例子告诉你为何你应当马上开始使用它，并不再回头。

## Async/await 101

对于那些没有听说过这个玩意儿的人，这里有一些快速扫盲：

- Async/await是编写异步代码新的方式，之前我们使用回调函数和Promise。
- Async/await实际上是基于Promise的。它不能与常规回调函数和node回调函数一起使用。
- Async/await跟Promise一样，不会阻塞程序。
- Async/await让异步代码看起来更像同步执行的。这就是它强大的地方所在。

## 语法

假设一个方法 `getJSON` 返回一个Promise，这个Promise解析(resolve)出一个JSON对象，我们想要调用他并且打印那个JSON，返回`"done"`。

你用Promise通常会这样实现：

```javascript
const makeRequest = () =>
  getJSON()
    .then(data => {
      console.log(data)
      return "done"
    })

makeRequest()
```

而如果你用async/await：

```javascript
const makeRequest = async () => {
  console.log(await getJSON())
  return "done"
}

makeRequest()
```

这里有一些区别：

1. 我们的方法前面有一个关键字`async`。而`await`关键字只能被用于以`async`定义的方法内部。**任何`async`方法都隐式地返回一个Promise，并且解析(resolve)的值是你从方法内`return`出来的东西(在这个例子中就是string `"done"`)。**

2. 上面第一点表示我们不能在外层是用`await`，因为我们只能在`async`方法中使用它。

```javascript
// 在外层不好使
// await makeRequest()

// 好使
makeRequest().then((result) => {
  // do something
})
```

1. `await getJSON()`表示console.log会等到`getJSON()`这个Promise resolve之后才会打印它的值。

## 为什么这样更好？

### 1\. 简洁、干净

瞧瞧我们少写了多少东西！就算是上面这么一个简短的例子，我们都明显少写了很多代码。不需要写`.then`，创建异步函数去处理response，或者对我们根本用不到的变量起`data`这个名字。我们也避免了把代码写成很多层。在下面的例子中，将会凸显这一点。

### 2\. 错误处理

Async/await让我们终于可以以同一种方法同时处理同步和异步错误：经典的`try/catch`。在下面这个Promise的例子中，`try/catch`不会处理`JSON.parse`错误，因为那是在Promise中发生的。我们得在Promise中去`.catch`，然后重新写一次我们的错误处理代码，这可能比你的生产环境代码中的那些`console.log`要冗余的多。

```javascript
const makeRequest = () => {
  try {
    getJSON()
      .then(result => {
        // JSON.parse可能会失败
        const data = JSON.parse(result)
        console.log(data)
      })
      // 处理JSON.parse可能抛出的错误
      .catch((err) => {
        console.log(err)
      })
  } catch (err) {
    console.log(err)
  }
}
```

现在我们用async/await写这段代码。`catch`代码块现在会处理JSON转换错误了。

```javascript
const makeRequest = async () => {
  try {
    // JSON.parse可能会失败
    const data = JSON.parse(await getJSON())
    console.log(data)
  } catch (err) {
    console.log(err)
  }
}
```

是不是好棒棒？

### 3\. 条件判断

想象一下这个场景，一段代码需要获取一些数据然后决定是返回这些数据，还是根据这些数据中的一些值来获取更多的数据。

```javascript
const makeRequest = () => {
  return getJSON()
    .then(data => {
      if (data.needsAnotherRequest) {
        return makeAnotherRequest(data)
          .then(moreData => {
            console.log(moreData)
            return moreData
          })
      } else {
        console.log(data)
        return data
      }
    })
}
```

是不是看着就头大？这六层的代码很容易让你头晕目眩。那些括号，以及return语句只是为了将最终结果传递到主要Promise中。

这个例子用async/await重写以后可读性大大提高。

```javascript
const makeRequest = async () => {
  const data = await getJSON()
  if (data.needsAnotherRequest) {
    const moreData = await makeAnotherRequest(data);
    console.log(moreData)
    return moreData
  } else {
    console.log(data)
    return data    
  }
}
```

### 4\. 中间值

你可能遇到这种情况：你需要调用`promise1`然后用它的返回值去调用`promise2`，然后用两者的返回值去调用`promise3`，你的代码很可能是这样的：

```javascript
const makeRequest = () => {
  return promise1()
    .then(value1 => {
      // do something
      return promise2(value1)
        .then(value2 => {
          // do something          
          return promise3(value1, value2)
        })
    })
}
```

如果`promise3`不需要`value1`，这个层级关系会清楚一些。如果你不能忍受，你可以用`promise.all`包装value1和value2：

```javascript
const makeRequest = () => {
  return promise1()
    .then(value1 => {
      // do something
      return Promise.all([value1, promise2(value1)])
    })
    .then(([value1, value2]) => {
      // do something          
      return promise3(value1, value2)
    })
}
```

这种写法牺牲了可读性，`value1`和`value2`不应当出于任何理由属于同一个数组。

同样的逻辑如果使用async/await来写会变得很简单，会让你怀疑以前为什么挣扎着让使用`Promise`的代码看起来更简单：

```javascript
const makeRequest = async () => {
  const value1 = await promise1()
  const value2 = await promise2(value1)
  return promise3(value1, value2)
}
```

### 5\. 错误栈

想像这样一个情景：连续链式调用多个`Promise`，然后其中某个地方可能会抛出异常：

```javascript
const makeRequest = () => {
  return callAPromise()
    .then(() => callAPromise())
    .then(() => callAPromise())
    .then(() => callAPromise())
    .then(() => callAPromise())
    .then(() => {
      throw new Error("oops");
    })
}

makeRequest()
  .catch(err => {
    console.log(err);
    // 输出
    // Error: oops at callAPromise.then.then.then.then.then (index.js:8:13)
  })
```

这个错误输出栈不能明确的指示错误发生在哪里。甚至会产生误导：整个错误只包含一个方法名`then`。

然而，async/await的错误栈会指向具体产生错误的方法：

```javascript
const makeRequest = async () => {
  await callAPromise()
  await callAPromise()
  await callAPromise()
  await callAPromise()
  await callAPromise()
  throw new Error("oops");
}

makeRequest()
  .catch(err => {
    console.log(err);
    // output
    // Error: oops at makeRequest (index.js:7:9)
  })
```

其实当你本地开发调试时，这一点意义不大。但是如果你需要在生产环境服务器上的代码上找错时，就非常实用了。在这种情况下，知道错误发生在`makeRequest`比知道错误发生在`then.then.then.then...`要好很多......

### 6\. 调试

最后，async/await一个巨大的优势是它非常容易调试。对`Promise`进行调试出于两个原因非常的痛苦：

1. 你不能在返回表达式的箭头函数上设置断点（没有函数体）。

  ```javascript
  const makeRequest = () =>{
  reutrn callAPromise()
  .then(()=>callAPromise())
  .then(()=>callAPromise())
  .then(()=>callAPromise())
  .then(()=>callAPromise())
  }
  ```

2. 如果你在一个`.then`块内设置了断点，然后用`step-over`等功能时，调试器不会进入`.then`代码因为他只会"step"进同步代码。

使用async/await你不需要那么多箭头函数，你可以直接"step"那些`await`调用，就像普通的同步调用一样。

```javascript
const makeRequest = async () =>{
  await callAPromise()
  await callAPromise()
  await callAPromise()
  await callAPromise()
  await callAPromise()
}
```

## 结论

Async/await是Javascript近几年来最具有革命性的特性之一，它让你体会到`Promise`是多么的混乱，然后给你一个方便的替代品。
