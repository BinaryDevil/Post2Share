# Apollo Mobile Framework With MobX
这是一个将[MobX](https://mobx.js.org)与现有apollo-mobile开发框架结合的Demo。MobX是一个负责状态(state)管理的JS库，并且有针对react的版本[mobx-react](https://github.com/mobxjs/mobx-react)。

## 安装&运行
```
$ npm install
$ npm run compileScript
$ npm start
```
---

# 什么是MobX
## 核心概念
状态(state)是每个应用程序的核心，如果出现对于state管理不当、state前后不一致的情况，很容易导致程序频繁出现BUG、无法有效管理。所以很多state管理方案尝试限制对state的更改，比如让state变成只读。

MobX采取更简单直接的方法：让state前后不一致的情况不可能发生。达成这点的策略也很简单：让所有能够从state中产生的项(可以是计算出来的值、生成的UI或是任何其他结果)，都能够完全自动地被产生。

## mobx-react
[mobx-react](https://github.com/mobxjs/mobx-react)是针对MobX针对React推出的解决方案。同时支持React和React Native。

---

# 结合React与MobX
## React组件生命周期
我们知道在React组件（即继承React.Component的那些）中，每次改变组件的state，即调用this.setState()之时，该组件会调用一次Render()以更新UI。因此我们需要常常更新state以完成UI更新的生命周期。

举一个简单的例子：
```javascript
class NameDisplayer extends React.Component{
    constructor(){
        super();
        this.setState({
            name: 'Alex Lee'
        });
    }
    render(){
        return (
            <p>Name is {{this.state.name}}</p>
        )
    }
}
```

当我们需要更新组件显示的名字时，就需要用this.setState()去改变state中name的值，就会自动更新到UI上。

但在比较复杂的界面里，大量对state的更新可能会难以管理，导致state失去一致性，维护起来也比较困难。

## 使用MobX：`@observer`和`@observable`修饰符
我们需要引入最基本的两个模块：

```javascript
import {observable} from 'mobx';
import {observer} from 'mobx-react';
```

mobx提供了一个@observable修饰符，mobx-react提供了一个@observer修饰符。被@observable修饰的变量（这个变量不必要在组件的state中）改变时，会被@observer（通常就是这个组件class本身）跟踪到，并自动调用render()方法。

举一个最简单的例子：
```javascript
@observer
class NameDisplayer extends React.Component{
    @observable name = 'Alex Lee'
    render(){
        return (
            <p>Name is {{this.name}}</p>
        )
    }
}
```

任何对this.name变量的更改，都会被组件本身追踪到，并自动调用render()更新到UI上去。

## 使用MobX：Provider与`@inject`
由于React官方文档中不建议使用context进行属性传递（具体原因参考[React Docs - Context](https://facebook.github.io/react/docs/context.html#passing-info-automatically-through-a-tree)），但是目前apollo-mobile使用context给每个组件demo页面传递了setTitleBar方法，用于改变标题栏：
```javascript
  static contextTypes = {
    setTitleBar: React.PropTypes.func,
    router: React.PropTypes.any
  }

  componentWillMount() {
    const router = this.context.router;
    this.context.setTitleBar({
      title: 'SegmentTab',
      back: true,
      leftTap(){
        router.goBack();
      }
    });
  }
```
mobx-react提供了Provider组件和@inject修饰符，专门用于向下传递属性和值，从而避免使用context，提供了一种更优的解决方案。

首先，在我们的上层组件（此处为App/index.js）Render方法中最外层加入Provider组件：
```javascript
<Provider setTitleBar={(arg) => {
        this.setState({
          header: arg
        })}}>
        <Drawer className="demo-app-drawer" sidebar={this.getSidebarView()} {...drawerProps}>

          ......

        </Drawer>
 </Provider>
```

之后在需要使用setTitleBar方法的组件中通过@inject注入该方法，就可以为之使用：
```javascript
@inject("setTitleBar") @observer
export default class TextareaDemo extends PureComponent {

  ......

  componentWillMount() {
    const router = this.context.router;
    this.props.setTitleBar({
      title: 'Textarea',
      back: true,
      leftTap(){
        router.goBack();
      }
    });
  }

  ......

}
```
## 可选：引入mobx-react-devtools
[mobx-react-devtools](https://github.com/mobxjs/mobx-react-devtools)是针对mobx-react的开发者工具，可以跟踪mobx工作情况、render耗费时间等，使用方法是：
```javascript
import DevTools from 'mobx-react-devtools';
```
然后在Render的UI中任何一处加入`<DevTools/>`。

# apollo-mobile集成MobX
本Demo选取了apollo-mobile中部分组件，按照上述方法实现MobX集成。

出现在本Demo界面中的所有元素都已经集成了MobX，并且开启了DevTools。

---

# 进行中：与IE8的兼容测试
代码中包含的ie8-demo文件夹，是正在进行的MobX与IE8兼容性解决方案。包含单独的package.json，是一个单独的项目。
```javascript
$ npm install
$ npm start
```
