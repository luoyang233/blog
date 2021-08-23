## 实现一个react

> 参考自https://pomb.us/build-your-own-react/

## 开始

一段基本的react代码

```jsx
const element = <h1 title="foo">Hello</h1>
const container = document.getElementById("root")
ReactDOM.render(element, container)
```

上面有第一句和第三句是我们需要自己实现的部分



## 实现`ReactDOM.createElement()`

```jsx
const element = <h1 title="foo">Hello</h1>
```

等同于

```js
const element = React.createElement(
  "h1",
  { title: "foo" },
  "Hello"
)

// =============> 最后会转换成

const element = {
  type: "h1",
  props: {
    title: "foo",
    children: "Hello",
  },
}
```

jsx如何转换成`React.createElement`的事情暂时不用管了

我们需要实现一个自己的`MyReact.createElement`

一个常见的嵌套结构

```jsx
const element = (
  <div id="foo">
    <a>bar</a>
    <b />
  </div>
)

// ====================> 会被这样调用

const element = React.createElement(
  "div",
  { id: "foo" },
  React.createElement("a", null, "bar"),
  React.createElement("b")
)
```

所以可以这样实现（文字类型需要单独照顾一下）

```jsx
function createElement(type, props, ...children) {
  return {
    type,
    props: {
      ...props,
      children: children.map(child =>
        typeof child === "object"
          ? child
          : createTextElement(child)
      ),
    },
  }
}

function createTextElement(text) {
  return {
    type: "TEXT_ELEMENT",
    props: {
      nodeValue: text,
      children: [],
    },
  }
}

const MyReact = {
  createElement
}
```

用我们自己的`createElement`

```js
/** @jsx MyReact.createElement */
const MyReact = {
  createElement
}
```

### 总结

`ReactDOM.createElement()`其实就是获取一个用**对象表示的react结构**

```js
{
  type,
  props: {
    ...props,
    children: []
  }
}
```



## 实现`ReactDOM.render()`

```js
ReactDOM.render(element, container)
```

换成我们自己的

```js
function render(element, container) {
  // TODO create dom nodes
}

const MyReact = {
  createElement,
  render
}

MyReact.render(element, container)
```

实现render

同样文字单独处理一下，前面已经给了文字一个`nodeValue: text`的prop，所以可以统一赋予标签属性

```js
function render(element, container) {
  // 1.创建dom
  const dom =
    element.type == "TEXT_ELEMENT"
      ? document.createTextNode("")
      : document.createElement(element.type)

  // 2.赋予属性，同时children属性做特殊处理
  const isProperty = key => key !== "children"
  Object.keys(element.props)
    .filter(isProperty)
    .forEach(name => {
      dom[name] = element.props[name]
    })

  // 3.递归添加dom
  element.props.children.forEach(child =>
    render(child, dom)
  )

  container.appendChild(dom)
}
```

### 总结

`ReactDom.render()`做的事情就是一句话概括：**创建dom，挂载dom**

创建的过程就是，通过前面转换成的react对象结构，生成dom节点，并通过递归一个一个的挂上到父节点上去

### 现在的全貌

```jsx
function createElement(type, props, ...children) {
  return {
    type,
    props: {
      ...props,
      children: children.map(child =>
        typeof child === "object" ? child : createTextElement(child)
      )
    }
  };
}

function createTextElement(text) {
  return {
    type: "TEXT_ELEMENT",
    props: {
      nodeValue: text,
      children: []
    }
  };
}

function render(element, container) {
  const dom =
    element.type == "TEXT_ELEMENT"
      ? document.createTextNode("")
      : document.createElement(element.type);
  const isProperty = key => key !== "children";
  Object.keys(element.props)
    .filter(isProperty)
    .forEach(name => {
      dom[name] = element.props[name];
    });
  element.props.children.forEach(child => render(child, dom));
  container.appendChild(dom);
}

const MyReact = {
  createElement,
  render
};

/** @jsx MyReact.createElement */
const element = (
  <div style="background: salmon">
    <h1>Hello World</h1>
    <h2 style="text-align:right">from MyReact</h2>
  </div>
);
const container = document.getElementById("root");
MyReact.render(element, container);

```



## Concurrent Mode

