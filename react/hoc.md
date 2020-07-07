# 深入理解React高阶组件

> [关于高阶组件的官方文档](https://react.docschina.org/docs/higher-order-components.html)
>
> **高阶组件是参数为组件，返回值为新组件的函数。**--React 官网

## 高阶组件可以做什么

总结起来指两件事：

- 属性代理：高阶组件操控传递给 WrappedComponent 的 props

- 反向继承：高阶组件继承（extends）WrappedComponent

展开来讲也就是：

- 代码复用，逻辑抽象
- 渲染劫持
- State 抽象和更改
- Props 更改

## Props Proxy（props代理）

```javascript
function ppHOC(WrappedComponent) {
  return class PP extends React.Component {
    render() {
      return <WrappedComponent {...this.props}/>
    }
  }
}

//foo.js
class Foo extends Component{}
export default ppHOC(Foo)
//或者
@ppHOC  //注意这里必须是一个class组件
class Foo extends Component{}

//index.js
import Foo from './foo.js'
<Foo {...}>
```

可以看到，这里高阶组件的 render 方法返回了一个 type 为 WrappedComponent 的组件，我们把高阶组件PP收到的 props 传递给它，因此得名 **Props Proxy**。

### 更改props

可以『**读取，添加，修改，删除**』将要传递给 WrappedComponent 的 props

```javascript
function ppHOC(WrappedComponent) {
  return class PP extends React.Component {
    render() {
      const newProps = {
        user: currentLoggedInUser
      }
      return <WrappedComponent {...this.props} {...newProps}/>
    }
  }
}
```

例子：添加新的props至WrappedComponent

### 通过 refs 获取组件实例

可以通过ref获取this（WrappedComponent 的实例），必须在一次正常的渲染后才能获得

```javascript
function refsHOC(WrappedComponent) {
  return class RefsHOC extends React.Component {
    ref=(wrappedComponentInstance)=> {
      wrappedComponentInstance.method()
    }
    render() {
      return <WrappedComponent {...props} ref={this.ref}/>
    }
  }
}
```

例子：获取ref，并以props传入WrappedComponent

### 抽象state

抽象state也就是常说的公用逻辑提取，参考官网上的例子，也就是将公共的逻辑和数据提取至HOC中，将原本的state提取至HOC，并以props的形式传入WrappedComponent

```javascript
function withSubscription(WrappedComponent, selectData) {
  return class extends React.Component {
      state = {
        data: selectData(DataSource, this.props)
      };

    componentDidMount() {
      // ...负责订阅相关的操作...
      // DataSource为一个全局变量
      DataSource.addChangeListener(this.handleChange);
    }

    componentWillUnmount() {
      DataSource.removeChangeListener(this.handleChange);
    }

    handleChange=()=> {
      this.setState({
        data: selectData(DataSource, this.props)
      });
    }

    render() {
      return <WrappedComponent data={this.state.data} {...this.props} />;
    }
  };
}
```

上面是官网中的例子，原本WrappedComponent中的state.data、监听方法、被监听执行事件都被抽象到HOC中来

## Inheritance Inversion（反向继承）

```javascript
function iiHOC(WrappedComponent) {
  return class Enhancer extends WrappedComponent {
    render() {
      return super.render()
    }
  }
}
```

上面是一个反向继承的例子，⚠️注意现在的HOC不再继承Component，而是继承`WrappedComponent`

这里的HOC组件`Enhancer`可以通过this获取到WrappedComponent中的state、props、生命周期函数、render，注意别忘记调用` super.[lifecycleHook]`，防止原本的WrappedComponent被破坏

P.S.因为如果在HOC中未调用生命周期函数，React将在原型链中去寻找生命周期函数并执行，所以如果HOC定义了生命周期函数，比如componentDidMount，那么将不再执行WrappedComponent中的componentDidMount，所以这里将super.componentDidMount()写入HOC中的componentDidMount则可以在不破坏原本生命周期的基础上进行扩展

>这里不是很懂暂时直接引用原文过来
>
>React Element 可以是两个种类其中的一种：String 或 Function。String 类型的 React Element 代表原声 DOM 节点，Function 类型的 React Element 代表通过 React.Component 创建的组件。想要了解更多关于 Elements 和 Components 的知识请阅读此[推文](https://link.jianshu.com?t=https://facebook.github.io/react/blog/2015/12/18/react-components-elements-and-instances.html)。
>
>Function 类型的 React Element 将在 reconciliation 阶段被解析成 DOM 类型的 React Element （最终结果一定都是 DOM 元素）。
>
>**反向继承的高阶组件不保证一定解析整个子元素树**

### 渲染劫持

```javascript
function iiHOC(WrappedComponent) {
  return class Enhancer extends WrappedComponent {
    render() {
      if (this.props.loggedIn) {
        return super.render()
      } else {
        return null
      }
    }
  }
}
```

上面是一个渲染劫持的例子，通过条件来选择是否渲染

```javascript
import React from 'react';
const MyContainer = (WrappedComponent) => {
    return class extends WrappedComponent {
        render() {
            const elementsTree = super.render();
            let newProps = {};
            if (elementsTree && elementsTree.type === 'input') {
                newProps = {value: 'may the force be with you'};
            }
            const props = Object.assign({}, elementsTree.props, newProps);
            const newElementsTree = React.cloneElement(elementsTree, props, elementsTree.props.children);
            return newElementsTree;
        }
    }
}

export default MyContainer;
```

上面是通过执行super.render()，来获取WrappedComponent中的render返回结果，最后再用过clone的方式重新创建组件Tree

通过继承的方式来实现后，下面是生命周期调用顺序，

```
didmount -> HOC didmount -> (HOCs didmount) -> will unmount -> HOC will unmount -> (HOCs will unmount)
```

### 操作state

```javascript
export function IIHOCDEBUGGER(WrappedComponent) {
  return class II extends WrappedComponent {
    render() {
      return (
        <div>
          <h2>HOC Debugger Component</h2>
          <p>Props</p> <pre>{JSON.stringify(this.props, null, 2)}</pre>
          <p>State</p><pre>{JSON.stringify(this.state, null, 2)}</pre>
          {super.render()}
        </div>
      )
    }
  }
}
```

通过this直接操作WrappedComponent中的state

## 与父组件渲染的区别

这里指的父组件渲染，实际就是常用的这种方式

```javascript
<div>this.props.children</div>
//或者
<div>this.props.children()</div>
```

- 渲染劫持
- 操作内部props
- 抽象state
- 包装React Element，这里通过render props的方式会比HOC方便很多
- Children的操控，HOC的方式会保证组件root单一，render props则不能保证

通常来讲，能使用父组件达到效果的，就不要使用高阶组件，但更多的灵活场景则还是需要用到HOC