> 引用原文的一段话
>
> Once we start rendering, we won’t stop until we have rendered the complete element tree. If the element tree is big, it may block the main thread for too long. And if the browser needs to do high priority stuff like handling user input or keeping an animation smooth, it will have to wait until the render finishes.

### `requestIdleCallback`

> MDN上的一段解释
>
> **`window.requestIdleCallback()`**方法将在浏览器的空闲时段内调用的函数排队。这使开发者能够在主事件循环上执行后台和低优先级工作，而不会影响延迟关键事件，如动画和输入响应。

浏览器会在闲暇时不断调用workLoop

```js
function workLoop() {
  requestIdleCallback(workLoop)
}

requestIdleCallback(workLoop)
```

我们需要实现一个随时可中断的机制，所以dom的创建不再一次性完成

当执行一次dom的创建后，应当返回下一个需要创建的dom，并判断能否进入下一次创建

```js
let nextUnitOfWork = null

// deadline参数是浏览器给出的剩余空闲时间
function workLoop(deadline) {
  let shouldYield = false
  // 初始的nextUnitOfWork从哪里来后面会说
  while (nextUnitOfWork && !shouldYield) {
    nextUnitOfWork = performUnitOfWork(
      nextUnitOfWork
    )
    shouldYield = deadline.timeRemaining() < 1
  }
  requestIdleCallback(workLoop)
}

requestIdleCallback(workLoop)

function performUnitOfWork(nextUnitOfWork) {
  // 这里要做的事情将是react的重点 fibers
  // TODO
}
```

## Fibers

上面的`nextUnitOfWork`实际就是一个fiber，也叫一个工作单元

![image-20210820190633696](https://github.com/luoyang233/blog/blob/master/images/image-20210820190633696.png)

`performUnitOfWork()`要做的事情有

- 创建一个真实dom并添加到父节点
- 创建所有子元素fiber
- 返回下一个工作单元(fiber)，优先级为child > sibling > uncle(parent.sibling) > uncle.sibling

改造一下`render()`，之前的render做的事情是创建并挂载dom，前面说了浏览器会在空闲时不停的调用`workLoop()`，所以render现在要做的应该是触发`performUnitOfWork()`的执行

```js
// 省略了不相关代码
// 所以能看出 performUnitOfWork 的执行条件是有一个初始的工作单元fiber
function workLoop(deadline) {
  while (nextUnitOfWork) {
    nextUnitOfWork = performUnitOfWork(
      nextUnitOfWork
    )
  }
}
```

```js
function render(element, container) {
  nextUnitOfWork = {
    dom: container,
    props: {
      children: [element]
    }
  }
}

// 这是一个全局的参数
let nextUnitOfWork = null
```

回顾一下`performUnitOfWork()`要做的第一件事

- 创建一个真实dom并添加到父节点

```js
function createDom(fiber) {
  const dom =
    fiber.type == "TEXT_ELEMENT"
      ? document.createTextNode("")
      : document.createElement(fiber.type)

  const isProperty = key => key !== "children"
  Object.keys(fiber.props)
    .filter(isProperty)
    .forEach(name => {
      dom[name] = fiber.props[name]
    })

  return dom
}

function performUnitOfWork(fiber) {
  // 1.创建一个真实dom并添加到父节点
  if (!fiber.dom) {
    fiber.dom = createDom(fiber)
  }

  if (fiber.parent) {
    fiber.parent.dom.appendChild(fiber.dom)
  }

  // TODO.2 创建所有子元素fiber
  // TODO.3 返回下一个工作单元
}
```

`performUnitOfWork()`要做的第二件事

- 创建所有子元素fiber

```js
function performUnitOfWork(fiber) {
  // 1.创建一个真实dom并添加到父节点
  // 。。

  // 2.创建所有子元素fiber
  // children可能是一个数组 所以每个都需要创建fiber
  const elements = fiber.props.children;
  let index = 0;
  let prevSibling = null;

  while (index < elements.length) {
    const element = elements[index];

    const newFiber = {
      type: element.type,
      props: element.props,
      parent: fiber,
      dom: null,
    };

    // 链接子fiber与子fiber.sibling
    if (index === 0) {
      fiber.child = newFiber;
    } else {
      prevSibling.sibling = newFiber;
    }

    prevSibling = newFiber;
    index++;
  }
   // TODO.3 返回下一个工作单元
}
```

`performUnitOfWork()`要做的第三件事

- 返回下一个工作单元

```js
function performUnitOfWork(fiber) {
  // 1.创建一个真实dom并添加到父节点
  // 。。

  // 2.创建所有子元素fiber
  // 。。

  // 3.返回下一个工作单元
  // 优先级 child > sibling
  if (fiber.child) {
    return fiber.child;
  }
  let nextFiber = fiber;
  while (nextFiber) {
    if (nextFiber.sibling) {
      return nextFiber.sibling;
    }
    nextFiber = nextFiber.parent;
  }
}
```

`performUnitOfWork()`全貌

```js
function performUnitOfWork(fiber) {
  // 1.创建一个真实dom并添加到父节点
  if (!fiber.dom) {
    fiber.dom = createDom(fiber)
  }

  if (fiber.parent) {
    fiber.parent.dom.appendChild(fiber.dom)
  }

  // 2.创建所有子元素fiber
  const elements = fiber.props.children
  let index = 0
  let prevSibling = null

  while (index < elements.length) {
    const element = elements[index]

    const newFiber = {
      type: element.type,
      props: element.props,
      parent: fiber,
      dom: null,
    }

    if (index === 0) {
      fiber.child = newFiber
    } else {
      prevSibling.sibling = newFiber
    }

    prevSibling = newFiber
    index++
  }

  // 3.返回下一个工作单元
  if (fiber.child) {
    return fiber.child
  }
  let nextFiber = fiber
  while (nextFiber) {
    if (nextFiber.sibling) {
      return nextFiber.sibling
    }
    nextFiber = nextFiber.parent
  }
}
```

回顾一下第一件事

```js
function performUnitOfWork(fiber) {
  // 1.创建一个真实dom并添加到父节点
  if (!fiber.dom) {
    fiber.dom = createDom(fiber)
  }

  // 这段需要移除
  if (fiber.parent) {
    fiber.parent.dom.appendChild(fiber.dom)
  }
}
```

这里有个问题，一开始就把dom挂载到了根节点上，我们的`workloop`可能在未完成所有元素的创建前就被中断，导致浏览器渲染不完整的dom

所以我们要在整颗fiber tree创建完成后一次性执行挂载的动作，在这之前，只需要把创建的真实dom保存在fiber中

如何判断fiber tree创建完成？**没有下一个`nextUnitOfWork`**!

```js
function workLoop(deadline) {
  let shouldYield = false
  while (nextUnitOfWork && !shouldYield) {
    nextUnitOfWork = performUnitOfWork(
      nextUnitOfWork
    )
    shouldYield = deadline.timeRemaining() < 1
  }
	// nextUnitOfWork是一个一直在变的全局变量
  // 所以我们需要保存一下根fiber wipRoot，以便执行提交
  if (!nextUnitOfWork && wipRoot) {
    commitRoot()
  }

  requestIdleCallback(workLoop)
}

function render(element, container) {
  wipRoot = {
    dom: container,
    props: {
      children: [element],
    },
  }
  nextUnitOfWork = wipRoot
}

let nextUnitOfWork = null
let wipRoot = null
```

执行提交

```js
function commitRoot() {
  commitWork(wipRoot.child)
  wipRoot = null
  // 这句可以省略 因为前面最后已经返回null赋值了
  // nextUnitOfWork = null
}

function commitWork(fiber) {
  if (!fiber) {
    return
  }
  const domParent = fiber.parent.dom
  domParent.appendChild(fiber.dom)
  commitWork(fiber.child)
  commitWork(fiber.sibling)
}
```



## 协调

> 就是常说的diff过程
>
> 如果没有协调，按照前面的思路，我们每次render都是从头开始创建dom并一层层提交，这样的效率显然是不合格的
>
> 所以有了reconcile过程，我们只需要对原有的dom节点做增删改即可

我们需要和上一颗fiber tree对比，所以需要储存上一颗fiber tree

```js
function commitRoot() {
  commitWork(wipRoot.child)
  // 提交时储存该fiber tree
  currentRoot = wipRoot
  wipRoot = null
}

function render(element, container) {
  wipRoot = {
    dom: container,
    props: {
      children: [element],
    },
    // 渲染时绑定old fiber	
    alternate: currentRoot,
  }
  nextUnitOfWork = wipRoot
}

// 新增一个全局变量
let currentRoot = null
```

再次回顾一下`performUnitOfWork()`

```js
function performUnitOfWork(fiber) {
  // 1.创建dom
  // 2.创建所有子元素fiber
  // 3.返回下一个工作单元
}
```

现在第二步的工作就要改变一下了，也就是执行`reconcile`过程

```js
function performUnitOfWork(fiber) {
  // 1.创建dom
  // 2.协调
  const elements = fiber.props.children
  reconcileChildren(fiber, elements)
  // 3.返回下一个工作单元
}

function reconcileChildren(wipFiber, elements) {
  // 注意这里的wipFiber指的是父节点fiber，elements是子节点元素对象
}
```

`reconcileChildren()`中要做的就是比较old fiber，给现在的每个fiber打tag，也就是做变化标记，然后再提交时根据tag执行动作

- type相同标记更新
- type不同创建新fiber
- type不同标记删除老fiber

```js
function reconcileChildren(wipFiber, elements) {
  let index = 0
  let oldFiber =
    wipFiber.alternate && wipFiber.alternate.child
  let prevSibling = null

  // diff当前节点下的所有子元素
  while (
    index < elements.length ||
    oldFiber != null
  ) {
    const element = elements[index]
    let newFiber = null

    const sameType =
      oldFiber &&
      element &&
      element.type == oldFiber.type

    // type相同标记更新
    if (sameType) {
      newFiber = {
        type: oldFiber.type,
        props: element.props,
        dom: oldFiber.dom,
        parent: wipFiber,
        alternate: oldFiber,
        effectTag: "UPDATE",
      }
    }
    // type不同标记创建
    if (element && !sameType) {
      newFiber = {
        type: element.type,
        props: element.props,
        dom: null,
        parent: wipFiber,
        alternate: null,
        effectTag: "PLACEMENT",
      }
    }
    // type不同 older fiber标记删除
    if (oldFiber && !sameType) {
      oldFiber.effectTag = "DELETION"
      // 全局数组对象 被标记删除的fiber
      deletions.push(oldFiber)
    }

    // sibling为循环下一个
    if (oldFiber) {
      oldFiber = oldFiber.sibling
    }

    // 链接新的fiber tree
    if (index === 0) {
      wipFiber.child = newFiber
    } else if (element) {
      prevSibling.sibling = newFiber
    }

    prevSibling = newFiber
    index++
  }
}
```

有个地方值得注意，在type相同时，`newFiber.dom`直接引用了`oldFiber.dom`

`reconcileChildren()`代码虽然很长，但总结起来很简单：**对比每个子元素older fiber，并创建被打上tag的new fiber，最后链接new fiber**

```js
function render(element, container) {
  // 。。。
  deletions = []
  // 。。。
}

// 这是上一段代码说到的新增的全局变量
// 标记删除的fiber，以便提交时使用
let deletions = null
```

提交

```js
function commitRoot() {
  // 遍历被删除fiber
  deletions.forEach(commitWork)
  // 下面的代码和之前相同
  commitWork(wipRoot.child)
  currentRoot = wipRoot
  wipRoot = null
}
```

所以现在对`commitWork()`改造一下，针对tag做处理

```js
function commitWork(fiber) {
  if (!fiber) {
    return;
  }
  const domParent = fiber.parent.dom;

  // 移除
  // domParent.appendChild(fiber.dom)

  if (fiber.effectTag === 'PLACEMENT' && fiber.dom != null) {
    // 添加
    domParent.appendChild(fiber.dom);
  } else if (fiber.effectTag === 'UPDATE' && fiber.dom != null) {
    // 更新
    updateDom(fiber.dom, fiber.alternate.props, fiber.props);
  } else if (fiber.effectTag === 'DELETION') {
    // 移除
    domParent.removeChild(fiber.dom);
  }
  commitWork(fiber.child);
  commitWork(fiber.sibling);
}

function updateDom(dom, prevProps, nextProps) {
  // TODO
}
```

```js
const isProperty = key => key !== "children"
const isNew = (prev, next) => key =>
  prev[key] !== next[key]
const isGone = (prev, next) => key => !(key in next)

function updateDom(dom, prevProps, nextProps) {
  // 移除oldProperty
  Object.keys(prevProps)
    .filter(isProperty)
    .filter(isGone(prevProps, nextProps))
    .forEach(name => {
      dom[name] = ""
    })

  // 添加newProperty
  Object.keys(nextProps)
    .filter(isProperty)
    .filter(isNew(prevProps, nextProps))
    .forEach(name => {
      dom[name] = nextProps[name]
    })
}
```

基于上面的代码做修改，为`on`开头的属性添加事件监听

```js
const isEvent = key => key.startsWith("on")
const isProperty = key =>
  key !== "children" && !isEvent(key)
const isNew = (prev, next) => key =>
  prev[key] !== next[key]
const isGone = (prev, next) => key => !(key in next)

function updateDom(dom, prevProps, nextProps) {
  // 移除old event
  Object.keys(prevProps)
    .filter(isEvent)
    .filter(
      key =>
        !(key in nextProps) ||
        isNew(prevProps, nextProps)(key)
    )
    .forEach(name => {
      const eventType = name
        .toLowerCase()
        .substring(2)
      dom.removeEventListener(
        eventType,
        prevProps[name]
      )
    })

  Object.keys(prevProps)
    .filter(isProperty)
    .filter(isGone(prevProps, nextProps))
    .forEach(name => {
      dom[name] = ""
    })

  Object.keys(nextProps)
    .filter(isProperty)
    .filter(isNew(prevProps, nextProps))
    .forEach(name => {
      dom[name] = nextProps[name]
    })

  // 添加new event
  Object.keys(nextProps)
    .filter(isEvent)
    .filter(isNew(prevProps, nextProps))
    .forEach(name => {
      const eventType = name
        .toLowerCase()
        .substring(2)
      dom.addEventListener(
        eventType,
        nextProps[name]
      )
    })
}
```

到目前为止已经算是半个完整的小react了，接下来再为之锦上添花，使其支持函数组件

## 函数组件

```jsx
function App(props) {
  return <h1>Hi {props.name}</h1>
}
const element = <App name="foo" />
const container = document.getElementById("root")
MyReact.render(element, container)
```

翻译过来就是

```js
function App(props) {
  return Didact.createElement(
    "h1",
    null,
    "Hi ",
    props.name
  )
}
const element = Didact.createElement(App, {
  name: "foo",
})
const container = document.getElementById("root")
MyReact.render(element, container)
```

函数组件和元素直接表示有一些不同

- `elements`不能直接通过元素对象得到，而是通过运行函数获取
- 函数组件对应的fiber上没有dom属性，或者dom属性为空（因为用jsx表示的函数组件并不是真正的dom）

再次回顾`performUnitOfWork()`

```js
function performUnitOfWork(fiber) {
  // 1.创建dom
  // 2.协调
  // 3.返回下一个工作单元
}
```

再次改造，把前面两步，针对函数和直接表示的元素分开做处理

```js
function performUnitOfWork(fiber) {
  // 1.分别处理
  const isFunctionComponent =
    fiber.type instanceof Function
  if (isFunctionComponent) {
    updateFunctionComponent(fiber)
  } else {
    updateHostComponent(fiber)
  }
   // 2.返回下一个工作单元
}

// 函数组件
function updateFunctionComponent(fiber) {
  // TODO
}

// 非函数组件，和之前相同处理
function updateHostComponent(fiber) {
  if (!fiber.dom) {
    fiber.dom = createDom(fiber)
  }
  reconcileChildren(fiber, fiber.props.children)
}
```

### `updateFunctionComponent`

函数组件只需要调用一次，得到对象结果

```js
function updateFunctionComponent(fiber) {
  // 这里并不需要处理结果中再次返回的函数组件(函数组件返回函数组件)
  // 直接在协调中，同样创建type为函数的fiber
  // 返回的函数组件会在下一个nextUnitOfWork中得到处理
  const children = [fiber.type(fiber.props)]
  reconcileChildren(fiber, children)
}
```

回顾一下`commitWork()`

```jsx
function commitWork(fiber) {
  if (!fiber) {
    return
  }

  const domParent = fiber.parent.dom
  // 省略针对tag做处理
  // 。。。
  
  commitWork(fiber.child)
  commitWork(fiber.sibling)
}
```

所以`domParent`不能再直接从`fiber.parent.dom`中获取，因为这个fiber可能是函数组件fiber，没有dom

```js
function commitWork(fiber) {
  // 省略了其余代码。。。
  
  // 移除
  // const domParent = fiber.parent.dom
  
  let domParentFiber = fiber.parent;
  while (!domParentFiber.dom) {
    domParentFiber = domParentFiber.parent;
  }
  const domParent = domParentFiber.dom;
}
```

为什么只需要向上遍历找到有dom的节点作为父节点即可，因为fiber tree同我们写的jsx结构是相同的，但这其中包含了函数组件fiber

举个例子

```jsx
const App = () => {
  return (
    <div>
      <App1></App1>
    </div>
  );
};

const App1 = () => {
  return <h>1</h>;
};
```

最终得出的真实dom

```html
<div>
  <h></h>
</div>
```

所以我们只需要向上遍历找到存在的真实dom节点作为父节点

移除节点的时候处理思路也是如此

```js

function commitWork(fiber) {
  // 省略了其余代码。。。
  
  if (fiber.effectTag === 'PLACEMENT' && fiber.dom != null) {
    // 添加
    domParent.appendChild(fiber.dom);
  } else if (fiber.effectTag === 'UPDATE' && fiber.dom != null) {
    // 更新
    updateDom(fiber.dom, fiber.alternate.props, fiber.props);
  } else if (fiber.effectTag === 'DELETION') {
    // 移除
    // domParent.removeChild(fiber.dom);
    commitDeletion(fiber, domParent);
  }
}

// 递归直到找到子节点且移除
// 以上的实现不支持函数+props.children的传入形式，所以必定能找到一个父节点dom
// react源码中会去寻找fiber.child.sibling
function commitDeletion(fiber, domParent) {
  if (fiber.dom) {
    domParent.removeChild(fiber.dom);
  } else {
    commitDeletion(fiber.child, domParent);
  }
}
```

## Hooks

```jsx
function Counter() {
  const [state, setState] = MyReact.useState(1)
  return (
    <h1 onClick={() => setState(c => c + 1)}>
      Count: {state}
    </h1>
  )
}
```

从使用来说我们知道Hooks的几个点

- useState是一个全局变量
- 只能在函数组件中使用
- 能同时存在n个`useState`
- `setState`能够触发render

```js
function useState(initial) {
  // TODO
}

const MyReact = {
  useState
};
```

实现在函数中使用，并且能够调用n次`useState`

修改`updateFunctionComponent()`，在函数重新执行时，我们需要拿到之前的`hook.state`，所以我们需要把所有hooks保存下来，链接到fiber上，以便后续读取

再者，hook可以反复调用，所以我们通过索引依次遍历，这也就是react中不允许条件调用hook的原因

```js
let wipFiber = null
let hookIndex = null

function updateFunctionComponent(fiber) {
  // 标记当前fiber
  wipFiber = fiber
  // 初始化索引
  hookIndex = 0
  // 初始化fiber.hooks（这里的wipFiber永远是newFiber）
  wipFiber.hooks = []
  const children = [fiber.type(fiber.props)]
  reconcileChildren(fiber, children)
}
```

触发渲染 --> 执行函数 --> 取oldFiber(如果有)中的`hook.state`

```js
function useState(initial) {
  const oldHook =
    wipFiber.alternate &&
    wipFiber.alternate.hooks &&
    wipFiber.alternate.hooks[hookIndex]
  const state = oldHook ? oldHook.state : initial
        
  wipFiber.hooks.push(hook)
  hookIndex++
  return [state]
}
```

### setState

`setState`能够触发"render"，而`render()`触发react执行，是赋予了`nextUnitOfWork`初始值，所以这里的`setState`触发work也应该做`render()`相同的事

```js
function useState(initial) {
  const oldHook =
    wipFiber.alternate &&
    wipFiber.alternate.hooks &&
    wipFiber.alternate.hooks[hookIndex];
  const hook = {
    state: oldHook ? oldHook.state : initial,
    queue: [],
  };

  const setState = (action) => {
    // setState可能被多次调用
    // 所以记录下每一个action，应对多次调用的情况
    hook.queue.push(action);
    wipRoot = {
      dom: currentRoot.dom,
      props: currentRoot.props,
      alternate: currentRoot,
    };
    nextUnitOfWork = wipRoot;
    deletions = [];
  };

  wipFiber.hooks.push(hook);
  hookIndex++;
  return [hook.state, setState];
}
```

`setState`做的只是触发work，真正的修改state动作，是在触发work后执行函数时，再次执行到`useState`时去修改的

```js
function useState(initial) {
  const oldHook =
    wipFiber.alternate &&
    wipFiber.alternate.hooks &&
    wipFiber.alternate.hooks[hookIndex];
  const hook = {
    state: oldHook ? oldHook.state : initial,
    queue: [],
  };
  
  // 再次执行时，遍历action
  const actions = oldHook ? oldHook.queue : []
  actions.forEach(action => {
    hook.state = action(hook.state)
  })

  const setState = (action) => {
    hook.queue.push(action);
    wipRoot = {
      dom: currentRoot.dom,
      props: currentRoot.props,
      alternate: currentRoot,
    };
    nextUnitOfWork = wipRoot;
    deletions = [];
  };

  wipFiber.hooks.push(hook);
  hookIndex++;
  return [hook.state, setState];
}
```

虽然这里只实现了传入函数修改的方式，但也不难解释，为何函数能做到每次获取最新值，而`setState(s+1)`这种方式做不到的问题，`s+1` 给react的是一个具体的值，假如现在的`s===1`，并且`setState(s+1)`被同步调用3次，可以理解为调用了三次`hook.queue.push(2)`，所以结果只有一个

## End

> 至此，这就是我们开发的react的全部
>
> [codesandbox](https://codesandbox.io/s/didact-8-21ost?file=/src/index.js:0-6313)

```jsx
function createElement(type, props, ...children) {
  return {
    type,
    props: {
      ...props,
      children: children.map(child =>
        typeof child === "object" ? child : createTextElement(child)
      )
    }
  };
}

function createTextElement(text) {
  return {
    type: "TEXT_ELEMENT",
    props: {
      nodeValue: text,
      children: []
    }
  };
}

function createDom(fiber) {
  const dom =
    fiber.type == "TEXT_ELEMENT"
      ? document.createTextNode("")
      : document.createElement(fiber.type);

  updateDom(dom, {}, fiber.props);

  return dom;
}

const isEvent = key => key.startsWith("on");
const isProperty = key => key !== "children" && !isEvent(key);
const isNew = (prev, next) => key => prev[key] !== next[key];
const isGone = (prev, next) => key => !(key in next);
function updateDom(dom, prevProps, nextProps) {
  //Remove old or changed event listeners
  Object.keys(prevProps)
    .filter(isEvent)
    .filter(key => !(key in nextProps) || isNew(prevProps, nextProps)(key))
    .forEach(name => {
      const eventType = name.toLowerCase().substring(2);
      dom.removeEventListener(eventType, prevProps[name]);
    });

  // Remove old properties
  Object.keys(prevProps)
    .filter(isProperty)
    .filter(isGone(prevProps, nextProps))
    .forEach(name => {
      dom[name] = "";
    });

  // Set new or changed properties
  Object.keys(nextProps)
    .filter(isProperty)
    .filter(isNew(prevProps, nextProps))
    .forEach(name => {
      dom[name] = nextProps[name];
    });

  // Add event listeners
  Object.keys(nextProps)
    .filter(isEvent)
    .filter(isNew(prevProps, nextProps))
    .forEach(name => {
      const eventType = name.toLowerCase().substring(2);
      dom.addEventListener(eventType, nextProps[name]);
    });
}

function commitRoot() {
  deletions.forEach(commitWork);
  commitWork(wipRoot.child);
  currentRoot = wipRoot;
  wipRoot = null;
}

function commitWork(fiber) {
  if (!fiber) {
    return;
  }

  let domParentFiber = fiber.parent;
  while (!domParentFiber.dom) {
    domParentFiber = domParentFiber.parent;
  }
  const domParent = domParentFiber.dom;

  if (fiber.effectTag === "PLACEMENT" && fiber.dom != null) {
    domParent.appendChild(fiber.dom);
  } else if (fiber.effectTag === "UPDATE" && fiber.dom != null) {
    updateDom(fiber.dom, fiber.alternate.props, fiber.props);
  } else if (fiber.effectTag === "DELETION") {
    commitDeletion(fiber, domParent);
  }

  commitWork(fiber.child);
  commitWork(fiber.sibling);
}

function commitDeletion(fiber, domParent) {
  if (fiber.dom) {
    domParent.removeChild(fiber.dom);
  } else {
    commitDeletion(fiber.child, domParent);
  }
}

function render(element, container) {
  wipRoot = {
    dom: container,
    props: {
      children: [element]
    },
    alternate: currentRoot
  };
  deletions = [];
  nextUnitOfWork = wipRoot;
}

let nextUnitOfWork = null;
let currentRoot = null;
let wipRoot = null;
let deletions = null;

function workLoop(deadline) {
  let shouldYield = false;
  while (nextUnitOfWork && !shouldYield) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    shouldYield = deadline.timeRemaining() < 1;
  }

  if (!nextUnitOfWork && wipRoot) {
    commitRoot();
  }

  requestIdleCallback(workLoop);
}

requestIdleCallback(workLoop);

function performUnitOfWork(fiber) {
  const isFunctionComponent = fiber.type instanceof Function;
  if (isFunctionComponent) {
    updateFunctionComponent(fiber);
  } else {
    updateHostComponent(fiber);
  }
  if (fiber.child) {
    return fiber.child;
  }
  let nextFiber = fiber;
  while (nextFiber) {
    if (nextFiber.sibling) {
      return nextFiber.sibling;
    }
    nextFiber = nextFiber.parent;
  }
}

let wipFiber = null;
let hookIndex = null;

function updateFunctionComponent(fiber) {
  wipFiber = fiber;
  hookIndex = 0;
  wipFiber.hooks = [];
  const children = [fiber.type(fiber.props)];
  reconcileChildren(fiber, children);
}

function useState(initial) {
  const oldHook =
    wipFiber.alternate &&
    wipFiber.alternate.hooks &&
    wipFiber.alternate.hooks[hookIndex];
  const hook = {
    state: oldHook ? oldHook.state : initial,
    queue: []
  };

  const actions = oldHook ? oldHook.queue : [];
  actions.forEach(action => {
    hook.state = action(hook.state);
  });

  const setState = action => {
    hook.queue.push(action);
    wipRoot = {
      dom: currentRoot.dom,
      props: currentRoot.props,
      alternate: currentRoot
    };
    nextUnitOfWork = wipRoot;
    deletions = [];
  };

  wipFiber.hooks.push(hook);
  hookIndex++;
  return [hook.state, setState];
}

function updateHostComponent(fiber) {
  if (!fiber.dom) {
    fiber.dom = createDom(fiber);
  }
  reconcileChildren(fiber, fiber.props.children);
}

function reconcileChildren(wipFiber, elements) {
  let index = 0;
  let oldFiber = wipFiber.alternate && wipFiber.alternate.child;
  let prevSibling = null;

  while (index < elements.length || oldFiber != null) {
    const element = elements[index];
    let newFiber = null;

    const sameType = oldFiber && element && element.type == oldFiber.type;

    if (sameType) {
      newFiber = {
        type: oldFiber.type,
        props: element.props,
        dom: oldFiber.dom,
        parent: wipFiber,
        alternate: oldFiber,
        effectTag: "UPDATE"
      };
    }
    if (element && !sameType) {
      newFiber = {
        type: element.type,
        props: element.props,
        dom: null,
        parent: wipFiber,
        alternate: null,
        effectTag: "PLACEMENT"
      };
    }
    if (oldFiber && !sameType) {
      oldFiber.effectTag = "DELETION";
      deletions.push(oldFiber);
    }

    if (oldFiber) {
      oldFiber = oldFiber.sibling;
    }

    if (index === 0) {
      wipFiber.child = newFiber;
    } else if (element) {
      prevSibling.sibling = newFiber;
    }

    prevSibling = newFiber;
    index++;
  }
}

const MyReact = {
  createElement,
  render,
  useState
};

/** @jsx MyReact.createElement */
function Counter() {
  const [state, setState] = MyReact.useState(1);
  return (
    <h1 onClick={() => setState(c => c + 1)} style="user-select: none">
      Count: {state}
    </h1>
  );
}
const element = <Counter />;
const container = document.getElementById("root");
MyReact.render(element, container);

```

